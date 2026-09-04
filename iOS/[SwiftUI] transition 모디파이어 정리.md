# [SwiftUI] transition 모디파이어 정리

`transition(_:)`은 **뷰가 뷰 계층(view hierarchy)에 삽입(insertion)되거나 제거(removal)될 때** 어떤 모습으로 나타나고 사라질지를 지정하는 모디파이어다.

한 줄로 요약하면 이렇다.

> 트랜지션은 **"어떤 모양으로"** 나타나고 사라질지를, 애니메이션은 **"어떤 타이밍으로"** 움직일지를 정한다. 애니메이션 없이 트랜지션만 지정하면 아무 일도 일어나지 않는다.

---

## 1. 기본 사용법

```swift
struct ContentView: View {
    @State private var isShown = false

    var body: some View {
        VStack(spacing: 20) {
            Button("토글") {
                withAnimation {
                    isShown.toggle()
                }
            }

            if isShown {
                Text("안녕하세요")
                    .padding()
                    .background(.yellow)
                    .transition(.slide)
            }
        }
    }
}
```

버튼을 누르면 `Text`가 leading(왼쪽)에서 미끄러져 들어오고, 다시 누르면 trailing(오른쪽)으로 빠져나간다.

트랜지션이 보이려면 세 가지가 모두 갖춰져야 한다.

1. **삽입/제거되는 뷰** — `if`, `switch` 등으로 뷰 계층에 들어오고 나가는 뷰여야 한다.
2. **그 뷰에 붙은 `.transition`** — 생략하면 기본값인 `.opacity`(페이드)가 쓰인다.
3. **애니메이션** — 상태 변경이 `withAnimation` 등 애니메이션이 있는 트랜잭션 안에서 일어나야 한다.

### 시그니처

| 선언 | 지원 버전 | 비고 |
| :--- | :--- | :--- |
| `func transition(_ t: AnyTransition) -> some View` | iOS 13+ | 타입이 지워진(type-erased) 트랜지션 |
| `func transition<T: Transition>(_ transition: T) -> some View` | iOS 17+ | `Transition` 프로토콜 기반 |

### 기본값이 opacity인 이유 — transition은 "트레잇"이다

Xcode SDK의 `SwiftUICore.swiftinterface`를 열어 보면 구현이 그대로 드러난다. (Xcode 26.6 / iOS 26.5 SDK 기준, 관련 부분만 발췌)

```swift
extension View {
    @inlinable @_disfavoredOverload
    nonisolated public func transition(_ t: AnyTransition) -> some View {
        return _trait(TransitionTraitKey.self, t)
    }

    @available(iOS 17.0, macOS 14.0, tvOS 17.0, watchOS 10.0, *)
    @_alwaysEmitIntoClient
    nonisolated public func transition<T>(_ transition: T) -> some View where T : Transition {
        self.transition(AnyTransition(transition))
    }
}

internal struct TransitionTraitKey : _ViewTraitKey {
    @inlinable internal static var defaultValue: AnyTransition {
        get { .opacity }
    }
}
```

여기서 세 가지를 알 수 있다.

- `.transition`은 뷰에 효과를 입히는 모디파이어가 아니라, **"이 뷰가 들어오고 나갈 때는 이 트랜지션을 써라"라는 트레잇(trait) 값을 기록**할 뿐이다. 그래서 뷰가 계층에 그대로 있는 한 아무 일도 일어나지 않는다.
- 트레잇의 기본값이 `.opacity`다. `.transition`을 붙이지 않아도 애니메이션 안에서 삽입/제거되는 뷰는 페이드된다. Apple의 SwiftUI 튜토리얼에서도 기본 트랜지션을 페이드 인/아웃으로 설명한다.
- iOS 17의 `Transition` 버전도 결국 `AnyTransition`으로 감싸 iOS 13 버전을 호출한다. 두 API는 같은 메커니즘 위에 있다.

---

## 2. 언제 "삽입/제거"가 일어나는가 — View Identity

