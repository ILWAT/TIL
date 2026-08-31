# [TCA] Reducer 개념 정리

TCA(The Composable Architecture)에서 **Reducer**는 앱의 상태 변화 로직을 담는 핵심 단위다. "현재 상태(State)와 발생한 액션(Action)을 받아, 상태를 바꾸고 필요하면 부수 효과(Effect)를 반환하는 함수"라고 보면 된다.

```text
(State, Action) -> (State 변경, Effect)
```

---

## 1. 구성 요소

하나의 Reducer는 보통 세 가지로 이루어진다.

| 요소 | 역할 |
|---|---|
| `State` | 화면이 필요로 하는 데이터. 값 타입(struct)으로 정의한다. |
| `Action` | 사용자 입력, 응답 도착 등 앱에서 일어날 수 있는 모든 사건. enum으로 정의한다. |
| `body` | 액션이 왔을 때 State를 어떻게 바꾸고 무엇을 실행할지 기술하는 곳. |

핵심 규칙은 **State는 오직 Reducer 안에서만 바뀐다**는 것이다. View는 상태를 직접 수정하지 않고 Action만 보낸다.

---

## 2. 기본 예시 — 카운터

```swift
import ComposableArchitecture

@Reducer
struct CounterFeature {

    @ObservableState
    struct State: Equatable {
        var count = 0
    }

    enum Action {
        case incrementButtonTapped
        case decrementButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none          // 실행할 부수 효과 없음

            case .decrementButtonTapped:
                state.count -= 1
                return .none
            }
        }
    }
}
```

- `state`는 `inout`이므로 직접 수정하면 된다.
- 반환값은 `Effect`다. 할 일이 없으면 `.none`을 돌려준다.

---

## 3. View와 연결하기

```swift
struct CounterView: View {
    let store: StoreOf<CounterFeature>

    var body: some View {
        VStack {
            Text("\(store.count)")

            Button("+") { store.send(.incrementButtonTapped) }
            Button("-") { store.send(.decrementButtonTapped) }
        }
    }
}

// 사용
CounterView(
    store: Store(initialState: CounterFeature.State()) {
        CounterFeature()
    }
)
```

`Store`는 State를 보관하고, `send`로 들어온 Action을 Reducer에 흘려보내는 런타임 역할을 한다.

---

## 4. 부수 효과(Effect) — 비동기 작업

네트워크 호출처럼 시간이 걸리는 작업은 Reducer 안에서 직접 하지 않고 `.run`으로 반환한다. 결과는 **다시 Action으로 되돌려 받는다.**

```swift
enum Action {
    case factButtonTapped
    case factResponse(String)
}

var body: some ReducerOf<Self> {
    Reduce { state, action in
        switch action {
        case .factButtonTapped:
            state.isLoading = true
            return .run { [count = state.count] send in
                let fact = try await numberFactClient.fetch(count)
                await send(.factResponse(fact))   // 결과를 Action으로 전달
            }

        case let .factResponse(fact):
            state.isLoading = false
            state.fact = fact
            return .none
        }
    }
}
```

상태 변경은 항상 동기적으로 Reducer 안에서 일어나고, 비동기 작업은 Effect가 담당한다. 이 분리 덕분에 Reducer는 순수 함수에 가까워지고 테스트하기 쉬워진다.

---

## 5. 정리

- Reducer = **(State, Action) → State 변경 + Effect**
- State 변경은 Reducer 안에서만, View는 Action만 보낸다.
- 즉시 끝나는 로직은 `state` 수정 후 `.none`.
- 오래 걸리는 로직은 `.run`으로 넘기고, 결과는 새 Action으로 받는다.

---

### References

[Reducer | The Composable Architecture Documentation](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/reducer)

[swift-composable-architecture | GitHub](https://github.com/pointfreeco/swift-composable-architecture)
