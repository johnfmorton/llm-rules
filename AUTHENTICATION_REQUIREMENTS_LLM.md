# Laravel Authentication System Requirements

This document defines a comprehensive authentication system for a Laravel application. It is intended as an instruction set for an LLM building or modifying a Laravel project. Implement all features described below unless the project maintainer specifies otherwise.

## Package Dependencies

Install the following packages via Composer:

```bash
composer require laravel/socialite        # OAuth social login
composer require pragmarx/google2fa-laravel  # TOTP two-factor authentication
composer require bacon/bacon-qr-code      # QR code generation for 2FA setup
composer require laragear/webauthn        # WebAuthn passkey support
```

After installing `laragear/webauthn`, publish its config and migration:

```bash
php artisan vendor:publish --provider="Laragear\WebAuthn\WebAuthnServiceProvider"
php artisan migrate
```

---

## 1. Email & Password Authentication

### Login

Users log in with an email address and password via a standard form POST.

- The login form includes: email input, password input, "Remember me" checkbox, and a submit button.
- Use Laravel's `Auth::attempt()` for credential validation.
- Rate-limit login attempts to **5 per email+IP combination** using `RateLimiter`. The throttle key format is `{lowercase_email}|{ip_address}`.
- On successful authentication, regenerate the session and redirect to the intended URL or dashboard.
- If the authenticated user has two-factor authentication enabled (see Section 6), **do not** complete the login. Instead, store the user ID and remember preference in the session, log the user out, and redirect to the two-factor challenge route.

### Registration

- Registration requires: name, email, password (with confirmation).
- After creating the user, send an email verification notification and log the user in.
- If the application uses invite codes or a beta system, validate the invite code during registration and set the user's subscription tier accordingly.

### Password Reset

Implement the standard Laravel password reset flow with custom email notifications:

1. **Forgot Password** (GET `/forgot-password`) - Form to enter email address.
2. **Send Reset Link** (POST `/forgot-password`) - Validates email and dispatches a password reset notification. Throttle to prevent abuse.
3. **Reset Form** (GET `/reset-password/{token}`) - Form for new password with the token.
4. **Store New Password** (POST `/reset-password`) - Validates token, hashes new password, regenerates remember token, fires `PasswordReset` event.

**Password reset tokens** expire after 60 minutes with a 60-second throttle between requests.

### Password Update (Authenticated)

Authenticated users can update their password from their profile page.

- Route: `PUT /password`
- Validate: `current_password` (required, must match), `password` (required, confirmed, meets `Password::defaults()`)
- Hash and save the new password.

---

## 2. User Roles & Permissions

### Role Enum

Create a `UserRole` enum with at least two values:

```php
enum UserRole: string
{
    case SuperAdmin = 'super_admin';
    case Subscriber = 'subscriber';
}
```

Add a `role` column to the users table as a string with a default of `'subscriber'`. Cast it to the enum in the User model.

### User Model Helper Methods

```php
public function isSuperAdmin(): bool
{
    return $this->role === UserRole::SuperAdmin;
}

public function isSubscriber(): bool
{
    return $this->role === UserRole::Subscriber;
}
```

### Super Admin Middleware

Create `EnsureSuperAdmin` middleware that aborts with 403 if the user is not a super admin:

```php
public function handle(Request $request, Closure $next): Response
{
    if (! $request->user()?->isSuperAdmin()) {
        abort(403, 'Access denied. Super admin privileges required.');
    }
    return $next($request);
}
```

Register it as a named alias (e.g., `super_admin`) in `bootstrap/app.php`.

### Admin Routes

All admin routes (user management, system settings) must use middleware: `['auth', 'verified', 'super_admin']`.

Admin capabilities include:
- View and search all users
- Suspend and unsuspend users (see Section 3)
- Send password reset emails to users
- Impersonate users (for support/debugging)

---

## 3. User Status & Suspension

### Database Fields

Add the following columns to the users table:

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `is_suspended` | boolean | `false` | Whether the user is suspended |
| `suspended_at` | timestamp | `null` | When the suspension occurred |
| `suspended_reason` | string | `null` | Optional reason for suspension |
| `deleted_at` | timestamp | `null` | Soft delete support |

Use Laravel's `SoftDeletes` trait on the User model.

### User Model Methods

