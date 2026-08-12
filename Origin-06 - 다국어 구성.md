# 다국어(i18n) 처리 흐름 프로그램 명세서

## 1. 개요

| 항목 | 값 |
|---|---|
| 프로젝트 | `nougat-resume-wc` (Nuxt 4 포트폴리오/이력서 사이트) |
| 대상 도메인 | `chu.plus` |
| 지원 로케일 | `ko`(기본), `en`, `ja` |
| i18n 모듈 | `@nuxtjs/i18n` v10.3.0 |
| 콘텐츠 백엔드 | Strapi v5 (`PUBLIC_STRAPI_URL`) |
| AI 채팅 백엔드 | `nougat-career-be` (`https://api.chu.plus/resume/stream`) |

다국어 처리는 **(A) UI 문자열**, **(B) 라우팅/URL**, **(C) CMS 콘텐츠**, **(D) SEO/hreflang/sitemap/RSS**, **(E) 정적 데이터(FAQ·SEO 페이지)**, **(F) AI 채팅 응답 언어** 6개 계층으로 분리되어 동작한다.

---

## 2. 로케일 식별자 체계

`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\types\i18n.ts" />`

- **앱 내부 정규 코드**: `Locale = "ko" | "en" | "ja"` (모든 로직의 기준)
- **BCP-47 매핑**: `ko→ko-KR`, `en→en-US`, `ja→ja-JP` (HTML `lang`, `Intl.DateTimeFormat`, JSON-LD `inLanguage`에 사용)
- **LLM 언어명 매핑**: `ko→Korean`, `en→English`, `ja→Japanese` (AI 프롬프트용)
- **RSS language 코드**: `ko-kr`, `en-us`, `ja-jp`
- `localeConfig`: UI 스위처용 `{ code, name, flag }[]` (🇰🇷 한국어 / 🇺🇸 English / 🇯🇵 日本語)

---

## 3. 모듈 설정 (`nuxt.config.ts`)

`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\nuxt.config.ts" lines="35-53" />`

| 설정 | 값 | 의미 |
|---|---|---|
| `strategy` | `prefix_and_default` | 기본 로케일(ko)은 URL prefix 없음, en/ja는 `/en`, `/ja` prefix |
| `defaultLocale` | `ko` | 한국어가 디폴트 |
| `langDir` | `locales` (+ `restructureDir: "."`) | `./locales/{ko,en,ja}.json` 로드 |
| `detectBrowserLanguage.useCookie` | `true` (key: `i18n_redirected`) | 최초 방문 시 브라우저 언어 감지 → 쿠키 저장 |
| `detectBrowserLanguage.redirectOn` | `root` | 루트(`/`) 방문 시에만 리다이렉트 |
| `detectBrowserLanguage.alwaysRedirect` | `false` | 이미 결정된 로케일은 유지 |
| `fallbackLocale` | `ko` | 누락 키/감지 실패 시 한국어로 폴백 |

---

## 4. UI 문자열 계층 (Vue I18n JSON 카탈로그)

### 4.1 리소스
- `locales/ko.json`, `locales/en.json`, `locales/ja.json` (각 약 918줄, 동일 키 구조)
- 네임스페이스 예: `nav.*`, `home.*`, `columns.*`, `chat.*`, `theme.*`, `language.*`, `common.*`

### 4.2 접근 API
- **단일 문자열**: `useI18n().t(key)` — 대부분의 컴포넌트에서 사용
- **배열/객체 메시지** (`chat.suggestions`, `chat.questions`): `useResolvedI18nMessage().tmResolved<T>(key)`
  - `tm()`은 Vue I18n AST를 반환하므로 `rt()`로 실제 문자열로 resolve하는 재귀 헬퍼
  - `<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useResolvedI18nMessage.ts" />`

### 4.3 공용 composable
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useLocale.ts" />`
- `lang`: `computed(() => locale.value as Locale)` — 반응형 로케일
- `href(path)`: `localePath(path)` 래퍼 (로케일 prefix가 적용된 URL 생성)
- `localePath`: `useLocalePath()` 직접 노출

### 4.4 사용 패턴 (컴포넌트 표준)
```ts
const { lang, href } = useLocale();
const { t } = useI18n();
usePageSeo({ lang: lang.value, path: "...", title: t("..."), description: t("...") });
```
- 39개 파일이 `useI18n`/`useLocale`/`localePath`/`switchLocalePath` 중 하나 이상 사용