SwiftUI는 뷰의 **identity(정체성)** 가 바뀌면 기존 뷰를 제거하고 새 뷰를 삽입한다. 트랜지션은 바로 이 순간에 적용된다.

| 상황 | 트랜지션 | 이유 |
| :--- | :---: | :--- |
| `if isShown { ... }`의 조건이 바뀜 | ⭕️ | 뷰가 계층에 들어오고 나간다 |
| `if/else`, `switch`의 분기가 바뀜 | ⭕️ | 분기마다 identity가 달라 한쪽은 제거, 한쪽은 삽입된다 |
| `ForEach`의 데이터가 추가/삭제됨 | ⭕️ | 해당 id의 뷰가 삽입/제거된다 |
| `.id(_:)`에 넘긴 값이 바뀜 | ⭕️ | identity가 바뀌어 기존 뷰 제거 + 새 뷰 삽입 |
| `.opacity(0)`, `.hidden()`으로 숨김 | ❌ | 뷰는 그대로 있고 속성만 바뀐다 |
| 크기·위치·색상·텍스트 내용 변경 | ❌ | 같은 뷰의 속성 변화 → 일반 애니메이션의 대상 |

### if/else — 제거와 삽입이 동시에 일어난다

```swift
if isLoggedIn {
    ProfileView()
        .transition(.move(edge: .trailing))
} else {
    LoginView()
        .transition(.move(edge: .leading))
}
```

`isLoggedIn`이 바뀌면 `LoginView`의 제거 트랜지션과 `ProfileView`의 삽입 트랜지션이 **동시에** 실행된다. 애니메이션이 진행되는 동안에는 두 뷰가 모두 트리에 존재한다.

### .id(_:) — 같은 자리의 뷰를 "교체"로 만들기

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
                .font(.largeTitle)
                .id(count)                         // 값이 바뀔 때마다 다른 뷰로 취급
                .transition(.push(from: .bottom))  // iOS 16+

            Button("+1") {
                withAnimation { count += 1 }
            }
        }
    }
}
```

값이 바뀔 때마다 이전 `Text`는 위로 밀려 나가고, 새 `Text`가 아래에서 밀고 들어온다.

단, **identity가 바뀐다는 것은 상태도 새로 시작한다는 뜻**이다. `.id`가 붙은 뷰와 그 하위 뷰의 `@State`는 값이 바뀔 때마다 초기화된다. 숫자 변경을 연출하는 게 목적이라면 identity를 건드리지 않는 `contentTransition(_:)`이 더 적합하다. (8절)

### 조건문 vs 모디파이어 — 무엇을 쓸까

WWDC21 *Demystify SwiftUI*에서는 기본적으로 **identity를 유지하는 쪽**을 권장한다. identity가 유지되어야 전환이 매끄럽고, 뷰의 수명과 상태도 보존되기 때문이다.

```swift
// 1) 분기 — 서로 다른 두 뷰. 바뀔 때 제거/삽입 트랜지션(기본값: 페이드)이 일어난다
if isExpired {
    CouponView()
        .opacity(0.5)
} else {
    CouponView()
}

// 2) 모디파이어 — 하나의 뷰. 속성만 부드럽게 애니메이션되고 상태도 유지된다
CouponView()
    .opacity(isExpired ? 0.5 : 1.0)
```

- **"나타난다/사라진다" 자체가 의미인 UI**(토스트, 팝업, 에러 배너 등) → 조건문 + `transition`
- **같은 요소의 상태가 바뀌는 UI**(활성/비활성, 강조 등) → 모디파이어 값 변경

---

## 3. 애니메이션을 연결하는 세 가지 방법

### 3-1. withAnimation — 가장 명확한 방법

```swift
Button("토글") {
    withAnimation(.easeInOut(duration: 0.3)) {
        isShown.toggle()
    }
}
```

iOS 17부터는 완료 콜백을 받을 수 있어서, **사라지는 트랜지션이 끝난 뒤** 작업을 이어갈 때 유용하다.

```swift
withAnimation(.easeInOut(duration: 0.3)) {
    isShown = false
} completion: {
    // 제거 트랜지션이 끝난 뒤 실행
    onDismiss()
}
```

### 3-2. .animation(_:value:) — 항상 존재하는 부모에 붙인다

```swift
// ❌ 삽입/제거되는 뷰 자신에 붙이면 트랜지션이 애니메이션되지 않는다
VStack {
    if isShown {
        DetailView()
            .transition(.slide)
            .animation(.easeInOut, value: isShown)
    }
}