```php
public function suspend(?string $reason = null): void
{
    $this->forceFill([
        'is_suspended' => true,
        'suspended_at' => now(),
        'suspended_reason' => $reason,
    ])->save();
}

public function unsuspend(): void
{
    $this->forceFill([
        'is_suspended' => false,
        'suspended_at' => null,
        'suspended_reason' => null,
    ])->save();
}

public function isSuspended(): bool
{
    return (bool) $this->is_suspended;
}
```

### Suspension Middleware

Create `EnsureNotSuspended` middleware that redirects suspended users to a suspension notice page. Allow access to a small set of routes so suspended users can still view the notice, log out, and edit their profile:

```php
protected array $allowedRoutes = ['suspended', 'logout', 'profile.edit'];

public function handle(Request $request, Closure $next): Response
{
    $user = $request->user();

    if ($user && $user->isSuspended()) {
        $currentRoute = $request->route()?->getName();
        if (! in_array($currentRoute, $this->allowedRoutes)) {
            return redirect()->route('suspended');
        }
    }

    return $next($request);
}
```

Register as `not_suspended` and add it to the web middleware stack so it runs on every authenticated request.

### Suspended View

Create a dedicated route and view (e.g., GET `/suspended`, name `suspended`) that informs the user their account is suspended and displays the reason if one was provided.

---

## 4. Email Verification

The User model must implement `MustVerifyEmail`.

### Flow

1. **Verification Notice** (GET `/verify-email`, name `verification.notice`) - Shows a page prompting the user to check their email. Redirects to dashboard if already verified.
2. **Send Verification** (POST `/email/verification-notification`, name `verification.send`) - Resends verification email. Throttle to `6,1`.
3. **Verify Email** (GET `/verify-email/{id}/{hash}`, name `verification.verify`) - Validates the signed URL, marks email as verified, fires `Verified` event. Uses `signed` and `throttle:6,1` middleware.

### Custom Notification

Override `sendEmailVerificationNotification()` on the User model to use a custom notification class with a branded email template rather than the default Laravel notification.

---

## 5. Profile & Security Settings Page

The user profile page (e.g., `/profile`) should contain these sections:

1. **Update Profile Information** - Name and email fields
2. **Update Password** - Current password, new password, confirmation
3. **Security Settings** - Contains three cards:
   - **Two-Factor Authentication** - Status indicator and setup/manage link
   - **Passkeys** - List registered passkeys, add new, remove existing
   - **Connected Accounts** - Show linked OAuth providers and option to connect
4. **Delete Account** - Danger zone with password confirmation modal

---

## 6. Two-Factor Authentication (2FA)

### Package

Use `pragmarx/google2fa-laravel` for TOTP generation and verification, and `bacon/bacon-qr-code` for rendering QR codes.

### Database Fields

Add to the users table:

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `two_factor_secret` | text | `null` | Encrypted TOTP secret |
| `two_factor_recovery_codes` | text | `null` | Encrypted JSON array of codes |
| `two_factor_confirmed_at` | timestamp | `null` | When 2FA was confirmed |

Hide `two_factor_secret` and `two_factor_recovery_codes` from serialization.

### User Model Helper

```php
public function hasTwoFactorEnabled(): bool
{
    return ! is_null($this->two_factor_secret) && ! is_null($this->two_factor_confirmed_at);
}
```

### Controller Methods & Routes

Create a `TwoFactorController` with the following:

#### Setup (GET `/two-factor`, name `two-factor.setup`)
- Middleware: `auth`
- If 2FA is already enabled, show the management view.
- Otherwise, generate a new secret via Google2FA, encrypt and store it on the user (but do NOT set `two_factor_confirmed_at` yet).
- Display a QR code and the manual entry key for the user to scan with their authenticator app.

#### Confirm (POST `/two-factor/confirm`, name `two-factor.confirm`)
- Middleware: `auth`
- Validate a 6-digit TOTP code from the user's authenticator app using `$google2fa->verifyKey($secret, $code)`.
- On success:
  - Generate **8 recovery codes** (each 10 random characters via `Str::random(10)`).
  - Set `two_factor_confirmed_at` to now.
  - Encrypt and store the recovery codes.
  - Flash the recovery codes to the session so the user can copy them.
- Redirect to the management view.

#### Disable (DELETE `/two-factor`, name `two-factor.disable`)
- Middleware: `auth`
- **Require password confirmation** (`current_password` validation rule).
- Nullify `two_factor_secret`, `two_factor_recovery_codes`, and `two_factor_confirmed_at`.
- Redirect with success status.