---

## 5. 라우팅 / URL 계층

### 5.1 URL 구조 (`prefix_and_default`)
| 로케일 | 홈 | 칼럼 리스트 | 칼럼 상세 |
|---|---|---|---|
| ko | `/` | `/columns` | `/columns/{slug}` |
| en | `/en/` | `/en/columns` | `/en/columns/{slug}` |
| ja | `/ja/` | `/ja/columns` | `/ja/columns/{slug}` |

정적 경로 목록은 `server/utils/create-sitemap.ts`의 `STATIC_PATHS`와 `components/RoutePrefetcher.vue`의 `staticPaths`에 하드코딩(동일 13개 경로).

### 5.2 로케일 전환 UI
- **데스크톱**: `<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\components\LocaleSwitcher.vue" />`
  - `switchLocalePath(code)` → `navigateTo(path)` (동일 라우트의 다른 로케일 버전으로 이동)
- **모바일**: `MobileMenu.vue` line 148-158 — `localePath(basePath, item.code)`를 `NuxtLink :to`에 직접 바인딩

### 5.3 사전 prefetch
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\components\RoutePrefetcher.vue" />`
- 모든 정적 경로 × 3 로케일의 숨겨진 `<NuxtLink prefetch>`를 렌더하여 라우트 캐시 워밍

---

## 6. CMS 콘텐츠 로케일 계층 (Strapi)

### 6.1 데이터 흐름
```
Page (lang.value) → useColumnsProvider() → getColumns(baseUrl, locale, ...) → Strapi /columns?locale=ko|en|ja
```

`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useColumns.ts" />`

### 6.2 Strapi 호출
`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\lib\data\columns.ts" lines="376-407" />`
- `getColumns(baseUrl, locale, appEnv, apiKey)`: `params.set("locale", locale)` 후 `/columns?...&locale=ko` 형태로 호출
- `getColumnBySlug`, `getRelatedColumns` 동일 패턴
- Dev 환경(`APP_ENV=development`)에서 Strapi 실패 시 `SAMPLE_COLUMNS`(영어 mock)로 폴백

### 6.3 SWR 캐시 키
- `columns:${lang.value}` (리스트), `column:${lang.value}:${slug}` (상세)
- 로케일이 캐시 키에 포함되어 로케일별로 독립 캐싱
- `useColumnSWR`은 SSR payload 캐시 + 클라이언트 마운트 후 백그라운드 재검증(SWR)

---

## 7. SEO / hreflang / Sitemap / RSS 계층

### 7.1 페이지 SEO (`usePageSeo`)
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\usePageSeo.ts" />`

- `useLocaleHead({ seo: true })`로부터 `hreflang` alternate 링크 획득
- 상대 href를 site origin으로 절대화
- `x-default` 누락 시 canonical을 x-default로 추가
- `<html lang>`, `dir` 속성 설정
- JSON-LD(`WebPage`, `BlogPosting`)에 `inLanguage: currentLocale` 주입
- 3개 로케일 RSS alternate link 추가 (ko/en/ja 각각)
- `lib/seo.ts`의 `getSiteName(lang)`: en일 때 "Heonnam Chu", 그 외 "Chu Heon Nam"

### 7.2 Sitemap (`/sitemap.xml`)
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\server\utils\create-sitemap.ts" />`

- 3개 로케일 칼럼을 병렬 fetch하여 `columnMaps[slug] = { ko?, en?, ja?, lastmod }` 구축
- 엔트리 그룹:
  1. 기본 로케일(ko) 정적 경로 — prefix 없음
  2. 기본 로케일 칼럼 — `/columns/{slug}`
  3. 비기본 로케일(en/ja) 정적+칼럼 — `/{lang}/...`
- 각 엔트리의 `alternates`:
  - 정적: `x-default`, `ko`, `en`, `ja` 4개
  - 칼럼: 해당 slug가 존재하는 로케일만 alternate에 포함 (존재하지 않으면 빠짐)
- `x-default`는 항상 ko(무prefix) URL

### 7.3 RSS
- `/rss.xml` → ko 피드 (기본)
- `/{locale}/rss.xml` → en, ja만 허용 (`server/routes/[locale]/rss.xml.ts`에서 `["en","ja"]` 검증, 그 외 404)
- `getChannelInfo(lang)`이 로케일별 channel title/description/language 코드 반환
- 아이템 link도 로케일 prefix 적용

