# [Xcode] Asset Symbol (ImageResource / ColorResource) 정리

Xcode 15부터 Asset Catalog의 이미지와 컬러에 대해 **Swift 심볼이 자동 생성**된다. `UIImage(named: "icon_home")` 같은 문자열 기반 접근 대신 `UIImage(resource: .iconHome)`으로 컴파일 타임에 검증된 접근이 가능하다. 그동안 직접 만들던 `UIImage` extension이나 SwiftGen / R.swift 같은 코드 생성 도구가 하던 일을 Xcode가 기본으로 해준다.

---

## 1. 기존 방식의 문제

```swift
// 문자열 기반 — 오타는 런타임까지 살아남는다
let image = UIImage(named: "icon_home")   // UIImage? — 이름이 틀리면 조용히 nil
```

- 반환 타입이 **Optional**이라 호출부마다 언래핑이 필요하다.
- 에셋 이름을 바꾸면 문자열을 쓰는 모든 곳을 직접 찾아 고쳐야 한다. 컴파일러가 잡아주지 않는다.
- 자동완성이 되지 않아 실제 에셋 이름을 Asset Catalog에서 눈으로 확인해야 한다.

그래서 보통 이런 식으로 감쌌다.

```swift
extension UIImage {
    static let iconHome = UIImage(named: "icon_home")!
}

// 또는 enum + rawValue
enum AppImage: String {
    case iconHome = "icon_home"
    var image: UIImage { UIImage(named: rawValue)! }
}
```

동작은 하지만 **에셋을 추가할 때마다 사람이 직접 한 줄씩 유지보수**해야 하고, 에셋을 지웠는데 extension을 안 지우면 `!`에서 크래시가 난다.

---

## 2. Asset Symbol이란

Asset Catalog Compiler(`actool`)가 빌드 시점에 카탈로그를 훑어서 **에셋 하나당 심볼 하나**를 담은 Swift 파일을 생성한다. 생성 위치는 다음과 같다.

```text
$(DERIVED_SOURCES_DIR)/GeneratedAssetSymbols.swift   // Swift
$(DERIVED_SOURCES_DIR)/GeneratedAssetSymbols.h       // Objective-C
$(DERIVED_SOURCES_DIR)/GeneratedAssetSymbols-Index.plist
```

DerivedData 안에 있으므로 직접 열어보려면 이렇게 찾으면 된다.

```bash
find ~/Library/Developer/Xcode/DerivedData -name "GeneratedAssetSymbols.swift"
```

실제 생성 결과는 다음과 같은 모양이다.

```swift
import Foundation
#if canImport(UIKit)
import UIKit
#endif
#if canImport(SwiftUI)
import SwiftUI
#endif
#if canImport(DeveloperToolsSupport)
import DeveloperToolsSupport
#endif

#if SWIFT_PACKAGE
private let resourceBundle = Foundation.Bundle.module
#else
private class ResourceBundleClass {}
private let resourceBundle = Foundation.Bundle(for: ResourceBundleClass.self)
#endif

// MARK: - Image Symbols -

@available(iOS 17.0, macOS 14.0, tvOS 17.0, watchOS 10.0, *)
extension DeveloperToolsSupport.ImageResource {

    /// The "icon_bookmark" asset catalog image resource.
    static let iconBookmark = DeveloperToolsSupport.ImageResource(name: "icon_bookmark", bundle: resourceBundle)

    /// The "btn_floating_archive" asset catalog image resource.
    static let btnFloatingArchive = DeveloperToolsSupport.ImageResource(name: "btn_floating_archive", bundle: resourceBundle)
}
```

핵심은 **번들 해석까지 생성 코드가 알아서 처리한다**는 점이다. SPM 타겟이면 `Bundle.module`, 일반 타겟이면 `Bundle(for:)`로 자기 모듈 번들을 잡는다. 프레임워크로 코드를 분리했을 때 `UIImage(named:in:compatibleWith:)`에 번들을 매번 넘겨주던 작업이 사라진다.

---

## 3. ImageResource / ColorResource의 정체

`DeveloperToolsSupport` 프레임워크에 정의된 값 타입이다.

```swift
@available(iOS 17.0, macOS 14.0, tvOS 17.0, watchOS 10.0, *)
public struct ImageResource: Hashable, Sendable {
    public init(name: String, bundle: Bundle)
}

@available(iOS 17.0, macOS 14.0, tvOS 17.0, watchOS 10.0, *)
public struct ColorResource: Hashable, Sendable {
    public init(name: String, bundle: Bundle)
}
```