#### Regenerate Recovery Codes (POST `/two-factor/recovery-codes`, name `two-factor.recovery-codes`)
- Middleware: `auth`
- Require password confirmation.
- Generate 8 new recovery codes, encrypt and store them.
- Flash new codes to the session.

#### Challenge (GET `/two-factor-challenge`, name `two-factor.challenge`)
- **No `auth` middleware** - the user is in a liminal state between login and 2FA verification.
- Check that `session('login.id')` exists; if not, redirect to login.
- Show a form that accepts either a TOTP code or a recovery code (with a toggle between the two).

#### Verify Challenge (POST `/two-factor-challenge`, name `two-factor.verify`)
- Accept either `code` (6-digit TOTP) or `recovery_code`.
- For TOTP: Verify via `$google2fa->verifyKey()`.
- For recovery code: Find the code in the decrypted array, remove it, and re-encrypt the updated array.
- On success: Clear `session('login.id')` and `session('login.remember')`, log the user in with the remember flag, regenerate the session, and redirect to dashboard.

### Login Flow Integration

In `AuthenticatedSessionController@store`, after successful `Auth::attempt()`:

```php
$user = Auth::user();

if ($user->hasTwoFactorEnabled()) {
    $request->session()->put('login.id', $user->id);
    $request->session()->put('login.remember', $request->boolean('remember'));
    Auth::guard('web')->logout();
    return redirect()->route('two-factor.challenge');
}

$request->session()->regenerate();
return redirect()->intended(route('dashboard'));
```

### Security Settings Display

In the profile security settings, show a card for 2FA with:
- A status indicator: green dot with "Enabled" or gray dot with "Not configured"
- A button linking to `route('two-factor.setup')` with label "Manage" (if enabled) or "Set up" (if not)

---

## 7. Passkeys (WebAuthn)

### Package & Configuration

Use `laragear/webauthn` (^4.1).

#### Critical Setup Steps

These are commonly missed and cause silent failures:

1. **User model** must both use the `WebAuthnAuthentication` trait AND declare `implements WebAuthnAuthenticatable`:

   ```php
   use Laragear\WebAuthn\Contracts\WebAuthnAuthenticatable;
   use Laragear\WebAuthn\WebAuthnAuthentication;

   class User extends Authenticatable implements MustVerifyEmail, WebAuthnAuthenticatable
   {
       use HasFactory, Notifiable, SoftDeletes, WebAuthnAuthentication;
   }
   ```

   If the interface is missing, the package's `AttestationRequest::authorize()` type-hints `?WebAuthnAuthenticatable` for the user parameter. Laravel's dependency injection resolves it to `null`, causing a 403 on the registration options endpoint.

2. **Auth provider** must be `eloquent-webauthn`, not `eloquent`, in `config/auth.php`:

   ```php
   'providers' => [
       'users' => [
           'driver' => 'eloquent-webauthn',
           'model' => App\Models\User::class,
       ],
   ],
   ```

   The `eloquent-webauthn` provider extends Laravel's standard Eloquent provider. It handles WebAuthn assertion validation when credentials contain a signed challenge, and falls back to normal password validation otherwise. Without this, `Auth::attempt()` tries to match WebAuthn credentials as a password and always fails.

3. **Environment variables** must include:

   ```env
   WEBAUTHN_NAME="${APP_NAME}"
   WEBAUTHN_ID=your-domain.com
   ```

   `WEBAUTHN_ID` must match the domain the app runs on. For local DDEV development, this is typically `your-app.ddev.site`. For production, it's the production domain. If missing, credential creation may fail with cryptographic errors.

#### Config File (`config/webauthn.php`)

```php
return [
    'relying_party' => [
        'name' => env('WEBAUTHN_NAME', config('app.name')),
        'id' => env('WEBAUTHN_ID'),
    ],
    'origins' => env('WEBAUTHN_ORIGINS'),
    'challenge' => [
        'bytes' => 16,
        'timeout' => 60,
        'key' => '_webauthn',
    ],
];
```

### Controllers

#### WebAuthnRegisterController (authenticated routes)

