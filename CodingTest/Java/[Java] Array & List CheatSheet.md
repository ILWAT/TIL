# Java 코딩 테스트 필수 메서드 및 개념 정리

자바 코딩 테스트에서 `Array`, `List`, `ArrayList`는 거의 모든 문제의 기본 뼈대가 됩니다. 이 세 가지를 다룰 때 **필수로 알아야 하는 핵심 메서드와 팁**을 상황별로 나누어 보기 쉽게 정리해 드립니다.

---

## 1. Array (배열) 기본 & `Arrays` 클래스

자바 기본 배열(`int[]`, `String[]` 등)은 크기가 고정되어 있으며, 주로 `java.util.Arrays` 클래스의 유틸리티 메서드와 함께 사용됩니다.

| 메서드 | 설명 | 예시 |
| :--- | :--- | :--- |
| **`arr.length`** | 배열의 **길이**(변수) | `int len = arr.length;` |
| **`Arrays.sort(arr)`** | 배열을 **오름차순 정렬** (원본 변경) | `Arrays.sort(arr);` |
| **`Arrays.copyOfRange(arr, from, to)`** | 배열의 특정 범위를 자른 **새 배열 생성** (to는 제외) | `int[] sub = Arrays.copyOfRange(arr, 0, 3);` |
| **`Arrays.fill(arr, val)`** | 배열의 모든 요소를 **특정 값으로 초기화** (DP 등에서 유용) | `Arrays.fill(arr, -1);` |
| **`Arrays.asList(arr)`** | 배열을 고정 크기의 **List로 변환** | `List<String> list = Arrays.asList(strArr);` |

> ⚠️ **주의**: `Arrays.sort()`에 별도의 정렬 기준(Comparator)을 넣고 싶다면, `int[]` 같은 기본형 배열이 아닌 `Integer[]` 같은 **참조형 객체 배열**이어야 합니다.

---

## 2. List & ArrayList

크기가 가변적인 데이터를 다룰 때 가장 많이 씁니다. 코테에서는 대다수 `List<T> list = new ArrayList<>();` 형태로 선언해 사용합니다.

| 메서드 | 설명 | 예시 |
| :--- | :--- | :--- |
| **`list.size()`** | 리스트의 요소 **개수** 반환 | `int size = list.size();` |
| **`list.add(value)`** | 리스트 끝에 **요소 추가** | `list.add(10);` |
| **`list.add(index, value)`** | 특정 인덱스에 **요소 삽입** (뒤 요소들은 밀림) | `list.add(0, 5);` |
| **`list.get(index)`** | 특정 인덱스의 **값 가져오기** | `int val = list.get(2);` |
| **`list.set(index, value)`** | 특정 인덱스의 **값 변경** | `list.set(1, 20);` |
| **`list.remove(index)`** | 특정 인덱스의 **요소 삭제** (삭제된 값 반환) | `list.remove(0);` |
| **`list.indexOf(value)`** | 특정 값이 위치한 **인덱스 찾기** (없으면 `-1`) | `int idx = list.indexOf("target");` |
| **`list.contains(value)`** | 특정 값이 리스트에 **포함되어 있는지 확인** (boolean) | `if (list.contains(5)) { ... }` |
| **`list.clear()`** | 리스트의 **모든 요소 삭제** | `list.clear();` |

---

## 3. Collections 클래스 (List 정렬 및 가공)

`List` 계열의 컬렉션을 다룰 때는 `java.util.Collections` 클래스를 필수적으로 사용해야 합니다.

- **오름차순 정렬:** `Collections.sort(list);`
- **내림차순 정렬:** `Collections.sort(list, Collections.reverseOrder());`
- **순서 뒤집기 (뒤죽박죽으로 만들기 아님):** `Collections.reverse(list);`
- **최댓값 / 최솟값 찾기:**

```java
int max = Collections.max(list);
int min = Collections.min(list);
```

---

## 4. 💡 코테 단골 변환 패턴 (형변환)

자바 코테 문제를 풀다 보면 배열을 리스트로, 리스트를 배열로 바꿔야 하는 경우가 정말 많습니다. 아래 패턴은 외워두시는 것이 좋습니다.

### ① List를 2차원 배열로 바꾸기 (이번 문제 구조)

```java
ArrayList<int[]> al = new ArrayList<>();
// ... al에 데이터 추가 ...
int[][] answer = al.toArray(new int[al.size()][]);
```

### ② `int[]` 배열을 `List<Integer>`로 변환 (Stream 활용)

```java
int[] arr = {1, 2, 3};
List<Integer> list = Arrays.stream(arr)
                           .boxed()
                           .collect(Collectors.toList());
```

### ③ `List<Integer>`를 `int[]`로 변환 (Stream 활용)

```java
List<Integer> list = new ArrayList<>();
// ... list에 데이터 추가 ...
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();
```

---

## 🚀 코테용 꿀팁 요약

1. 크기가 고정되어 있고 인덱스로 빠른 접근이 필요하다 ➡️ `Array`
2. 데이터 개수가 계속 변하고, 중간 삽입/삭제나 탐색 메서드가 필요하다 ➡️ `ArrayList`
3. 람다식을 쓸 때 리스트 안의 변수나 인덱스를 사용하려면 외부 변수가 고정값(`final` 혹은 변하지 않는 상태)이어야 한다는 점을 항상 인지하기!