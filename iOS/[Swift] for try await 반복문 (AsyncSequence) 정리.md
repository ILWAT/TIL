# [Swift] for try await 반복문 (AsyncSequence) 정리

아래 코드를 이해하는 것이 목표다.

```swift
for try await chunk in stream {
    try Task.checkCancellation()
    await send(.photoBoothChunkLoaded(chunk, generation: generation))
}
```

한 줄로 요약하면 이렇다.

> `stream`에서 값이 **하나씩 도착할 때마다** 루프 본문이 실행되고, 값을 기다리는 동안에는 **스레드를 점유하지 않고 중단(suspend)** 된다.

일반 `for in`이 "이미 있는 배열을 순회"하는 것이라면, `for try await`는 **"아직 오지 않은 값을 기다리며 순회"** 하는 것이다.

---

## 1. 먼저 한 줄씩 뜯어보기

| 키워드 | 의미 |
| :--- | :--- |
| `for ... in stream` | `stream`은 `AsyncSequence`를 채택한 타입 |
| `await` | 다음 값이 올 때까지 **기다린다**(중단점) |
| `try` | 다음 값을 가져오는 도중 **에러가 날 수 있다** |
| `chunk` | 도착한 값 하나 (`AsyncSequence.Element`) |

즉 `for try await`는 세 개의 서로 다른 문법이 합쳐진 것이 아니라, **"에러를 던질 수 있는 비동기 시퀀스를 순회한다"** 는 하나의 선언이다.

`try`와 `await`의 **순서는 고정**이다.

```swift
for try await chunk in stream { }   // ⭕️
for await try chunk in stream { }   // ❌ 컴파일 에러
```

---

## 2. AsyncSequence — 무엇을 순회하는가

`Sequence`와 `AsyncSequence`를 나란히 놓으면 차이가 명확하다.

```swift
protocol Sequence {
    associatedtype Element
    func makeIterator() -> Iterator
}

protocol IteratorProtocol {
    mutating func next() -> Element?          // 즉시 반환
}
```

```swift
protocol AsyncSequence {
    associatedtype Element
    func makeAsyncIterator() -> AsyncIterator
}

protocol AsyncIteratorProtocol {
    mutating func next() async throws -> Element?   // 기다렸다 반환, 실패 가능
}
```

**차이는 `next()` 하나뿐이다.**

- `async` → 값이 준비될 때까지 기다릴 수 있다.
- `throws` → 값을 만드는 도중 실패할 수 있다.
- `Element?` → `nil`을 반환하면 **시퀀스 종료**.

`for try await`의 `await`와 `try`는 결국 **이 `next()`에 붙는 것**이다.

---

## 3. 컴파일러가 실제로 만드는 코드

`for try await` 루프는 아래 `while` 루프로 풀어쓸 수 있다. 이 형태를 기억해 두면 대부분의 의문이 해결된다.

```swift
// for try await chunk in stream { body }
// 위 코드는 아래와 (거의) 같다

var iterator = stream.makeAsyncIterator()
while let chunk = try await iterator.next() {   // ← try, await가 여기 붙는다
    body
}
// next()가 nil을 반환하면 루프 정상 종료
// next()가 throw하면 루프 밖으로 에러 전파
```

여기서 읽어내야 할 세 가지.

1. **중단점은 `next()` 한 곳**이다. 루프를 도는 매 회차마다 스레드를 반납하고 값을 기다린다.
2. **루프 본문은 순차적으로 실행된다.** 본문이 끝나야 다음 값을 요청한다. (→ 8절)
3. **에러는 루프를 탈출시킨다.** `continue`처럼 다음 값으로 넘어가지 않고 루프 자체가 끝난다.

---

## 4. for await vs for try await

`try` 유무는 취향이 아니라 **시퀀스 타입이 결정**한다.

```swift
// AsyncStream — 실패하지 않는 스트림
let stream = AsyncStream<Int> { continuation in
    continuation.yield(1)
    continuation.finish()
}

for await value in stream { }        // ⭕️ try 불필요
for try await value in stream { }    // ⚠️ 경고: no calls to throwing functions
```

