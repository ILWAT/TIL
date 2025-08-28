# [Swift] 통합 로깅 시스템 (Logger, os_log)

### 개요

개발을 진행하다 보면 디버깅을 위해 실제로 어떻게 동작하는지 로그를 확인할 필요가 있다.

대개 간단하게 print()를 사용하는데 print문을 통해 특정 메서드의 실행 여부, 실행 순서, 데이터 확인 등을 확인하기에는 충분하지만 치명적이게 아쉬운 점이 있다.

Xcode에서 앱을 실행하는 환경. Xcode 콘솔을 통해서만 확인이 가능하다는 점이다.

앱 개발을 하면서 개발 환경말고 QA환경, 또는 Release 환경 등 다양한 사람, 다양한 환경에서 사용하다가 발생하는 흔치 않은 이슈가 발생할 때가 있는데 이런 문제를 디버깅해야할 때에도 로그를 확인할 수 있었으면 좋겠다라는 생각이 들곤 한다.

이럴 때 유용한 것이 통합 로깅 시스템이다.

### 장점 (통합 로깅 시스템의 사용 이유)

통합 로깅 시스템에 대해서 톺아보기 위해서 퍼플렉시티에 물어보니 잘 정리해준 표를 그대로 인용 해보았다.

| 항목 | Print문 | Logger(통합 로깅 시스템) |
| --- | --- | --- |
| 목적 | 빠른 디버깅, 콘솔 출력 (개발자 임시 용도) | 운영 환경에서 구조화된 로그, 모니터링, 분석 |
| 출력 장소 | Xcode 콘솔 (디버깅 세션) | Xcode 콘솔, macOS Console 앱, 디바이스 로그 파일 등 |
| 로그 구조화 | 없음 (평문 메시지) | 서브시스템, 카테고리, 로그 레벨 등 구조화 |
| 로그 레벨 | 없음 (모든 로그가 동일하게 출력) | Default, Info, Debug, Error, Fault 등 다중 레벨 지원 |
| 프라이버시 | 모든 정보가 노출됨 | 사생활 정보 자동 마스킹 가능 (privacy 옵션) |
| 성능 | 빠르나, 빈번한 호출 시 성능 저하 | 메시지 병합, lazy interpolation 등으로 높은 성능 |
| 검색/필터링 | 기본 제공 안 됨 | Console 앱, `log` 명령어 등에서 강력한 필터링 가능 |
| 운영 환경 | 디버깅 환경에서만 주로 사용 | 개발·운영 환경 모두에서 사용, 장애 분석에 유리 |
| 커스터 마이즈 | 거의 불가 | 로거별 커스터마이즈, 확장 가능 |
| 데이터 유실 | 메모리 부족 등에서 유실될 수 있음 | 디스크/메모리 이중 저장으로 데이터 유실 최소화 |
| 공식 권장 | 간단한 디버깅용 | 모든 Apple 플랫폼에서 공식적으로 권장 |


이처럼 정말 다양한 기능들을 제공하여 이를 잘 활용하면 강력한 디버깅 도구가 되어준다.

### Log 파일 추출

```bash
sudo log collect --device-name 'apple의 iPhone'  --start '2025-08-27 12:00:00' --output extractLog.logarchive
```

### References

[Explore logging in Swift - WWDC20 - 비디오 - Apple Developer](https://developer.apple.com/kr/videos/play/wwdc2020/10168/)

[Logging | Apple Developer Documentation](https://developer.apple.com/documentation/os/logging)