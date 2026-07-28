# 삼성인터넷 네이티브 "맨 위로" 버튼 겹침 회피 로직
## 프로그램 설계서 (재사용 단위)

---

## 1. 문서 개요

| 항목 | 내용 |
|------|------|
| 문서명 | 삼성인터넷 네이티브 플로팅 버튼 겹침 회피 로직 설계서 |
| 버전 | 1.0 |
| 적용 범위 | 모바일 웹 고정 하단 UI를 가지는 Vue 3 / Nuxt 프로젝트 |
| 재사용 단위 | 3개 composable + 1개 적용 가이드 (프로젝트 비의존) |
| 전제 조건 | Vue 3 Composition API, Nuxt `useState`/`useRouter`, SSR 환경 지원 |

---

## 2. 문제 정의

### 2.1 배경
- 삼성인터넷 브라우저는 문서 스크롤 시 화면 **하단 중앙**에 네이티브 "맨 위로" 플로팅 버튼을 표시한다.
- 해당 버튼은 웹 API(`CSS`/`JS`)로 숨기거나 비활성화할 수 없다.
- 서비스의 고정 하단 UI(채팅 바, FAB, 후속 질문 카드 등)와 시각적·조작적 겹침이 발생한다.

### 2.2 해결 전략
- 네이티브 버튼을 제거할 수 없으므로, **스크롤 활동 중에만 웹 고정 UI를 위로 올려** 겹침을 회피한다.
- 스크롤 정지 후 일정 시간이 경과하면 원위치로 복귀한다.
- 문서 상단 근처(네이티브 버튼이 노출되지 않는 영역)에서는 지연 없이 즉시 원위치한다.

### 2.3 요구사항

| ID | 요구사항 |
|----|----------|
| REQ-01 | 삼성인터넷 브라우저 환경을 UA 기반으로 감지할 것 |
| REQ-02 | 스크롤 활동 중 하단 UI를 상단 오프셋으로 올릴 것 |
| REQ-03 | 스크롤 정지 후 지정된 지연 시간 경과 시 원위치할 것 |
| REQ-04 | 문서 상단 임계값 이내에서는 지연 없이 즉시 원위치할 것 |
| REQ-05 | 비삼성 환경에서는 모든 로직이 no-op로 동작할 것 |
| REQ-06 | 라우트 전환 시 올림 상태 및 지연 타이머를 초기화할 것 |
| REQ-07 | 복수의 고정 하단 UI 컴포넌트가 동일 상태를 공유할 것 (싱글턴) |
| REQ-08 | 채팅 바/키보드 상호작용으로 발생한 스크롤은 활동으로 간주하지 않을 것 |
| REQ-09 | 코치마크 등 오버레이 표시 중에는 스크롤 활동 감지를 억제할 수 있을 것 |
| REQ-10 | SSR 환경에서 안전하게 동작할 것 (클라이언트 전환 후 감지) |

---

## 3. 모듈 구성

### 3.1 모듈 계층

```
┌─────────────────────────────────────────────────────────────┐
│  적용 컴포넌트 레이어 (프로젝트별 구현)                      │
│  - 고정 하단 UI 컴포넌트들이 computed 클래스를 바인딩       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  모듈 1: useSamsungChatChrome (오케스트레이션)              │
│  - 상태 관리, watch/라우터 훅 등록, computed 출력            │
└───────┬──────────────────────────────┬──────────────────────┘
        │                              │
┌───────▼──────────────────────┐  ┌───▼───────────────────────┐
│ 모듈 2: useScrollChrome      │  │ 모듈 3: useSamsungInternet │
│ Visibility                   │  │ Browser                    │
│ - scrollActive 상태 제공     │  │ - UA 감지                  │
│ - 스크롤 이벤트 통합 처리    │  │ - isSamsungInternet ref    │
└──────────────────────────────┘  └────────────────────────────┘
```

### 3.2 모듈 목록

| 모듈 | 파일명 (권장) | 역할 | 의존 |
|------|---------------|------|------|
| 모듈 1 | `useSamsungChatChrome.ts` | 올림 상태 관리, watch/라우터 훅, computed 출력 | 모듈 2, 3 |
| 모듈 2 | `useScrollChromeVisibility.ts` | 스크롤 활동 감지, `scrollActive` 상태 제공, 억제 API | 없음 |
| 모듈 3 | `useSamsungInternetBrowser.ts` | UA 기반 삼성인터넷 감지 | 없음 |

