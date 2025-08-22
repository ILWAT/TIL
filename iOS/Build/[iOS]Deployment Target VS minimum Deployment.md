# [iOS]Deployment Target VS minimum Deployment

⚠️ Xcode가 변했는지 최근 Xcdoe에는 Project Info에서 Deployment Target 과 minimum Deployment 의 구분이 사라진 듯 하다.

<img width="1374" height="786" alt="스크린샷_2025-08-22_오후_1 42 23" src="https://github.com/user-attachments/assets/0546d832-d320-4633-bfbb-a451a0fc42ce" />

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
            <img width="993" height="483" alt="스크린샷_2025-08-22_오후_11 25 49" src="https://github.com/user-attachments/assets/bdf5b246-e859-4f19-8a06-7fd76841c8ba" />
            

### References
[Configuring the build settings of a target | Apple Developer Documentation](https://developer.apple.com/documentation/Xcode/configuring-the-build-settings-of-a-target)

[Xcode target Deployment Target vs. project Deployment Target](https://stackoverflow.com/questions/7105127/xcode-target-deployment-target-vs-project-deployment-target)