**이미지 자체가 아니라 "이미지를 가리키는 이름 + 번들"** 이다. 실제 디코딩은 `UIImage(resource:)`나 `Image(_:)`를 호출하는 시점에 일어난다. 이 성질 덕분에 다음이 가능하다.

- `Hashable` → `Set`, `Dictionary` 키로 사용 가능
- `Sendable` → 액터 경계를 넘어 안전하게 전달 가능
- 생성 비용이 사실상 없으므로 **모델 계층에서 UIKit 의존 없이 이미지를 지정**할 수 있다

```swift
// 모델은 UIKit을 import하지 않고도 "어떤 이미지인지"를 표현할 수 있다
struct MenuItem {
    let title: String
    let icon: ImageResource   // UIImage가 아니다
}

// 렌더링 시점에만 실제 이미지로 변환
imageView.image = UIImage(resource: item.icon)
```

---

## 4. 사용법

### UIKit

```swift
@available(iOS 17.0, tvOS 17.0, *)
extension UIImage {
    convenience init(resource: ImageResource)   // Optional이 아니다
}

@available(iOS 17.0, tvOS 17.0, *)
extension UIColor {
    convenience init(resource: ColorResource)
}
```

```swift
imageView.image = UIImage(resource: .iconBookmark)
view.backgroundColor = UIColor(resource: .brandPrimary)
```

`init?`가 아니라 `init`이다. 심볼이 존재한다는 것은 빌드 시점에 에셋이 카탈로그에 있었다는 뜻이므로 옵셔널이 아니다.

### SwiftUI

```swift
extension Image { public init(_ resource: ImageResource) }
extension Color { public init(_ resource: ColorResource) }
```

```swift
Image(.iconBookmark)
    .resizable()
    .foregroundStyle(Color(.brandPrimary))
```

SwiftUI의 여러 API가 `ImageResource`를 직접 받는 오버로드를 제공한다. `Label`, `Button`, `Toggle`, `Picker`, `Menu` 등에서 `image:` 파라미터로 그대로 넘길 수 있다.

```swift
Label("북마크", image: .iconBookmark)
Toggle("알림", image: .iconBell, isOn: $isOn)
```

### AppKit

```swift
NSImage(resource: .iconBookmark)
NSColor(resource: .brandPrimary)
```

---

## 5. 심볼 이름 생성 규칙과 충돌

에셋 이름은 **camelCase Swift 식별자로 변환**된다. 구분자(`_`, `-`, 공백)는 제거되고 뒤 단어의 첫 글자가 대문자가 된다.

| 에셋 이름 | 생성 심볼 (Swift) | 생성 심볼 (ObjC) |
|---|---|---|
| `icon_bookmark` | `iconBookmark` | `ACImageNameIconBookmark` |
| `btn_floating_archive` | `btnFloatingArchive` | `ACImageNameBtnFloatingArchive` |
| `icon_bookmark_20_white_fill` | `iconBookmark20WhiteFill` | `ACImageNameIconBookmark20WhiteFill` |

문제는 **서로 다른 에셋 이름이 같은 심볼로 변환되는 경우**다. 예를 들어 `block_lv-1`과 `block_lv1`은 둘 다 `blockLv1`이 된다. 이때 뒤에 오는 쪽은 심볼이 생성되지 않고 대신 `#warning`이 삽입된다.

```swift
#warning("The \"block_lv1\" image asset name resolves to the symbol \"blockLv1\" which already exists. Try renaming the asset.")
```

```objc
#warning The "block_lv1" image asset name resolves to the symbol "ACImageNameBlockLv1" which already exists. Try renaming the asset.
```

즉 **에셋 접근에 실패하는 게 아니라 심볼 자체가 안 만들어진다.** 빌드 경고를 무시하고 있었다면 `.blockLv1`이 왜 자동완성에 안 뜨는지 한참 헤맬 수 있다. `-`와 `_` 혼용, 대소문자만 다른 이름은 피하는 게 좋다.

Swift 키워드와 겹치는 이름(`default`, `class` 등)은 백틱으로 감싸서 생성된다.

---

## 6. 관련 빌드 세팅

