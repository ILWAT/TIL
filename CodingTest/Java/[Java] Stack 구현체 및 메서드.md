# [Java] Stack 구현체 및 메서드

Java Stack 사용시 필수 숙지 사항들

### Stack 클래스

```java
import java.util.Stack;
```

| Modifier and Type | Method | Description |
| --- | --- | --- |
| boolean | empty() | 스택이 비어있는지 확인 |
| T | peek() | 스택 최상단 객체 확인(pop 되지 않음) |
| T | pop() | 스택 최상단 객체 pop(삭제 및 리턴) |
| T | push(T item) | 파라미터로 넘겨진 item Stack push |

### ArrayDeque

오라클은 Stack을 Deque 인터페이스와 그 구현체(대표적으로 ArrayDeque)를 추천한다.

```java
import java.util.ArrayDeque;
```

| Modifier and Type | Method | Description |
| --- | --- | --- |
| boolean | add(T t) | deque의 마지막 원소로 삽입 |
| void | addLast(T t) | deque의 마지막 원소로 삽입 |
| void | push(T t) | duque의 Stack push |
| T | peekLast() | 원소 삭제하지 않고 마지막 원소 확인 Empty시 null 리턴 |
| T | removeLast() | qeue의 마지막 원소 삭제및 리턴 |
| T | pop() | duque의 Stack pop |
| int | size() | deque 내의 원소 갯수 반환 |

필자 성향상 Last만 표기했지만 First로 해도 무방하다.(addFirst(), removeFirst(), peekFirst())

### References

[Stack (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/Stack.html)