# [CoreBluetooth] BLE Write Type 정리

iOS에서 BLE 기기에 데이터를 쓸 때 `.withResponse` / `.withoutResponse` 중 무엇을 쓸지 결정해야 한다. 그런데 `.withoutResponse`로 보냈는데 아무 일도 일어나지 않고, 에러 콜백조차 오지 않는 경우가 있다. Android에서는 잘 되는데 iOS에서만 무시되는 상황도 흔하다. 그 이유를 정리한다.

---

## 1. GATT와 Characteristic Properties

- **GATT(Generic Attribute Profile)** 는 BLE 기기가 제공하는 Service와 Characteristic, 그리고 각 데이터에 허용된 동작을 정의하는 명세다.
- iPhone은 일반적으로 **GATT Client**, 연결된 BLE 기기(Peripheral)는 **GATT Server**로 동작한다.
- Characteristic의 `properties`는 해당 데이터에 허용된 Read / Write / Notify 등의 방식을 비트 플래그로 나타낸다.

| 속성 | 값 | 의미 |
|---|---:|---|
| Read | `0x02` | 값 읽기 |
| Write Without Response | `0x04` | 응답 없는 쓰기 |
| Write | `0x08` | 응답 있는 쓰기 |
| Notify | `0x10` | 알림 수신 |
| Indicate | `0x20` | 수신 확인이 있는 알림 |

비트 플래그이므로 값을 더해서 읽으면 된다.

- `properties = 0x18` → `Write(0x08) + Notify(0x10)` → **Write Without Response는 지원하지 않음**
- `properties = 0x1C` → `Write Without Response(0x04) + Write(0x08) + Notify(0x10)` → 두 Write 방식 모두 지원

즉 Peripheral이 `0x04`를 선언하지 않았다면, 앱에서 아무리 `.withoutResponse`를 요청해도 그 Characteristic은 응답 없는 쓰기를 지원하지 않는 것이다.

---

## 2. 두 Write 방식의 차이

### With Response

- ATT **Write Request / Response**를 사용한다.
- 성공 또는 오류를 `peripheral(_:didWriteValueFor:error:)`에서 확인할 수 있다.
- 신뢰성이 높지만 매 전송마다 응답을 기다리므로 처리량(throughput)이 낮아질 수 있다.

### Without Response

- ATT **Write Command**를 사용한다.
- Peripheral의 성공·실패 응답이 없다. → **콜백도 없다.**
- 빠른 연속 전송에 유리하지만, 성공 확인이 필요하면 Notify나 별도의 애플리케이션 레벨 ACK 프로토콜이 필요하다.

---

## 3. iOS에서 요청이 조용히 무시되는 이유

iOS CoreBluetooth는 `.withoutResponse` 호출 전에 Characteristic의 `writeWithoutResponse` 속성을 **먼저 검사한다.** `0x04`가 선언되어 있지 않으면 다음과 같은 경고를 콘솔에 출력하고 요청을 **로컬에서 폐기한다.** 패킷 자체가 나가지 않는다.

```text
does not specify the "Write Without Response" property
- ignoring response-less write
```

에러 콜백이 없는 API이므로 앱 입장에서는 "보냈는데 반응이 없다"로만 보인다.

> Android에서 동작하더라도, 일부 Android BLE 스택이 GATT 선언을 엄격하게 검사하지 않고 그대로 ATT Write Command를 보내는 것일 수 있다. **Android에서 되는 것은 iOS에서도 된다는 보장이 아니다.**

정상적인(비탈옥) iOS 기기의 공개 API로는 이 검사를 우회하거나 Raw ATT / HCI 패킷을 강제로 보낼 수 없다.

---

## 4. 구현 시 필수 확인

Characteristic discovery 직후 지원 속성을 확인한다.

```swift
let properties = characteristic.properties

let supportsWithResponse    = properties.contains(.write)
let supportsWithoutResponse = properties.contains(.writeWithoutResponse)
```

`.withoutResponse` 전송 전에는 아래 두 가지를 **각각 구분해서** 확인해야 한다.

```swift
// 1) GATT 지원 여부
guard characteristic.properties.contains(.writeWithoutResponse) else {
    // 지원하지 않으면 .withResponse로 폴백하거나 전송을 중단
    return
}

// 2) iOS 송신 버퍼 상태
guard peripheral.canSendWriteWithoutResponse else {
    // 큐에 보관하고 peripheralIsReady(toSendWriteWithoutResponse:)에서 재시도
    return
}

peripheral.writeValue(data, for: characteristic, type: .withoutResponse)
```

- `properties.contains(.writeWithoutResponse)` → **기능 지원 여부** (Peripheral의 GATT 선언)
- `canSendWriteWithoutResponse` → **현재 송신 버퍼 상태** (iOS 내부 상태). GATT 지원 여부를 협상하거나 우회하지 않는다.
- 한 번에 보낼 수 있는 최대 크기는 `maximumWriteValueLength(for:)`로 확인한다. 두 write type의 값이 다를 수 있다.

---

## 5. 유의사항

- 앱 자체 조건(기기 모델, 펌웨어 버전, 국가 코드 등)만으로 Write 타입을 결정하면 **실제 GATT 속성과 불일치할 수 있다.** 앱 정책이 `.withoutResponse`를 선택하는 것과 GATT가 그것을 지원하는 것은 별개의 문제다. 최종 판단은 항상 `characteristic.properties`를 기준으로 한다.
- `.withoutResponse`에는 성공 콜백이 없으므로, 중요한 명령에는 애플리케이션 수준 ACK(Notify 응답 등)를 설계에 포함한다.
- Peripheral 펌웨어가 Write Command를 실제로 처리하더라도, GATT에 `0x04`를 선언하지 않으면 iOS에서는 사용할 수 없다.
- 펌웨어에서 GATT 속성을 변경했는데 iOS에 이전 값이 보인다면 **GATT 캐시**를 의심한다. Peripheral 측의 Service Changed Indication 처리, 또는 재연결/재페어링이 필요할 수 있다.
- 결국 iOS에서 안정적으로 지원하려면 **Peripheral의 GATT 선언과 실제 처리 동작을 일치시켜야 한다.**

---

## 6. 결론

```text
앱 정책상 withoutResponse 대상
        +
GATT에 Write Without Response(0x04) 선언
        +
iOS 송신 버퍼 준비 (canSendWriteWithoutResponse)
        =
정상적인 withoutResponse 전송
```

세 조건 중 하나라도 충족되지 않으면 `.withoutResponse` 전송을 보장할 수 없다.

---

### References

[CBCharacteristic.properties | Apple Developer Documentation](https://developer.apple.com/documentation/corebluetooth/cbcharacteristic/properties)

[writeValue(_:for:type:) | Apple Developer Documentation](https://developer.apple.com/documentation/corebluetooth/cbperipheral/writevalue(_:for:type:))

[CBCharacteristicWriteType.withoutResponse | Apple Developer Documentation](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicwritetype/withoutresponse)
