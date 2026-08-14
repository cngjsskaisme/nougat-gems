# Multi-layer i18n Architecture

## 1. Locale as a Contract

애플리케이션 내부에서는 짧고 안정적인 Locale ID를 하나의 Source of Truth로 사용합니다.

```text
Locale ID
  ├─ UI Catalog
  ├─ Route Prefix
  ├─ CMS Query
  ├─ BCP-47 / Intl
  ├─ SEO / hreflang
  └─ LLM Response Language
```

표시용 언어명, BCP-47, RSS language, LLM용 언어명은 Locale ID에서 중앙 매핑합니다.

## 2. Six Layers

### A. UI Strings

문자열 Catalog는 동일한 Key 구조를 유지합니다. 문자열뿐 아니라 배열·객체 메시지를 사용할 경우 Runtime Resolve 규칙을 한곳에 둡니다.

### B. Routing

Locale Prefix 정책을 명확히 정합니다. 기본 Locale의 Prefix 생략 여부, Browser Language Redirect, Cookie 유지 정책을 중앙 설정으로 관리합니다.

### C. CMS Content

CMS 요청과 Cache Key에 Locale을 포함합니다.

```text
content:<locale>:<resource>
```

Locale이 빠진 Cache Key 때문에 다른 언어 콘텐츠가 섞이지 않도록 합니다.

### D. SEO

Canonical, `hreflang`, `x-default`, `<html lang>`, JSON-LD `inLanguage`, Sitemap, RSS를 같은 Locale Mapping에서 파생합니다.

### E. Static Data

FAQ나 SEO 설명처럼 코드에 저장된 다국어 데이터가 별도 체계로 존재하면 Locale 추가 시 누락이 발생하기 쉽습니다. 가능하면 Catalog/CMS로 통합하거나 최소한 하나의 Selector API 뒤로 숨깁니다.

### F. AI Response Language

Frontend Locale을 LLM Prompt의 Target Language로 변환합니다. System Rule에 특정 언어가 하드코딩되어 Locale 지시와 충돌하지 않는지 검증합니다.

## 3. Central Locale Helper

애플리케이션 곳곳에서 `locale === ... ? ...` 분기를 반복하지 않습니다. 다음을 중앙화합니다.

- BCP-47 mapping
- localized path generation
- date/number Intl locale
- LLM language name
- fallback locale
- site/channel display name where needed

## 4. Adding a Locale

새 Locale을 추가할 때 최소한 다음을 함께 확인합니다.

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

- Sitemap과 App Router가 서로 다른 경로 목록을 유지
- Locale별 Static Data가 여러 파일에 중복 정의
- 인라인 삼항 분기로 Locale Mapping이 분산
- LLM System Prompt의 하드코딩 언어와 Runtime Locale 충돌
- CMS Cache Key에서 Locale 누락
- 존재하지 않는 번역 페이지까지 hreflang에 포함

## 6. Validation

- 모든 Locale Catalog의 Key Set 비교
- Localized Route Snapshot
- hreflang reciprocal link 검사
- CMS locale/cache isolation
- Locale별 JSON-LD language 검사
- LLM Target Language와 UI Locale 일치 확인