```php
use Laragear\WebAuthn\Http\Requests\AttestationRequest;
use Laragear\WebAuthn\Http\Requests\AttestedRequest;
use Laragear\WebAuthn\Models\WebAuthnCredential;

class WebAuthnRegisterController
{
    // POST /webauthn/register/options (name: webauthn.register.options)
    public function options(AttestationRequest $request): Responsable
    {
        return $request->fastRegistration()->toCreate();
    }

    // POST /webauthn/register (name: webauthn.register)
    public function register(AttestedRequest $request): Response
    {
        // IMPORTANT: Pass alias through save(), not through the request body.
        // The package's AttestedRequest validation rules do not include
        // response.alias, so $this->validated() strips it out before the
        // JsonTransport is created. The alias must be passed via save()
        // which forceFill()s it onto the credential before persisting.
        $request->save([
            'alias' => $request->input('response.alias', 'Passkey'),
        ]);

        return response()->noContent();
    }

    // DELETE /webauthn/credentials/{credential} (name: webauthn.credentials.destroy)
    public function destroy(WebAuthnCredential $credential): RedirectResponse
    {
        abort_unless($credential->authenticatable_id === auth()->id(), 403);
        $credential->delete();
        return back()->with('status', 'Passkey deleted.');
    }
}
```

#### WebAuthnLoginController (guest routes)

```php
use Laragear\WebAuthn\Http\Requests\AssertedRequest;
use Laragear\WebAuthn\Http\Requests\AssertionRequest;

class WebAuthnLoginController
{
    // POST /webauthn/login/options (name: webauthn.login.options)
    public function options(AssertionRequest $request): Responsable
    {
        return $request->toVerify($request->validate(['email' => 'sometimes|email|string']));
    }

    // POST /webauthn/login (name: webauthn.login)
    public function login(AssertedRequest $request): Response
    {
        return response()->noContent($request->login() ? 204 : 422);
    }
}
```

### Routes

```php
// Guest routes
Route::middleware('guest')->group(function () {
    Route::post('webauthn/login/options', [WebAuthnLoginController::class, 'options'])
        ->name('webauthn.login.options');
    Route::post('webauthn/login', [WebAuthnLoginController::class, 'login'])
        ->name('webauthn.login');
});

// Authenticated routes
Route::middleware('auth')->group(function () {
    Route::post('webauthn/register/options', [WebAuthnRegisterController::class, 'options'])
        ->name('webauthn.register.options');
    Route::post('webauthn/register', [WebAuthnRegisterController::class, 'register'])
        ->name('webauthn.register');
    Route::delete('webauthn/credentials/{credential}', [WebAuthnRegisterController::class, 'destroy'])
        ->name('webauthn.credentials.destroy');
});
```

### JavaScript: Base64URL Encoding

The WebAuthn browser API uses `ArrayBuffer` objects, but the Laragear package expects **base64url** encoding (not standard base64). Both the login and registration pages need these helper functions:

```javascript
function base64UrlDecode(input) {
    input = input.replace(/-/g, '+').replace(/_/g, '/');
    const pad = input.length % 4;
    if (pad) input += '='.repeat(4 - pad);
    const binary = atob(input);
    return Uint8Array.from(binary, c => c.charCodeAt(0)).buffer;
}

function arrayToBase64Url(buffer) {
    return btoa(String.fromCharCode(...new Uint8Array(buffer)))
        .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}
```

Using standard base64 (`btoa` without the replacements) will cause signature verification failures because the package decodes with `ByteBuffer::decodeBase64Url()`.

### JavaScript: Login Page (Passkey Sign-In)

Add a "Sign in with a passkey" button to the login page, visible only when the browser supports WebAuthn (`typeof PublicKeyCredential !== 'undefined'`).

The flow:

1. POST to `/webauthn/login/options` with CSRF token to get challenge options.
2. Decode `options.challenge` and each `options.allowCredentials[].id` from base64url to `ArrayBuffer`.
3. Call `navigator.credentials.get({ publicKey: options })` to prompt the browser/password manager.
4. Encode the credential response fields (`rawId`, `clientDataJSON`, `authenticatorData`, `signature`, `userHandle`) as base64url.
5. POST the encoded credential to `/webauthn/login`.
6. On 204 success, redirect to dashboard. On failure, show error message.
7. Handle `NotAllowedError` (user cancelled the prompt) gracefully.

### JavaScript: Registration Page (Add Passkey)

Use an Alpine.js component for the registration flow in the profile security settings.

The flow:

