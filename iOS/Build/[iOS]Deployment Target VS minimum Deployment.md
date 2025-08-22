# [iOS]Deployment Target VS minimum Deployment

⚠️ Xcode가 변했는지 최근 Xcdoe에는 Project Info에서 Deployment Target 과 minimum Deployment 의 구분이 사라진 듯 하다.

![스크린샷 2025-08-22 오후 1.42.23.png](%5BiOS%5DDeployment%20Target%20VS%20minimum%20Deployment%202578ef220c3b803e9987e2e901094595/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-08-22_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_1.42.23.png)

### ~~Deployment Target~~

- ~~Project 단에서 Info 설정 화면에서 설정할 수 있는 앱을 실행할 수 있는 iOS 최소 버전~~
- ~~Project → Target의 프로젝트 구조에 따라 모든 Target에 영향을 줄 *수* 있음~~
    - ~~모든 Target들의 기준을 세우는 역할의 용도~~

⇒ 최근 Xcode에서는 Project Info에서 Deployment Target 설정이 사라지고 Build Setting에서 수정하도록 유도됨.

### Minimum Deployment

- Target 단에서 Info 설정 화면에서 설정할 수 있는 앱을 실행할 수 있는 iOS 최소 버전
- 실제 빌드에 반영됨

### **프로젝트가 설정을 상속하는 방식**

- 모든 Target은 상위 프로젝트와 플랫폼 SDK로부터 설정을 상속
    - 물론 Target 별로 설정을 자유롭게 수정할 수 있음
    - 즉, Target에서 Build Setting을 수정하거나 할당하지 않으면 Project Build Setting을 상속받음
        - 이를 이해하기 제일 좋은 방법은 Target의 Build Settings를 열고 필터로 설정
            
            ![스크린샷 2025-08-22 오후 11.25.49.png](%5BiOS%5DDeployment%20Target%20VS%20minimum%20Deployment%202578ef220c3b803e9987e2e901094595/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-08-22_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.25.49.png)
            

[Configuring the build settings of a target | Apple Developer Documentation](https://developer.apple.com/documentation/Xcode/configuring-the-build-settings-of-a-target)

[Xcode target Deployment Target vs. project Deployment Target](https://stackoverflow.com/questions/7105127/xcode-target-deployment-target-vs-project-deployment-target)