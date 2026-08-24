# [Swift] Hasher 정리

`Hasher`는 Swift 표준 라이브러리가 제공하는 **범용 해시 함수 구현체**다. `Set`, `Dictionary`가 값을 어느 버킷에 넣을지 정할 때 내부적으로 이것을 사용한다.

한 줄로 요약하면 이렇다.

> 값을 `Hasher`에 **밀어 넣으면(combine)**, `Hasher`가 그것들을 섞어 하나의 `Int` 해시값을 만들어 준다.

---

## 1. Hashable과의 관계

`Hashable`은 두 가지 요구사항을 갖는다.

```swift
protocol Hashable: Equatable {
    func hash(into hasher: inout Hasher)
    var hashValue: Int { get }   // Swift 4.2부터 직접 구현하지 않음
}
```

직접 구현할 것은 `hash(into:)` 하나뿐이다. `hashValue`는 기본 구현이 `hash(into:)`를 대신 호출해 준다.

```swift
struct User {
    let id: String
    let nickname: String
    var lastLoginedAt: Date
}

extension User: Hashable {
    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)       // == 에서 쓴 프로퍼티만 넣는다
    }
}
```

---

## 2. 자동 합성 (Synthesized Conformance)

저장 프로퍼티(또는 연관값)가 **전부 `Hashable`이면** 컴파일러가 `hash(into:)`를 자동으로 만들어 준다.

```swift
struct Point: Hashable {     // 이것만으로 끝
    let x: Int
    let y: Int
}

enum Status: Hashable {
    case idle
    case loading(progress: Double)
}
```

합성된 구현은 **모든 저장 프로퍼티를 순서대로 `combine`** 한다. 그래서 1절처럼 `==`가 일부 프로퍼티만 비교한다면 자동 합성을 쓰면 안 된다.

---

## 3. 직접 사용하기 — combine과 finalize

`Hashable` 구현이 아니라 해시값 자체가 필요할 때는 직접 만들어 쓸 수 있다.

```swift
var hasher = Hasher()
hasher.combine("Moon")
hasher.combine(2026)
let value = hasher.finalize()   // Int
```

| 메서드 | 설명 |
| :--- | :--- |
| `init()` | 프로세스별 랜덤 시드로 초기화 |
| `combine(_:)` | `Hashable` 값을 해시 상태에 반영 |
| `combine(bytes:)` | raw 바이트를 직접 반영 |
| `finalize()` | 최종 `Int` 반환 (**consuming** — 이후 재사용 불가) |

`hash(into:)` 안에서는 **`finalize()`를 호출하지 않는다.** 넘겨받은 `hasher`는 상위 호출자의 것이라 컴파일 에러가 난다.

```swift
func hash(into hasher: inout Hasher) {
    hasher.combine(id)
    _ = hasher.finalize()   // ❌ inout 파라미터를 consume 할 수 없음
}
```

---

## 4. ⚠️ 해시값은 실행마다 달라진다

`Hasher`는 **SipHash-1-3** 기반이며, 시드가 **프로세스 실행마다 랜덤하게 생성**된다. 해시 충돌을 노린 DoS 공격을 막기 위한 설계다.

```swift
// 실행 1: -3287524981223441291
// 실행 2:  8391238471023948123
print("hello".hashValue)
```

여기서 나오는 실무 규칙은 하나다.

> **해시값을 저장하거나 전송하지 않는다.**

UserDefaults·DB·서버 전송·캐시 키 파일명 등에 `hashValue`를 쓰면 앱을 재실행하는 순간 값이 달라져 조회에 실패한다. 영속적인 식별자가 필요하면 `SHA256`(CryptoKit) 같은 **결정적(deterministic) 해시**를 써야 한다.

디버깅 목적으로 고정하고 싶다면 스킴의 환경 변수에 `SWIFT_DETERMINISTIC_HASHING=1`을 넣으면 된다. 테스트 재현용이며 프로덕션에서 의존할 값은 아니다.

---

## 5. 지켜야 하는 계약 (Hashable Contract)

```
a == b  이면  반드시  a.hashValue == b.hashValue
```

역은 성립하지 않는다. **해시가 같다고 값이 같은 것은 아니다**(충돌). 그래서 `Set`/`Dictionary`는 해시로 후보를 좁힌 뒤 최종 판정은 `==`로 한다.

이 계약이 깨지면 크래시 없이 **조용히 동작이 이상해진다.**

```swift
struct Broken: Hashable {
    let id: String
    var count: Int

    static func == (lhs: Broken, rhs: Broken) -> Bool {
        lhs.id == rhs.id            // id만 비교
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(count)       // ⚠️ ==에 없는 프로퍼티를 넣음
    }
}

var set: Set<Broken> = [Broken(id: "A", count: 0)]
print(set.contains(Broken(id: "A", count: 5)))   // false — 있는데도 못 찾음
```

정리하면 **`==`에서 쓰는 프로퍼티의 집합 ⊇ `hash(into:)`에 넣는 프로퍼티의 집합**이어야 한다.

같은 이유로, `Set`이나 `Dictionary` 키로 넣은 뒤 해시에 참여하는 프로퍼티를 변경하면 안 된다. 클래스처럼 참조 타입을 키로 쓸 때 특히 주의해야 한다.

---

## 6. 알아 두면 좋은 것들

- **combine 순서가 결과에 영향을 준다.** `combine(a); combine(b)`와 `combine(b); combine(a)`는 다른 값이다. 순서가 의미 없는 데이터(집합 등)라면 정렬 후 넣거나 XOR 같은 교환법칙이 성립하는 방식을 써야 한다.
- **모든 프로퍼티를 넣을 필요는 없다.** 배열처럼 큰 값은 `count`와 앞 몇 개 원소만 넣는 식으로 비용을 줄일 수 있다. 다만 구별력이 떨어지면 충돌이 늘어 조회가 O(n)에 가까워진다.
- **품질이 균일하다.** 옛날 방식처럼 `x.hashValue ^ y.hashValue`로 직접 조합하면 `(1,2)`와 `(2,1)`이 같은 값이 되는 등 충돌이 쉽게 생긴다. `Hasher`에 맡기는 편이 안전하다.
- **`actor`에서 채택할 때는 `nonisolated`가 필요하다.** `Hashable`은 동기 프로토콜이기 때문이다. → [[Swift] Actor 개념과 예시 정리](<[Swift] Actor 개념과 예시 정리.md>)

---

## References

- [Apple Developer - Hasher](https://developer.apple.com/documentation/swift/hasher)
- [Apple Developer - Hashable](https://developer.apple.com/documentation/swift/hashable)
- [SE-0206: Hashable Enhancements](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0206-hashable-enhancements.md)