1. POST to `/webauthn/register/options` to get attestation options.
2. Decode `options.challenge`, `options.user.id`, and `options.excludeCredentials[].id` from base64url.
3. Call `navigator.credentials.create({ publicKey: options })` to create the credential.
4. **Show an in-app modal** prompting the user to name the passkey (e.g., "MacBook", "iPhone"). Default to "Passkey" if left empty. Do NOT use `prompt()` or any browser-native dialog.
5. Encode the credential response and include `response.alias` with the chosen name.
6. POST to `/webauthn/register`.
7. On success, reload the page to show the new passkey in the list.

### Security Settings: Passkey Management UI

Display a card in the security settings section with:

- Count of registered passkeys (e.g., "2 passkeys registered" or "No passkeys registered").
- A list of each passkey showing its **alias** (name) and creation date (e.g., "MacBook - Added 3 days ago").
- A **Remove** button next to each passkey that opens an in-app confirmation modal. Do NOT use `confirm()`.
- An **Add a passkey** button that triggers the registration flow.
- A message if the browser doesn't support WebAuthn.

---

## 8. Social/OAuth Login (GitHub)

### Package

Use `laravel/socialite`.

### Configuration

In `config/services.php`:

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => env('GITHUB_REDIRECT_URL', '/auth/github/callback'),
],
```

Required environment variables:

```env
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_REDIRECT_URL="${APP_URL}/auth/github/callback"
```

### Social Account Model & Migration

Create a `SocialAccount` model with migration:

```php
Schema::create('social_accounts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('provider');          // e.g., 'github'
    $table->string('provider_id');       // GitHub user ID
    $table->string('provider_token')->nullable();
    $table->string('provider_refresh_token')->nullable();
    $table->string('provider_avatar')->nullable();
    $table->timestamps();
    $table->unique(['provider', 'provider_id']);
});
```

Hide `provider_token` and `provider_refresh_token` from serialization.

Add a `socialAccounts()` HasMany relationship on the User model.

### Critical Implementation Notes

These are commonly missed and cause broken functionality:

1. **OAuth redirect/callback routes must NOT be inside a `guest` middleware group.** These routes serve two purposes: allowing guests to log in via OAuth, AND allowing authenticated users to link their social account from the profile page. If placed under `guest` middleware, authenticated users clicking "Connect GitHub" on their profile will be silently redirected to the dashboard instead of starting the OAuth flow.

2. **The callback must handle authenticated users.** When an authenticated user initiates OAuth from the profile page, the callback should detect `Auth::check()` and link the account to the current user rather than trying to log them in or create a new account. It must also check that the social account isn't already linked to a different user.

3. **The registration page must offer OAuth as a sign-up option.** If the login page shows "Continue with GitHub", the registration page must also show "Sign up with GitHub". Otherwise, new users are forced into email/password registration even when they prefer OAuth. Show the button conditionally with the same `config('services.github.client_id')` check.

### Routes

```php
// Outside both guest and auth groups — accessible to everyone
Route::get('auth/{provider}/redirect', [SocialiteController::class, 'redirect'])
    ->name('socialite.redirect');
Route::get('auth/{provider}/callback', [SocialiteController::class, 'callback'])
    ->name('socialite.callback');