| 빌드 세팅 | 표시 이름 | 기본값 | 설명 |
|---|---|---|---|
| `ASSETCATALOG_COMPILER_GENERATE_ASSET_SYMBOLS` | Generate Asset Symbols | `YES` | 심볼 생성 자체를 켜고 끈다 |
| `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS` | Generate Swift Asset Symbol Extensions | `NO`* | `UIImage`/`Color` 등에 정적 프로퍼티도 함께 생성 |
| `ASSETCATALOG_COMPILER_GENERATE_ASSET_SYMBOL_FRAMEWORKS` | Generate Swift Asset Symbol Framework Support | `SwiftUI UIKit AppKit` | 어떤 UI 프레임워크용 확장을 생성할지 |
| `ASSETCATALOG_COMPILER_GENERATE_ASSET_SYMBOL_BACKWARDS_DEPLOYMENT_SUPPORT` | — | 자동 (Deployment Target 기준) | 하위 버전 지원 코드 생성 |
| `ASSETCATALOG_COMPILER_GENERATE_ASSET_SYMBOL_WARNINGS` | — | `YES` | 심볼 관련 경고 출력 |
| `ASSETCATALOG_COMPILER_GENERATE_ASSET_SYMBOL_ERRORS` | — | `YES` | 심볼 관련 에러 출력 |
| `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOLS_PATH` | — | `$(DERIVED_SOURCES_DIR)/GeneratedAssetSymbols.swift` | 생성 파일 경로 |

\* 툴 명세상 기본값은 `NO`지만, Xcode 15 이후 **새 프로젝트 템플릿은 `project.pbxproj`에 `= YES`로 넣어준다.** 오래된 프로젝트를 Xcode 15+로 올린 경우에는 꺼져 있을 수 있으므로 Build Settings에서 직접 확인해야 한다.

### Symbol Extensions를 켜면

`ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS = YES`일 때 아래 코드가 **추가로** 생성된다.

```swift
#if canImport(UIKit)
@available(iOS 17.0, tvOS 17.0, *)
@available(watchOS, unavailable)
extension UIKit.UIImage {

    /// The "icon_bookmark" asset catalog image.
    static var iconBookmark: UIKit.UIImage {
#if !os(watchOS)
        .init(resource: .iconBookmark)
#else
        .init()
#endif
    }
}
#endif
```

덕분에 이렇게도 쓸 수 있다.

```swift
imageView.image = .iconBookmark          // UIImage.iconBookmark
```

다만 **정적 프로퍼티는 접근할 때마다 새로 디코딩**된다(`static var`, 캐시 없음). 반복 호출이 잦은 경로에서는 `ImageResource`를 들고 있다가 필요할 때 `UIImage(resource:)`로 변환하는 편이 의도가 더 분명하다.

---

## 7. iOS 17 미만 지원 (Backwards Deployment)

`ImageResource`/`ColorResource` 타입 자체는 iOS 17 / macOS 14 이상이다. 하지만 Deployment Target이 그보다 낮으면 Xcode가 **모듈 내부에 동일한 이름의 구조체를 직접 생성**해서 하위 버전에서도 동작하게 만든다.

```swift
// MARK: - Backwards Deployment Support -

/// An image resource.
struct ImageResource: Swift.Hashable, Swift.Sendable {
    fileprivate let name: Swift.String
    fileprivate let bundle: Foundation.Bundle

    init(name: Swift.String, bundle: Foundation.Bundle) {
        self.name = name
        self.bundle = bundle
    }
}

#if canImport(UIKit)
@available(iOS 11.0, tvOS 11.0, *)
@available(watchOS, unavailable)
extension UIKit.UIImage {
    /// Initialize a `UIImage` with an image resource.
    convenience init(resource: ImageResource) {
        self.init(named: resource.name, in: resource.bundle, compatibleWith: nil)!
    }
}
#endif
```

즉 iOS 11까지는 그대로 쓸 수 있다. 다만 아래 두 가지를 유의해야 한다.

- 이때의 `ImageResource`는 `DeveloperToolsSupport.ImageResource`가 **아니라 모듈 내부(internal) 타입**이다. 모듈 경계를 넘는 public API의 파라미터 타입으로는 쓸 수 없다.
- 내부 구현이 `UIImage(named:...)!` 강제 언래핑이다. 심볼 생성 이후에 에셋을 지웠다면 런타임 크래시가 난다. (심볼도 함께 사라지므로 보통은 컴파일 에러가 먼저 나지만, 카탈로그를 별도 번들로 분리했다면 얘기가 다르다.)

---

## 8. Objective-C

ObjC 헤더는 타입이 아니라 **문자열 상수**를 만든다.