---

## 4. 모듈 상세 설계

### 4.1 모듈 3 — `useSamsungInternetBrowser`

#### 4.1.1 인터페이스

```ts
function useSamsungInternetBrowser(): {
  isSamsungInternet: Ref<boolean>
}
```

#### 4.1.2 설계 명세

| 항목 | 내용 |
|------|------|
| 목적 | 삼성인터넷 브라우저 환경 감지 |
| 감지 방식 | `navigator.userAgent` 정규식 매칭 |
| 감지 정규식 | `/SamsungBrowser/i` |
| 감지 시점 | `onMounted` (클라이언트 사이드) |
| 초기값 | `false` |
| SSR 동작 | `typeof navigator === 'undefined'` 시 초기값 유지 |

#### 4.1.3 처리 순서

1. `isSamsungInternet` ref를 `false`로 초기화
2. `onMounted` 훅 실행
3. `navigator` 미존재 시(SSR) 즉시 종료
4. `navigator.userAgent`에 `SamsungBrowser` 포함 여부 평가
5. 매칭 시 `isSamsungInternet.value = true`

#### 4.1.4 참고 사양
- Samsung Internet UA 형식: `Mozilla/5.0 ... SamsungBrowser/<version> ...`
- 정규식은 대소문자 무시(`/i` 플래그)하여 변형 대응

---

### 4.2 모듈 2 — `useScrollChromeVisibility`

#### 4.2.1 인터페이스

```ts
function useScrollChromeVisibility(): {
  chromeVisible: Ref<boolean>           // 하위 호환 (항상 true)
  scrollActive: Ref<boolean>            // 스크롤 활동 중 여부
  suppressScrollChromeForChatBarMs: (ms?: number) => void
  suppressScrollChromeForKeyboardMs: (ms?: number) => void
}

// 모듈 외부 export 함수
function isWindowScrollWithinTopResetPx(): boolean
function setCoachmarkScrollChromeBlocked(blocked: boolean): void
```

#### 4.2.2 상수 정의

| 상수명 | 값 | 설명 |
|--------|-----|------|
| `SCROLL_IDLE_MS` | 150ms | 스크롤 정지 판정 시간 (이 시간 동안 추가 스크롤 이벤트가 없으면 비활성) |
| `VIEWPORT_SCROLL_IGNORE_MULTIPLIER` | 0.75 | 상단 무시 영역 배수. `scrollY < innerHeight * 0.75`인 영역에서는 스크롤 활동으로 간주하지 않음 |
| `SCROLL_TOP_ACTIVE_RESET_PX` | 200px | 상단 강제 비활성 임계값. `scrollY <= 200`이면 `scrollActive`를 항상 `false`로 강제 |
| `VV_RESIZE_SUPPRESS_DEBOUNCE_MS` | 220ms | `visualViewport.resize` 디바운스 시간 |
| `VV_RESIZE_SUPPRESS_DURATION_MS` | 850ms | 키보드로 인한 스크롤 억제 지속 시간 |

#### 4.2.3 상태

| 상태명 | 타입 | 스코프 | 설명 |
|--------|------|--------|------|
| `chromeVisible` | `boolean` | `useState` | 하위 호환용. 항상 `true` |
| `scrollActive` | `boolean` | `useState` | 스크롤 활동 중 여부. 모듈 1이 watch |
| `suppressScrollChromeUntil` | `number` | 모듈 변수 | 억제 종료 timestamp |
| `coachmarkBlocksScrollChrome` | `boolean` | 모듈 변수 | 오버레이 표시 중 억제 플래그 |
| `idleTimer` | `setTimeout 반환 \| null` | 모듈 변수 | 스크롤 정지 판정 타이머 |
| `vvResizeSuppressDebounce` | `setTimeout 반환 \| null` | 모듈 변수 | vv.resize 디바운스 타이머 |
| `listenerAttached` | `boolean` | 모듈 변수 | 리스너 중복 등록 방지 |
| `routeHooksRegistered` | `boolean` | 모듈 변수 | 라우터 훅 중복 등록 방지 |

#### 4.2.4 핵심 처리 — `markScrollActivity()`