// Guest-only routes — social registration requires an invite code
Route::middleware('guest')->group(function () {
    Route::get('auth/social/register', [SocialiteController::class, 'showRegisterForm'])
        ->name('socialite.register');
    Route::post('auth/social/register', [SocialiteController::class, 'completeRegistration'])
        ->name('socialite.register.complete');
});
```

### Controller Flow

Create a `SocialiteController` with these methods:

#### Redirect (GET `/auth/{provider}/redirect`, name `socialite.redirect`)
- Validate the provider is supported (e.g., `'github'`).
- Return `Socialite::driver($provider)->redirect()`.

#### Callback (GET `/auth/{provider}/callback`, name `socialite.callback`)
- Get the user from Socialite: `Socialite::driver($provider)->user()`.
- **If the current user is authenticated** (linking from profile page): Create or update the social account for the current user and redirect back to the profile page with a status message. Check that the social account isn't already linked to a different user.
- **If social account exists** (matching provider + provider_id): Update tokens, log in the existing linked user.
- **If email matches an existing user** (but no social account yet): Create the social account linked to that user, log them in.
- **If no matching user**: Store the Socialite user data in the session and redirect to a social registration form where the user can complete account creation.

#### Social Registration Form (GET `/auth/social/register`, name `socialite.register`)
- Display a form pre-filled with name and email from the Socialite session data.
- User can adjust their name and provide any additional required fields (e.g., invite code).

#### Complete Registration (POST `/auth/social/register`, name `socialite.register.complete`)
- Validate input, create the user, create the linked social account.
- Log in and redirect to dashboard.

### Login Page Display

Show the GitHub OAuth button on the login page **only if** `config('services.github.client_id')` is set. Use a divider ("or") between the OAuth button and the email/password form.

### Registration Page Display

Show the GitHub OAuth button on the registration page as well, with "Sign up with GitHub" label and an "or" divider above the registration form. Only show if `config('services.github.client_id')` is set. When a new user registers via OAuth, they will be redirected to the social registration form to provide an invite code.

### Security Settings Display

In the Connected Accounts card on the profile page:
- If GitHub is connected: Show the GitHub icon with a "Connected" badge.
- If not connected but configured: Show a "Connect GitHub" link pointing to the OAuth redirect route.
- If not configured: Show "GitHub login is not configured."

### Adding Other OAuth Providers

This pattern is designed to be provider-agnostic. To add another provider (e.g., Google):
1. Add the provider config to `config/services.php`.
2. Add the provider name to the validation in the controller's `redirect()` method.
3. The `social_accounts` table already supports multiple providers via the `provider` column.

---

## 9. Login Page Layout

The login page presents all authentication methods in a clear hierarchy:

1. **OAuth buttons** (top) - e.g., "Continue with GitHub". Only shown if configured.
2. **Divider** - "or" separator between OAuth and form login.
3. **Email/password form** - Standard form with email, password, remember me, submit.
4. **Passkey button** (below form) - "Sign in with a passkey". Only shown if browser supports WebAuthn.
5. **Footer links** - "Forgot password?" and "Create account".

---

## 10. UX Rules

- **Never use browser-native dialogs** (`alert()`, `confirm()`, `prompt()`). These break the application's visual design, cannot be styled, and behave inconsistently across browsers. Use in-app modal components for all confirmations and user input.
- Use loading states on buttons that trigger server-side processing (disable button, show spinner text).
- All sensitive operations (disable 2FA, regenerate recovery codes, delete account) should require password confirmation.

---

## 11. Migration Summary

The complete set of auth-related database changes:

### Users Table (base)
- `id`, `name`, `email` (unique), `email_verified_at`, `password`, `remember_token`, `timestamps`

### Users Table (additions)
- `role` - string, default `'subscriber'`
- `subscription_tier` - string, default `'trial'` (adjust for your app)
- `is_suspended` - boolean, default `false`
- `suspended_at` - timestamp, nullable
- `suspended_reason` - string, nullable
- `deleted_at` - timestamp (soft deletes)
- `two_factor_secret` - text, nullable
- `two_factor_recovery_codes` - text, nullable
- `two_factor_confirmed_at` - timestamp, nullable

### Password Reset Tokens Table
- `email` (primary), `token`, `created_at`

### Sessions Table
- `id`, `user_id`, `ip_address`, `user_agent`, `payload`, `last_activity`

### Social Accounts Table
- `id`, `user_id` (FK), `provider`, `provider_id`, `provider_token`, `provider_refresh_token`, `provider_avatar`, `timestamps`
- Unique: `[provider, provider_id]`

### WebAuthn Credentials Table
- Created via `Laragear\WebAuthn\Models\WebAuthnCredential::migration()` in the published migration.

---

## 12. Middleware Stack Summary

| Alias | Class | Purpose |
|-------|-------|---------|
| `super_admin` | `EnsureSuperAdmin` | Blocks non-super-admin users with 403 |
| `not_suspended` | `EnsureNotSuspended` | Redirects suspended users to notice page |

`not_suspended` runs in the web middleware stack (every request). `super_admin` is applied to admin route groups.

### Route Middleware Groups

- **Guest routes** (`guest`): login, register, forgot password, social registration form, WebAuthn login
- **Auth routes** (`auth`): profile, password update, email verification, 2FA setup, WebAuthn registration, logout
- **Admin routes** (`auth`, `verified`, `super_admin`): user management, system administration
- **No middleware group** (accessible to both guests and authenticated users): OAuth redirect/callback routes, two-factor challenge. The OAuth routes must be outside both groups so guests can log in and authenticated users can link accounts from their profile. The 2FA challenge routes sit outside both groups because the user is in a liminal authentication state.
