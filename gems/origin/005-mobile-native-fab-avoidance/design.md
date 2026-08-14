# Mobile Native Floating Control Avoidance

## Problem

일부 모바일 브라우저는 스크롤 중 화면 하단에 네이티브 Floating Control을 표시합니다. 이 UI는 웹 페이지가 직접 숨길 수 없고, 서비스의 Fixed Bottom Bar나 FAB와 겹칠 수 있습니다.

## Intent

Native Control을 제거하려 하지 않고, 노출 가능성이 높은 동안에만 웹의 Fixed UI를 위로 이동하고 활동이 끝나면 원위치합니다.

## State

```text
browserMatchesTarget
scrollActive
uiElevated
suppressedUntil
```

여러 Fixed UI가 동일한 `uiElevated` 상태를 공유하도록 합니다.

## Flow

```text
scroll input
   ↓
Suppressed?
 ├─ yes → ignore
 └─ no
      ↓
Near top?
 ├─ yes → uiElevated = false
 └─ no  → scrollActive = true
             ↓
          uiElevated = true
             ↓ idle delay
          uiElevated = false
```

## Suppression

키보드 표시/숨김, Fixed UI 자체의 상호작용, Coachmark/Overlay처럼 스크롤 이벤트를 실제 사용자 탐색으로 보기 어려운 구간은 짧은 Suppression Window를 둘 수 있습니다.

## Browser Detection

특정 브라우저에 대한 workaround는 Client Runtime에서만 활성화합니다. User-Agent 감지는 불완전할 수 있으므로 Feature Detection이 가능한 문제라면 Feature Detection을 우선하고, UA가 필요한 경우 영향 범위를 이 Gem 내부로 제한합니다.

## Configurable Values

다음 값은 제품 Layout과 브라우저 버전에 따라 바뀔 수 있으므로 Gem의 고정 상수가 아니라 Configuration으로 취급합니다.

- scroll idle delay
- top reset threshold
- base bottom offset
- elevated bottom offset
- keyboard suppression duration

## Cleanup

Route 전환과 Component lifecycle에서 Timer와 Scroll State를 정리합니다. Listener는 중복 등록되지 않도록 단일 소유자를 둡니다.

## Accessibility

위치 이동에는 `prefers-reduced-motion`을 고려하고, Native Control 회피 때문에 주요 Action이 Viewport 밖으로 이동하지 않는지 검증합니다.

## Validation

- 대상 브라우저에서 스크롤 시 겹침이 줄어드는지
- 비대상 브라우저에서 no-op인지
- 문서 상단에서 즉시 원위치하는지
- 키보드/Overlay 이벤트가 오탐되지 않는지
- Route 전환 후 상태가 남지 않는지
