# Multi-layer i18n Architecture

## 1. Locale as a Contract

Use a short, stable Locale ID as the single source of truth inside the application.

```text
Locale ID
  ├─ UI Catalog
  ├─ Route Prefix
  ├─ CMS Query
  ├─ BCP-47 / Intl
  ├─ SEO / hreflang
  └─ LLM Response Language
```

Map display names, BCP-47 tags, RSS language values, and LLM-facing language names centrally from that Locale ID.

## 2. Six Layers

### A. UI Strings

Keep the same key structure across message catalogs. If catalogs contain arrays or objects as well as strings, centralize the runtime resolution rules.

### B. Routing

Define the Locale Prefix policy explicitly. Central configuration should decide whether the default locale omits its prefix, how Browser Language Redirect works, and how locale preference is persisted.

### C. CMS Content

Include Locale in both CMS requests and Cache Keys.

```text
content:<locale>:<resource>
```

This prevents content from different languages from being mixed because of locale-agnostic caching.

### D. SEO

Derive Canonical URLs, `hreflang`, `x-default`, `<html lang>`, JSON-LD `inLanguage`, Sitemap entries, and RSS metadata from the same Locale Mapping.

### E. Static Data

When localized FAQ or SEO data lives in a separate code-based system, adding a locale can easily miss one of those sources. Prefer consolidating such data into the Catalog/CMS, or at minimum hide it behind a single Selector API.

### F. AI Response Language

Map the Frontend Locale to the Target Language used by LLM prompts. Validate that hard-coded language instructions in System Rules do not conflict with the runtime Locale.

## 3. Central Locale Helper

Avoid repeating `locale === ... ? ...` branches throughout the application. Centralize at least:

- BCP-47 mapping
- localized path generation
- date/number Intl locale
- LLM language name
- fallback locale
- site/channel display name where needed

## 4. Adding a Locale

When adding a new Locale, review at least:

- Locale type/config
- UI message catalog
- Route/prefix strategy
- CMS locale availability
- static localized data
- date/number formatting
- SEO alternate URLs
- Sitemap/RSS
- AI response language mapping
- fallback behavior

## 5. Common Failure Modes

- Sitemap and App Router maintain different hard-coded path lists
- localized Static Data is duplicated across multiple files
- Locale Mapping is scattered through inline ternary expressions
- hard-coded language rules in the LLM System Prompt conflict with the runtime Locale
- Locale is missing from CMS Cache Keys
- `hreflang` advertises translated pages that do not actually exist

## 6. Validation

- compare Key Sets across every Locale Catalog
- snapshot localized Routes
- validate reciprocal `hreflang` links
- verify CMS locale/cache isolation
- validate JSON-LD language per Locale
- ensure LLM Target Language matches the UI Locale
