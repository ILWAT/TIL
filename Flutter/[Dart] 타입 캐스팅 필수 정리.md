# [Dart] 타입 캐스팅 필수 정리

Dart는 **정적 타입 + 런타임 타입 검사**를 모두 하는 언어다.
컴파일러가 통과시켜도 런타임에 `type 'X' is not a subtype of type 'Y' in type cast`로 죽는 경우가 많기 때문에,
캐스팅 연산자별 동작과 실패 시점을 정확히 알아둬야 한다.

---

## 1. 핵심 연산자 4가지

| 연산자 | 하는 일 | 실패하면 |
|---|---|---|
| `as` | 타입 캐스팅 | `TypeError` **throw** |
| `is` / `is!` | 타입 검사 + 타입 승격(promotion) | `false` 반환 (안전) |
| `!` | null 아님 단언 | `TypeError` **throw** |
| `?.` / `??` | null 안전 접근 / 기본값 | 안전 |

```dart
Object value = 'hello';

final a = value as String;      // OK
final b = value as int;         // 런타임 throw
final c = value is String;      // true (절대 throw 안 함)
```

> **Dart에는 Swift의 `as?`(옵셔널 캐스팅)가 없다.**
> "실패하면 null" 이 필요하면 직접 만들어야 한다. → [3번 참고](#3-실패해도-안전한-캐스팅-패턴)

---

## 2. `is`의 타입 승격 (Type Promotion)

`is`로 검사하면 그 블록 안에서는 **캐스팅 없이** 해당 타입으로 쓸 수 있다.

```dart
void printLength(Object obj) {
  if (obj is String) {
    print(obj.length);   // obj가 String으로 승격됨. (obj as String) 불필요
  }
}
```

### 승격이 안 되는 경우 (자주 걸리는 함정)

타입 승격은 **지역 변수**와 **final 필드**에서만 동작한다.

```dart
class Foo {
  String? name;          // non-final 필드

  void bar() {
    if (name != null) {
      print(name.length);   // 컴파일 에러! 승격 안 됨
    }
  }

  void baz() {
    final n = name;         // 지역 변수로 복사
    if (n != null) {
      print(n.length);      // OK
    }
  }
}
```

- 다른 객체의 필드(`other.name`), `var` 필드, getter는 승격되지 않는다.
- 해결책은 **지역 변수로 한 번 받는 것**이다.

---

## 3. 실패해도 안전한 캐스팅 패턴

### 3-1. 확장(extension)으로 `asOrNull` 만들기

```dart
extension SafeCast on Object? {
  T? asOrNull<T>() => this is T ? this as T : null;
}

final name = json['name'].asOrNull<String>() ?? 'unknown';
```

### 3-2. Dart 3 패턴 매칭 (권장)

```dart
// if-case
if (value case String s) {
  print(s.toUpperCase());
}

// switch
final result = switch (value) {
  int i    => 'int: $i',
  String s => 'string: $s',
  null     => 'null',
  _        => 'unknown',
};
```

패턴 매칭은 `is` + `as`를 한 번에 처리하면서 **절대 throw하지 않는다.**

---

## 4. 숫자 타입 — `as`로 변환되지 않는다

`int`와 `double`은 서로 **상속 관계가 아니다.** 둘 다 `num`을 구현할 뿐이다.

```dart
        num
       /   \
    int     double     // int → double 캐스팅 불가
```

```dart
Object n = 1;

n as double;        // 런타임 throw (native)
(1).toDouble();     // 1.0  ✅
(1.7).toInt();      // 1 (버림, 반올림 아님) ✅
1.7.round();        // 2
```

### num으로 받고 변환하기 (JSON에서 필수)

서버가 `10`을 주면 `int`, `10.5`를 주면 `double`로 파싱되므로 `as double`은 언제든 터진다.

```dart
// 위험: 서버가 정수를 내려주면 throw
final price = json['price'] as double;

// 안전
final price = (json['price'] as num).toDouble();
```

### ⚠️ 웹(dart2js)과 네이티브의 동작 차이

JS에는 정수 타입이 없어서 웹에서는 모든 숫자가 double로 표현된다.

```dart
1 is double   // native(VM/AOT): false  /  web: true
1.0 is int    // native: false          /  web: true
```

**로컬 테스트는 통과했는데 웹 배포 후 터지는(혹은 그 반대) 대표적인 원인**이다.
숫자는 `is`/`as`로 분기하지 말고 `num`으로 받아 `.toInt()` / `.toDouble()`로 변환하는 것이 정답이다.

---

## 5. 문자열 → 숫자

```dart
int.parse('42');          // 42
int.parse('abc');         // FormatException throw
int.tryParse('abc');      // null  ✅ 안전
double.tryParse('3.14');  // 3.14
int.tryParse('0xFF', radix: 16);  // 255

// 실무 패턴
final count = int.tryParse(input) ?? 0;
```

`parse`는 예외, `tryParse`는 null. **사용자 입력에는 무조건 `tryParse`.**

---

## 6. 컬렉션 캐스팅 (가장 많이 터지는 곳)

`List<dynamic>`을 `as List<String>`으로 캐스팅하면 **실패한다.** 제네릭 타입 인자까지 런타임에 검사하기 때문이다.

```dart
List<dynamic> raw = ['a', 'b', 'c'];

raw as List<String>;   // 🚫 throw: List<dynamic> is not a subtype of List<String>
```

### 3가지 해법 비교

```dart
// ① cast<T>() — 지연(lazy) 뷰. 원소에 접근할 때 캐스팅/에러 발생
final a = raw.cast<String>();

// ② List<T>.from() — 즉시(eager) 복사. 생성 시점에 에러 발생
final b = List<String>.from(raw);

// ③ map + as — 명시적, 새 리스트 생성
final c = raw.map((e) => e as String).toList();

// ④ whereType<T>() — 타입이 다른 원소는 버리고 걸러냄 (throw 없음)
final d = raw.whereType<String>().toList();
```

| 방법 | 복사 | 에러 시점 | 추천 상황 |
|---|---|---|---|
| `cast<T>()` | ❌ 뷰 | 원소 접근 시 (**디버깅 어려움**) | 짧게 스쳐 지나갈 때 |
| `List<T>.from()` | ✅ | 변환 시점 | JSON 파싱 등 대부분 |
| `map + as` | ✅ | 변환 시점 | 변환 로직이 함께 필요할 때 |
| `whereType<T>()` | ✅ | 없음(무시) | 이종 리스트 필터링 |

> `cast()`는 에러가 **한참 뒤 엉뚱한 곳에서** 터지므로, 파싱 경계에서는 `List<T>.from()`을 쓰는 편이 낫다.

### Map도 동일

```dart
final raw = jsonDecode(body);                       // dynamic
final map = raw as Map<String, dynamic>;            // OK (jsonDecode가 이 타입으로 만듦)
final typed = Map<String, String>.from(map);        // 값 타입까지 바꿀 때
```

---

## 7. JSON 파싱 실전

`jsonDecode`의 반환 타입은 `dynamic`이며, 실제로는
객체 → `Map<String, dynamic>`, 배열 → `List<dynamic>`으로 만들어진다.

```dart
class User {
  final int id;
  final String name;
  final double score;
  final List<String> tags;
  final String? nickname;

  User.fromJson(Map<String, dynamic> json)
      : id       = json['id'] as int,
        name     = json['name'] as String,
        score    = (json['score'] as num).toDouble(),          // num 경유 필수
        tags     = List<String>.from(json['tags'] as List),    // as List<String> ❌
        nickname = json['nickname'] as String?;                // nullable은 ? 붙이기
}
```

### 방어적으로 쓰기

서버 응답을 100% 믿을 수 없다면 헬퍼를 두는 게 안전하다.

```dart
extension JsonX on Map<String, dynamic> {
  T? get<T>(String key) => this[key] is T ? this[key] as T : null;
}

final name  = json.get<String>('name') ?? '';
final score = (json.get<num>('score') ?? 0).toDouble();
```

### 중첩 리스트

```dart
final users = (json['users'] as List)
    .map((e) => User.fromJson(e as Map<String, dynamic>))
    .toList();
```

---

## 8. `dynamic` vs `Object?`

```dart
dynamic d = 'hello';
d.foo();          // 컴파일 통과 → 런타임 NoSuchMethodError 💣

Object? o = 'hello';
o.foo();          // 컴파일 에러 ✅ (미리 잡힘)
```

| | 정적 검사 | 용도 |
|---|---|---|
| `dynamic` | **꺼짐** | JSON 등 진짜 타입을 모를 때만 |
| `Object?` | 켜짐 | "아무 값" 을 표현하는 기본 선택지 |

**"모든 타입 허용"이 필요하면 `dynamic`이 아니라 `Object?`를 쓴다.**

---

## 9. 제네릭 공변성(covariance) 함정

Dart의 제네릭은 공변(covariant)이라 `List<Dog>`는 `List<Animal>`의 서브타입이다.
읽기는 안전하지만 **쓰기에서 런타임 에러**가 난다.

```dart
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

List<Dog> dogs = [Dog()];
List<Animal> animals = dogs;   // 컴파일 OK (공변)

animals.add(Cat());            // 💣 런타임 throw: Cat is not a subtype of Dog
```

- 실제 객체는 여전히 `List<Dog>`이므로 `Cat`을 넣을 수 없다.
- 업캐스팅한 리스트에는 **쓰기(add/insert)를 하지 말 것**. 필요하면 `List<Animal>.from(dogs)`로 복사한다.

---

## 10. null 관련 캐스팅

```dart
String? maybe = fetch();

maybe!;                 // null이면 throw (의미: "null 아님을 내가 보장")
maybe as String;        // null이면 throw (의미: "타입 캐스팅")
maybe ?? 'default';     // 안전 ✅
maybe?.length;          // 안전 ✅ (결과 int?)
```

- `!`와 `as String`은 실패 시 결과가 같지만, **의도가 다르므로 null 문제엔 `!`를 쓴다.**
- `as String?`은 null을 통과시킨다. `as String`은 통과시키지 않는다.

```dart
Object? v = null;
v as String?;   // OK → null
v as String;    // throw
```

---

## 11. 런타임 타입 확인

```dart
final v = 42;

v.runtimeType;              // int (디버깅용. 분기 조건으로 쓰지 말 것)
v.runtimeType == int;       // 🚫 나쁨: 서브타입/난독화/제네릭에 취약
v is int;                   // ✅ 좋음
```

- `runtimeType` 비교는 서브클래스를 걸러내고, Flutter 릴리스 빌드 난독화에서도 깨질 수 있다.
- **분기는 항상 `is`로.**

---

## 12. 체크리스트

- [ ] 실패 가능성이 있으면 `as` 대신 `is` / 패턴 매칭
- [ ] 숫자는 `as double` 금지 → `(x as num).toDouble()`
- [ ] 사용자 입력은 `parse`가 아니라 `tryParse`
- [ ] `as List<String>` 금지 → `List<String>.from(...)`
- [ ] `cast()`는 에러가 늦게 터진다는 점 인지
- [ ] "아무 타입"은 `dynamic`이 아니라 `Object?`
- [ ] 업캐스팅한 제네릭 컬렉션에 쓰기 금지
- [ ] 타입 분기에 `runtimeType` 대신 `is`
- [ ] 필드는 지역 변수로 받아야 타입 승격됨

---

## 참고

- [Dart Docs — Type system](https://dart.dev/language/type-system)
- [Dart Docs — Patterns](https://dart.dev/language/patterns)
- [Dart Docs — Numbers in Dart (web/native 차이)](https://dart.dev/resources/language/number-representation)
