# Multilingual (i18n) Processing Flow Program Specification

## 1. Overview

| Item | Value |
|---|---|
| Project | `nougat-resume-wc` (Nuxt 4 portfolio/resume site) |
| Target domain | `chu.plus` |
| Supported locales | `ko` (default), `en`, `ja` |
| i18n module | `@nuxtjs/i18n` v10.3.0 |
| Content backend | Strapi v5 (`PUBLIC_STRAPI_URL`) |
| AI chat backend | `nougat-career-be` (`https://api.chu.plus/resume/stream`) |

Multilingual processing is separated into 6 layers: **(A) UI strings**, **(B) routing/URL**, **(C) CMS content**, **(D) SEO/hreflang/sitemap/RSS**, **(E) static data (FAQ/SEO pages)**, and **(F) AI chat response language**.

---

## 2. Locale Identifier System

`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\types\i18n.ts" />`

- **App internal canonical code**: `Locale = "ko" | "en" | "ja"` (basis for all logic)
- **BCP-47 mapping**: `ko→ko-KR`, `en→en-US`, `ja→ja-JP` (used for HTML `lang`, `Intl.DateTimeFormat`, JSON-LD `inLanguage`)
- **LLM language name mapping**: `ko→Korean`, `en→English`, `ja→Japanese` (for AI prompts)
- **RSS language code**: `ko-kr`, `en-us`, `ja-jp`
- `localeConfig`: `{ code, name, flag }[]` for UI switcher (🇰🇷 한국어 / 🇺🇸 English / 🇯🇵 日本語)

---

## 3. Module Configuration (`nuxt.config.ts`)

`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\nuxt.config.ts" lines="35-53" />`

| Setting | Value | Meaning |
|---|---|---|
| `strategy` | `prefix_and_default` | Default locale (ko) has no URL prefix, en/ja use `/en`, `/ja` prefix |
| `defaultLocale` | `ko` | Korean is the default |
| `langDir` | `locales` (+ `restructureDir: "."`) | Loads `./locales/{ko,en,ja}.json` |
| `detectBrowserLanguage.useCookie` | `true` (key: `i18n_redirected`) | Detect browser language on first visit → save to cookie |
| `detectBrowserLanguage.redirectOn` | `root` | Redirect only on root (`/`) visit |
| `detectBrowserLanguage.alwaysRedirect` | `false` | Keep already-determined locale |
| `fallbackLocale` | `ko` | Fall back to Korean on missing key/detection failure |

---

## 4. UI String Layer (Vue I18n JSON Catalog)

### 4.1 Resources
- `locales/ko.json`, `locales/en.json`, `locales/ja.json` (approximately 918 lines each, identical key structure)
- Namespace examples: `nav.*`, `home.*`, `columns.*`, `chat.*`, `theme.*`, `language.*`, `common.*`

### 4.2 Access API
- **Single string**: `useI18n().t(key)` — used in most components
- **Array/object messages** (`chat.suggestions`, `chat.questions`): `useResolvedI18nMessage().tmResolved<T>(key)`
  - `tm()` returns Vue I18n AST, so a recursive helper that resolves to actual strings using `rt()`
  - `<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useResolvedI18nMessage.ts" />`

### 4.3 Common composable
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useLocale.ts" />`
- `lang`: `computed(() => locale.value as Locale)` — reactive locale
- `href(path)`: `localePath(path)` wrapper (generates URL with locale prefix applied)
- `localePath`: `useLocalePath()` direct exposure

### 4.4 Usage Pattern (Component Standard)
```ts
const { lang, href } = useLocale();
const { t } = useI18n();
usePageSeo({ lang: lang.value, path: "...", title: t("..."), description: t("...") });
```
- 39 files use at least one of `useI18n`/`useLocale`/`localePath`/`switchLocalePath`

---

## 5. Routing / URL Layer

### 5.1 URL Structure (`prefix_and_default`)
| Locale | Home | Column list | Column detail |
|---|---|---|---|
| ko | `/` | `/columns` | `/columns/{slug}` |
| en | `/en/` | `/en/columns` | `/en/columns/{slug}` |
| ja | `/ja/` | `/ja/columns` | `/ja/columns/{slug}` |

Static path list is hardcoded in `STATIC_PATHS` in `server/utils/create-sitemap.ts` and `staticPaths` in `components/RoutePrefetcher.vue` (same 13 paths).

### 5.2 Locale Switch UI
- **Desktop**: `<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\components\LocaleSwitcher.vue" />`
  - `switchLocalePath(code)` → `navigateTo(path)` (navigate to a different locale version of the same route)
- **Mobile**: `MobileMenu.vue` line 148-158 — `localePath(basePath, item.code)` directly bound to `NuxtLink :to`

### 5.3 Prefetch
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\components\RoutePrefetcher.vue" />`
- Renders hidden `<NuxtLink prefetch>` for all static paths × 3 locales to warm the route cache

---

## 6. CMS Content Locale Layer (Strapi)

