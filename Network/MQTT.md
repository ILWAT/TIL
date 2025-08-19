# MQTT

> [MQTT](https://mqtt.org/)는 게시 및 구독 브로커 아키텍처를 사용하여 양방향 메시지를 제공하는 연결된 기기 애플리케이션용 OASIS 표준 프로토콜입니다. MQTT 프로토콜은 네트워크 오버헤드를 줄이기 위한 경량 프로토콜로서 MQTT 클라이언트가 매우 작아 제한된 기기의 리소스 사용이 최소화됩니다
*- [Google의 MQTT 설명](https://cloud.google.com/architecture/connected-devices/mqtt-broker-architecture?hl=ko)*
> 

## MQTT

- Message Queuing Telemetry Transport
- Client - Broker 게시/구독 형태 프로토콜
    - IoT 기기와 같이 대역폭이 제한적이거나 네트워크가 불안정한 환경에서 소량 데이터를 효율적으로 교환하도록 설계
    - TCP/IP 위에서 동작
    - 스마트홈, 헬스케어, 교통, 산업 자동화등에 널리 사용되는 프로토콜
- 핵심 구성 요소
    - MQTT Broker
    - MQTT Client
        - 게시자(Publisher)
        - 구독자(Subscriber)

### MQTT의 장점

- 가볍고 효율적임
    
    메시지 헤더 크기가 작은 등의 통신에 필요한 리소스가 적음.
    
- 양방향 통신
- 확장성
    
    수백만개의 연결된 장치를 지원
    
- 메시지 전달 보장
    
    3가지 QoS 레벨 설정을 통한 메시지 전달 안정성 확보 가능
    
    - **QoS 0**: At most once (최대 한 번) – 전달 보장 없음
    - **QoS 1**: At least once (최소 한 번) – 중복 가능
    - **QoS 2**: Exactly once (정확히 한 번) – 가장 안전하지만 오버헤드 큼
- 영구적인 세션을 통한 재 연결 최적화
    
    영구 세션을 통한 불안정적인(Unreliable) 환경에서 연결이 지속적으로 끊겼을 때, 재 연결 시간을 절약할 수 있도록 지원
    
- 보안 요소 적용 가능
    
    TLS를 통한 메시지 암호화 가능
    
    - OAuth와 같은 최신 인증 프로토콜을 사용할 수도 있음.

### MQTT Broker

발신자와 수신자 사이의 중개자로서, 구독자를 관리하고 구독자에게 메시지를 전송하는 역할을 수행.

### MQTT Client

토픽을 구독하여 메시지를 수신하거나 메시지를 게시하는 역할

### MQTT 프로토콜 패킷 구조

- 명령/명령 ACK 형식 통신
- Exmaple
    - connect 요청 → Connect ACK
    - 구독 요청 → 구독 ACK
    - 게시 요청 → 게시 ACK

### MQTT Topic

- MQTT에서 토픽은 게시자와 구독자 간에 메시지를 라우팅하는 방식의 핵심 요소.
    - 메시지가 전달될 위치를 정의하는 주소의 역할
    - UTF-8 문자열 형태로 표현
- 주로 계층적으로 구성되며 각 계층은 슬래시(/) 로 구분된다 (파일 경로와 유사)
    - 각 계층은 topic level이라고 칭한다.

- 예시
    - southkorea/seoul/gangnam/temperature
        - 서울시 강남구의 온도를 모니터링 하는데 사용
    - company/device/12345678/status
        - 12345678 기기의 시스템 상태를 모니터링하는데 사용

- Topic Name VS Topic Filter
    - Topic Name
        - 게시자가 메시지를 보낼 때 사용하는 정확한 주소
    - Topic Filter
        - 구독자가 메시지를 받고 싶을 때 사용하는 패턴
        - **와일드 카드를 통해 여러 토픽을 동시에 구독 가능**

### MQTT Wildcard

- WildCard를 통해 여러 토픽을 동시에 구독 가능
- single-level과 multi-level이 있음
- single-level
    - 더하기(+) 기호를 통해 하나의 topic level을 대체 가능
    - 대체된 topic level에 대응하는 모든 topic을 구독
    - 예시
        - southkorea/seoul/+/temperature
            - 서울시의 모든 구에 대한 온도를 모니터링 가능
            - ✅ southkorea/seoul/gangnam/temperature
            - ✅ southkorea/seoul/nowon/temperature
            - ✅ southkorea/seoul/jongno/temperature
            - ❌ southkorea/seoul/gangnam/sinsa/temperature
- multi-level
    - 해시(#) 기호를 마지막 topic level에 붙여 동일 레벨 및 하위 레벨까지 구독 가능
    - 예시
        - southkorea/seoul/#
            - 서울시의 모든 구에 대한 온도를 모니터링 가능
            - ✅ southkorea/seoul/gangnam/temperature
            - ✅ southkorea/seoul/jongno/temperature
            - ✅ southkorea/seoul/jongno/humidity
            - ❌ southkorea/chungnam/gangnam/temperature
                - 실제론 충남에 강남이 없지만 단순 예시

### 참고 문헌

[MQTT - The Standard for IoT Messaging](https://mqtt.org/)

[MQTT Essentials: Your 2025 Learning Hub for IoT & IIoT Data Streaming](https://www.hivemq.com/mqtt/)

[MQTT Client, MQTT Broker, and MQTT Server Connection Establishment Explained – MQTT Essentials: Part 3](https://www.hivemq.com/blog/mqtt-essentials-part-3-client-broker-connection-establishment/)

[MQTT Topics, Wildcards, & Best Practices – MQTT Essentials: Part 5](https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/)

[Beginners Guide To The MQTT Protocol](http://www.steves-internet-guide.com/mqtt/)

[Understanding the MQTT Protocol Packet Structure](http://www.steves-internet-guide.com/mqtt-protocol-messages-overview/)