| 단계 | 조건 | 동작 |
|------|------|------|
| 1 | `coachmarkBlocksScrollChrome == true` | 종료 (억제) |
| 2 | `scrollY <= SCROLL_TOP_ACTIVE_RESET_PX` | `idleTimer` 해지, `scrollActive = false`, 종료 |
| 3 | `Date.now() < suppressScrollChromeUntil` | 종료 (억제) |
| 4 | `scrollY < innerHeight * VIEWPORT_SCROLL_IGNORE_MULTIPLIER` | 종료 (상단 영역 무시) |
| 5 | (위 조건 모두 불만족) | `scrollActive = true`, `idleTimer` 재설정(`SCROLL_IDLE_MS` 후 `false`), `chromeVisible = true` |

#### 4.2.5 이벤트 리스너 (최초 1회 등록, `listenerAttached` 플래그)

| 대상 | 이벤트 | 옵션 | 핸들러 | 목적 |
|------|--------|------|--------|------|
| `window` | `scroll` | passive | `markScrollActivity` | 문서 전체 스크롤 |
| `document` | `scroll` | capture, passive | `markScrollActivity` | 내부 overflow 스크롤 (자식 요소) |
| `window` | `wheel` | passive | `markScrollActivity` | 마우스휠/트랙패드 |
| `window` | `touchmove` | passive | `markScrollActivity` | 터치 스크롤/관성 |
| `visualViewport` | `scroll` | passive | `markScrollActivity` | 레이아웃 뷰포트 스크롤 (주소창 등) |
| `visualViewport` | `resize` | passive | `onVisualViewportResizeForKeyboard` | 소프트 키보드 표시/숨김 |
| `document` | `focusin` | capture | `onFocusInMaybeKeyboard` | 입력 요소 포커스 시 억제 |
| `document` | `focusout` | capture | `onFocusOutMaybeKeyboard` | 입력 요소 포커스 해제 시 억제 |

#### 4.2.6 라우터 훅 (최초 1회 등록, `routeHooksRegistered` 플래그)

| 훅 | 동작 |
|----|------|
| `router.afterEach` | `nextTick(() => resetScrollActivity())` |

**`resetScrollActivity()` 처리**
1. `idleTimer` 해지
2. `vvResizeSuppressDebounce` 해지
3. `scrollActive = false`
4. `chromeVisible = true`

#### 4.2.7 외부 API 함수

| 함수 | 시그니처 | 용도 |
|------|----------|------|
| `suppressScrollChromeForChatBarMs` | `(ms = 600) => void` | 고정 UI 상호작용 후 스크롤 무시 |
| `suppressScrollChromeForKeyboardMs` | `(ms = 1000) => void` | 키보드 표시/숨김 후 스크롤 무시 |
| `setCoachmarkScrollChromeBlocked` | `(blocked: boolean) => void` | 오버레이 표시 중 스크롤 활동 감지 억제 |
| `isWindowScrollWithinTopResetPx` | `() => boolean` | `scrollY <= 200` 판정. 모듈 1과 임계값 공유 |

#### 4.2.8 억제 메커니즘 상세

**`suppressScrollChromeUntil` 기반 억제**
- `bumpSuppressChromeUntil(ms)`: `suppressScrollChromeUntil = max(현재값, Date.now() + ms)`
- `markScrollActivity()`에서 `Date.now() < suppressScrollChromeUntil`이면 활동 처리 스킵
- 억제는 시간 기반이므로 연속 호출해도 종료 시각이 뒤로 밀리지 않고 `max`로 보정됨

**`visualViewport.resize` 디바운스**
- `resize` 이벤트는 연속 발생하므로 220ms 디바운스 후 1회만 억제 적용
- 억제 지속 시간: 850ms

**포커스 기반 키보드 억제**
- `focusin`/`focusout`에서 타깃이 `input, textarea, select` 또는 `contenteditable`이면 850ms 억제

---

### 4.3 모듈 1 — `useSamsungChatChrome`

#### 4.3.1 인터페이스

```ts
function useSamsungChatChrome(): {
  floatingBarBottomClass: ComputedRef<string>
  suggestedCardBottomClass: ComputedRef<string>
  samsungBarElevated: Ref<boolean>
}
```

#### 4.3.2 상수 정의

| 상수명 | 값 | 설명 |
|--------|-----|------|
| `SAMSUNG_LOWER_DELAY_MS` | 1800ms | 스크롤 정지 후 하향(원위치) 지연 시간 |

