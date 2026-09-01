# [Swift] Actor 개념과 예시 정리

Swift 5.5(SE-0306)에서 도입된 `actor`는 **가변 상태(mutable state)를 데이터 경쟁(Data Race)으로부터 보호하기 위한 참조 타입**이다.

한 줄로 요약하면 이렇다.

> 하나의 actor 인스턴스가 가진 상태에는 **한 번에 하나의 작업(task)만** 접근할 수 있다.

---

## 1. 왜 필요한가 — Data Race

```swift
class Counter {
    var value = 0

    func increment() -> Int {
        value = value + 1
        return value
    }
}

let counter = Counter()

Task.detached { print(counter.increment()) }
Task.detached { print(counter.increment()) }
```

기대값은 `1, 2` 또는 `2, 1`이지만, 두 스레드가 동시에 `value`를 읽으면 `1, 1` 또는 `2, 2`가 나올 수 있다.

Data Race는 **공유(shared) + 가변(mutable)**이 겹칠 때 발생한다. 기존에는 이를 `DispatchQueue`나 `NSLock`으로 직접 막았지만, 이는 **개발자가 규칙을 지켜야만 성립하는 안전**이다. lock 거는 것을 한 군데라도 빠뜨리면 컴파일은 그대로 통과한다.

actor는 이 규칙을 **컴파일러가 강제**하도록 만든 장치다.

---

## 2. 기본 사용법

```swift
actor Counter {
    private var value = 0

    func increment() -> Int {
        value += 1          // actor 내부에서는 그냥 접근
        return value
    }

    func current() -> Int {
        value
    }
}
```

외부에서 접근할 때는 `await`가 필요하다.

```swift
let counter = Counter()

Task {
    let result = await counter.increment()   // await 필수
    print(result)
}
```

`await`를 빼면 컴파일 에러가 난다.

```swift
counter.increment()
// ❌ Actor-isolated instance method 'increment()' can not be
//    referenced from a non-isolated context
```

즉 **"실수로 lock을 안 걸었다"가 런타임 버그가 아니라 컴파일 에러**가 된다. 이것이 actor의 가장 큰 가치다.

---

## 3. Actor Isolation (액터 격리)

actor의 저장 프로퍼티와 메서드는 기본적으로 **isolated(격리됨)** 상태다.

| 접근 위치 | 동기 접근 | 설명 |
| :--- | :--- | :--- |
| actor 내부 | ⭕️ 가능 | 이미 그 actor 위에서 실행 중 |
| actor 외부 | ❌ 불가 | `await`로 비동기 접근만 가능 |

외부에서의 프로퍼티 **쓰기**는 `await`를 붙여도 불가능하다.

```swift
actor Counter {
    var value = 0
}

let counter = Counter()
print(await counter.value)   // ⭕️ 읽기는 await로 가능
await counter.value = 10     // ❌ 외부에서 직접 쓰기는 불가
```

상태를 바꾸려면 반드시 actor **내부에 메서드를 만들어** 그 메서드를 호출해야 한다. 상태 변경 로직이 actor 안에 모이도록 언어가 유도하는 구조다.

```swift
actor Counter {
    private(set) var value = 0

    func update(to newValue: Int) {
        value = newValue
    }
}

await counter.update(to: 10)   // ⭕️
```

---

## 4. nonisolated — 격리가 필요 없는 멤버

가변 상태를 건드리지 않는 멤버는 `nonisolated`로 표시해 동기적으로 호출할 수 있다.

```swift
actor User {
    let id: String              // let 상수는 애초에 경쟁 대상이 아님
    private var nickname: String

    init(id: String, nickname: String) {
        self.id = id
        self.nickname = nickname
    }

    nonisolated var description: String {
        "User(\(id))"           // ⭕️ 불변 프로퍼티만 사용
    }

    // nonisolated func broken() -> String { nickname }
    // ❌ Actor-isolated property 'nickname' can not be referenced
    //    from a nonisolated context
}

let user = User(id: "A01", nickname: "문")
print(user.id)              // await 없이 접근 가능 (let)
print(user.description)     // await 없이 접근 가능 (nonisolated)
```

`Hashable`, `Equatable`, `CustomStringConvertible` 같은 **동기 프로토콜을 채택할 때** 주로 쓰인다.

```swift
extension User: Equatable {
    nonisolated static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }
}
```

---

## 5. ⚠️ Actor Reentrancy — 가장 많이 걸리는 함정

actor는 **한 번에 하나의 작업만 실행**하지만, **중단(suspension) 없이 실행한다는 뜻은 아니다**.

`await`를 만나 작업이 중단되면 actor는 **그 사이 다른 작업을 처리한다.** 이를 재진입(reentrancy)이라고 한다.

### 문제 상황 — 중복 요청 캐시

```swift
actor ImageDownloader {
    private var cache: [URL: UIImage] = [:]

    func image(from url: URL) async throws -> UIImage {
        if let cached = cache[url] {
            return cached
        }

        let image = try await download(url)   // ⚠️ 여기서 중단됨
        cache[url] = image
        return image
    }
}
```

같은 URL로 거의 동시에 두 번 호출하면,

1. Task A: 캐시 미스 → `download` 시작 → **중단**
2. (actor가 비어 다른 작업 수행) Task B: 캐시 여전히 비어 있음 → `download` 시작 → **중단**
3. 결국 **같은 이미지를 두 번 다운로드**