// ⭕️ 조건문을 감싸고 있는, 계속 살아 있는 컨테이너에 붙인다
VStack {
    if isShown {
        DetailView()
            .transition(.slide)
    }
}
.animation(.easeInOut, value: isShown)
```

`.animation(_:value:)`은 **자신이 계층에 존재하면서 `value`의 변화를 지켜봐야** 동작한다. 조건부 뷰에 붙이면 삽입될 때는 모디파이어가 막 생겨나 비교할 이전 값이 없고, 제거될 때는 뷰와 함께 모디파이어도 사라진다. 결국 삽입/제거 어느 쪽에도 애니메이션을 걸 기회가 없다.

### 3-3. 트랜지션 자체에 애니메이션 붙이기 — `.animation(_:)`

```swift
if isShown {
    DetailView()
        .transition(.opacity.animation(.easeInOut(duration: 0.3)))
}
```

`withAnimation` 없이 상태만 바꿔도 이 트랜지션은 지정한 애니메이션으로 동작한다. 삽입과 제거에 서로 다른 타이밍을 줄 때 특히 유용하다.

```swift
.transition(
    .asymmetric(
        insertion: .move(edge: .top).animation(.spring()),
        removal: .opacity.animation(.easeOut(duration: 0.15))
    )
)
```

다만 알고 써야 할 점이 있다.

- 애니메이션이 **트랜지션에만** 붙는다. 뷰가 들어오고 나가면서 밀려나는 주변 뷰의 레이아웃 변화까지 부드럽게 하려면 여전히 `withAnimation`이나 `.animation(_:value:)`이 필요하다.
- 트랜지션 종류에 따라 이 방식이 기대대로 동작하지 않았다는 사례가 있다(The SwiftUI Lab). 기본은 3-1, 3-2 방식으로 두고, 타이밍을 따로 줘야 할 때 쓰는 편이 안전하다.

---

## 4. 기본 제공 트랜지션

| 트랜지션 | 삽입 | 제거 | 지원 |
| :--- | :--- | :--- | :--- |
| `.opacity` | 투명 → 불투명 | 불투명 → 투명 | iOS 13+ (**기본값**) |
| `.scale` | 0에 가까운 크기 → 원래 크기 | 원래 크기 → 0에 가까운 크기 | iOS 13+ |
| `.scale(scale:anchor:)` | 지정 배율 → 원래 크기 | 원래 크기 → 지정 배율 | iOS 13+ |
| `.slide` | leading에서 들어옴 | trailing으로 나감 | iOS 13+ |
| `.move(edge:)` | 지정한 edge에서 들어옴 | **같은** edge로 나감 | iOS 13+ |
| `.offset(x:y:)` | 지정 오프셋 → 제자리 | 제자리 → 지정 오프셋 | iOS 13+ |
| `.push(from:)` | 지정한 edge에서 페이드 인하며 들어옴 | **반대쪽** edge로 페이드 아웃하며 나감 | iOS 16+ |
| `.blurReplace` | 블러가 걷히며 커짐 | 블러되며 작아짐 | iOS 17+ |
| `.symbolEffect` | 뷰 안의 SF Symbol에 Appear 효과 | 뷰 안의 SF Symbol에 Disappear 효과 | iOS 17+ |
| `.identity` | 효과 없음 | 효과 없음 | iOS 13+ |

- `.blurReplace`의 기본 설정은 `.downUp`이다. `.blurReplace(.upUp)`을 쓰면 사라질 때도 커지면서 사라진다.
- `.symbolEffect`는 삽입/제거되는 뷰 안의 **SF Symbol 이미지에만** 적용되고, 다른 뷰에는 영향을 주지 않는다.

### 방향 비교 — move vs slide vs push

```text
                         삽입(insertion)              제거(removal)
