# Craft CMS 5 Twig Templating Requirements

This document defines conventions and patterns for Twig templating in a Craft CMS 5 project. It is intended as an instruction set for an LLM building or modifying Craft CMS templates. Follow all patterns described below unless the project maintainer specifies otherwise.

## 1. Template Structure & Conventions

### Directory Layout

Organize templates under `templates/` using the following structure:

```
templates/
├── _layouts/          # Base layouts (e.g., _layouts/base.twig)
├── _partials/         # Reusable partial templates (header, footer)
├── _components/       # Small, self-contained UI components
├── _macros/           # Twig macro files
├── index.twig         # Homepage
├── blog/
│   ├── index.twig     # Blog listing page
│   └── _entry.twig    # Blog detail template
├── pages/
│   └── _entry.twig    # Generic page detail template
└── search/
    └── index.twig     # Search results page
```

### Naming Conventions

- **Underscore prefix** (`_layouts/`, `_partials/`, `_entry.twig`) denotes templates that are not directly accessible via URL. Craft returns a 404 for any URL that resolves to an underscore-prefixed template.
- **`index.twig`** is the default template Craft renders when a URL resolves to a directory. For example, `/blog` renders `templates/blog/index.twig`.
- **`_entry.twig`** is the conventional name for a section's detail template. Craft renders this when an entry's URI matches a section's URI format (e.g., `blog/{slug}`).

### Content-Type Templates

Craft selects templates based on the requested format. Place alternate-format templates alongside the default:

```
templates/
├── blog/
│   ├── index.twig       # HTML (default)
│   ├── index.rss.twig   # RSS feed
│   └── index.json.twig  # JSON endpoint
```

Access them via URL format parameter: `/blog?format=rss` or `/blog.json`.

```twig
{# index.rss.twig #}
{% header "Content-Type: application/rss+xml; charset=utf-8" %}
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>{{ siteName }}</title>
    {% for entry in craft.entries().section('blog').limit(20).all() %}
    <item>
      <title>{{ entry.title }}</title>
      <link>{{ entry.url }}</link>
      <pubDate>{{ entry.postDate|rss }}</pubDate>
    </item>
    {% endfor %}
  </channel>
</rss>
```

### Layout Inheritance

Use a base layout with `{% block %}` tags for page-specific content:

```twig
{# _layouts/base.twig #}
<!DOCTYPE html>
<html lang="{{ currentSite.language }}">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{% block title %}{{ siteName }}{% endblock %}</title>
  {% block head %}{% endblock %}
</head>
<body>
  {% include '_partials/header' %}

  <main>
    {% block content %}{% endblock %}
  </main>

  {% include '_partials/footer' %}

  {% block scripts %}{% endblock %}
</body>
</html>
```

```twig
{# blog/_entry.twig #}
{% extends '_layouts/base' %}

{% block title %}{{ entry.title }} — {{ siteName }}{% endblock %}

{% block content %}
  <article>
    <h1>{{ entry.title }}</h1>
    <time datetime="{{ entry.postDate|atom }}">{{ entry.postDate|date('F j, Y') }}</time>
    {{ entry.articleBody }}
  </article>
{% endblock %}
```

### Including Partials and Components

Use `{% include %}` for partials and components. Pass variables explicitly:

```twig
{% include '_components/card' with {
  title: entry.title,
  url: entry.url,
  image: entry.featuredImage.one(),
} only %}
```

The `only` keyword prevents the included template from accessing the parent's variables, keeping components self-contained.

---

## 2. Entry Queries

### Builder Pattern

All element queries in Craft use a builder pattern. Construct with `craft.entries()`, chain parameters, then execute with a terminal method.

```twig
{% set posts = craft.entries()
  .section('blog')
  .type('article')
  .limit(10)
  .orderBy('postDate DESC')
  .all() %}
```

### Terminal Methods

| Method | Returns | Use When |
|--------|---------|----------|
| `.all()` | Array of elements | You need to iterate the results |
| `.collect()` | Laravel Collection | You need collection methods (e.g., `groupBy`, `pluck`) |
| `.one()` | Single element or `null` | You expect exactly one result |
| `.exists()` | `true` / `false` | You only need to know if results exist |
| `.count()` | Integer | You need the total count without loading elements |
| `.ids()` | Array of IDs | You only need element IDs |

### Common Query Parameters

