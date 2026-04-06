---
name: feature-flag
description: "Use when adding, toggling, or removing feature flags in meloming projects, or when the user says: 피쳐플래그 추가, feature flag, 기능 토글, 킬 스위치, 새 기능 플래그"
---

# Feature Flag Management

PostHog Feature Flags로 meloming 클라이언트(Front/iOS/Android)의 기능을 독립적으로 켜고 끈다. 백엔드 변경 없이 클라이언트 UI만 제어.

**Security boundary:** Flag는 UX 게이팅일 뿐, 보안/인가가 아니다. API는 항상 열려있음.

## When to Use

- 새 기능에 feature flag 추가할 때
- 기존 기능에 킬 스위치 감쌀 때
- feature flag 제거할 때
- PostHog 대시보드 설정 안내할 때

## Flag Registry Locations

| Platform | File | Format |
|----------|------|--------|
| Front | `src/shared/lib/feature-flags.ts` | `{ key: string, default: boolean }` |
| iOS | `Meloming/Core/FeatureFlags/FeatureFlags.swift` | `enum FeatureFlag: String` with `defaultValue` |
| Android | `core/analytics/.../FeatureFlag.kt` | `enum class FeatureFlag(key, default)` |

## Adding a New Flag

### 1. Flag key 결정

- Format: **kebab-case** (e.g. `smart-song-addition`)
- 3개 플랫폼 동일한 key 문자열 사용

### 2. Default value 결정

- **신규 기능** (미릴리즈): `default = false`
- **기존 기능 킬 스위치**: `default = true`

### 3. 레지스트리 추가 (3개 플랫폼)

**Front** (`meloming-front/src/shared/lib/feature-flags.ts`):
```typescript
export const FeatureFlags = {
  // ... existing flags
  myNewFeature: { key: 'my-new-feature', default: false },
} as const
```

**iOS** (`meloming-ios/Meloming/Core/FeatureFlags/FeatureFlags.swift`):
```swift
enum FeatureFlag: String, CaseIterable {
    // ... existing cases
    case myNewFeature = "my-new-feature"

    var defaultValue: Bool {
        switch self {
        // ... existing cases
        case .myNewFeature: return false
        }
    }
}
```

**Android** (`meloming-android/core/analytics/.../FeatureFlag.kt`):
```kotlin
enum class FeatureFlag(val key: String, val default: Boolean) {
    // ... existing entries
    MY_NEW_FEATURE("my-new-feature", false),
}
```

### 4. UI에서 flag 사용

**Front:**
```tsx
const enabled = useFeatureFlag('myNewFeature')
if (enabled) { /* show feature */ }
```

**iOS:**
```swift
@ObservedObject private var flags = FeatureFlagManager.shared
if flags.isEnabled(.myNewFeature) { /* show feature */ }
```

**Android:**
```kotlin
val revision by featureFlagManager.flagsRevision.collectAsStateWithLifecycle()
if (featureFlagManager.isEnabled(FeatureFlag.MY_NEW_FEATURE)) { /* show feature */ }
```

### 5. PostHog 대시보드

1. Flag 생성: key = `my-new-feature`
2. QA condition: `environment = qa` → 100% rollout
3. 테스트 후 prod condition 추가

## Removing a Flag

1. PostHog 대시보드에서 flag 삭제
2. 코드에서 flag 분기 제거 (항상 활성화 경로만 남김)
3. 3개 플랫폼 레지스트리에서 flag 삭제

## Evaluation Wrappers (DO NOT call PostHog directly)

| Platform | Wrapper | Reactivity |
|----------|---------|------------|
| Front | `useFeatureFlag(name)` hook | `useFeatureFlagEnabled` — 자동 리렌더링 |
| iOS | `FeatureFlagManager.shared.isEnabled(flag)` | `ObservableObject` + `NotificationCenter` |
| Android | `featureFlagManager.isEnabled(flag)` | `StateFlow` + `PostHogOnFeatureFlags` |

## Common Mistakes

- **PostHog API 직접 호출** — 항상 래퍼를 통해 사용. 래퍼가 default 처리 + reactivity 보장
- **플랫폼별 key 불일치** — 반드시 동일한 kebab-case key 사용
- **Android에서 revision collect 안 함** — `flagsRevision.collectAsStateWithLifecycle()` 없으면 flag 변경 시 UI 안 바뀜
- **iOS에서 @ObservedObject 안 씀** — `FeatureFlagManager.shared.isEnabled()` 직접 호출만으로는 SwiftUI 리렌더링 안 됨