#### 4.3.3 상태

| 상태명 | 타입 | 스코프 | 설명 |
|--------|------|--------|------|
| `samsungBarElevated` | `boolean` | `useState('samsung-chat-chrome-elevated')` | 하단 UI 올림 여부. 복수 컴포넌트 공유 |
| `samsungLowerTimer` | `setTimeout 반환 \| null` | 모듈 변수 | 하향 지연 타이머 |
| `samsungChromeWatchRegistered` | `boolean` | 모듈 변수 | watch/리스너 중복 등록 방지 |

#### 4.3.4 초기화 처리 (최초 1회, `samsungChromeWatchRegistered` 플래그)

**전제**: `import.meta.client && !samsungChromeWatchRegistered`일 때만 실행

**A. 스크롤 리스너 등록 (상단 근처 즉시 하향용)**

| 대상 | 이벤트 | 옵션 | 핸들러 |
|------|--------|------|--------|
| `window` | `scroll` | passive | `lowerSamsungBarIfNearTopFromScroll` |
| `visualViewport` | `scroll` | passive | `lowerSamsungBarIfNearTopFromScroll` |

**`lowerSamsungBarIfNearTopFromScroll()` 처리**

| 단계 | 조건 | 동작 |
|------|------|------|
| 1 | `isSamsungInternet == false` | 종료 |
| 2 | `isWindowScrollWithinTopResetPx() == false` | 종료 |
| 3 | (조건 만족) | `clearSamsungLowerTimer()`, `samsungBarElevated = false` |

**B. watch: `scrollActive`**

| `scrollActive` 값 | 추가 조건 | 동작 |
|-------------------|-----------|------|
| `true` | - | `clearSamsungLowerTimer()`, `samsungBarElevated = true` (즉시 올림) |
| `false` | `isWindowScrollWithinTopResetPx() == true` | `clearSamsungLowerTimer()`, `samsungBarElevated = false` (즉시 하향) |
| `false` | `isWindowScrollWithinTopResetPx() == false` | `samsungLowerTimer = setTimeout(하향, SAMSUNG_LOWER_DELAY_MS)` (지연 하향) |

**C. watch: `isSamsungInternet`**

| 값 | 동작 |
|----|------|
| `true` | 종료 (no-op) |
| `false` | `clearSamsungLowerTimer()`, `samsungBarElevated = false` (비삼성 전환 시 정리) |

**D. 라우터 훅**

| 훅 | 동작 |
|----|------|
| `router.afterEach` | `nextTick(() => { clearSamsungLowerTimer(); samsungBarElevated = false })` |

#### 4.3.5 보조 함수

| 함수 | 처리 내용 |
|------|-----------|
| `clearSamsungLowerTimer()` | `samsungLowerTimer` 존재 시 `clearTimeout` 후 `null` 할당 |

#### 4.3.6 출력 computed

**`floatingBarBottomClass`** — 고정 하단 바용

| 조건 | 반환값 | 의미 |
|------|--------|------|
| `isSamsungInternet == false` | `'bottom-7'` | 기본 위치 |
| `isSamsungInternet == true && samsungBarElevated == true` | `'bottom-16'` | 올림 (네이티브 버튼 피함) |
| `isSamsungInternet == true && samsungBarElevated == false` | `'bottom-7'` | 복귀 |

**`suggestedCardBottomClass`** — 후속 UI 카드용

| 조건 | 반환값 | 의미 |
|------|--------|------|
| `isSamsungInternet == true && samsungBarElevated == true` | `'bottom-[calc(6rem+2.25rem)]'` | 올림 (바와 동일 델타 2.25rem 추가) |
| 그 외 | `'bottom-[6rem]'` | 기본 위치 |

> **주**: `2.25rem`은 `bottom-16`과 `bottom-7`의 차이(`4rem - 1.75rem = 2.25rem`)와 일치. 고정 바와 후속 카드가 동일 간격을 유지하도록 정합.

---

## 5. 상태 전이 명세

### 5.1 `samsungBarElevated` 상태 전이