---

## 8. 정적 데이터 로케일 계층 (JSON 카탈로그 외부)

Vue I18n JSON 대신 **TS 데이터 파일에 필드를 3개 로케일별로 병렬 정의**하는 별도 계층이 존재한다.

### 8.1 FAQ 데이터
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\data\faq-data.ts" />`
- `faqDataKo`, `faqDataEn`, `faqDataJa` 3개 배열 + `getFaqDataByLang(lang)` 셀렉터 (ko 폴백)
- `pages/faq.vue`에서 `getFaqDataByLang(lang.value)`로 사용
- 동일 데이터가 FAQPage JSON-LD 스키마에도 사용

### 8.2 SEO 페이지 데이터
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\data\seo-pages-data.ts" />`
- `SeoPageItem` 인터페이스: `titleKo/titleEn/titleJa`, `metaKo/metaEn/metaJa`, `descriptionKo/descriptionEn/descriptionJa`, `chatCtas[].{label,query}{Ko,En,Ja}`
- 3개 페이지가 사용: `who-is-chuheonam.vue`, `nougat-developer.vue`, `chuheonam-projects.vue` (모두 `getSeoPageBySlug` + `lang.value` 분기)
- `SeoPageContent.vue`가 `lang.value === "ko" ? ... : "ja" ? ... : ...` 3항 분기로 필드 선택

> ⚠️ 이 계층은 JSON 카탈로그와 중복 없이 병행 운영된다. 신규 로케일 추가 시 두 체계를 모두 갱신해야 한다.

---

## 9. AI 채팅 응답 언어 계층

### 9.1 프론트엔드 흐름
`<ref_snippet file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\composables\useChatState.ts" lines="13-15" />`
```
locale.value (Locale) ──getChatLanguage()──▶ "Korean"|"Japanese"|"English"
```
- `useChatState.sendMessage()`가 스트림 요청 payload에 `language: getChatLanguage(locale.value)` 포함
- 폴백 에러 메시지는 `locale.value === "ko" ? ... : "ja" ? ... : ...` 인라인 3항 분기 (line 506-510, 536-540)

### 9.2 전송
`<ref_file file="C:\Users\Nuts\Documents\workspace\nougat-career\nougat-resume-wc\lib\streamClient.ts" />`
- `StreamRequest.language` 필드 → 암호화된 body로 POST `https://api.chu.plus/resume/stream`

### 9.3 백엔드(`nougat-career-be`) 처리
1. `src/wedding/stream/capability.ts validateStreamRequest`: `language` 추출, 기본 `"Korean"`
2. `streamController.ts`: `IntegrationSession`에 language 주입, `callGPTStream(message, language, systemPrompt)` 호출
3. `tools/callGPT.ts`: vendor(OpenAI/Gemini/InceptionLabs) 디스패치 시 language 전달
4. `tools/getFormattedPrompt.ts` line 444-457:
   ```
   TARGET ANSWERING LANGUAGE
   ${language}
   ```
   시스템 프롬프트에 "TARGET ANSWERING LANGUAGE: Korean/English/Japanese" 블록 주입 → LLM이 해당 언어로 응답

> 참고: `catalog.ts`의 일부 커스텀 룰은 "Always respond in Korean"으로 하드코딩되어 있어, 비 ko 로케일에서 프롬프트 충돌 가능성이 존재한다.

---

## 10. 날짜/시간 포맷팅

- 칼럼 리스트/상세 페이지에서 `intl` computed:
  ```ts
  const intl = computed(() =>
    lang.value === "ko" ? "ko-KR" : lang.value === "ja" ? "ja-JP" : "en-US",
  );
  ```
- `formatDateTime(dateStr, intl, ...)`에 전달하여 `Intl.DateTimeFormat` 기반 로케일 포맷 적용

---

