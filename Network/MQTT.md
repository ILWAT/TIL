# MQTT

> [MQTT](https://mqtt.org/)는 게시 및 구독 브로커 아키텍처를 사용하여 양방향 메시지를 제공하는 연결된 기기 애플리케이션용 OASIS 표준 프로토콜입니다. MQTT 프로토콜은 네트워크 오버헤드를 줄이기 위한 경량 프로토콜로서 MQTT 클라이언트가 매우 작아 제한된 기기의 리소스 사용이 최소화됩니다
*- [Google의 MQTT 설명](https://cloud.google.com/architecture/connected-devices/mqtt-broker-architecture?hl=ko)*
> 

## MQTT

- Message Queuing Telemetry Transport
- Client - Server 게시/구독 형태 프로토콜
    - IoT(Internet of Things)기기와 같이 대역폭이 제한적이거나 네트워크가 불안정적인 환경에서 소량의 데이터를 통신하는 것에 중점을 둔 프로토콜
    - TCP/IP를 통해 설계
    - 대개 스마트홈, 헬스케어, 교통, 산업 자동화등에 쓰이는 프로토콜
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

바이너리 기반 프로토콜

명령/명령 ACK 포맷을 사용하여 통신

ex) 

- connect 요청 → Connect ACK
- 구독 요청 → 구독 ACK
- 게시 요청 → 게시 ACK

### 참고 문헌

[MQTT - The Standard for IoT Messaging](https://mqtt.org/)

[Beginners Guide To The MQTT Protocol](http://www.steves-internet-guide.com/mqtt/)