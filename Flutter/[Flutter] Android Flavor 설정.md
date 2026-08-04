# Flutter Android Flavor 설정

> 릴리스 타입이나 개발 환경별로 빌드 Flavor를 만드는 방법
> *- [Flutter Docs, Set up Flutter flavors for Android](https://docs.flutter.dev/deployment/flavors)*

## Flavor란

Flutter의 **Flavor**는 Android에서 여러 플랫폼별 기능을 하나로 묶어 부르는 개념이다.
예를 들어 앱 버전마다 다음과 같은 요소를 다르게 가져갈 수 있다.

- 앱 아이콘
- 앱 이름
- API Key
- Feature Flag
- 로깅 레벨

Android에서는 이를 **Product Flavor**라고 부른다.

### Product Flavor × Build Type = Build Variant

Product Flavor 2개(`staging`, `production`)와 Build Type 2개(`debug`, `release`)를 조합하면
총 4개의 Build Variant가 만들어진다.

| Product Flavor | Build Type | 생성되는 Build Variant |
| :--- | :--- | :--- |
| staging | debug / release | `stagingDebug`, `stagingRelease` |
| production | debug / release | `productionDebug`, `productionRelease` |

---

## 1. Product Flavor 설정하기

`flavors_example`라는 새 프로젝트에 `staging` / `production` 두 개의 Flavor를 추가하는 예시.

### 프로젝트 생성

```console
flutter create --android-language kotlin flavors_example
```

기본적으로 `debug`, `release` Build Type이 포함되어 있다.

### build.gradle.kts 수정

`android/app/build.gradle.kts`를 열고 `android {}` 블록 안에
`flavorDimensions`와 `productFlavors`를 추가한다.

```kotlin
android {
    ...
    buildTypes {
      getByName("debug") {...}
      getByName("release") {...}
    }
    ...
    flavorDimensions += "default"
    productFlavors {
        create("staging") {
            dimension = "default"
            applicationIdSuffix = ".staging"
        }
        create("production") {
            dimension = "default"
            applicationIdSuffix = ".production"
        }
    }
}
```

> `applicationIdSuffix`를 지정하면 패키지명이 달라지므로
> 한 기기에 여러 Flavor를 **동시에 설치**할 수 있다.

### 동작 확인

에뮬레이터를 켜거나 개발자 옵션이 활성화된 실기기를 연결한 뒤 실행한다.

```console
flutter run --flavor staging
```

아직 설정값을 바꾸지 않았으므로 눈에 보이는 차이는 없지만,
정상적으로 빌드/실행되는지만 확인하면 된다. `production`도 동일하게 확인한다.

---

## 2. 특정 Flavor로 실행 / 빌드하기

```console
flutter (run | build <subcommand>) --flavor <flavor_name>
```

| 항목 | 설명 |
| :--- | :--- |
| `run` | 디버그 모드로 앱 실행 |
| `build <subcommand>` | `apk` 또는 `appbundle` 빌드 |
| `--flavor <flavor_name>` | `staging`, `production` 등 Flavor 이름 |

예시

```console
flutter build apk --flavor staging
```

---

## 3. Dart 코드에서 Flavor 사용하기

Flutter 프레임워크는 현재 Flavor 이름을 `String`으로 알려주는 **`appFlavor`** 상수를 제공한다.
이 값은 `--flavor` 플래그로 전달한 이름과 일치한다.

### import

```dart
import 'package:flutter/services.dart';
```

### 분기 처리

```dart
void main() {
  // appFlavor는 build.gradle.kts에 정의한 flavor 이름과 일치한다
  if (appFlavor == 'production') {
    // 프로덕션 환경 로직
    Config.apiUrl = 'https://api.flavors_example.com';
  } else if (appFlavor == 'staging') {
    // 스테이징 환경 로직
    Config.apiUrl = 'https://staging.api.flavors_example.com';
  }

  runApp(const MyApp());
}
```

> ⚠️ 빌드 시 Flavor를 지정하지 않으면 `appFlavor`는 **`null`** 을 반환한다.

---

## 4. Flavor별 커스터마이징

<details>
<summary><b>앱 이름 다르게 하기</b></summary>

<br>

Flavor가 여러 개일 때 앱 이름이 다르면 지금 설치된 게 어떤 빌드인지 바로 알 수 있다.

**① `android/app/build.gradle.kts`** — 각 Flavor에 `resValue()`로 `app_name`을 정의한다.

```kotlin
android {
    ...
    flavorDimensions += "default"
    productFlavors {
        create("staging") {
            dimension = "default"
            resValue(
                type = "string",
                name = "app_name",
                value = "Flavors staging")
            applicationIdSuffix = ".staging"
        }
        create("production") {
            dimension = "default"
            resValue(
                type = "string",
                name = "app_name",
                value = "Flavors production")
            applicationIdSuffix = ".production"
        }
    }
}
```

**② `android/app/src/main/AndroidManifest.xml`** — `android:label`을 `@string/app_name`으로 교체한다.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
      android:label="@string/app_name"
      ...
    />
</manifest>
```

**③ 확인** — 각 Flavor로 실행한 뒤 앱 목록에서 이름이 다르게 표시되는지 확인한다.

</details>

<details>
<summary><b>앱 아이콘 다르게 하기</b></summary>

<br>

**① 아이콘 준비** — 각 Flavor용 아이콘을 아래 사이즈의 PNG로 생성한다.

| 디렉터리 | 크기 |
| :--- | :--- |
| `mipmap-mdpi` | 48 × 48 |
| `mipmap-hdpi` | 72 × 72 |
| `mipmap-xhdpi` | 96 × 96 |
| `mipmap-xxhdpi` | 144 × 144 |
| `mipmap-xxxhdpi` | 192 × 192 |

> [App Icon Generator](https://www.appicon.co/) 같은 도구로 한 번에 생성할 수 있다.

**② Flavor별 리소스 디렉터리 생성** — `android/app/src` 아래에 Flavor 이름의 디렉터리를 만든다.

```
android/app/src/
├── main/
├── staging/
│   └── res/
│       ├── mipmap-mdpi/ic_launcher.png
│       ├── mipmap-hdpi/ic_launcher.png
│       ├── mipmap-xhdpi/ic_launcher.png
│       ├── mipmap-xxhdpi/ic_launcher.png
│       └── mipmap-xxxhdpi/ic_launcher.png
└── production/
    └── res/
        └── ... (동일 구조)
```

> 파일명은 **모두 `ic_launcher.png`** 로 통일해야 한다.
> Gradle이 Flavor 디렉터리의 리소스를 `main`보다 우선해서 병합한다.

**③ Manifest 확인** — `AndroidManifest.xml`의 `android:icon` 값이 `@mipmap/ic_launcher`인지 확인한다.

**④ 확인** — 각 Flavor로 실행해서 아이콘이 다르게 나오는지 확인한다.

</details>

<details>
<summary><b>Flavor별 에셋 번들링</b></summary>

<br>

특정 Flavor에서만 쓰는 에셋이 있다면, 그 Flavor로 빌드할 때만 번들에 포함되도록 설정할 수 있다.
사용하지 않는 에셋 때문에 앱 번들 크기가 커지는 걸 막을 수 있다.

`pubspec.yaml`의 `assets` 필드에 **`flavors` 하위 필드**를 추가하면 된다.

</details>

<details>
<summary><b>기본 Flavor 지정</b></summary>

<br>

`--flavor` 없이 실행했을 때 사용할 기본 Flavor를 지정할 수 있다.
`pubspec.yaml`에 **`default-flavor`** 필드를 추가한다.

</details>

<details>
<summary><b>추가 빌드 설정</b></summary>

<br>

그 외 Flavor별 빌드 설정은 Android의 [Configure build variants](https://developer.android.com/build/build-variants) 문서를 참고한다.

> ⚠️ **`abiFilters` 주의**
> Product Flavor에 `abiFilters`를 설정하는 것은 **권장되지 않는다**. 가능하면 `defaultConfig`에 설정할 것.
> Product Flavor에 설정해야 한다면 빌드/실행 시 `-Pdisable-abi-filtering=true`를 넘겨야 한다.

</details>

---

## 참고 자료

- [Flutter Docs — Set up Flutter flavors for Android](https://docs.flutter.dev/deployment/flavors)
- [Android Developers — Configure build variants](https://developer.android.com/build/build-variants)
- [Build flavors in Flutter (Android and iOS) with Firebase](https://medium.com/@animeshjain/build-flavors-in-flutter-android-and-ios-with-different-firebase-projects-per-flavor-27c5c5dac10b)
- [How to Setup Flutter & Firebase with Multiple Flavors using the FlutterFire CLI](https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/)