## 11. 전체 데이터 흐름도

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 사용자 진입                                                   │
│    URL prefix (/en/...) ── 또는 ── 쿠키 i18n_redirected (루트)   │
│                              ↓                                  │
│    @nuxtjs/i18n → useI18n().locale = "ko"|"en"|"ja"             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
   [UI 문자열]           [라우팅]              [CMS 콘텐츠]
   t(key)                localePath()           useColumnsProvider
   tmResolved()          switchLocalePath()     → Strapi ?locale=
   locales/*.json        prefix_and_default     (ko/en/ja 병렬)
        ↓                     ↓                     ↓
        └─────────────────────┼─────────────────────┘
                              ↓
                     [정적 데이터 병렬 계층]
                     faq-data.ts (Ko/En/Ja 배열)
                     seo-pages-data.ts (Ko/En/Ja 필드)
                     getFaqDataByLang(lang) / 3항 분기
                              ↓
                     [SEO 메타]
                     usePageSeo → useLocaleHead
                     hreflang alternate + x-default
                     JSON-LD inLanguage
                     <html lang>
                              ↓
                     [Sitemap / RSS (서버)]
                     3 로케일 칼럼 병렬 fetch
                     hreflang alternate 매트릭스
                     /rss.xml (ko), /{lang}/rss.xml (en/ja)
                              ↓
                     [AI 채팅]
                     getChatLanguage(locale) → "Korean"/...
                     streamClient POST /resume/stream
                     BE: getFormattedPrompt에 주입 → LLM 응답 언어 결정
```

---

## 12. 신규 로케일 추가 시 체크리스트

신규 로케일(예: `zh`) 추가 시 **6개 계층 모두** 수정 필요:

1. `nuxt.config.ts` `i18n.locales` 배열에 `{ code, name, language, file }` 추가
2. `types/i18n.ts`: `Locale` 유니언, `locales` 배열, `localeConfig` 항목 추가
3. `locales/zh.json` 신규 파일 (기존 키 구조 전체 복제)
4. `data/faq-data.ts`: `faqDataZh` 배열 + `getFaqDataByLang` 분기 추가
5. `data/seo-pages-data.ts`: 모든 `SeoPageItem`에 `titleZh/metaZh/descriptionZh` 및 `chatCtas[].{labelZh,queryZh}` 추가, `SeoPageContent.vue` 분기 추가
6. `server/utils/create-sitemap.ts`: `columnMaps` 타입, `getStaticAlternates`, `getColumnAlternates`에 zh 분기 추가
7. `server/utils/create-rss.ts`: `getChannelInfo` zh 분기 추가, `server/routes/[locale]/rss.xml.ts` `supportedLangs`에 zh 추가
8. `composables/useChatState.ts`: `getChatLanguage` zh 매핑 + 인라인 폴백 메시지 분기 추가
9. `lib/seo.ts` `getSiteName`: zh 분기 (필요 시)
10. 칼럼 리스트/상세 페이지의 `intl` computed: zh → `zh-CN` 매핑 추가
11. `components/RoutePrefetcher.vue` `localizedPaths`: zh 키 추가
12. Strapi 콘텐츠에 zh 로케일 데이터 입력

---

## 13. 주의사항 / 개선 여지

1. **정적 경로 중복 정의**: `STATIC_PATHS`(create-sitemap.ts)와 `staticPaths`(RoutePrefetcher.vue)가 동일 13개 경로를 하드코딩. 단일 소스로 통합 권장.
2. **정적 데이터 i18n 이원화**: FAQ·SEO 페이지가 JSON 카탈로그가 아닌 TS 필드 병렬 정의 방식. 유지보수 비용이 높고 누락 위험 존재. JSON 카탈로그 또는 Strapi i18n 플러그인으로 통합 검토.
3. **인라인 3항 분기 반복**: `lang.value === "ko" ? ... : "ja" ? ... : ...` 패턴이 `SeoPageContent.vue`, `useChatState.ts`, 칼럼 페이지 `intl` 등 다수 분산. `useLocale()`에 BCP-47/LLM 언어명/폴백 메시지 헬퍼를 추가해 중앙화 권장.
4. **AI 프롬프트 충돌**: `catalog.ts`의 "Always respond in Korean" 하드코딩 룰과 `getFormattedPrompt`의 `TARGET ANSWERING LANGUAGE`가 비 ko 로케일에서 충돌 가능. catalog 룰을 로케일 파라미터화 필요.
5. **Sitemap `x-default` 정책**: 항상 ko(무prefix)를 x-default로 고정. 다국어 사용자 비중에 따라 `x-default`를 브라우저 언어 자동 감지 페이지(`/`)로 두는 정책 검토 여지.
6. **RSS 비대칭**: ko는 `/rss.xml`, en/ja는 `/{lang}/rss.xml`. 경로 일관성 확보 또는 ko도 `/ko/rss.xml` 별칭 제공 검토.