| 현재 상태 | 이벤트 | 조건 | 다음 상태 | 지연 |
|-----------|--------|------|-----------|------|
| `false` | `scrollActive: false → true` | 삼성 | `true` | 없음 (즉시) |
| `true` | `scrollActive: true → false` | 삼성, `scrollY <= 200` | `false` | 없음 (즉시) |
| `true` | `scrollActive: true → false` | 삼성, `scrollY > 200` | `false` | 1800ms 후 |
| `true` | `scroll` 이벤트 | 삼성, `scrollY <= 200` | `false` | 없음 (즉시) |
| `true` | `isSamsungInternet: true → false` | - | `false` | 없음 (즉시) |
| `true` | `router.afterEach` | - | `false` | `nextTick` 후 즉시 |
| `false` | (위 모든 이벤트) | - | `false` | - (변화 없음) |

### 5.2 `scrollActive` 상태 전이

| 현재 상태 | 이벤트 | 조건 | 다음 상태 | 타이머 |
|-----------|--------|------|-----------|--------|
| `false` | 스크롤 입력 | 억제 중 아님, 상단 무시 영역 아님, `scrollY > 200` | `true` | `idleTimer` 150ms 설정 |
| `true` | `idleTimer` 만료 | - | `false` | - |
| `true`/`false` | 스크롤 입력 | `scrollY <= 200` | `false` | `idleTimer` 해지 |
| `true`/`false` | 스크롤 입력 | 억제 중 | (변화 없음) | - |
| `true`/`false` | 스크롤 입력 | `scrollY < innerHeight * 0.75` | (변화 없음) | - |
| `true`/`false` | 스크롤 입력 | `coachmarkBlocksScrollChrome == true` | (변화 없음) | - |
| `true`/`false` | `router.afterEach` | - | `false` | `idleTimer` 해지 |

---

## 6. 적용 가이드

### 6.1 적용 컴포넌트 요건

고정 하단 UI 컴포넌트는 다음을 만족해야 한다.

| 요건 | 설명 |
|------|------|
| `position: fixed` | 하단 고정 위치 |
| `bottom` 오프셋 클래스 바인딩 | 모듈 1의 computed를 `:class`에 바인딩 |
| `transition-[bottom]` | 오프셋 변경 시 부드러운 전환 (권장) |
| `motion-reduce:transition-none` | `prefers-reduced-motion` 대응 (권장) |

### 6.2 적용 예시

```vue
<script setup lang="ts">
import { useSamsungChatChrome } from '~/composables/useSamsungChatChrome'

const { floatingBarBottomClass } = useSamsungChatChrome()
</script>

<template>
  <div
    class="fixed left-0 right-0 transition-[bottom] duration-300 ease-out motion-reduce:transition-none"
    :class="floatingBarBottomClass"
  >
    <!-- 고정 하단 UI 내용 -->
  </div>
</template>
```

### 6.3 복수 컴포넌트 적용 시 주의

- `samsungBarElevated`는 `useState` 기반이므로 **동일 키**를 사용하는 모든 컴포넌트가 상태를 공유한다.
- watch/리스너는 모듈 변수 플래그(`samsungChromeWatchRegistered`)로 **1회만 등록**된다.
- 복수 컴포넌트가 `useSamsungChatChrome()`을 호출해도 부작용이 없다.

### 6.4 오프셋 값 커스터마이징

| 변경 대상 | 위치 | 설명 |
|-----------|------|------|
| 기본/올림 오프셋 | 모듈 1 computed 반환값 | `bottom-7` / `bottom-16`을 프로젝트 레이아웃에 맞게 조정 |
| 후속 UI 오프셋 | 모듈 1 computed 반환값 | `bottom-[6rem]` / `bottom-[calc(6rem+2.25rem)]` 조정 |
| 올림 델타 | `2.25rem` | `bottom-16 - bottom-7` 차이와 후속 카드 델타를 일치시킬 것 |
| 하향 지연 | `SAMSUNG_LOWER_DELAY_MS` | 네이티브 버튼 소거 타이밍에 맞춰 조정 |
| 상단 임계값 | `SCROLL_TOP_ACTIVE_RESET_PX` | 모듈 2에서 export. 모듈 1과 동일 값 공유 필수 |

### 6.5 억제 API 사용 가이드

| 시나리오 | 호출 함수 | 시점 |
|----------|----------|------|
| 고정 UI 버튼 클릭 직후 | `suppressScrollChromeForChatBarMs(600)` | `pointerdown` / `click` 핸들러 |
| 입력 요소 포커스/블러 | `suppressScrollChromeForKeyboardMs(850)` | 자동 적용 (포커스 리스너) |
| 오버레이/코치마크 표시 | `setCoachmarkScrollChromeBlocked(true)` | 오버레이 open 시 |
| 오버레이/코치마크 해제 | `setCoachmarkScrollChromeBlocked(false)` | 오버레이 close 시 |

