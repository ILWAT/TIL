# WebP

> WebP는 웹상의 이미지를 위해 뛰어난 무손실 압축과 손실 압축을 제공하는 최신 이미지 형식입니다. WebP를 사용하면 웹마스터와 웹 개발자가 더 작고 풍부한 이미지를 만들어 웹의 속도를 높일 수 있습니다.
> *- [Google Developers, An image format for the Web](https://developers.google.com/speed/webp)*

## WebP란

- 2010년 Google이 발표한 이미지 포맷 (확장자 `.webp`, MIME 타입 `image/webp`)
- 영상 코덱 **VP8의 키 프레임(인트라 프레임) 압축 기술**을 정지 이미지에 적용한 것에서 출발
    - 컨테이너는 **RIFF** 포맷을 사용 (`RIFF....WEBP` 헤더로 시작)
- JPEG / PNG / GIF가 각각 담당하던 역할을 **하나의 포맷으로 통합**하는 것이 목표
    - 손실 압축 (JPEG 대체)
    - 무손실 압축 (PNG 대체)
    - 알파 채널(투명도) 지원
    - 애니메이션 지원 (GIF 대체)
    - EXIF / ICC 프로파일 등 메타데이터 지원

### 압축률 (Google 공식 수치)

| 비교 대상 | 절감 효과 |
| :--- | :--- |
| 무손실 WebP vs PNG | 약 **26%** 더 작음 |
| 손실 WebP vs JPEG | 동일 SSIM 기준 약 **25~34%** 더 작음 |
| 무손실 + 알파 vs PNG | 알파 채널 자체를 약 **22%** 더 작게 저장 |

> ⚠️ 위 수치는 벤더(Google)가 자체 벤치마크로 발표한 값입니다. 실제 절감률은 이미지의 종류(사진 / 일러스트 / 스크린샷)와 품질 파라미터에 따라 크게 달라지므로, **도입 전 자신의 에셋으로 직접 측정**해 보는 것이 좋습니다.

---

## 압축 방식

<details>
<summary><b>1. 손실 압축 (Lossy)</b> — 예측 부호화 + DCT + 산술 부호화</summary>

<br>

VP8의 인트라 프레임 인코딩과 동일한 방식으로 동작합니다.

- **예측 부호화(Predictive Coding)**
    - 이미지를 블록(매크로블록) 단위로 나눔
    - 이미 복원된 **인접 블록의 픽셀 값으로 현재 블록을 예측**하고, 예측값과 실제값의 **차이(residual)만 저장**
    - 인접 픽셀은 대체로 비슷하다는 이미지의 공간적 중복성을 이용
- **변환 및 양자화**
    - residual에 DCT(이산 코사인 변환) / WHT(Walsh-Hadamard 변환)를 적용하고 양자화
    - 이 양자화 단계에서 정보 손실이 발생 (= 품질 파라미터가 조절하는 지점)
- **엔트로피 부호화**
    - 최종 계수를 **불리언 산술 부호화(Boolean Arithmetic Coding)** 로 압축
    - JPEG이 쓰는 허프만 부호화보다 일반적으로 압축률이 높음

</details>

<details>
<summary><b>2. 무손실 압축 (Lossless)</b> — 공간 예측 + LZ77 + 허프만 + 컬러 캐시</summary>

<br>

- 픽셀을 완전히 동일하게 복원하며, 다음 기법들을 조합해서 사용
    - **공간 예측 변환**: 주변 픽셀로 현재 픽셀을 예측하고 차이만 기록 (14가지 예측 모드)
    - **색 변환(Color Decorrelation)**: R, G, B 채널 간 상관관계를 제거
    - **컬러 인덱싱**: 사용된 색이 적으면 팔레트로 치환
    - **LZ77 + 허프만 부호화**: 반복 패턴을 백워드 레퍼런스로 치환 후 엔트로피 부호화
    - **컬러 캐시**: 최근 사용한 색을 해시 테이블에 캐싱해서 짧은 코드로 참조

</details>

<details>
<summary><b>3. 알파 채널</b> — 손실 / 무손실 모두 8비트 투명도 지원</summary>

<br>

- 손실 / 무손실 **양쪽 모두에서 8비트 알파 채널을 지원**
    - JPEG은 투명도를 아예 지원하지 못하고, PNG는 무손실뿐이라는 점과 대비되는 강점
- 내부적으로 **컬러는 손실 압축 + 알파는 무손실 압축** 조합이 가능
    - 반투명 UI 리소스, 아이콘 등에서 PNG 대비 용량을 크게 줄일 수 있는 이유

</details>

---

## 다른 포맷과의 비교

| 포맷 | 손실 | 무손실 | 투명도 | 애니메이션 | 비고 |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **JPEG** | ✅ | ❌ | ❌ | ❌ | 사진용 표준, 프로그레시브 로딩 지원 |
| **PNG** | ❌ | ✅ | ✅ | ❌ | 무손실 + 투명도, 사진에는 용량이 큼 |
| **GIF** | ❌ | ✅ | 1비트 | ✅ | 256색 제한, 애니메이션 용량이 매우 큼 |
| **WebP** | ✅ | ✅ | ✅ (8비트) | ✅ | 위 셋을 대체 가능, 지원 범위 넓음 |
| **AVIF** | ✅ | ✅ | ✅ | ✅ | AV1 기반, WebP보다 압축률↑ / 인코딩 느림 |
| **HEIC** | ✅ | ✅ | ✅ | ✅ | HEVC 기반, Apple 기본 포맷, 라이선스 이슈 |

### WebP의 한계

- **손실 WebP는 YUV 4:2:0 크로마 서브샘플링만 지원**
    - 색 정보를 가로/세로 절반으로 줄여서 저장하므로, 붉은 계열 텍스트나 선명한 색 경계에서 번짐이 발생할 수 있음
    - 그래서 텍스트가 많은 스크린샷 이미지는 손실 WebP보다 **무손실 WebP나 PNG**가 나을 때가 많음
- **8비트 색심도만 지원** → HDR / 10비트 이상 이미지는 표현 불가 (AVIF, HEIC와 차이나는 지점)
- **프로그레시브 / 인터레이스 디코딩이 없음**
    - 프로그레시브 JPEG처럼 저해상도가 먼저 뜨는 연출이 불가능하고, 전부 받아야 표시됨
- **최대 크기가 16383 × 16383 픽셀**로 제한됨
- **품질을 아주 높게(q 90 이상) 잡으면** MozJPEG 같은 최적화된 JPEG 인코더보다 오히려 용량이 커질 수 있음
- 인코딩 속도가 JPEG보다 느림 (다만 AVIF보다는 훨씬 빠름)

---

## 지원 현황

### 브라우저

| 브라우저 | 지원 버전 |
| :--- | :--- |
| Chrome / Opera | 초기부터 지원 |
| Firefox | 65+ |
| Edge | 18+ |
| Safari | **14+** (macOS Big Sur, iOS 14) |

- 사실상 최신 브라우저에서는 모두 사용 가능하지만, 구형 환경을 지원해야 한다면 폴백이 필요

### 모바일 / OS

- **iOS**: iOS 14부터 Image I/O가 WebP 디코딩을 기본 지원 → `UIImage(data:)`로 바로 로드 가능
    - iOS 13 이하 지원이나 인코딩이 필요하다면 `libwebp` 기반 라이브러리를 사용
    - 예: [SDWebImageWebPCoder](https://github.com/SDWebImage/SDWebImageWebPCoder), Nuke의 WebP 플러그인
- **Android**: API 14(4.0)부터 손실 WebP, API 17(4.2.1)부터 무손실 / 투명도 지원
    - 애니메이션 WebP 디코딩은 API 28(9.0)의 `ImageDecoder`부터
    - Android Studio에서 드로어블을 WebP로 일괄 변환하는 기능 제공

---

## 실전 사용법

### 1. HTML `<picture>` 폴백

지원하지 않는 브라우저를 위해 원본 포맷을 함께 제공합니다. 브라우저는 위에서부터 순서대로 확인해 **자신이 디코딩 가능한 첫 번째 소스**를 선택합니다.

```html
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg"  type="image/jpeg">
  <img src="image.jpg" alt="설명" width="800" height="600">
</picture>
```

### 2. 서버 콘텐츠 협상 (Content Negotiation)

브라우저가 보내는 `Accept` 헤더를 보고 서버가 포맷을 골라 응답하는 방식입니다.

```http
# Request
GET /image.jpg
Accept: image/avif,image/webp,image/apng,*/*

# Response
Content-Type: image/webp
Vary: Accept
```

> ⚠️ **`Vary: Accept` 헤더는 필수**입니다. 이게 없으면 CDN이나 프록시가 WebP 응답을 캐싱한 뒤 WebP를 지원하지 않는 클라이언트에게도 그대로 내려줘서 이미지가 깨집니다.

### 3. CLI 변환 (libwebp)

```bash
# 손실 압축 (품질 0~100, 기본값 75)
cwebp -q 80 input.png -o output.webp

# 무손실 압축
cwebp -lossless input.png -o output.webp

# 니어 로스리스 (눈에 안 띄는 수준으로만 손실을 허용해 용량 절감)
cwebp -near_lossless 60 input.png -o output.webp

# 알파 채널 품질만 따로 지정
cwebp -q 80 -alpha_q 100 input.png -o output.webp

# WebP → PNG 디코딩
dwebp output.webp -o restored.png

# GIF → 애니메이션 WebP
gif2webp -q 80 input.gif -o output.webp
```

macOS에서는 `brew install webp`로 설치할 수 있습니다.

---

## 언제 쓰고, 언제 쓰지 말아야 할까

**쓰면 좋은 경우**

- 웹/앱의 일반적인 사진, 썸네일, 배너 이미지 → JPEG 대비 확실한 용량 이득
- **투명도가 있는 이미지** → PNG 대비 절감 폭이 가장 큰 영역
- 애니메이션 → GIF 대비 압축률이 압도적 (수 배 차이가 흔함)

**다시 생각해 볼 경우**

- 텍스트·도표가 많고 색 경계가 선명한 이미지 → 손실 WebP의 4:2:0 서브샘플링 때문에 번질 수 있음
- 원본 보관용 마스터 이미지 → 8비트 제한, 편집 툴 호환성 이슈
- 아이콘·단순 도형 → **SVG**가 거의 항상 더 나은 선택
- 최신 브라우저만 대응하면 되고 인코딩 시간에 여유가 있다면 → **AVIF**가 더 작음

---

## 참고: CVE-2023-4863

2023년 `libwebp`의 무손실 디코더(허프만 테이블 생성부)에서 **힙 버퍼 오버플로 취약점**이 발견되어 실제 공격에 악용되었습니다. Chrome, Firefox, Electron 등 libwebp를 내장한 거의 모든 소프트웨어가 영향을 받아 긴급 패치가 배포되었습니다.

- 이미지 포맷 라이브러리는 **신뢰할 수 없는 외부 입력을 파싱하는 코드**라는 점을 상기시켜 준 사례
- 앱에 libwebp를 직접 번들링하고 있다면 **의존성 버전 관리를 게을리하지 말 것**

---

## 정리

- WebP = VP8 기반의 이미지 포맷으로, **손실 / 무손실 / 알파 / 애니메이션을 하나로 통합**
- 손실은 예측 부호화 + DCT + 산술 부호화, 무손실은 예측 변환 + LZ77 + 허프만 조합
- JPEG 대비 25~34%, PNG 대비 약 26% 절감 (단, 실제 에셋으로 검증 필요)
- 약점은 **4:2:0 고정, 8비트 한정, 프로그레시브 미지원**
- Safari 14 / iOS 14부터 지원되어 현재는 사실상 전 플랫폼에서 사용 가능
- 서버에서 협상 방식으로 내려줄 땐 **`Vary: Accept`를 잊지 말 것**

---

### 참고 문헌

[WebP  |  An image format for the Web  |  Google Developers](https://developers.google.com/speed/webp)

[Compression Techniques  |  WebP](https://developers.google.com/speed/webp/docs/compression)

[WebP Container Specification](https://developers.google.com/speed/webp/docs/riff_container)

[cwebp - Command line tool documentation](https://developers.google.com/speed/webp/docs/cwebp)

[WebP image format - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types#webp)

[Using WebP images - Android Developers](https://developer.android.com/studio/write/convert-webp)

[CVE-2023-4863 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-4863)