```twig
{# By section and entry type #}
{% set articles = craft.entries()
  .section('blog')
  .type('article')
  .all() %}

{# By slug #}
{% set about = craft.entries()
  .section('pages')
  .slug('about')
  .one() %}

{# By ID #}
{% set entry = craft.entries().id(123).one() %}

{# Date filtering #}
{% set recent = craft.entries()
  .section('blog')
  .postDate('>= ' ~ now|date_modify('-30 days')|atom)
  .all() %}

{# Status — by default, only 'live' entries are returned #}
{% set drafts = craft.entries()
  .section('blog')
  .status('pending')
  .all() %}

{# Custom field conditions #}
{% set featured = craft.entries()
  .section('blog')
  .isFeatured(true)
  .all() %}

{# Offset and limit for manual pagination #}
{% set page2 = craft.entries()
  .section('blog')
  .limit(10)
  .offset(10)
  .all() %}
```

### Relational Queries

Query entries related to another element:

```twig
{# Entries related to the current entry (any field) #}
{% set related = craft.entries()
  .relatedTo(entry)
  .section('blog')
  .limit(3)
  .all() %}

{# Related via a specific field #}
{% set related = craft.entries()
  .relatedTo({
    targetElement: entry,
    field: 'relatedArticles',
  })
  .all() %}

{# Related to multiple elements (AND) #}
{% set related = craft.entries()
  .relatedTo(['and', category, tag])
  .all() %}

{# Related to any of multiple elements (OR — the default) #}
{% set related = craft.entries()
  .relatedTo([category, tag])
  .all() %}
```

### Search

```twig
{% set query = craft.app.request.getQueryParam('q') %}
{% set results = craft.entries()
  .search(query)
  .orderBy('score')
  .all() %}
```

### Structure Queries

For entries in Structure sections, use tree-traversal methods:

```twig
{# Direct children of an entry #}
{% set children = entry.children.all() %}

{# All descendants #}
{% set descendants = entry.descendants.all() %}

{# Parent #}
{% set parent = entry.parent %}

{# Siblings #}
{% set siblings = entry.siblings.all() %}

{# Ancestors (breadcrumb) #}
{% set ancestors = entry.ancestors.all() %}

{# Root-level entries in a structure #}
{% set roots = craft.entries()
  .section('documentation')
  .level(1)
  .all() %}
```

---

## 3. Eager Loading

### The N+1 Problem

Without eager loading, accessing a relational field inside a loop triggers a separate database query per iteration. Always eager load relational fields when iterating over entries.

```twig
{# BAD — N+1 queries #}
{% set entries = craft.entries().section('blog').all() %}
{% for entry in entries %}
  {% set image = entry.featuredImage.one() %}
{% endfor %}

{# GOOD — eager loaded #}
{% set entries = craft.entries()
  .section('blog')
  .with(['featuredImage'])
  .all() %}
{% for entry in entries %}
  {% set image = entry.featuredImage[0] ?? null %}
{% endfor %}
```

### `.with()` Syntax

Pass an array of field handles to `.with()`:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    'featuredImage',
    'categories',
    'author',
    'relatedArticles',
  ])
  .all() %}
```

### Nested Eager Loading (Dot Notation)

Eager load fields on related elements using dot notation:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    'relatedArticles.featuredImage',
    'categories.categoryImage',
  ])
  .all() %}
```

### Custom Parameters on Eager-Loaded Elements

Pass query parameters as a nested array:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    ['featuredImage', { withTransforms: ['thumbnail'] }],
    ['relatedArticles', { limit: 3, orderBy: 'postDate DESC' }],
    ['categories', { limit: 5 }],
  ])
  .all() %}
```

### Entry-Type-Scoped Eager Loading

In Craft 5, matrix/nested entries are scoped to entry types. Use `entryTypeHandle:fieldHandle` syntax:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    'contentBlocks',
    'contentBlocks.imageBlock:image',
    'contentBlocks.galleryBlock:images',
    'contentBlocks.quoteBlock:author',
  ])
  .all() %}
```

### `.eagerly()` — Lazy Eager Loading (Craft 5)

When you cannot predict which fields will be used (e.g., in partials loaded via switch), use `.eagerly()` to trigger batch loading on first access:

```twig
{% for entry in entries %}
  {% set image = entry.featuredImage.eagerly().one() %}
{% endfor %}
```

Craft batches these into a single query for all entries in the loop. This is less efficient than `.with()` but far better than unmanaged N+1 queries.

### `.andWith()` — Appending Eager-Load Paths

Use `.andWith()` to add eager-load paths without overwriting those already set:

```twig
{% set query = craft.entries().section('blog').with(['featuredImage']) %}
{% set query = query.andWith(['categories']) %}
{% set entries = query.all() %}
```

### `.withTransforms()` — Eager-Loading Image Transforms

Eager load named transforms to avoid per-image transform generation queries:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    ['featuredImage', { withTransforms: ['thumbnail', 'hero'] }],
  ])
  .all() %}