.move(edge: .leading)    leading에서 등장              leading으로 퇴장 (같은 쪽)
.slide                   leading에서 등장              trailing으로 퇴장 (반대쪽)
.push(from: .leading)    leading에서 등장 + 페이드 인   trailing으로 퇴장 + 페이드 아웃
```

`.slide`는 `.asymmetric(insertion: .move(edge: .leading), removal: .move(edge: .trailing))`와 같은 움직임이다. 방향을 바꾸고 싶다면 `.push(from:)`이나 `.asymmetric`으로 직접 조합한다.

### AnyTransition 버전과 Transition 버전의 차이

iOS 17부터는 같은 이름의 트랜지션이 `Transition` 프로토콜을 따르는 구조체(`OpacityTransition`, `MoveTransition`, `PushTransition` 등)로도 제공된다. 사용법은 거의 같지만 SDK를 보면 몇 가지 차이가 있다.

```swift
.transition(.scale(scale: 0.5, anchor: .top))  // AnyTransition — iOS 13+
.transition(.scale(0.5, anchor: .top))         // ScaleTransition — iOS 17+ (라벨 없음)
```

- `scale`은 두 버전의 **인자 라벨이 다르다.**
- 인자 없는 `.scale`은 iOS 17 버전 기준 `ScaleTransition(1e-5)`로 선언되어 있다. 정확히 0이 아니라 0에 아주 가까운 배율에서 시작한다.
- `.asymmetric`은 `Transition` 쪽에 정적 메서드가 없고 `AsymmetricTransition(insertion:removal:)` 구조체로 제공된다. `.transition(.asymmetric(...))`으로 쓰면 `AnyTransition` 버전이 선택되므로 쓰는 법은 같다.

---

## 5. 트랜지션 조합하기

### asymmetric(insertion:removal:) — 들어올 때와 나갈 때를 다르게

```swift
.transition(
    .asymmetric(
        insertion: .move(edge: .bottom),
        removal: .opacity
    )
)
```

### combined(with:) — 여러 효과를 동시에

```swift
.transition(.move(edge: .bottom).combined(with: .opacity))
```

`combined(with:)`는 여러 번 체이닝할 수 있고, `asymmetric` 안에 넣을 수도 있다.

### 예시 — 위에서 내려왔다가 페이드로 사라지는 토스트

```swift
struct ToastExampleView: View {
    @State private var isToastShown = false

