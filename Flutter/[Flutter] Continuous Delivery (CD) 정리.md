# Flutter Continuous Delivery (CD)

> Flutter 앱의 빌드·테스트·배포를 자동화하는 방법
> *- [Flutter Docs, Continuous delivery with Flutter](https://docs.flutter.dev/deployment/cd)*

앱을 스토어에 올리는 과정(빌드 → 서명 → 업로드)을 매번 손으로 하면 실수가 나기 쉽다.
CI/CD를 붙여두면 커밋 → 빌드 → TestFlight/Play Store 업로드까지 자동으로 흘러가게 만들 수 있다.

---

## 1. CI/CD 선택지

### 올인원 (Flutter 기능이 내장된 서비스)

별도 설정 없이 Flutter 빌드를 바로 지원한다.

| 서비스 | 링크 |
| :--- | :--- |
| Codemagic | [시작 가이드](https://blog.codemagic.io/getting-started-with-codemagic/) |
| Bitrise | [시작 가이드](https://devcenter.bitrise.io/en/getting-started/quick-start-guides/getting-started-with-flutter-apps) |
| Appcircle | [시작 가이드](https://appcircle.io/blog/guide-to-automated-mobile-ci-cd-for-flutter-projects-with-appcircle/) |

### 기존 워크플로에 fastlane 연동

이미 쓰고 있는 CI가 있다면 거기에 fastlane을 붙이는 방식.

- [GitHub Actions](https://github.com/features/actions) — [예제 프로젝트](https://github.com/nabilnalakath/flutter-githubaction)
- [Cirrus](https://cirrus-ci.org)
- [Travis](https://travis-ci.org/)
- [GitLab CI](https://docs.gitlab.com/ee/ci/)
- [CircleCI](https://circleci.com) — [Flutter + Fastlane 배포 글](https://circleci.com/blog/deploy-flutter-android)

---

## 2. fastlane

[fastlane](https://docs.fastlane.tools)은 앱의 릴리스와 배포를 자동화하는 오픈소스 툴 모음이다.

### 로컬 설정

**① 설치**

```console
gem install fastlane
```

```console
brew install fastlane
```

**② `FLUTTER_ROOT` 환경 변수 설정**

Flutter SDK 루트 디렉터리를 값으로 지정한다. (iOS 배포 스크립트에서 필요)

**③ 빌드 확인**

fastlane을 붙이기 전에, 프로젝트가 정상적으로 빌드되는지부터 확인한다.

| 플랫폼 | 명령어 |
| :--- | :--- |
| Android | `flutter build appbundle` |
| iOS | `flutter build ipa` |

**④ 플랫폼별 fastlane 초기화**

각 플랫폼 디렉터리에서 따로 초기화한다.

```console
cd [project]/android && fastlane init
```

```console
cd [project]/ios && fastlane init
```

**⑤ `Appfile` 메타데이터 확인**

| 플랫폼 | 확인할 것 |
| :--- | :--- |
| Android | `android/fastlane/Appfile`의 `package_name`이 `AndroidManifest.xml`의 패키지명과 일치하는지 |
| iOS | `ios/fastlane/Appfile`의 `app_identifier`가 `Info.plist`의 Bundle Identifier와 일치하는지<br>+ `apple_id`, `itc_team_id`, `team_id` 채우기 |

**⑥ 스토어 로그인 자격증명 설정**

- **Android** — [Supply 설정 절차](https://docs.fastlane.tools/getting-started/android/setup/#setting-up-supply)를 따라 `fastlane supply init`이 Play Console 데이터를 정상적으로 가져오는지 확인한다.
  > ⚠️ 발급받은 `.json` 키 파일은 **비밀번호처럼** 취급해야 한다. 공개 저장소에 절대 커밋하지 말 것.
- **iOS** — iTunes Connect 계정은 이미 `Appfile`의 `apple_id`에 들어있다. 비밀번호는 `FASTLANE_PASSWORD` 셸 환경 변수로 넣어둔다. 지정하지 않으면 업로드할 때마다 입력을 요구한다.

**⑦ 코드 서명 설정**

- **Android** — [Android 앱 서명 절차](https://docs.flutter.dev/deployment/android#signing-the-app)를 따른다.
- **iOS** — TestFlight/App Store 배포는 개발용(development)이 아니라 **배포용(distribution) 인증서**로 서명해야 한다.
  1. [Apple Developer 콘솔](https://developer.apple.com/account/ios/certificate/)에서 distribution certificate를 생성·다운로드
  2. `open [project]/ios/Runner.xcworkspace/` 후 타겟 설정에서 해당 인증서를 선택

**⑧ 플랫폼별 `Fastfile` 작성**

- **Android** — [fastlane Android 베타 배포 가이드](https://docs.fastlane.tools/getting-started/android/beta-deployment/) 참고.
  `upload_to_play_store`를 호출하는 `lane` 하나만 추가해도 된다.
  이때 `aab` 인자를 `../build/app/outputs/bundle/release/app-release.aab`로 지정하면
  `flutter build`가 이미 만들어둔 app bundle을 **재빌드 없이** 그대로 쓸 수 있다.

- **iOS** — [fastlane iOS 베타 배포 가이드](https://docs.fastlane.tools/getting-started/ios/beta-deployment/) 참고.
  archive 경로를 지정해 재빌드를 피할 수 있다.

```ruby
build_app(
  skip_build_archive: true,
  archive_path: "../build/ios/archive/Runner.xcarchive",
)
upload_to_testflight
```

### 로컬에서 배포 실행해보기

CI로 옮기기 전에, 로컬에서 먼저 성공시키는 게 순서다.

**① 릴리스 모드로 빌드**

```console
flutter build appbundle
```

```console
flutter build ipa
```

**② 각 플랫폼 디렉터리에서 lane 실행**

```console
cd android && fastlane [lane 이름]
```

```console
cd ios && fastlane [lane 이름]
```

---

## 3. 클라우드 빌드 & 배포

핵심 전제는 **클라우드 인스턴스는 일회성(ephemeral)이고 신뢰할 수 없다**는 것이다.
Play Store 서비스 계정 JSON이나 iTunes 배포 인증서 같은 자격증명을 서버에 그대로 두면 안 된다.

CI 시스템은 보통 **암호화된 환경 변수**를 지원하므로 비밀값은 거기에 넣는다.
앱 빌드 시에는 `--dart-define MY_VAR=MY_VALUE`로 전달할 수 있다.

> ⚠️ **비밀값 노출 주의**
> - 테스트 스크립트에서 환경 변수 값을 **다시 콘솔에 echo하지 말 것.**
> - 이런 변수들은 PR이 머지되기 전까지는 접근할 수 없다. 악의적인 사용자가 시크릿을 출력하는 PR을 만드는 것을 막기 위한 장치다.
> - 그러므로 **머지하는 PR의 내용을 주의 깊게 확인**해야 한다.

### ① 로그인 자격증명을 일회성으로 만들기

<details>
<summary><b>Android</b></summary>

<br>

`Appfile`에서 `json_key_file` 필드를 제거하고, JSON 문자열 내용을 CI의 암호화 변수에 저장한다.
그리고 `Fastfile`에서 환경 변수를 직접 읽는다.

```ruby
upload_to_play_store(
  ...
  json_key_data: ENV['<variable name>']
)
```

업로드 키(keystore)는 base64 등으로 직렬화해서 암호화 환경 변수로 저장하고,
CI의 install 단계에서 복원한다.

```bash
echo "$PLAY_STORE_UPLOAD_KEY" | base64 --decode > [업로드 keystore 경로]
```

</details>

<details>
<summary><b>iOS</b></summary>

<br>

- 로컬에서 쓰던 `FASTLANE_PASSWORD`를 CI의 암호화 환경 변수로 옮긴다.
- CI가 배포 인증서에 접근할 수 있어야 한다. 여러 머신 간 인증서 동기화에는 fastlane의 [Match](https://docs.fastlane.tools/actions/match/)를 사용하는 것이 권장된다.

</details>

### ② Gemfile 사용 (선택)

CI에서 매번 `gem install fastlane`을 실행하면 버전이 그때그때 달라진다.
`Gemfile`로 고정하면 로컬과 클라우드의 fastlane 의존성이 **안정적이고 재현 가능**해진다.

`[project]/android`와 `[project]/ios` 양쪽에 `Gemfile`을 만든다.

```ruby
source "https://rubygems.org"

gem "fastlane"
```

두 디렉터리에서 `bundle update`를 실행하고, `Gemfile`과 `Gemfile.lock`을 모두 소스 컨트롤에 커밋한다.

> 이후 로컬에서 실행할 때는 `fastlane`이 아니라 **`bundle exec fastlane`** 을 사용한다.

### ③ CI 테스트 스크립트 작성

저장소 루트에 `.travis.yml`, `.cirrus.yml` 같은 CI 설정 파일을 만든다.
CI별 세부 설정은 [fastlane CI 문서](https://docs.fastlane.tools/best-practices/continuous-integration)를 참고한다.

- 스크립트를 **Linux와 macOS 양쪽에서 돌도록 샤딩(shard)** 한다.

**Setup 단계에서 할 일**

| 항목 | 내용 |
| :--- | :--- |
| Bundler | `gem install bundler`로 Bundler 확보 |
| 의존성 설치 | `[project]/android` 또는 `[project]/ios`에서 `bundle install` |
| Flutter SDK | SDK를 준비하고 `PATH`에 등록 |
| Android | Android SDK 준비 + `ANDROID_SDK_ROOT` 경로 설정 |
| iOS | Xcode 의존성 명시 (예: `osx_image: xcode9.2`) |

**Script 단계에서 할 일**

플랫폼에 따라 빌드한 뒤, 해당 디렉터리로 이동해 lane을 실행한다.

```bash
flutter build appbundle
cd android
bundle exec fastlane [lane 이름]
```

```bash
flutter build ios --release --no-codesign --config-only
cd ios
bundle exec fastlane [lane 이름]
```

---

## 4. Xcode Cloud

[Xcode Cloud](https://developer.apple.com/xcode-cloud)는 Apple 플랫폼 앱·프레임워크의 빌드, 테스트, 배포를 위한 Apple의 CI/CD 서비스다.

### 요구사항

- Xcode **13.4.1 이상**
- [Apple Developer Program](https://developer.apple.com/programs) 가입

### 커스텀 빌드 스크립트

Xcode Cloud는 지정된 시점에 추가 작업을 수행하는 [커스텀 빌드 스크립트](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)를 인식한다.
또한 클론된 저장소 위치를 담고 있는 `$CI_WORKSPACE` 같은 [사전 정의된 환경 변수](https://developer.apple.com/documentation/xcode/environment-variable-reference)를 제공한다.

> Xcode Cloud의 임시 빌드 환경에는 macOS와 Xcode에 포함된 도구(예: Python)가 들어있고,
> 서드파티 의존성 설치를 위해 **Homebrew**도 함께 제공된다.

#### Post-clone 스크립트

Xcode Cloud가 Git 저장소를 클론한 **직후**에 실행되는 스크립트다.
Flutter는 Xcode Cloud 환경에 기본 설치되어 있지 않으므로, 여기서 직접 설치해줘야 한다.

`ios/ci_scripts/ci_post_clone.sh` 파일을 만들고 아래 내용을 작성한다.

```sh
#!/bin/sh

# 하위 명령이 하나라도 실패하면 스크립트를 실패 처리
set -e

# 이 스크립트의 기본 실행 위치는 ci_scripts 디렉터리다.
cd $CI_PRIMARY_REPOSITORY_PATH # 클론된 저장소 루트로 이동

# git으로 Flutter 설치
git clone https://github.com/flutter/flutter.git --depth 1 -b stable $HOME/flutter
export PATH="$PATH:$HOME/flutter/bin"

# iOS(--ios) 또는 macOS(--macos) 플랫폼용 Flutter 아티팩트 설치
flutter precache --ios

# Flutter 의존성 설치
flutter pub get

# Homebrew로 CocoaPods 설치
HOMEBREW_NO_AUTO_UPDATE=1 # homebrew 자동 업데이트 비활성화
brew install cocoapods

# CocoaPods 의존성 설치
cd ios && pod install # ios 디렉터리에서 pod install 실행

exit 0
```

이 파일은 git에 추가하면서 **실행 권한**을 함께 부여해야 한다.

```console
git add --chmod=+x ios/ci_scripts/ci_post_clone.sh
```

### 워크플로 설정

[Xcode Cloud 워크플로](https://developer.apple.com/documentation/xcode/xcode-cloud-workflow-reference)는 트리거됐을 때 수행할 CI/CD 단계들을 정의한다.

> 프로젝트가 이미 Git으로 초기화되어 있고 **원격 저장소에 연결**되어 있어야 한다.

Xcode에서 새 워크플로를 만드는 순서.

1. **Product > Xcode Cloud > Create Workflow** 선택
2. 워크플로를 연결할 제품(앱)을 선택하고 **Next**
3. Xcode가 제안하는 기본 워크플로 개요가 나오며, **Edit Workflow**로 커스터마이징

#### Branch Changes

기본값은 **기본 브랜치에 변경이 생길 때마다** 새 빌드를 시작하는 Branch Changes 조건이다.

Flutter 앱의 iOS 변형이라면, 보통 다음이 바뀌었을 때만 워크플로를 돌리고 싶을 것이다.

- Flutter 패키지(의존성)를 수정했을 때
- `lib/` 디렉터리의 Dart 소스를 수정했을 때
- `ios/` 디렉터리의 iOS 소스를 수정했을 때

이는 **Files and Folders 조건**으로 지정할 수 있다.

### Next Build Number

Xcode Cloud는 새 워크플로의 빌드 번호를 **`1`부터 시작**해서 빌드가 성공할 때마다 증가시킨다.
이미 더 높은 빌드 번호를 쓰고 있는 앱이라면, `Next Build Number`를 직접 지정해줘야 한다.

> 자세한 내용은 [Setting the next build number for Xcode Cloud builds](https://developer.apple.com/documentation/xcode/setting-the-next-build-number-for-xcode-cloud-builds#Set-the-next-build-number-to-a-custom-value) 참고.

---

## 정리

| 단계 | 핵심 |
| :--- | :--- |
| 도구 선택 | 빠르게 붙이려면 올인원(Codemagic 등), 기존 CI가 있다면 fastlane 연동 |
| 로컬 설정 | `flutter build`가 먼저 성공해야 하고, `Appfile` / 서명 / `Fastfile` 순서로 구성 |
| 로컬 검증 | CI로 옮기기 전에 로컬에서 lane 실행을 반드시 성공시킬 것 |
| 클라우드 이전 | 자격증명은 전부 암호화 환경 변수로. 서버에 남기지 않는다 |
| 재현성 | `Gemfile` + `bundle exec fastlane`으로 fastlane 버전 고정 |
| Xcode Cloud | post-clone 스크립트에서 Flutter를 직접 설치해야 함 |

---

## 참고 자료

- [Flutter Docs — Continuous delivery with Flutter](https://docs.flutter.dev/deployment/cd)
- [fastlane Docs](https://docs.fastlane.tools)
- [fastlane — Continuous Integration 모범 사례](https://docs.fastlane.tools/best-practices/continuous-integration)
- [fastlane — Match (인증서 동기화)](https://docs.fastlane.tools/actions/match/)
- [Apple — Xcode Cloud](https://developer.apple.com/xcode-cloud)
- [Apple — Writing custom build scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)