```objc
#if __has_attribute(swift_private)
#define AC_SWIFT_PRIVATE __attribute__((swift_private))
#else
#define AC_SWIFT_PRIVATE
#endif

/// The "icon_bookmark" asset catalog image resource.
static NSString * const ACImageNameIconBookmark AC_SWIFT_PRIVATE = @"icon_bookmark";
```

`AC_SWIFT_PRIVATE`(= `swift_private`) 때문에 이 상수들은 Swift에서 그대로 보이지 않는다. Swift는 위의 `ImageResource` 심볼을 쓰라는 의도다.

---

## 9. 기존 UIImage extension에서 넘어갈 때

| | 직접 만든 extension | Asset Symbol |
|---|---|---|
| 유지보수 | 에셋 추가/삭제 시 수동 반영 | 빌드마다 자동 동기화 |
| 오타 | 런타임 nil 또는 크래시 | 컴파일 에러 |
| 번들 처리 | `Bundle(for:)` 직접 전달 | 생성 코드가 자동 처리 (SPM 포함) |
| 삭제된 에셋 | `!` 강제 언래핑 크래시 | 심볼이 사라져 컴파일 에러 |
| 이름 충돌 | 사람이 인지 못 함 | 빌드 경고 |
| 외부 도구 | SwiftGen / R.swift 필요 | Xcode 기본 제공 |

마이그레이션은 대체로 다음 순서가 무난하다.

1. Build Settings에서 `Generate Swift Asset Symbol Extensions`를 켠다.
2. 한 번 빌드해서 `GeneratedAssetSymbols.swift`를 확인하고 **이름 충돌 경고를 먼저 정리**한다.
3. 기존 extension의 프로퍼티를 하나씩 생성 심볼로 위임한다.
   ```swift
   extension UIImage {
       // static let iconHome = UIImage(named: "icon_home")!
       static let iconHome = UIImage(resource: .iconHome)
   }
   ```
   이러면 호출부를 건드리지 않고 안전성만 먼저 확보할 수 있다.
4. 호출부를 `UIImage(resource: .iconHome)` / `Image(.iconHome)`로 정리한 뒤 기존 extension을 제거한다.

---

## 10. 유의사항

- **심볼은 컴파일 타임 산물이다.** 서버에서 받은 문자열로 이미지를 고르는 등 런타임에 이름이 결정되는 경우는 여전히 `UIImage(named:)`를 써야 한다. 이때 `ImageResource(name:bundle:)`를 직접 호출할 수도 있지만, 존재 여부를 보장하지 않으므로 `UIImage(resource:)`에서 크래시 위험이 있다.
- **SF Symbol은 대상이 아니다.** Asset Catalog에 들어 있는 에셋만 심볼이 생성된다. SF Symbol은 그대로 `UIImage(systemName:)` / `Image(systemName:)`을 쓴다.
- **에셋 이름을 바꾸면 심볼도 바뀐다.** 컴파일 에러로 잡히므로 안전하지만, 리네임 시 영향 범위를 미리 감안해야 한다.
- **`Bundle.module` 경로 확인.** SPM에서는 `#if SWIFT_PACKAGE` 분기를 타므로, 리소스를 `Package.swift`의 `resources:`에 제대로 선언했는지 확인해야 한다.
- 실제로 어떤 심볼이 만들어졌는지 의심스러우면 DerivedData의 `GeneratedAssetSymbols.swift`를 직접 열어보는 게 가장 빠르다.

---

## 11. 결론

```text
Asset Catalog에 에셋 추가
        ↓
actool이 빌드 시점에 GeneratedAssetSymbols.swift 생성
        ↓
ImageResource / ColorResource 정적 심볼
        ↓
UIImage(resource: .iconHome)  /  Image(.iconHome)
        =
컴파일 타임 검증 + 자동완성 + 번들 자동 해석
```

직접 관리하던 `UIImage` extension이나 SwiftGen 같은 코드 생성 도구의 역할을 Xcode가 대신한다. `ImageResource`가 이미지가 아니라 "이미지에 대한 가벼운 참조"라는 점을 활용하면, UI 레이어 밖에서도 UIKit 의존 없이 이미지를 지정할 수 있다.

---

### References

[ImageResource | Apple Developer Documentation](https://developer.apple.com/documentation/developertoolssupport/imageresource)

[ColorResource | Apple Developer Documentation](https://developer.apple.com/documentation/developertoolssupport/colorresource)

[UIImage.init(resource:) | Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uiimage/init(resource:))

[Image.init(_:) | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/image/init(_:)-5tqmg)

[WWDC23 - What's new in Xcode 15](https://developer.apple.com/videos/play/wwdc2023/10165/)