    var body: some View {
        ZStack(alignment: .top) {
            Button("저장") {
                withAnimation(.spring()) {
                    isToastShown = true
                }
                Task {
                    try? await Task.sleep(for: .seconds(2))
                    withAnimation(.easeOut) {
                        isToastShown = false
                    }
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            if isToastShown {
                Text("저장되었습니다")
                    .padding(.horizontal, 20)
                    .padding(.vertical, 12)
                    .background(.thinMaterial, in: Capsule())
                    .transition(
                        .asymmetric(
                            insertion: .move(edge: .top).combined(with: .opacity),
                            removal: .opacity
                        )
                    )
                    .zIndex(1)  // ZStack에서 사라지는 애니메이션이 가려지지 않게 (7-2 참고)
            }
        }
    }
}
```

---

## 6. 커스텀 트랜지션 만들기

### 6-1. iOS 17+ — Transition 프로토콜

```swift
// SDK 선언에서 동시성 어노테이션과 내부용 멤버를 생략해 정리
public protocol Transition {
    associatedtype Body : View
    @ViewBuilder func body(content: Content, phase: TransitionPhase) -> Body
    static var properties: TransitionProperties { get }   // 기본 구현 제공
    typealias Content = PlaceholderContentView<Self>
}
```

`body(content:phase:)` 하나만 구현하면 된다. `phase`는 지금이 트랜지션의 어느 단계인지 알려준다.

#### TransitionPhase

| phase | 의미 | `isIdentity` | `value` |
| :--- | :--- | :---: | :---: |
| `.willAppear` | 삽입되기 직전 (시작 모습) | `false` | `-1.0` |
| `.identity` | 화면에 정상적으로 떠 있는 상태 | `true` | `0` |
| `.didDisappear` | 제거된 직후 (끝 모습) | `false` | `1.0` |

```text
삽입:  .willAppear ──(애니메이션)──▶ .identity
제거:  .identity   ──(애니메이션)──▶ .didDisappear
```

들어오는 도중에 제거되면 `.didDisappear`로, 나가는 도중에 다시 추가되면 `.identity`로 되돌아간다.

#### 예시 — 뒤집히며 나타나는 카드

```swift
struct FlipTransition: Transition {
    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            // willAppear: -90° → identity: 0° → didDisappear: 90°
            .rotation3DEffect(.degrees(phase.value * 90), axis: (x: 1, y: 0, z: 0))
            .opacity(phase.isIdentity ? 1 : 0)
    }
}

extension Transition where Self == FlipTransition {
    static var flip: FlipTransition { FlipTransition() }
}

// 사용
CardView()
    .transition(.flip)
```

`phase.value`를 곱하면 시작 모습(`-1`)과 끝 모습(`1`)의 부호가 반대가 되어, **한 방향으로 이어지는** 효과를 한 줄로 표현할 수 있다. 위 카드는 -90°에서 0°로 회전하며 나타났다가, 사라질 때는 같은 방향으로 계속 회전해 90°로 넘어가며 사라진다.

- 대칭 효과(나타날 때와 사라질 때 모양이 같음) → `phase.isIdentity`
- 비대칭 효과(방향이 다름) → `phase.value` 또는 `switch phase`

#### 작성 규칙 (Apple 문서 기준)

1. **`.identity` 단계에서는 뷰를 바꾸지 않는다.** 이 단계의 수정 사항은 뷰가 화면에 있는 내내 적용된다.
2. **`.willAppear` / `.didDisappear`에서는 애니메이션 가능한 변화를 준다.** 애니메이션될 변화가 없으면 트랜지션은 아무 효과가 없다.
3. **`content`에 `.id`, `if`, `switch` 같은 identity를 바꾸는 코드를 쓰지 않는다.** 뷰의 상태가 초기화되어 불필요한 작업과 예상치 못한 동작이 생긴다.

#### properties — 동작 줄이기(Reduce Motion) 대응

`Transition`에는 `static var properties: TransitionProperties`가 있고, `TransitionProperties`는 현재 `hasMotion` 하나를 가진다(기본값 `true`). Apple 문서에 따르면 `hasMotion`이 `true`인 트랜지션은 **'동작 줄이기'(설정 > 손쉬운 사용 > 동작)가 켜져 있으면 opacity 트랜지션으로 대체**된다.

즉 위의 `FlipTransition`은 별도 처리 없이도 동작 줄이기 사용자에게는 페이드로 보인다. 반대로 움직임이 없는 트랜지션이라면 `false`로 선언해 원래 효과가 유지되게 한다.

```swift
struct BlurFadeTransition: Transition {
    static var properties: TransitionProperties {
        TransitionProperties(hasMotion: false)   // 움직임이 없으므로 동작 줄이기에서도 그대로 사용
    }

    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .blur(radius: phase.isIdentity ? 0 : 10)
            .opacity(phase.isIdentity ? 1 : 0)
    }
}
```

### 6-2. iOS 16 이하 — AnyTransition.modifier(active:identity:)

```swift
struct FlipModifier: ViewModifier {
    let angle: Double

    func body(content: Content) -> some View {
        content
            .rotation3DEffect(.degrees(angle), axis: (x: 1, y: 0, z: 0))
            .opacity(angle == 0 ? 1 : 0)
    }
}