---

## 7. 제약 및 주의사항

| ID | 내용 |
|----|------|
| CON-01 | UA 기반 감지이므로, 삼성인터넷이 UA 문자열을 변경하거나 다른 브라우저가 `SamsungBrowser`를 포함하는 경우 오감지 가능 |
| CON-02 | `visualViewport` API 미지원 환경(구형 브라우저)에서는 vv 기반 스크롤 감지·키보드 억제가 동작하지 않음 (null 체크로 안전 장치) |
| CON-03 | 모듈 변수(`listenerAttached` 등)는 모듈 로드 단위로 1회만 초기화됨. HMR 환경에서는 중복 등록 가능성 있음 |
| CON-04 | `useState` 키 문자열이 충돌하지 않도록 프로젝트 전역에서 고유해야 함 (`'samsung-chat-chrome-elevated'`, `'scroll-chrome-active'`, `'scroll-chrome-visible'`) |
| CON-05 | 올림/하향 오프셋 클래스(`bottom-7`, `bottom-16`)는 Tailwind CSS 기준. 다른 CSS 프레임워크 사용 시 computed 반환값 조정 필요 |
| CON-06 | `SCROLL_TOP_ACTIVE_RESET_PX`(200px)는 모듈 1과 모듈 2가 공유하는 임계값. 단일 소스에서 export하여 정합성 보장 |
| CON-07 | 네이티브 "맨 위로" 버튼의 정확한 노출/소거 스크롤 임계값은 삼성인터넷 버전에 따라 다를 수 있음. 200px은 경험적 근사치 |

---

## 8. 검증 항목

| ID | 항목 | 방법 |
|----|------|------|
| VER-01 | 삼성인터넷에서 스크롤 시 고정 UI가 올라가는가 | 실기기/에뮬레이터 UA 변경 후 스크롤 |
| VER-02 | 스크롤 정지 1800ms 후 원위치하는가 | 스크롤 후 타이밍 측정 |
| VER-03 | 상단 200px 이내에서 즉시 원위치하는가 | `scrollTo(0, 100)` 후 확인 |
| VER-04 | 비삼성 환경에서 no-op인가 | Chrome/Safari에서 `samsungBarElevated` 항상 `false` |
| VER-05 | 라우트 전환 시 상태가 초기화되는가 | 페이지 이동 후 `samsungBarElevated == false` |
| VER-06 | 복수 컴포넌트가 동일 상태를 공유하는가 | 두 컴포넌트 동시 렌더 시 오프셋 동기 |
| VER-07 | 키보드 표시 시 스크롤 활동으로 오인하지 않는가 | 입력 포커스 후 `scrollActive` 변화 없음 |
| VER-08 | SSR 환경에서 에러 없이 렌더되는가 | SSR 빌드 후 페이지 소스 확인 |
| VER-09 | 코치마크 표시 중 스크롤 활동이 억제되는가 | 오버레이 open 상태에서 스크롤 시 `scrollActive == false` |
| VER-10 | `prefers-reduced-motion` 시 전환이 생략되는가 | OS 설정 후 `transition` 미적용 확인 |

---

## 9. 파일 구성 (재사용 패키지)

```
composables/
├── useSamsungInternetBrowser.ts   # 모듈 3: UA 감지 (무의존)
├── useScrollChromeVisibility.ts    # 모듈 2: 스크롤 활동 감지 (무의존)
└── useSamsungChatChrome.ts         # 모듈 1: 오케스트레이션 (모듈 2, 3 의존)
```

**의존 그래프**

```
useSamsungChatChrome
  ├── useScrollChromeVisibility
  │     └── (isWindowScrollWithinTopResetPx export)
  └── useSamsungInternetBrowser
```

**외부 의존**: Vue 3 (`ref`, `computed`, `watch`, `nextTick`, `onMounted`), Nuxt (`useState`, `useRouter`, `import.meta.client`/`import.meta.server`)

---

이 설계서는 프로젝트 비의존적 단위로 작성되었으며, 오프셋 값·임계값·지연 시간을 조정하면 다른 Vue 3/Nuxt 프로젝트에 그대로 적용 가능합니다.