```swift
// AsyncThrowingStream — 실패할 수 있는 스트림
let stream = AsyncThrowingStream<Int, Error> { continuation in
    continuation.yield(1)
    continuation.finish(throwing: NetworkError.timeout)
}

for try await value in stream { }    // ⭕️
for await value in stream { }        // ❌ 에러: call can throw but is not marked with 'try'
```

정리하면,

| 시퀀스 | 순회 문법 |
| :--- | :--- |
| `AsyncStream` | `for await` |
| `AsyncThrowingStream` | `for try await` |
| `URLSession.bytes`, `URL.lines` | `for try await` |
| `NotificationCenter.notifications` | `for await` |

> 맨 처음 코드가 `for try await`인 것은 **`stream`이 실패할 수 있는 시퀀스**(대개 `AsyncThrowingStream`)라는 뜻이다.

---

## 5. 에러 처리 — 어디서 잡히는가

`for try await`가 던지는 에러는 **두 군데**에서 나온다.

```swift
do {
    for try await chunk in stream {   // ① next()가 던지는 에러 (스트림 자체의 실패)
        try process(chunk)            // ② 본문에서 던지는 에러
    }
} catch {
    // ①, ② 모두 여기로 온다
}
```

둘 다 **루프를 즉시 종료**시킨다는 점이 중요하다. "한 청크가 실패해도 다음 청크는 계속 받고 싶다"면 본문에서 개별로 감싸야 한다.

```swift
for try await chunk in stream {
    do {
        try process(chunk)
    } catch {
        logger.error("청크 처리 실패: \(error)")   // 루프는 계속
    }
}
```

`break` / `return`으로 빠져나가는 것도 정상 종료다. 이때 이터레이터는 해제되고, `AsyncStream`이라면 `onTermination`이 호출된다.

---

## 6. 취소(Cancellation)와 `try Task.checkCancellation()`

처음 코드에서 가장 궁금한 줄이 이것이다.

```swift
for try await chunk in stream {
    try Task.checkCancellation()   // 왜 필요한가?
    await send(...)
}
```

### Swift의 취소는 "협조적(cooperative)"이다

`Task.cancel()`은 실행을 **강제로 멈추지 않는다.** 단지 `isCancelled` 플래그를 켤 뿐이고, **확인하는 쪽이 스스로 멈춰야** 한다.

| API | 동작 |
| :--- | :--- |
| `Task.isCancelled` | `Bool` 반환. 직접 분기 처리 |
| `try Task.checkCancellation()` | 취소되었으면 `CancellationError`를 **throw** |
| `try await Task.sleep(...)` | 취소되면 자동으로 throw |

### 그래서 왜 루프 안에 넣는가

`AsyncStream` / `AsyncThrowingStream`은 취소를 감지하면 스스로 스트림을 종료한다. 하지만,

1. **모든 `AsyncSequence`가 그렇지는 않다.** 직접 구현한 시퀀스나 서드파티 시퀀스는 취소를 확인하지 않을 수 있다.
2. **루프 본문이 무거울 수 있다.** 값은 이미 도착해 있고 본문을 처리하는 중이라면, 중단점이 없어 취소를 알아채지 못한다.
3. **취소 이후의 `send`를 막는다.** 취소된 화면에 상태를 밀어 넣는 것을 방지한다.

즉 이 한 줄은 **"매 청크마다 이 작업이 아직 유효한지 확인하고, 아니면 루프 밖으로 던져 나간다"** 는 안전장치다.

```swift
// checkCancellation()이 하는 일
if Task.isCancelled { throw CancellationError() }
```

### TCA에서의 의미

TCA의 `.run` 이펙트는 내부에서 `CancellationError`를 **조용히 무시**하도록 되어 있다. 그래서 취소로 인해 throw가 발생해도 에러 액션이 전달되거나 런타임 경고가 뜨지 않고, 이펙트만 깔끔하게 끝난다.

