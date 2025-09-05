# [JAVA] chained assignment (연쇄할당)

### Chained assignment (연쇄 할당)

특정 언어 들에서는 위와 같이 하나의 값을 여러 변수에 할당하는 동작을 한줄로 작성할 수 있도록 지원해주기도 한다. 결합 순서는 **오른쪽에서 왼쪽**이다.

```java
int a, b, c, d, f;
a = b = c = d = f = 0;
```

위 예시는 a, b, c, d, f에 대한 변수에 0이라는 값을 할당한 경우이다.

### 사용 예시

```java
public class Main {
    public static void main(String[] args) {
        int[] src = {1, 2, 3, 4, 5};
        int[] dst = new int[5];

        // 연쇄 할당을 이용해 복사
        dst[0] = dst[1] = dst[2] = dst[3] = dst[4] = src[2];

        for (int value : dst) {
            System.out.println(value);  // 모두 3 출력
        }
    }
}
```

결합 순서가 오른쪽에서 왼쪽이기에 src[2]의 값으로 모든 값이 바뀐다.

```java
public class Main {
    static int f() {
        System.out.println("f() called");
        return 10;
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4, 5};
        int i = 2;

        i = arr[i] = f();

        System.out.println("i = " + i);        // 10
        System.out.println("arr[2] = " + arr[2]); // 10
    }
}
```

함수의 결과 값을 통해서도 연쇄 할당이 가능