```

Or on a standalone asset query:

```twig
{% set images = craft.assets()
  .volume('images')
  .withTransforms(['thumbnail', 'hero'])
  .all() %}
```

### Eager-Loadable Attributes

Beyond custom fields, these native attributes can be eager loaded:

| Attribute | Notes |
|-----------|-------|
| `photo` | User photo |
| `author` | Entry author (Craft 5) |
| `ancestors` | Structure ancestors |
| `children` | Direct structure children |
| `descendants` | All structure descendants |
| `parent` | Structure parent |

---

## 4. Asset Handling & Image Transforms

### Querying Assets

```twig
{# All images in a volume #}
{% set images = craft.assets()
  .volume('images')
  .kind('image')
  .all() %}

{# Single asset by filename #}
{% set logo = craft.assets()
  .volume('images')
  .filename('logo.svg')
  .one() %}

{# Assets from an entry's field #}
{% set image = entry.featuredImage.one() %}
```

### Named Transforms

Define transforms in the control panel (Settings → Image Transforms) or in `config/image-transforms.php`, then reference by handle:

```twig
{% set image = entry.featuredImage.one() %}
{% if image %}
  <img src="{{ image.getUrl('thumbnail') }}"
       width="{{ image.getWidth('thumbnail') }}"
       height="{{ image.getHeight('thumbnail') }}"
       alt="{{ image.alt ?? image.title }}">
{% endif %}
```

### Inline Transforms

Define transforms directly in the template:

```twig
{% set transform = {
  mode: 'crop',
  width: 800,
  height: 600,
  quality: 80,
  format: 'webp',
} %}

{% set image = entry.featuredImage.one() %}
{% if image %}
  <img src="{{ image.getUrl(transform) }}"
       width="800"
       height="600"
       alt="{{ image.alt ?? image.title }}">
{% endif %}
```

### `getSrcset()` — Responsive Images

Generate a `srcset` attribute with multiple widths:

```twig
{% set image = entry.featuredImage.one() %}
{% if image %}
  <img src="{{ image.getUrl({ width: 800 }) }}"
       srcset="{{ image.getSrcset([400, 800, 1200, 1600]) }}"
       sizes="(max-width: 800px) 100vw, 800px"
       width="800"
       height="{{ 800 * (image.height / image.width)|round }}"
       alt="{{ image.alt ?? image.title }}">
{% endif %}
```

### `getImg()` — Quick Image Tag

Generate a complete `<img>` tag:

```twig
{{ image.getImg({ width: 800, height: 600 }) }}
```

### Responsive `<picture>` Pattern

```twig
{% set image = entry.featuredImage.one() %}
{% if image %}
  <picture>
    <source
      type="image/avif"
      srcset="{{ image.getSrcset([400, 800, 1200], { format: 'avif' }) }}"
      sizes="(max-width: 800px) 100vw, 800px">
    <source
      type="image/webp"
      srcset="{{ image.getSrcset([400, 800, 1200], { format: 'webp' }) }}"
      sizes="(max-width: 800px) 100vw, 800px">
    <img
      src="{{ image.getUrl({ width: 800 }) }}"
      srcset="{{ image.getSrcset([400, 800, 1200]) }}"
      sizes="(max-width: 800px) 100vw, 800px"
      width="{{ image.width }}"
      height="{{ image.height }}"
      alt="{{ image.alt ?? image.title }}"
      loading="lazy"
      decoding="async">
  </picture>
{% endif %}
```

### Eager-Loading Transforms

Always eager load transforms when displaying images in a loop:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    ['featuredImage', { withTransforms: ['thumbnail', 'hero'] }],
  ])
  .all() %}

{% for entry in entries %}
  {% set image = entry.featuredImage[0] ?? null %}
  {% if image %}
    <img src="{{ image.getUrl('thumbnail') }}" alt="{{ image.alt ?? image.title }}">
  {% endif %}
{% endfor %}
```

---

## 5. Matrix Fields (Craft 5 Entries)

In Craft 5, Matrix blocks have been replaced by nested entries. Each "block type" is now an entry type within the Matrix field.

### Iterating with a Switch

```twig
{% set blocks = entry.contentBlocks.all() %}

{% for block in blocks %}
  {% switch block.type.handle %}

    {% case 'richText' %}
      <div class="prose">
        {{ block.body }}
      </div>

    {% case 'image' %}
      {% set image = block.image.one() %}
      {% if image %}
        <figure>
          <img src="{{ image.getUrl({ width: 1200 }) }}"
               alt="{{ image.alt ?? image.title }}"
               loading="lazy">
          {% if block.caption %}
            <figcaption>{{ block.caption }}</figcaption>
          {% endif %}
        </figure>
      {% endif %}

    {% case 'quote' %}
      <blockquote>
        <p>{{ block.quoteText }}</p>
        {% if block.attribution %}
          <cite>{{ block.attribution }}</cite>
        {% endif %}
      </blockquote>

    {% case 'codeSnippet' %}
      <pre><code class="language-{{ block.language }}">{{ block.code|e }}</code></pre>

  {% endswitch %}
{% endfor %}
```

### Iterating with Partials (Preferred for Large Projects)

Delegate each entry type to its own template for cleaner organization:

```twig
{% set blocks = entry.contentBlocks.all() %}

{% for block in blocks %}
  {% include '_components/blocks/' ~ block.type.handle ignore missing %}
{% endfor %}
```

```twig
{# _components/blocks/richText.twig #}
<div class="prose">
  {{ block.body }}
</div>
```

```twig
{# _components/blocks/image.twig #}
{% set image = block.image.one() %}
{% if image %}
  <figure>
    <img src="{{ image.getUrl({ width: 1200 }) }}"
         alt="{{ image.alt ?? image.title }}"
         loading="lazy">
    {% if block.caption %}
      <figcaption>{{ block.caption }}</figcaption>
    {% endif %}
  </figure>
{% endif %}
```

### Filtering by Type

```twig
{# Only image blocks #}
{% set images = entry.contentBlocks
  .type('image')
  .all() %}

{# Count specific types #}
{% set imageCount = entry.contentBlocks
  .type('image')
  .count() %}
```

### Eager Loading Matrix Content

Scope eager loading to specific entry types within a Matrix field:

```twig
{% set entries = craft.entries()
  .section('blog')
  .with([
    'contentBlocks',
    'contentBlocks.image:image',
    'contentBlocks.gallery:images',
    'contentBlocks.quote:authorPhoto',
    ['contentBlocks.image:image', { withTransforms: ['contentImage'] }],
  ])
  .all() %}
```

### `.eagerly()` in Matrix Loops

When using the partials approach and you cannot predict all needed fields upfront:

```twig
{# _components/blocks/image.twig #}
{% set image = block.image.eagerly().one() %}
```

---

## 6. Navigation

### Structure Navigation

```twig
{% set pages = craft.entries()
  .section('pages')
  .level(1)
  .all() %}

<nav>
  <ul>
    {% nav page in pages %}
      <li>
        <a href="{{ page.url }}" {{ page.url == entry.url ? 'aria-current="page"' }}>
          {{ page.title }}
        </a>
        {% ifchildren %}
          <ul>
            {% children %}
          </ul>
        {% endifchildren %}
      </li>
    {% endnav %}
  </ul>
</nav>
```

> **Gotcha:** The `{% nav %}` tag requires entries from a Structure section with `lft` ordering. If you apply a custom `orderBy()` that overrides the default structure order, `{% nav %}` will render a flat list or produce unexpected nesting. If you need custom ordering, use a regular `{% for %}` loop instead.

### Breadcrumbs

```twig
{% if entry is defined and entry.ancestors|length %}
  <nav aria-label="Breadcrumb">
    <ol>
      <li><a href="{{ siteUrl }}">Home</a></li>
      {% for ancestor in entry.ancestors.all() %}
        <li><a href="{{ ancestor.url }}">{{ ancestor.title }}</a></li>
      {% endfor %}
      <li aria-current="page">{{ entry.title }}</li>
    </ol>
  </nav>
{% endif %}
```

---

## 7. Global Sets, Categories, Tags & Users

### Global Sets

Access global sets by their handle:

```twig
{# Access a global set directly #}
{{ siteInfo.companyName }}
{{ siteInfo.contactEmail }}

{# Asset field on a global set #}
{% set logo = siteInfo.logo.one() %}

{# Matrix/entries field on a global set #}
{% set socialLinks = siteInfo.socialLinks.all() %}
```

When working with multi-site setups, globals return content for the current site by default. To access a different site's globals:

```twig
{% set otherSiteInfo = craft.globals()
  .handle('siteInfo')
  .siteId(2)
  .one() %}
```

### Categories

```twig
{# All categories in a group #}
{% set categories = craft.categories()
  .group('blogCategories')
  .all() %}

{# Categories related to an entry #}
{% set entryCategories = entry.blogCategories.all() %}

{# Category with children (structure) #}
{% set categories = craft.categories()
  .group('blogCategories')
  .level(1)
  .all() %}

{% nav category in categories %}
  <li>
    <a href="{{ category.url }}">{{ category.title }}</a>
    {% ifchildren %}
      <ul>
        {% children %}
      </ul>
    {% endifchildren %}
  </li>
{% endnav %}
```

### Tags

```twig
{# All tags in a group #}
{% set tags = craft.tags()
  .group('blogTags')
  .all() %}

{# Tags related to an entry #}
{% set entryTags = entry.blogTags.all() %}

{# Render as a list #}
{% for tag in entryTags %}
  <a href="{{ url('blog', { tag: tag.slug }) }}">{{ tag.title }}</a>
  {{ not loop.last ? ',' }}
{% endfor %}
```

### Users

```twig
{# Query users by group #}
{% set authors = craft.users()
  .group('authors')
  .all() %}

{# Current logged-in user #}
{% if currentUser %}
  <p>Welcome, {{ currentUser.friendlyName }}!</p>
{% endif %}

{# Entry author #}
{% set author = entry.author %}
{% if author %}
  <p>By {{ author.fullName }}</p>
  {% set photo = author.photo %}
  {% if photo %}
    <img src="{{ photo.getUrl({ width: 80, height: 80 }) }}" alt="{{ author.fullName }}">
  {% endif %}
{% endif %}
```

### Addresses (Craft 5)

```twig
{# Addresses from an address field #}
{% set address = entry.officeAddress.one() %}
{% if address %}
  <address>
    {{ address.addressLine1 }}<br>
    {% if address.addressLine2 %}{{ address.addressLine2 }}<br>{% endif %}
    {{ address.locality }}, {{ address.administrativeArea }} {{ address.postalCode }}<br>
    {{ address.countryCode }}
  </address>
{% endif %}
```

---

## 8. Craft-Specific Twig Tags

### `{% cache %}`

Cache a template fragment to reduce database queries:

```twig
{% cache %}
  {# This block is cached until any element within it changes #}
  {% set entries = craft.entries().section('blog').limit(10).all() %}
  {% for entry in entries %}
    <article>
      <h2><a href="{{ entry.url }}">{{ entry.title }}</a></h2>
    </article>
  {% endfor %}
{% endcache %}
```

With options:

```twig
{# Cache with a key and expiration #}
{% cache using key "blog-sidebar" for 1 hour %}
  {# ... #}
{% endcache %}

{# Cache globally (shared across pages) #}
{% cache globally using key "global-footer" %}
  {# ... #}
{% endcache %}

{# Cache per-user #}
{% cache using key "user-dashboard-#{currentUser.id}" %}
  {# ... #}
{% endcache %}
```

> **Gotcha:** Never cache CSRF tokens or any personalized/session-dependent content inside a `{% cache %}` block. The cached CSRF token will be stale for subsequent requests, causing form submissions to fail. See Section 14 for details.

### `{% redirect %}`

```twig
{% if not currentUser %}
  {% redirect '/login' %}
{% endif %}

{# With status code #}
{% redirect '/new-url' 301 %}
```

### `{% requireLogin %}`

Immediately redirects guests to the login page:

```twig
{% requireLogin %}
{# Everything below only renders for logged-in users #}
```

### `{% exit %}`

Halt template rendering and return an HTTP status code:

```twig
{% if entry is not defined %}
  {% exit 404 %}
{% endif %}
```

### `{% dd %}`

Dump and die — output a variable's contents and stop rendering. For debugging only:

```twig
{% dd entry %}
{% dd craft.entries().section('blog').all() %}
```

### `{% tag %}`

Generate HTML tags with dynamic attributes:

```twig
{% tag 'div' with {
  class: 'card',
  id: 'card-' ~ entry.id,
  'data-entry-id': entry.id,
} %}
  <h3>{{ entry.title }}</h3>
{% endtag %}
```

### `{% switch %}`

Craft's multi-case switch (more readable than Twig's ternary chains):

```twig
{% switch entry.type.handle %}
  {% case 'article' %}
    {% include '_components/article' %}
  {% case 'video' %}
    {% include '_components/video' %}
  {% case 'gallery' %}
    {% include '_components/gallery' %}
  {% default %}
    {% include '_components/default' %}
{% endswitch %}
```

### `{% nav %}`

Renders hierarchical/nested navigation from structure elements. See Section 6 for full examples.

```twig
{% nav entry in craft.entries().section('pages').all() %}
  <li>
    <a href="{{ entry.url }}">{{ entry.title }}</a>
    {% ifchildren %}
      <ul>
        {% children %}
      </ul>
    {% endifchildren %}
  </li>
{% endnav %}
```

### `{% css %}` and `{% js %}`

Register inline CSS or JavaScript for the page:

```twig
{% css %}
  .hero { background-color: {{ entry.heroColor }}; }
{% endcss %}

{% js %}
  console.log('Entry ID:', {{ entry.id }});
{% endjs %}

{# With a file URL #}
{% css '/assets/css/lightbox.css' %}
{% js '/assets/js/lightbox.js' %}
```

---

## 9. Craft-Specific Twig Filters

### Translation — `|t`

Translate strings using Craft's static translation system:

```twig
{{ 'Read more'|t }}

{# With a translation category #}
{{ 'Welcome back, {name}'|t('site', { name: currentUser.friendlyName }) }}
```

### Date & Time Filters

```twig
{# Standard date formatting #}
{{ entry.postDate|date('F j, Y') }}          {# January 15, 2025 #}
{{ entry.postDate|date('m/d/Y g:ia') }}      {# 01/15/2025 3:30pm #}

{# Atom format (ISO 8601 — for datetime attributes) #}
<time datetime="{{ entry.postDate|atom }}">{{ entry.postDate|date('F j, Y') }}</time>

{# RSS format (RFC 2822 — for RSS feeds) #}
{{ entry.postDate|rss }}
```

### Number & Currency

```twig
{{ 1234567.89|number }}              {# 1,234,567.89 #}
{{ 1234567.89|number(0) }}           {# 1,234,568 #}
{{ 49.99|currency('USD') }}          {# $49.99 #}
```

### Text Processing

```twig
{# Parse Markdown to HTML #}
{{ entry.markdownField|markdown }}

{# Purify HTML (strip unsafe tags) #}
{{ entry.richTextField|purify }}
```

### Collection Filters

```twig
{# Group entries by a field value #}
{% set grouped = entries|group(e => e.postDate|date('Y')) %}
{% for year, yearEntries in grouped %}
  <h2>{{ year }}</h2>
  {% for entry in yearEntries %}
    <p>{{ entry.title }}</p>
  {% endfor %}
{% endfor %}

{# Extract a single property from an array of objects #}
{% set titles = entries|column('title') %}

{# Sort by multiple criteria #}
{% set sorted = entries|multisort('postDate', SORT_DESC, 'title', SORT_ASC) %}
```

### `|literal`

Escape a string for use in an element query's `search` parameter, preventing operators from being interpreted:

```twig
{% set query = craft.app.request.getQueryParam('q') %}
{% set results = craft.entries()
  .search(query|literal)
  .all() %}
```

---

## 10. Forms

### CSRF Protection

All forms must include a CSRF token. Craft provides the `csrfInput()` function:

```twig
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('entries/save-entry') }}
  {# form fields #}
</form>
```

### Hidden Inputs

| Function | Purpose |
|----------|---------|
| `csrfInput()` | CSRF token hidden field |
| `actionInput('controller/action')` | Sets the controller action to invoke |
| `redirectInput('/thank-you')` | Sets the post-action redirect URL |

### Contact Form Example

```twig
<form method="post" accept-charset="UTF-8">
  {{ csrfInput() }}
  {{ actionInput('entries/save-entry') }}
  {{ redirectInput('/thank-you') }}
  {{ hiddenInput('sectionId', craft.app.sections.getSectionByHandle('contactSubmissions').id) }}
  {{ hiddenInput('typeId', craft.app.sections.getSectionByHandle('contactSubmissions').entryTypes[0].id) }}
  {{ hiddenInput('enabled', '0') }}

  <label for="title">Name</label>
  <input type="text" id="title" name="title" value="{{ entry is defined ? entry.title }}" required>

  <label for="email">Email</label>
  <input type="email" id="email" name="fields[emailAddress]" value="{{ entry is defined ? entry.emailAddress }}" required>

  <label for="message">Message</label>
  <textarea id="message" name="fields[message]" required>{{ entry is defined ? entry.message }}</textarea>

  <button type="submit">Send</button>
</form>
```

### Flash Messages

After a successful form submission with a redirect:

```twig
{% if craft.app.session.hasFlash('notice') %}
  <div role="alert" class="alert alert--success">
    {{ craft.app.session.getFlash('notice') }}
  </div>
{% endif %}

{% if craft.app.session.hasFlash('error') %}
  <div role="alert" class="alert alert--error">
    {{ craft.app.session.getFlash('error') }}
  </div>
{% endif %}
```

### Validation Errors

When a form submission fails validation, Craft routes back to the form with an `entry` variable (or `submission`, depending on context) containing the errors:

```twig
{% if entry is defined and entry.hasErrors() %}
  <div class="errors" role="alert">
    <ul>
      {% for error in entry.getErrors()|flatten %}
        <li>{{ error }}</li>
      {% endfor %}
    </ul>
  </div>
{% endif %}

{# Per-field errors #}
<label for="title">Name</label>
<input type="text" id="title" name="title"
       value="{{ entry is defined ? entry.title }}"
       class="{{ entry is defined and entry.getErrors('title') ? 'has-error' }}">
{% if entry is defined and entry.getErrors('title') %}
  <p class="error-message">{{ entry.getErrors('title')|join(', ') }}</p>
{% endif %}
```

### AJAX Form Submission

Use the `Accept: application/json` header to receive JSON responses from Craft actions:

```twig
{% js %}
document.querySelector('#contact-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const form = e.target;
  const response = await fetch('/', {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
    },
    body: new FormData(form),
  });
  const result = await response.json();
  if (response.ok) {
    form.reset();
    // Show success message
  } else {
    // Handle result.errors
  }
});
{% endjs %}
```

---

## 11. Pagination

### `{% paginate %}` Tag

```twig
{% set query = craft.entries()
  .section('blog')
  .limit(12) %}

{% paginate query as pageInfo, entries %}

{% for entry in entries %}
  {% include '_components/card' with { entry: entry } only %}
{% endfor %}

{% if pageInfo.totalPages > 1 %}
  <nav aria-label="Pagination">
    {% if pageInfo.prevUrl %}
      <a href="{{ pageInfo.prevUrl }}">&larr; Previous</a>
    {% endif %}

    {% for page, url in pageInfo.getPrevUrls(2) %}
      <a href="{{ url }}">{{ page }}</a>
    {% endfor %}

    <span aria-current="page">{{ pageInfo.currentPage }}</span>

    {% for page, url in pageInfo.getNextUrls(2) %}
      <a href="{{ url }}">{{ page }}</a>
    {% endfor %}

    {% if pageInfo.nextUrl %}
      <a href="{{ pageInfo.nextUrl }}">Next &rarr;</a>
    {% endif %}
  </nav>
{% endif %}
```

### `pageInfo` Properties

| Property | Type | Description |
|----------|------|-------------|
| `pageInfo.first` | int | First element's 1-based index on this page |
| `pageInfo.last` | int | Last element's 1-based index on this page |
| `pageInfo.total` | int | Total number of matched elements |
| `pageInfo.currentPage` | int | Current page number |
| `pageInfo.totalPages` | int | Total number of pages |
| `pageInfo.prevUrl` | string/false | URL to previous page, or `false` |
| `pageInfo.nextUrl` | string/false | URL to next page, or `false` |
| `pageInfo.getPageUrl(n)` | string | URL for page number `n` |
| `pageInfo.getPrevUrls(n)` | array | Up to `n` previous page URLs |
| `pageInfo.getNextUrls(n)` | array | Up to `n` next page URLs |

> **Gotcha:** Pass the query object to `{% paginate %}`, **not** the result of `.all()`. The tag needs the query to apply limit/offset internally. Writing `{% paginate query.all() as ... %}` will throw an error.

---

## 12. Environment & Configuration

### Accessing Environment Variables

```twig
{# In Craft 5, use App::env() via a global or access in config #}
{# Environment variables should generally be used in config files, not templates #}

{# Use aliases (preferred) — set in config/general.php #}
{{ alias('@baseAssetsUrl') }}
{{ alias('@web') }}
```

### Multi-Environment Configuration

Craft uses `config/general.php` for environment-aware settings:

```php
// config/general.php
use craft\config\GeneralConfig;

return GeneralConfig::create()
    ->defaultWeekStartDay(1)
    ->omitScriptNameInUrls()
    ->devMode(App::env('DEV_MODE') ?? false)
    ->allowAdminChanges(App::env('ALLOW_ADMIN_CHANGES') ?? false)
    ->aliases([
        '@web' => App::env('PRIMARY_SITE_URL'),
        '@webroot' => App::env('WEB_ROOT_PATH') ?? dirname(__DIR__) . '/web',
        '@baseAssetsUrl' => App::env('PRIMARY_SITE_URL') . '/assets',
    ]);
```

### Useful Template Variables

| Variable | Description |
|----------|-------------|
| `siteName` | Current site's name |
| `siteUrl` | Current site's base URL |
| `currentSite` | Current `Site` object (`.language`, `.handle`, `.name`) |
| `currentUser` | Logged-in `User` object or `null` |
| `devMode` | `true` if `devMode` is enabled |
| `now` | Current `DateTime` object |

### Conditional Dev Mode Content

```twig
{% if devMode %}
  <div class="debug-bar">
    Template: {{ _self }}
    Entry ID: {{ entry.id ?? 'N/A' }}
  </div>
{% endif %}
```

---

## 13. Development of Custom Plugins and Modules

### Modules

- Use `ddev craft make module` to scaffold new modules using Craft's built-in generator.
- Module namespaces should follow the pattern `modules\<modulename>`.
- Register modules in `config/app.php` under the `modules` key.

### Plugins

- Use `ddev craft make plugin` to scaffold new plugins using Craft's built-in generator.
- Custom plugins are located in `local-plugins/` and `plugins/`. Use Composer for dependencies and follow Craft CMS plugin development best practices.

---

## 14. Common Pitfalls

### N+1 Queries

**Problem:** Accessing relational fields inside loops without eager loading triggers a separate query per iteration.

```twig
{# BAD — triggers N+1 queries #}
{% for entry in craft.entries().section('blog').all() %}
  {% set image = entry.featuredImage.one() %}
  {% set categories = entry.categories.all() %}
{% endfor %}
```

**Fix:** Use `.with()` to eager load, or `.eagerly()` as a fallback:

```twig
{# GOOD #}
{% set entries = craft.entries()
  .section('blog')
  .with(['featuredImage', 'categories'])
  .all() %}
```

### `.all()|length` vs `.count()`

**Problem:** `.all()|length` loads every element into memory just to count them.

```twig
{# BAD — loads all entries into memory #}
{% set total = craft.entries().section('blog').all()|length %}

{# GOOD — COUNT(*) query only #}
{% set total = craft.entries().section('blog').count() %}
```

### Caching CSRF Tokens

**Problem:** A `{% cache %}` block containing `{{ csrfInput() }}` stores a stale token. Subsequent page loads serve the cached (expired) token, and form submissions fail with 400 errors.

```twig
{# BAD #}
{% cache %}
  <form method="post">
    {{ csrfInput() }}
    {# ... #}
  </form>
{% endcache %}

{# GOOD — CSRF token outside cache #}
{% cache %}
  <form method="post" id="my-form">
    {# ... #}
  </form>
{% endcache %}
{{ csrfInput() }}
```

Or inject the CSRF token via JavaScript using safe DOM methods:

```twig
{% cache %}
  <form method="post" id="my-form">
    <span id="csrf-container"></span>
    {# ... #}
  </form>
{% endcache %}

{% js %}
  fetch('/actions/users/session-info', {
    headers: { 'Accept': 'application/json' }
  })
  .then(r => r.json())
  .then(data => {
    const container = document.getElementById('csrf-container');
    const input = document.createElement('input');
    input.type = 'hidden';
    input.name = data.csrfTokenName;
    input.value = data.csrfTokenValue;
    container.appendChild(input);
  });
{% endjs %}
```

### Caching Personalized Content

**Problem:** Content that varies per user (e.g., "Welcome, John") is cached and shown to all users.

```twig
{# BAD #}
{% cache %}
  <p>Welcome, {{ currentUser.friendlyName }}</p>
{% endcache %}

{# GOOD — either skip caching or use a per-user cache key #}
{% cache using key "welcome-#{currentUser.id}" %}
  <p>Welcome, {{ currentUser.friendlyName }}</p>
{% endcache %}
```

### Missing `lft ASC` for `{% nav %}`

**Problem:** Using `{% nav %}` with a custom `orderBy()` that doesn't include `lft ASC` breaks the hierarchical rendering. The tag relies on the `lft` column to determine nesting.

```twig
{# BAD — custom order breaks {% nav %} hierarchy #}
{% nav entry in craft.entries().section('pages').orderBy('title ASC').all() %}

{# GOOD — use default structure order for {% nav %} #}
{% nav entry in craft.entries().section('pages').all() %}

{# If you need custom ordering, use {% for %} instead #}
{% for entry in craft.entries().section('pages').orderBy('title ASC').all() %}
  <li><a href="{{ entry.url }}">{{ entry.title }}</a></li>
{% endfor %}
```

### Passing `.all()` to `{% paginate %}`

**Problem:** The `{% paginate %}` tag expects an element query object, not an array. Calling `.all()` before passing it to `{% paginate %}` will throw an error.

```twig
{# BAD — .all() returns an array, not a query #}
{% paginate craft.entries().section('blog').all() as pageInfo, entries %}

{# GOOD — pass the query directly #}
{% paginate craft.entries().section('blog').limit(12) as pageInfo, entries %}
```

### Forgetting `|e` for User-Supplied Content in Attributes

**Problem:** Twig auto-escapes in content, but not within `{{ }}` inside HTML attributes in all contexts. Always escape user-supplied or dynamic content used in attributes.

```twig
{# Be explicit when inserting into attributes #}
<a title="{{ entry.title|e('html_attr') }}">{{ entry.title }}</a>
```

### Using `entry.url` Without Null Checks

**Problem:** If a section's entries don't have URIs (URI Format is blank), `entry.url` is `null` and renders as an empty string, producing broken links.

```twig
{# GOOD — check before linking #}
{% if entry.url %}
  <a href="{{ entry.url }}">{{ entry.title }}</a>
{% else %}
  <span>{{ entry.title }}</span>
{% endif %}
```
