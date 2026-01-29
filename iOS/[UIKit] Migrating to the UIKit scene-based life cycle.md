# [UIKit] Migrating to the UIKit scene-based life cycle

Scene-based lifey cycle(Scene Delegate 도입)을 채택 하지 않은 프로젝트에서 경고가 떴었다.

![스크린샷 2025-09-08 오후 3.11.08.png](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-09-08_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.11.08.png)

하지만 iOS 26, iPadOS 26, Mac Catalyst 26, tvOS 26, visionOS 26 에서 부터는 UIScene Life Cycle을 무조건 채택해야만 하게 되었다. 공식 문서에 따르면 **앱이 제대로 실행되지 않을 것이라고 경고했다**.

> In the next major release following iOS 26, UIScene lifecycle will be required when building with the latest SDK; otherwise, your app won’t launch.
> 

[TN3187: Migrating to the UIKit scene-based life cycle | Apple Developer Documentation](https://developer.apple.com/documentation/technotes/tn3187-migrating-to-the-uikit-scene-based-life-cycle)

앱 실행이 막히기 전에 Migration을 하도록 하자...

---

## SceneDelegate Migration 방법

1. **SceneDelegate 도입을 위한 info.plist 파일 UIApplicationSceneManifest 값 추가**
    - info.plist 파일 우클릭 → Open As → Source Code → 아래 코드 붙여 넣기
        
        ![스크린샷 2026-01-28 오후 11.14.37.png](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2026-01-28_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.14.37.png)
        
        ![스크린샷 2026-01-29 오전 12.37.33.png](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2026-01-29_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.37.33.png)
        
        ```swift
        <key>UIApplicationSceneManifest</key>
        <dict>
            <key>UIApplicationSupportsMultipleScenes</key>
            <false/> 
            <key>UISceneConfigurations</key>
            <dict>
                <key>UIWindowSceneSessionRoleApplication</key>
                <array>
                    <dict>
                        <key>UISceneConfigurationName</key>
                        <string>Default Configuration</string>
                        <key>UISceneDelegateClassName</key>
                        <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
                        <key>UISceneStoryboardFile</key>
                        <string>Main</string> 
                    </dict>
                </array>
            </dict>
        </dict>
        ```
        
    
    ### Key / value 부연 설명
    
    - UISceneConfigurationName(Configuration Name)
        - UISceneConfiguration의 이름 (구분자)
    - UISceneDelegateClassName(Delegate Class Name)
        - 해당 Configuration의 **Window Scene Delegate 역할을 수행하는 구현체 클래스** 이름
    - UISceneStoryboardFile(Storyboard Name)
        - 해당 Configuration이 연결할 Storyboard 파일 명
    
    ### (필요시) AppDelegate에 application(_:configurationForConnecting:options:) 구현하여 configuration 구성하기
    
    Info.plist 파일 UIApplicationSceneManifest를 제대로 설정해주었다면 AppDelegate에 application(_:configurationForConnecting:options:)를 구현할 필요가 없지만 아래 2가지 경우에는 구현 하도록 하자.
    
    1. **info.plist 파일 UIApplicationSceneManifest를 추가하지 않는 경우**
        
        AppDelegate에 application(_:configurationForConnecting:options:) 구현하는 방법도 있다.
        
        ```swift
        @main
        class AppDelegate: UIResponder, UIApplicationDelegate {
        
        		// ...
        
            // MARK: UISceneSession Lifecycle
        
            func application(_ application: UIApplication, configurationForConnecting connectingSceneSession: UISceneSession, options: UIScene.ConnectionOptions) -> UISceneConfiguration {
                let scene = UISceneConfiguration(name: "Default Configuration", sessionRole: connectingSceneSession.role)
                scene.delegateClass = SceneDelegate.self
                scene.storyboard = UIStoryboard(name: "Main", bundle: nil)
                
                return scene
            }
        }
        ```
        
    2. **동적 UISceneConfigurations 설정이 필요한 경우**
        
        사용자 활동 또는 세션별 데이터에 따라 다른 장면을 로드할 필요가 있는 경우, UISceneConfiguration을 생성해주고 싶다면 먼저 Info.plist에 추가적으로 UIWindowSceneSessionRoleApplication 값을 추가해주어야 한다.
        
        <aside>
        💡
        
        UIApplicationSupportsMultipleScenes을 true로 설정해주어야 한다
        
        false로 설정 되어있으면 정적으로 동작하게 되어 application(_:configurationForConnecting:options:)가 제대로 호출되지 않는다.
        
        </aside>
        
        ![스크린샷 2026-01-29 오전 1.39.49.png](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2026-01-29_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.39.49.png)
        
        ```swift
        <key>UIApplicationSceneManifest</key>
        	<dict>
        		<key>UIApplicationSupportsMultipleScenes</key>
        		<true/>
        		<key>UISceneConfigurations</key>
        		<dict>
        			<key>UIWindowSceneSessionRoleApplication</key>
        			<array>
        				<dict>
        					<key>UISceneConfigurationName</key>
        					<string>Default Configuration</string>
        					<key>UISceneDelegateClassName</key>
        					<string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
        					<key>UISceneStoryboardFile</key>
        					<string>Main</string>
        				</dict>
        				<dict>
        					<key>UISceneConfigurationName</key>
        					<string>Secondary Configuration</string>
        					<key>UISceneDelegateClassName</key>
        					<string>$(PRODUCT_MODULE_NAME).SecondarySceneDelegate</string>
        					<key>UISceneStoryboardFile</key>
        					<string>Main2</string>
        				</dict>
        			</array>
        		</dict>
        	</dict>
        ```
        
        UIWindowSceneSessionRoleApplication 값을 추가후 다시 AppDelegate application(_:configurationForConnecting:options:)로 돌아가서 상황에 맞는 UISceneConfiguration를 생성후 리턴해주자.
        
        ```swift
        class AppDelegate: UIResponder, UIApplicationDelegate {
            func application(
                _ application: UIApplication,
                configurationForConnecting connectingSceneSession: UISceneSession,
                options: UIScene.ConnectionOptions
            ) -> UISceneConfiguration {
            
                var configurationName: String!
            
                switch options.userActivities.first?.activityType {
                case UserActivity.GalleryOpenInspectorActivityType: //예시: 공식 문서 발췌
                    configurationName = "Secondary Configuration"
                default:
                    configurationName = "Default Configuration"
                }
                
                return UISceneConfiguration(
                    name: configurationName,
                    sessionRole: connectingSceneSession.role
                )
            }
        }
        ```
        

1. SceneDelegate 구현
    
    UIWindowSceneDelegate를 구현한 SceneDelegate class를 구현 해야한다. 
    
    새로 프로젝트를 생성해서 SceneDelegate 파일을 복붙해오는 방법도 있지만, 여기서 주의해야할 점은 Info.plist의 UIApplicationSceneManifest에서 **UISceneDelegateClassName 키에 할당한 Value 값의 이름 그대로 구현해주어야 한다는 점**이다.
    
    ```swift
    class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    
        var window: UIWindow?
    
        func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
            guard let _ = (scene as? UIWindowScene) else { return }
        }
    
        func sceneDidDisconnect(_ scene: UIScene) {
        }
    }
    ```
    
    ### 기존의 window rootViewController 연결을 코드 베이스로 했었다면
    
    ```swift
    window?.rootViewController = viewController
    window?.makeKeyAndVisible()
    ```
    
    만약 위 코드처럼 window.rootViewController를 코드베이스로 했었다면 해당 로직을 scene(_:willConnectTo:options:)로 옮기자
    
    ```swift
    class SceneDelegate: UIResponder, UIWindowSceneDelegate {
        var window: UIWindow?
        
        func scene(
            _ scene: UIScene,
            willConnectTo session: UISceneSession,
            options connectionOptions: UIScene.ConnectionOptions
        ) {
            guard let windowScene = scene as? UIWindowScene else { return }
                    
            window = UIWindow(windowScene: windowScene)
            window?.rootViewController = YourRootViewController()
            window?.makeKeyAndVisible()
        }
    }
    ```
    
2. AppDelegate에 있는 Deprecated Method migration
    
    이제 SceneDelegate 도입은 완료되었지만 **SceneDelegate를 도입하면서 AppDelegate에 있는 Deprecated 메서드들은 실행되지 않게 되는데 Migration을 해주어야 한다.**
    
    Migration 해주어야하는 메서드들은 아래와 같다.
    
    | **UIApplicationDelegate** | **UISceneDelegate** |
    | --- | --- |
    | [`applicationDidBecomeActive(_:)`](https://developer.apple.com/documentation/UIKit/UIApplicationDelegate/applicationDidBecomeActive(_:)) | [`sceneDidBecomeActive(_:)`](https://developer.apple.com/documentation/UIKit/UISceneDelegate/sceneDidBecomeActive(_:)) |
    | [`applicationWillResignActive(_:)`](https://developer.apple.com/documentation/UIKit/UIApplicationDelegate/applicationWillResignActive(_:)) | [`sceneWillResignActive(_:)`](https://developer.apple.com/documentation/UIKit/UISceneDelegate/sceneWillResignActive(_:)) |
    | [`applicationDidEnterBackground(_:)`](https://developer.apple.com/documentation/UIKit/UIApplicationDelegate/applicationDidEnterBackground(_:)) | [`sceneDidEnterBackground(_:)`](https://developer.apple.com/documentation/UIKit/UISceneDelegate/sceneDidEnterBackground(_:)) |
    | [`applicationWillEnterForeground(_:)`](https://developer.apple.com/documentation/UIKit/UIApplicationDelegate/applicationWillEnterForeground(_:)) | [`sceneWillEnterForeground(_:)`](https://developer.apple.com/documentation/UIKit/UISceneDelegate/sceneWillEnterForeground(_:)) |

### SceneDelegate 도입 전후 비교

상술한 것 처럼 migration 해야하는 메서드들에 print문을 달아서 확인 해본 결과는 아래와 같다.

앱을 실행하고 홈으로 나갔다가 다시 돌아 왔을 때의 결과이다.

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
		
		//...
		    
    //MARK: Legacy
    func applicationDidBecomeActive(_ application: UIApplication) {
        print(String(describing: self), #function)
    }
    
    func applicationWillResignActive(_ application: UIApplication) {
        print(String(describing: self), #function)
    }
    
    func applicationDidEnterBackground(_ application: UIApplication) {
        print(String(describing: self), #function)
    }
    
    func applicationWillEnterForeground(_ application: UIApplication) {
        print(String(describing: self), #function)
    }

}
```

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    func sceneDidBecomeActive(_ scene: UIScene) {
        // Called when the scene has moved from an inactive state to an active state.
        // Use this method to restart any tasks that were paused (or not yet started) when the scene was inactive.
        print(String(describing: self), #function)
    }

    func sceneWillResignActive(_ scene: UIScene) {
        // Called when the scene will move from an active state to an inactive state.
        // This may occur due to temporary interruptions (ex. an incoming phone call).
        print(String(describing: self), #function)
    }

    func sceneWillEnterForeground(_ scene: UIScene) {
        // Called as the scene transitions from the background to the foreground.
        // Use this method to undo the changes made on entering the background.
        print(String(describing: self), #function)
    }

    func sceneDidEnterBackground(_ scene: UIScene) {
        // Called as the scene transitions from the foreground to the background.
        // Use this method to save data, release shared resources, and store enough scene-specific state information
        // to restore the scene back to its current state.
        print(String(describing: self), #function)
    }
}
```

![SceneDelegate 도입 전](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2026-01-28_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.40.03.png)

SceneDelegate 도입 전

![SceneDelegate 도입 후](%5BUIKit%5D%20Migrating%20to%20the%20UIKit%20scene-based%20life%20cy/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2026-01-28_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.42.49.png)

SceneDelegate 도입 후

Scene Delegate 도입 전에는 AppDelegate에서 실행되던 메서드들이 Scene base life cycle을 적용하고 나서부터는 제대로 실행되지 않는 것을 확인할 수 있다.

이렇게 하면 Scene base life cycle 적용을 완료한 것이다.

---

## ⚠️ AppDelegate의 window 변수 migration

---

### References

[UIKit Scene-based Life Cycle 마이그레이션](https://phillip5094.tistory.com/310)

[TN3187: Migrating to the UIKit scene-based life cycle | Apple Developer Documentation](https://developer.apple.com/documentation/technotes/tn3187-migrating-to-the-uikit-scene-based-life-cycle)