### 6.1 Data Flow
```
Page (lang.value) → useColumnsProvider() → getColumns(baseUrl, locale, ...) → Strapi /columns?locale=ko|en|ja
```

`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useColumns.ts" />`

### 6.2 Strapi Calls
`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\lib\data\columns.ts" lines="376-407" />`
- `getColumns(baseUrl, locale, appEnv, apiKey)`: Sets `params.set("locale", locale)` then calls `/columns?...&locale=ko`
- `getColumnBySlug`, `getRelatedColumns` follow the same pattern
- In Dev environment (`APP_ENV=development`), falls back to `SAMPLE_COLUMNS` (English mock) on Strapi failure

### 6.3 SWR Cache Keys
- `columns:${lang.value}` (list), `column:${lang.value}:${slug}` (detail)
- Locale is included in the cache key for independent caching per locale
- `useColumnSWR` uses SSR payload cache + background revalidation (SWR) after client mount

---

## 7. SEO / hreflang / Sitemap / RSS Layer

### 7.1 Page SEO (`usePageSeo`)
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\usePageSeo.ts" />`

- Obtain `hreflang` alternate links from `useLocaleHead({ seo: true })`
- Convert relative href to absolute using site origin
- If `x-default` is missing, add canonical as x-default
- Set `<html lang>`, `dir` attributes
- Inject `inLanguage: currentLocale` into JSON-LD (`WebPage`, `BlogPosting`)
- Add 3-locale RSS alternate links (ko/en/ja each)
- `getSiteName(lang)` in `lib/seo.ts`: "Heonnam Chu" for en, "Chu Heon Nam" otherwise

### 7.2 Sitemap (`/sitemap.xml`)
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\server\utils\create-sitemap.ts" />`

- Fetch 3-locale columns in parallel to build `columnMaps[slug] = { ko?, en?, ja?, lastmod }`
- Entry groups:
  1. Default locale (ko) static paths — no prefix
  2. Default locale columns — `/columns/{slug}`
  3. Non-default locale (en/ja) static+columns — `/{lang}/...`
- `alternates` for each entry:
  - Static: 4 entries — `x-default`, `ko`, `en`, `ja`
  - Columns: only locales where the slug exists are included in alternates (omitted if not present)
- `x-default` is always the ko (no-prefix) URL

### 7.3 RSS
- `/rss.xml` → ko feed (default)
- `/{locale}/rss.xml` → only en and ja allowed (validated with `["en","ja"]` in `server/routes/[locale]/rss.xml.ts`, 404 otherwise)
- `getChannelInfo(lang)` returns channel title/description/language code per locale
- Item links also have locale prefix applied

---

## 8. Static Data Locale Layer (Outside JSON Catalog)

Instead of Vue I18n JSON, a separate layer exists that **defines fields in parallel for 3 locales in TS data files**.

### 8.1 FAQ Data
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\data\faq-data.ts" />`
- 3 arrays: `faqDataKo`, `faqDataEn`, `faqDataJa` + `getFaqDataByLang(lang)` selector (ko fallback)
- Used in `pages/faq.vue` via `getFaqDataByLang(lang.value)`
- Same data also used in FAQPage JSON-LD schema

### 8.2 SEO Page Data
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\data\seo-pages-data.ts" />`
- `SeoPageItem` interface: `titleKo/titleEn/titleJa`, `metaKo/metaEn/metaJa`, `descriptionKo/descriptionEn/descriptionJa`, `chatCtas[].{label,query}{Ko,En,Ja}`
- Used by 3 pages: `who-is-chuheonam.vue`, `nougat-developer.vue`, `chuheonam-projects.vue` (all use `getSeoPageBySlug` + `lang.value` branching)
- `SeoPageContent.vue` selects fields via `lang.value === "ko" ? ... : "ja" ? ... : ...` ternary branching

> ⚠️ This layer operates in parallel with the JSON catalog without duplication. When adding a new locale, both systems must be updated.

---

## 9. AI Chat Response Language Layer

### 9.1 Frontend Flow
`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useChatState.ts" lines="13-15" />`
```
locale.value (Locale) ──getChatLanguage()──▶ "Korean"|"Japanese"|"English"
```
- `useChatState.sendMessage()` includes `language: getChatLanguage(locale.value)` in the stream request payload
- Fallback error messages use `locale.value === "ko" ? ... : "ja" ? ... : ...` inline ternary branching (line 506-510, 536-540)

### 9.2 Transmission
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\lib\streamClient.ts" />`
- `StreamRequest.language` field → encrypted body POST to `https://api.chu.plus/resume/stream`

### 9.3 Backend (`nougat-career-be`) Processing
1. `src/wedding/stream/capability.ts validateStreamRequest`: Extract `language`, default `"Korean"`
2. `streamController.ts`: Inject language into `IntegrationSession`, call `callGPTStream(message, language, systemPrompt)`
3. `tools/callGPT.ts`: Pass language when dispatching to vendor (OpenAI/Gemini/InceptionLabs)
4. `tools/getFormattedPrompt.ts` line 444-457:
   ```
   TARGET ANSWERING LANGUAGE
   ${language}
   ```
   Inject "TARGET ANSWERING LANGUAGE: Korean/English/Japanese" block into system prompt → LLM responds in that language