```swift
case .startStreaming:
    return .run { send in
        for try await chunk in stream {
            try Task.checkCancellation()
            await send(.photoBoothChunkLoaded(chunk, generation: generation))
        }
    }
    .cancellable(id: CancelID.photoBooth, cancelInFlight: true)
    // cancelInFlight: 새 요청이 오면 이전 루프의 Task를 취소 → checkCancellation()이 걸림
```

---

## 7. `generation` 파라미터 — 뒤늦게 도착한 값 걸러내기

```swift
await send(.photoBoothChunkLoaded(chunk, generation: generation))
```

취소는 즉시 반영되지 않는다. 취소 신호와 실제 루프 종료 사이에 **이미 진행 중이던 `send`가 한두 개 빠져나갈 수 있다.** 그래서 요청마다 번호를 매겨 리듀서에서 걸러낸다.

```swift
case let .photoBoothChunkLoaded(chunk, generation):
    guard generation == state.currentGeneration else { return .none }  // 철 지난 응답 폐기
    state.chunks.append(chunk)
    return .none
```

`checkCancellation()`이 **생산 측 방어**라면, `generation` 비교는 **소비 측 방어**다. 두 겹으로 막는 구조.

---

## 8. ⚠️ 루프는 순차적이다 — 동시 처리가 아니다

가장 많이 오해하는 지점이다.

```swift
for try await url in urlStream {
    let image = try await download(url)   // 이게 끝나야 다음 url을 받는다
}
```

`for try await`는 **동시성(concurrency)을 만들지 않는다.** 본문이 완료되어야 `next()`를 다시 호출한다. 본문이 느리면 그동안 생산자의 값은 버퍼에 쌓이거나(버퍼링 정책에 따라) 버려진다.

동시에 처리하려면 `TaskGroup`으로 넘겨야 한다.

```swift
try await withThrowingTaskGroup(of: UIImage.self) { group in
    for try await url in urlStream {
        group.addTask { try await download(url) }   // 던져만 놓고 즉시 다음 url로
    }

    for try await image in group {                  // TaskGroup도 AsyncSequence다
        images.append(image)
    }
}
```

> `TaskGroup` 자체가 `AsyncSequence`이므로 `for try await`로 결과를 수확한다. 결과 **순서는 완료 순**이라는 점에 주의.

---

## 9. 스트림 만들기 — 콜백을 AsyncSequence로 바꾸기

기존 delegate/completion 기반 API를 `for try await`로 소비 가능하게 감싸는 것이 실무에서 가장 흔한 용례다.

```swift
func photoBoothStream(id: String) -> AsyncThrowingStream<Chunk, Error> {
    AsyncThrowingStream(bufferingPolicy: .bufferingNewest(10)) { continuation in

        let task = api.startStreaming(id: id) { result in
            switch result {
            case .success(let chunk):
                continuation.yield(chunk)            // 값 방출
            case .failure(let error):
                continuation.finish(throwing: error) // 에러로 종료
            }
        }

        continuation.onTermination = { termination in
            // 소비자가 루프를 break 하거나 Task가 취소되면 호출된다
            task.cancel()                            // ← 리소스 정리는 여기서
        }
    }
}
```

**버퍼링 정책**은 본문이 느릴 때의 동작을 결정한다.

| 정책 | 동작 |
| :--- | :--- |
| `.unbounded` (기본값) | 무제한 버퍼링. 소비가 느리면 **메모리 증가** |
| `.bufferingNewest(n)` | 최신 n개 유지, 오래된 값 폐기 (UI 상태 갱신에 적합) |
| `.bufferingOldest(n)` | 먼저 온 n개 유지, 이후 값 폐기 |

`onTermination`을 비워두면 **루프를 빠져나와도 원본 작업이 계속 도는** 누수가 생긴다. 반드시 정리 코드를 넣는다.

---

## 10. 자주 쓰는 오퍼레이터

`AsyncSequence`도 `map` / `filter` 등을 지원하며, 결과는 **또 다른 `AsyncSequence`** 다(지연 평가).