extension AnyTransition {
    static var flip: AnyTransition {
        .modifier(
            active: FlipModifier(angle: 90),   // 나타나기 전 / 사라진 후의 모습
            identity: FlipModifier(angle: 0)   // 화면에 떠 있을 때의 모습
        )
    }
}
```

- 삽입은 `active` → `identity`, 제거는 `identity` → `active`로 보간된다.
- `active` 하나로 시작과 끝을 모두 표현하므로 **방향이 대칭**이다. 들어올 때와 나갈 때를 다르게 하려면 `.asymmetric`으로 두 개를 조합한다.
- 모디파이어 안의 효과(`rotation3DEffect`, `opacity` 등)가 애니메이션 가능해야 보간된다.

### AnyTransition(_:) — Transition을 타입 소거하기

`Transition`은 제네릭이라 삼항 연산자로 서로 다른 트랜지션 타입을 섞을 수 없다. 조건에 따라 트랜지션을 고르고 싶다면 `AnyTransition(_:)`(iOS 17+)으로 감싸 타입을 통일한다.

```swift
.transition(isCompact ? .opacity : AnyTransition(FlipTransition()))
```

---

## 7. 자주 겪는 문제

### 7-1. 트랜지션이 전혀 보이지 않는다

아래 순서로 확인한다.

1. 상태 변경에 **애니메이션**이 있는가? (`withAnimation` / 부모의 `.animation(_:value:)` / `transition.animation(_:)`)
2. `.animation(_:value:)`을 **삽입/제거되는 뷰 자신**에 붙이지 않았는가? (3-2)
3. 실제로 **삽입/제거**가 일어나는가? `.opacity(0)`, `.hidden()`은 트랜지션 대상이 아니다. (2절)
4. `List` 안인가? (7-4)

### 7-2. ZStack에서 사라질 때만 애니메이션이 안 된다

```swift
ZStack {
    MainContentView()

    if isPopupShown {
        PopupView()
            .transition(.move(edge: .bottom))
            .zIndex(1)   // ⭕️ 0이 아닌 zIndex를 명시
    }
}
```

`ZStack`에서 모든 뷰의 `zIndex`가 기본값 0이면, 제거되는 뷰의 그리기 순서가 유지되지 않아 **다른 뷰 뒤로 들어가 버린다.** 애니메이션은 실행되지만 가려져서 보이지 않는 것이다. 트랜지션이 걸린 뷰에 `.zIndex(1)`처럼 0이 아닌 값을 명시하면 해결된다. 반대로 아래에 깔리는 뷰에 `.zIndex(-1)`을 줘도 된다.

### 7-3. move(edge:)가 화면 끝에서 들어오지 않는다

`move(edge:)`는 **화면이 아니라 뷰 자신의 가장자리**를 기준으로, 자기 크기만큼 이동한다. 화면 가운데 있는 작은 뷰에 `.move(edge: .bottom)`을 주면 화면 맨 아래에서 올라오는 게 아니라 제자리 바로 아래에서 올라온다.

- 화면 끝에서 들어오게 하려면 뷰를 화면 끝에 배치하거나(`.frame(maxHeight: .infinity, alignment: .bottom)` 등), `.offset(y:)` 트랜지션에 충분한 거리를 준다.
- 컨테이너 밖으로 삐져나오는 부분이 보기 싫다면 컨테이너에 `.clipped()`를 준다.

### 7-4. List에서 커스텀 트랜지션이 먹지 않는다

`List`는 행의 삽입·삭제 애니메이션을 스스로 관리하기 때문에 행에 적용할 수 있는 트랜지션에 제약이 있다. 원하는 트랜지션이 꼭 필요하다면 `ScrollView` + `LazyVStack`으로 구성한다.

### 7-5. 다시 나타난 뷰의 상태가 초기화되어 있다

제거된 뷰는 identity와 함께 상태도 사라진다. 다시 삽입되면 **완전히 새로운 뷰**이므로 `@State`는 초기값부터 시작한다. 유지해야 하는 값은 부모로 끌어올리거나(`@Binding`) 모델에 둔다.

---

## 8. 헷갈리는 관련 API

| API | 대상 | 쓰는 상황 | 지원 |
| :--- | :--- | :--- | :--- |
| `transition(_:)` | 뷰의 **삽입/제거** | 뷰가 나타나고 사라질 때 | iOS 13+ |
| `contentTransition(_:)` | **같은 뷰**의 내용 변경 | 숫자·텍스트·SF Symbol이 바뀔 때 | iOS 16+ |
| `matchedGeometryEffect(id:in:)` | **서로 다른 두 뷰**의 위치·크기 | 사라지는 뷰와 나타나는 뷰를 하나처럼 이어줄 때 | iOS 14+ |
| `navigationTransition(_:)` + `matchedTransitionSource(id:in:)` | **화면 전환** (push, sheet, fullScreenCover) | 탭한 요소에서 화면이 커지는 줌 전환 | iOS 18+ |

`contentTransition`은 identity를 바꾸지 않고 **한 뷰 안의 내용 변화**를 애니메이션한다. 이것 역시 애니메이션 컨텍스트 안에서만 효과가 있다.

```swift
Text("\(count)")
    .contentTransition(.numericText(value: Double(count)))   // iOS 17+ (iOS 16은 .numericText())