> Note: Some custom rules in `catalog.ts` are hardcoded to "Always respond in Korean", which may cause prompt conflicts in non-ko locales.

---

## 10. Date/Time Formatting

- `intl` computed in column list/detail pages:
  ```ts
  const intl = computed(() =>
    lang.value === "ko" ? "ko-KR" : lang.value === "ja" ? "ja-JP" : "en-US",
  );
  ```
- Passed to `formatDateTime(dateStr, intl, ...)` to apply locale formatting based on `Intl.DateTimeFormat`

---

## 11. Overall Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User entry                                                   │
│    URL prefix (/en/...) ── or ── cookie i18n_redirected (root)  │
│                              ↓                                  │
│    @nuxtjs/i18n → useI18n().locale = "ko"|"en"|"ja"             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
   [UI strings]           [Routing]            [CMS content]
   t(key)                localePath()           useColumnsProvider
   tmResolved()          switchLocalePath()     → Strapi ?locale=
   locales/*.json        prefix_and_default     (ko/en/ja parallel)
        ↓                     ↓                     ↓
        └─────────────────────┼─────────────────────┘
                              ↓
                     [Static data parallel layer]
                     faq-data.ts (Ko/En/Ja arrays)
                     seo-pages-data.ts (Ko/En/Ja fields)
                     getFaqDataByLang(lang) / ternary branching
                              ↓
                     [SEO meta]
                     usePageSeo → useLocaleHead
                     hreflang alternate + x-default
                     JSON-LD inLanguage
                     <html lang>
                              ↓
                     [Sitemap / RSS (server)]
                     3-locale column parallel fetch
                     hreflang alternate matrix
                     /rss.xml (ko), /{lang}/rss.xml (en/ja)
                              ↓
                     [AI chat]
                     getChatLanguage(locale) → "Korean"/...
                     streamClient POST /resume/stream
                     BE: Inject into getFormattedPrompt → determines LLM response language
```

---

## 12. Checklist for Adding a New Locale

When adding a new locale (e.g., `zh`), **all 6 layers** need modification:

1. Add `{ code, name, language, file }` to `i18n.locales` array in `nuxt.config.ts`
2. `types/i18n.ts`: Add to `Locale` union, `locales` array, and `localeConfig` entry
3. Create new `locales/zh.json` file (replicate entire existing key structure)
4. `data/faq-data.ts`: Add `faqDataZh` array + `getFaqDataByLang` branching
5. `data/seo-pages-data.ts`: Add `titleZh/metaZh/descriptionZh` and `chatCtas[].{labelZh,queryZh}` to all `SeoPageItem`s, add branching in `SeoPageContent.vue`
6. `server/utils/create-sitemap.ts`: Add zh branching to `columnMaps` type, `getStaticAlternates`, `getColumnAlternates`
7. `server/utils/create-rss.ts`: Add zh branching to `getChannelInfo`, add zh to `supportedLangs` in `server/routes/[locale]/rss.xml.ts`
8. `composables/useChatState.ts`: Add zh mapping to `getChatLanguage` + inline fallback message branching
9. `lib/seo.ts` `getSiteName`: Add zh branching (if needed)
10. `intl` computed in column list/detail pages: Add zh → `zh-CN` mapping
11. `components/RoutePrefetcher.vue` `localizedPaths`: Add zh key
12. Enter zh locale data in Strapi content

---

## 13. Notes / Improvement Opportunities

1. **Static path duplicate definition**: `STATIC_PATHS` (create-sitemap.ts) and `staticPaths` (RoutePrefetcher.vue) hardcode the same 13 paths. Recommend consolidating to a single source.
2. **Static data i18n dualization**: FAQ/SEO pages use TS field parallel definition instead of JSON catalog. High maintenance cost and risk of omissions. Consider consolidating to JSON catalog or Strapi i18n plugin.
3. **Inline ternary branching repetition**: The `lang.value === "ko" ? ... : "ja" ? ... : ...` pattern is scattered across `SeoPageContent.vue`, `useChatState.ts`, column page `intl`, etc. Recommend centralizing by adding BCP-47/LLM language name/fallback message helpers to `useLocale()`.
4. **AI prompt conflict**: The hardcoded "Always respond in Korean" rule in `catalog.ts` and the `TARGET ANSWERING LANGUAGE` in `getFormattedPrompt` may conflict in non-ko locales. The catalog rule needs to be parameterized by locale.
5. **Sitemap `x-default` policy**: Always fixes ko (no-prefix) as x-default. Depending on the proportion of multilingual users, there is room to consider a policy where `x-default` points to the browser language auto-detection page (`/`).
