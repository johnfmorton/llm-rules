# llm-rules

Reusable instruction documents for LLM coding tools like [Claude Code](https://claude.ai/code), Cursor, and GitHub Copilot. Each document is a self-contained specification that an LLM can follow to implement a common feature from scratch.

Created by [John F Morton](https://github.com/johnfmorton).

## What this is

When I build applications, I frequently implement the same features — authentication systems, role-based access, OAuth integrations, and so on. Rather than re-explaining these requirements to an LLM every time, I maintain this collection of detailed specification documents that can be provided as context to any LLM coding tool.

Each `*_LLM.md` file contains everything an LLM needs to implement that feature: package dependencies, database schemas, model changes, controller logic, routes, frontend code, and common pitfalls to avoid.

## Tech stack coverage

The primary focus is **PHP** and **Laravel**, but since my work also spans HTML, TypeScript, Craft CMS, and other stacks, documents covering those areas may appear here as well.

## Current documents

| File | Description |
|------|-------------|
| `AUTHENTICATION_REQUIREMENTS_LLM.md` | Full Laravel authentication system — email/password login, registration, password reset, roles & permissions, user suspension, email verification, 2FA (TOTP), passkeys (WebAuthn), and OAuth (GitHub) |

## Usage

Copy or reference the relevant `*_LLM.md` file when starting a new project or adding a feature. For example, with Claude Code you can reference the file directly in your prompt:

```
Implement the auth system described in @AUTHENTICATION_REQUIREMENTS_LLM.md
```

## License

MIT