Image(systemName: isMuted ? "speaker.slash" : "speaker.wave.2")
    .contentTransition(.symbolEffect(.replace))              // iOS 17+
```

- 2절의 `.id()` + `.transition` 방식과 달리 상태가 초기화되지 않으므로, 숫자·아이콘 변경이라면 이쪽이 의도에 맞다.
- SDK를 보면 `contentTransition(_:)`은 내부적으로 환경값(`\.contentTransition`)을 설정한다. 상위 뷰에 한 번 걸어 두면 하위 뷰 전체에 적용된다.

---

## 9. 정리

```text
상태 변경 + 애니메이션 (withAnimation / 부모의 .animation(_:value:))
        ↓
identity 변화 → 뷰 삽입 / 제거  (if · switch · ForEach · .id)
        ↓
SwiftUI가 그 뷰의 transition 트레잇을 읽음  (기본값 .opacity)
        ↓
삽입: .willAppear → .identity
제거: .identity   → .didDisappear
```

- `transition`은 **삽입/제거에만** 반응하고, **애니메이션**이 있어야 보인다.
- `.animation(_:value:)`은 조건부 뷰가 아니라 **항상 존재하는 부모**에 붙인다.
- 들어올 때와 나갈 때를 다르게 → `asymmetric`, 여러 효과를 함께 → `combined(with:)`.
- 커스텀 트랜지션은 iOS 17+ `Transition` 프로토콜, 그 이하는 `modifier(active:identity:)`.
- `ZStack`에서 사라지는 애니메이션이 안 보이면 `zIndex`부터 의심한다.

---

## References

- [transition(_:) | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/view/transition(_:))
- [AnyTransition | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/anytransition)
- [Transition | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/transition)
- [TransitionPhase | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/transitionphase)
- [TransitionProperties.hasMotion | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/transitionproperties/hasmotion)
- [ContentTransition | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/contenttransition)
- [SwiftUI Tutorials - Animating views and transitions](https://developer.apple.com/tutorials/swiftui/animating-views-and-transitions)
- [Demystify SwiftUI - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10022/)
- [Enhance your UI animations and transitions - WWDC24](https://developer.apple.com/videos/play/wwdc2024/10145/)
- [Transitions in SwiftUI - objc.io](https://www.objc.io/blog/2022/04/14/transitions/)
- [Advanced SwiftUI Transitions - The SwiftUI Lab](https://swiftui-lab.com/advanced-transitions/)
- [How to fix ZStack's views disappear transition not animated in SwiftUI - Sarunw](https://sarunw.com/posts/how-to-fix-zstack-transition-animation-in-swiftui/)
- [List or LazyVStack - Fatbobman](https://fatbobman.com/en/posts/list-or-lazyvstack/)
- Xcode 26.6 / iOS 26.5 SDK — `SwiftUICore.framework/.../arm64e-apple-ios.swiftinterface`
