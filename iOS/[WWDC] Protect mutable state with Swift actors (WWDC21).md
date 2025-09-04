# [WWDC] Protect mutable state with Swift actors (WWDC21)

*Italic체는 필자가 덧붙인 설명*

동시성 프로그래밍을 하는 데에 있어 가장 어려운 문제중 하나가 Data race(경쟁 상태)를 피하는 것.

### Data race의 발생 상황

- 다른 2개의 스레드가 동시에 같은 데이터에 접근한 상황
- 그중 하나라도 데이터를 변경하려고 할 때 발생함

```swift
class Counter {
    var value = 0
    
    func increment() -> Int {
        value = value + 1
        return value
    }
}
```

```swift
let counter = Counter()
Task.detached {
    print(counter.increment())
}

Task.detached {
    print(counter.increment())
}
```

위 예제로 우리는 counter가 일관된 상태로 1, 2 또는 2, 1의 값이 나오길 예상하지만 Data race가 발생할 경우 2, 2(Task가 value를 1로 읽을 경우) 또는 1, 1(Task가 value를 0으로 읽을 경우)이 나올 수도 있다.

이처럼 Data race는 발생하지 않도록 만드는 것도, 디버깅 하기도 매우 까다로운 요소이다.

- 경쟁상태를 유발하는 접근이 프로그램의 서로 다른 부분들에서 발생할 수 있기 때문에 비국소적 추론(nonlocal reasoning)이 필요
    - *프로그램이 실행되다 보면  어느 상황, 어느 시점에 Data race가 발생하는 지 예측하기 어려움*
- 운영체제의 스케쥴러가 매 실행마다 동시 태스크들을 다른 방식으로 교차 실행 하기 때문에 실행 순서가 보장되지 않음.

Data race는 공유 가능하면서도 mutable(변경 가능)한 점 때문에 일어난다.

이를 해결 하기 위한 방법

1. Value (Semantics) Type 사용
    
    Value Type을 사용하면 모든 변경 사항은 local화 된다.
    
    ```swift
    var array1 = [1, 2]
    var array2 = array1
    
    array1.append(3)
    array2.append(4)
    
    print(array1) // [1, 2, 3]
    print(array2) // [1, 2, 4]
    ```
    
    Value Type에서 let은 아예 변경이 불가하기에 동시성 작업에서 매우 안정적이다.
    

### References

[Protect mutable state with Swift actors - WWDC21 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10133/?time=167)