크래시는 안 나지만 네트워크 요청이 중복된다. **`await` 앞뒤로 actor의 상태가 바뀔 수 있다**는 점이 핵심이다.

### 해결 — 진행 중인 작업을 함께 저장

```swift
actor ImageDownloader {

    private enum CacheEntry {
        case inProgress(Task<UIImage, Error>)
        case ready(UIImage)
    }

    private var cache: [URL: CacheEntry] = [:]

    func image(from url: URL) async throws -> UIImage {
        // 1. 이미 끝났거나 진행 중이면 그것을 재사용
        if let entry = cache[url] {
            switch entry {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value
            }
        }

        // 2. 없으면 Task를 만들어 '먼저' 등록 — 중단 전에 등록하는 것이 포인트
        let task = Task { try await download(url) }
        cache[url] = .inProgress(task)

        do {
            let image = try await task.value
            cache[url] = .ready(image)
            return image
        } catch {
            cache[url] = nil                 // 실패 시 캐시 정리
            throw error
        }
    }
}
```

정리하면 이렇다.

- **`await` 전에 상태를 확정**해 놓는다 (여기서는 `.inProgress` 등록).
- `await` 이후에는 **이전에 읽은 값이 유효하다고 가정하지 않는다.**
- 불변식(invariant)이 깨진 상태로 `await`를 만나지 않게 한다.

---

## 6. Sendable — 경계를 넘는 값

actor 경계를 넘나드는 인자와 반환값은 **`Sendable`(동시성 안전)** 해야 한다. 참조 타입을 그대로 넘기면 actor 밖에서 같은 객체를 건드릴 수 있어 격리가 무의미해지기 때문이다.

```swift
final class MutableBox {          // Sendable 아님
    var value = 0
}

actor Store {
    func save(_ box: MutableBox) { }
    // ⚠️ Non-sendable type 'MutableBox' ... 경고/에러
}
```

| 타입 | Sendable 여부 |
| :--- | :--- |
| `Int`, `String`, `struct`(구성 요소가 모두 Sendable) | ⭕️ 자동 |
| `enum`(연관값이 모두 Sendable) | ⭕️ 자동 |
| `actor` | ⭕️ 자동 (격리로 보장) |
| `final class` + 모든 프로퍼티 `let` + Sendable | ⭕️ 명시 시 인정 |
| 가변 상태를 가진 `class` | ❌ |

Swift 6 언어 모드에서는 이 검사가 **경고가 아니라 에러**로 승격된다.

---

## 7. Global Actor / MainActor

`@globalActor`는 **프로세스 전역에 하나만 존재하는 actor**를 선언한다. 대표적인 것이 UI 작업을 메인 스레드에 묶는 `@MainActor`다.

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published var nickname: String = ""

    func load(id: String) async {
        // 여기는 항상 메인 스레드
        let user = await repository.fetch(id: id)   // 백그라운드에서 수행
        nickname = user.nickname                    // 다시 메인 스레드로 복귀
    }
}
```

`DispatchQueue.main.async { }`를 직접 쓰던 코드를 **타입 선언 한 줄로 대체**하고, 컴파일러가 검사하게 만드는 방식이다.

직접 정의할 수도 있다.

```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}

@DatabaseActor
func migrate() { /* 모든 DB 작업을 한 곳에 직렬화 */ }
```

---

## 8. actor vs class vs struct

| | `struct` | `class` | `actor` |
| :--- | :--- | :--- | :--- |
| 타입 | 값 타입 | 참조 타입 | 참조 타입 |
| 상속 | ❌ | ⭕️ | ❌ |
| 데이터 경쟁 방지 | 복사로 회피 | 직접 lock 필요 | 컴파일러가 강제 |
| 외부 접근 | 동기 | 동기 | `await` (비동기) |
| Sendable | 조건부 자동 | 수동 | 자동 |

**선택 기준**

- 상태를 공유할 필요가 없다 → `struct` (가장 우선)
- 공유되는 가변 상태가 있고 여러 작업이 동시에 접근한다 → `actor`
- 단일 스레드에서만 쓰거나 상속이 필요하다 → `class`

---

## 9. 사용 시 주의점

1. **모든 것을 actor로 만들지 않는다.** 호출부마다 `await`가 붙고 suspension 비용이 생긴다. 값 타입으로 해결되면 그쪽이 낫다.
2. **actor는 스레드가 아니다.** 특정 스레드를 점유하지 않고, 실행 스레드는 매번 달라질 수 있다. 스레드에 종속적인 API(일부 레거시 C/Objective-C 라이브러리)에는 부적합하다.
3. **`await` 지점마다 상태 재확인.** 5절의 재진입 문제가 실무에서 가장 자주 겪는 버그다.
4. **actor 안에서 오래 걸리는 동기 작업 금지.** 그동안 다른 모든 호출이 대기한다. 무거운 계산은 `nonisolated`로 빼거나 밖에서 처리한다.
5. **교착 상태는 발생하지 않는다.** actor 호출은 blocking이 아니라 suspension이므로 lock처럼 데드락에 빠지지는 않는다. 다만 순환 대기로 인한 성능 저하는 있을 수 있다.

---

## References

- [SE-0306: Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [SE-0316: Global actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [Protect mutable state with Swift actors - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [The Swift Programming Language - Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [[WWDC] Protect mutable state with Swift actors (WWDC21)](<[WWDC] Protect mutable state with Swift actors (WWDC21).md>)