```swift
let bigChunks = stream
    .filter { $0.size > 1024 }
    .map { $0.data }

for try await data in bigChunks { }
```

| 오퍼레이터 | 설명 |
| :--- | :--- |
| `map` / `compactMap` / `filter` | 변환·필터 |
| `prefix(n)` | 앞 n개만 받고 종료 |
| `dropFirst(n)` | 앞 n개 건너뛰기 |
| `first(where:)` | 조건 만족하는 첫 값 (`await` 필요) |
| `reduce` | 전체를 접어 하나의 값으로 |

배열로 한 번에 모으고 싶다면,

```swift
var all: [Chunk] = []
for try await chunk in stream { all.append(chunk) }
```

> `Array(stream)` 같은 초기화는 없다. 표준 라이브러리에는 `AsyncSequence → Array` 변환이 없으므로 직접 모으거나 [swift-async-algorithms](https://github.com/apple/swift-async-algorithms)의 `reduce` 등을 쓴다.

---

## 11. 실무 주의점

1. **무한 스트림은 스스로 끝나지 않는다.** `NotificationCenter.notifications` 같은 시퀀스는 `nil`을 반환하지 않으므로, `break` 또는 Task 취소가 유일한 탈출구다.
2. **루프 뒤의 코드는 스트림이 끝나야 실행된다.** 무한 스트림 뒤에 정리 코드를 두면 영원히 실행되지 않는다.
3. **`for await`를 `@MainActor` 컨텍스트에서 돌려도 메인 스레드를 막지 않는다.** 값을 기다리는 동안 중단될 뿐이다. 반대로 본문에 무거운 동기 연산을 넣으면 그대로 메인 스레드가 멈춘다.
4. **하나의 `AsyncStream`은 하나의 소비자만** 갖는다고 생각하는 편이 안전하다. 여러 곳에서 순회하면 값이 나눠 들어간다. 브로드캐스트가 필요하면 `AsyncChannel`(swift-async-algorithms)이나 별도 멀티캐스트 구현이 필요하다.
5. **Swift 6의 타입 지정 에러(typed throws).** `AsyncSequence`가 `Failure` 연관 타입을 갖게 되어, `for try await`에서 잡히는 에러 타입이 구체화된다. `AsyncStream`처럼 `Failure == Never`인 시퀀스에 `try`를 붙이면 경고가 뜬다.

---

## 12. 다시 처음 코드로

```swift
for try await chunk in stream {
    try Task.checkCancellation()
    await send(.photoBoothChunkLoaded(chunk, generation: generation))
}
```

이제 이렇게 읽힌다.

1. `stream`(실패 가능한 비동기 시퀀스)에서 **다음 청크가 도착할 때까지 중단하며 대기**한다.
2. 대기 중 스트림이 실패하면 `try`가 에러를 루프 밖으로 전파한다.
3. 청크가 도착하면, **먼저 이 작업이 아직 취소되지 않았는지 확인**한다. 취소되었으면 `CancellationError`를 던져 루프를 빠져나가고, TCA는 이를 조용히 무시한다.
4. 유효하다면 `generation`(요청 세대 번호)을 함께 실어 액션을 보낸다. 리듀서는 철 지난 세대의 청크를 폐기한다.
5. `send`가 끝나야 **다음 청크를 요청**한다. (순차 처리)
6. 스트림이 `finish()`되면 `next()`가 `nil`을 반환하고 루프가 정상 종료된다.

---

## References

- [SE-0298: Async/Await: Sequences](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0298-asyncsequence.md)
- [SE-0314: AsyncStream and AsyncThrowingStream](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)
- [SE-0421: Generalize effect polymorphism for AsyncSequence](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0421-generalize-async-sequence.md)
- [Apple Developer - AsyncSequence](https://developer.apple.com/documentation/swift/asyncsequence)
- [Meet AsyncSequence - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10058/)
- [[Swift] Actor 개념과 예시 정리](<[Swift] Actor 개념과 예시 정리.md>)
- [[TCA] Reducer 개념 정리](<[TCA] Reducer 개념 정리.md>)
