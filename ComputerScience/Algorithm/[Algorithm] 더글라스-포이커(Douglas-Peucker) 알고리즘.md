# [Algorithm] 더글라스-포이커(Douglas-Peucker) 알고리즘

> 선의 형태를 최대한 보존하면서 점의 개수를 줄여 단순화(Simplification)하는 대표적인 기하 알고리즘

## 개요

- 곡선(폴리라인)을 이루는 점들 중 **형태에 기여하지 않는 점을 골라내 제거**하는 알고리즘
- 1972년 **Urs Ramer**, 1973년 **David Douglas & Thomas Peucker**가 각각 독립적으로 제안
    - 그래서 **RDP(Ramer-Douglas-Peucker)** 알고리즘이라고도 부름
    - *iterative end-point fit*, *split-and-merge* 라는 이름으로도 불림
- 결과물은 항상 **원본 점들의 부분집합(subset)**
    - 새로운 좌표를 만들어내지 않으므로, 각 점에 붙어 있던 메타데이터(타임스탬프, 속도, 고도)가 그대로 유효
    - 이 성질 때문에 GPS 트랙 경량화에 특히 잘 맞음

---

## 왜 필요한가 — 맵 트랙라인의 리소스 문제

GPS는 보통 1초에 1개(1Hz) 좌표를 기록합니다. 한 시간을 달리면 **3,600개**, 하루 종일 기록하면 수만 개의 좌표가 쌓입니다.

문제는 이 좌표 대부분이 **화면에서는 구분조차 되지 않는다**는 점입니다.

- 직선 구간을 1초 간격으로 찍은 좌표 100개는, 지도상에서는 **양 끝점 2개와 똑같이 보임**
- 줌 아웃 상태에서는 1픽셀에 수십 개의 좌표가 겹쳐 그려짐

그런데 비용은 그대로 발생합니다.

| 항목 | 영향 |
| :--- | :--- |
| 렌더링 | `MKPolyline` / `Polyline` 이 점 개수만큼 선분을 그림 → 프레임 드랍, 스크롤 버벅임 |
| 메모리 | 좌표 배열 + 렌더러 내부 버퍼가 점 개수에 비례해 증가 |
| 네트워크 | 서버로 트랙을 올리거나 받을 때 페이로드가 그대로 커짐 |
| 저장 공간 | DB row 수 / 파일 크기 증가 |

**사람 눈에 보이지 않는 점을 미리 걷어내자**는 것이 트랙라인 단순화의 목적이고, 그 표준 도구가 더글라스-포이커입니다.

---

## 동작 원리

허용 오차 임계값 $\epsilon$ (Epsilon)을 기준으로 **분할 정복(Divide & Conquer)** 방식으로 동작합니다.

### 1. 기준선 설정

곡선의 **시작점**과 **끝점**을 잇는 가상의 직선을 긋습니다. 이 두 점은 **절대 제거되지 않습니다.**

### 2. 최대 거리 점 탐색

시작점과 끝점 사이의 점들 중, 기준선과의 **수직 거리가 가장 먼 점**을 찾습니다.

### 3. 임계값 비교 및 분할

- 최대 거리 $\le \epsilon$ → 중간 점들이 형태에 영향을 주지 않는다고 보고 **모두 제거**
- 최대 거리 $> \epsilon$ → 그 점은 **형태를 결정하는 점**이므로 **보존**하고, 그 점을 기준으로 곡선을 **두 구간으로 분할**

### 4. 반복

분할된 두 구간에 대해 각각 1~3단계를 재귀적으로 반복합니다.

```
[1단계] 시작점 A, 끝점 B를 잇고 가장 먼 점 C를 찾는다

            C
            ●  ← dmax > ε  → C 보존, [A~C] / [C~B] 로 분할
        ●      ●
      ●            ●
A ●----------------------● B
      ↑ 기준선

[2단계] 각 구간에서 다시 반복

            C
      ●     ●     ●  ← 모두 dmax ≤ ε  → 중간 점 전부 제거
A ●---'-----●-----'---● B

[결과] A - C - B  (5개 → 3개)
```

### 핵심 성질

이 방식 덕분에 **제거된 모든 원본 점은 단순화된 선분으로부터 $\epsilon$ 이내**임이 보장됩니다. "가장 먼 점이 $\epsilon$ 이하면 나머지는 당연히 그 이하"이기 때문입니다.

즉 $\epsilon$ 은 **"이만큼의 오차는 감수하겠다"는 값**이고, 그 안에서 최대한 점을 줄여줍니다.

---

## 수직 거리 계산

점 $P$ 에서 두 점 $A, B$ 를 지나는 직선까지의 거리는 2D 외적(cross product)으로 구합니다.

$$d = \frac{|(B - A) \times (P - A)|}{|B - A|}$$

$$(B - A) \times (P - A) = (B_x - A_x)(P_y - A_y) - (B_y - A_y)(P_x - A_x)$$

외적의 절댓값은 세 점이 이루는 평행사변형의 넓이이고, 이를 밑변 $|B - A|$ 로 나누면 높이(= 수직 거리)가 됩니다.

> ⚠️ $A = B$ 인 경우(시작점과 끝점이 같은 닫힌 경로 등) 직선이 정의되지 않으므로 `hypot(P - A)` 로 대체해야 합니다. 이 예외 처리를 빠뜨리면 0으로 나누기가 발생합니다.

**직선까지의 거리 vs 선분까지의 거리**

원논문은 **무한 직선까지의 수직 거리**를 사용하지만, 구현체에 따라 **선분까지의 거리**(수선의 발이 선분 밖이면 가까운 끝점까지의 거리)를 쓰기도 합니다. 후자가 조금 더 보수적(점이 덜 지워짐)이며, 대부분의 트랙 데이터에서는 결과 차이가 미미합니다.

---

## 의사 코드

```
DouglasPeucker(points, ε):
    dmax  = 0
    index = 0
    end   = points.length

    # 시작점과 끝점을 잇는 직선에서 가장 먼 점 찾기
    for i = 1 to end - 2:
        d = perpendicularDistance(points[i], Line(points[0], points[end - 1]))
        if d > dmax:
            index = i
            dmax  = d

    if dmax > ε:
        # 가장 먼 점을 기준으로 분할 후 재귀
        left  = DouglasPeucker(points[0 ... index], ε)
        right = DouglasPeucker(points[index ... end - 1], ε)
        return left[0 ... left.length - 2] + right   # 겹치는 index 점 하나 제거
    else:
        # 중간 점 전부 버리고 양 끝점만 남김
        return [points[0], points[end - 1]]
```

---

## Swift 구현 (MapKit)

실무에서는 **재귀 대신 명시적 스택**을 쓰고, **위경도가 아닌 투영 좌표계에서** 계산하는 편이 안전합니다. (이유는 아래 "주의할 점" 참고)

```swift
import CoreLocation
import MapKit

extension Array where Element == CLLocationCoordinate2D {

    /// 더글라스-포이커 단순화
    /// - Parameter tolerance: 허용 오차 (meter)
    func simplified(tolerance: CLLocationDistance) -> [CLLocationCoordinate2D] {
        guard count > 2 else { return self }

        // 위경도(degree)로 바로 계산하면 위도에 따라 경도 1도의 실제 거리가 달라진다.
        // Web Mercator 평면(MKMapPoint)으로 옮겨서 계산한다.
        let points = map { MKMapPoint($0) }
        let midLatitude = self[count / 2].latitude
        let epsilon = tolerance * MKMapPointsPerMeterAtLatitude(midLatitude)

        var keep = [Bool](repeating: false, count: count)
        keep[0] = true
        keep[count - 1] = true

        // 트랙 포인트가 수만 개면 재귀는 스택 오버플로 위험이 있다.
        var stack: [(first: Int, last: Int)] = [(0, count - 1)]

        while let (first, last) = stack.popLast() {
            guard last > first + 1 else { continue }

            var maxDistance = 0.0
            var index = first

            for i in (first + 1)..<last {
                let distance = perpendicularDistance(points[i],
                                                     from: points[first],
                                                     to: points[last])
                if distance > maxDistance {
                    maxDistance = distance
                    index = i
                }
            }

            if maxDistance > epsilon {
                keep[index] = true
                stack.append((first, index))
                stack.append((index, last))
            }
        }

        return zip(self, keep).compactMap { $1 ? $0 : nil }
    }
}

/// 점 p에서 a-b가 놓인 직선까지의 수직 거리
private func perpendicularDistance(_ p: MKMapPoint,
                                   from a: MKMapPoint,
                                   to b: MKMapPoint) -> Double {
    let dx = b.x - a.x
    let dy = b.y - a.y

    // 시작점과 끝점이 같으면 직선이 정의되지 않는다
    guard dx != 0 || dy != 0 else {
        return hypot(p.x - a.x, p.y - a.y)
    }

    // |(b - a) × (p - a)| / |b - a|
    let cross = dx * (p.y - a.y) - dy * (p.x - a.x)
    return abs(cross) / hypot(dx, dy)
}
```

사용:

```swift
let simplified = trackPoints.simplified(tolerance: 5) // 5m 오차 허용
let polyline = MKPolyline(coordinates: simplified, count: simplified.count)
mapView.addOverlay(polyline)
```

---

## 시간 복잡도

| 구분 | 복잡도 | 설명 |
| :--- | :--- | :--- |
| 평균 | $O(n \log n)$ | 분할이 대체로 균등하게 일어나는 경우 |
| 최악 | $O(n^2)$ | 매번 한쪽 끝에서 분할되어 구간이 1개씩만 줄어드는 경우 |
| 개선 | $O(n \log n)$ | Hershberger & Snoeyink(1992), convex hull 기반 *path hull* 자료구조 사용 |

일반적인 구현은 단순한 $O(n^2)$ 최악 버전을 씁니다. 트랙 포인트 수만 개 수준에서는 충분히 빠르고, 대신 아래 전처리로 실효 성능을 크게 올릴 수 있습니다.

### 전처리: Radial Distance 필터

DP를 돌리기 전에 **직전에 유지한 점과의 거리가 tolerance 미만인 점을 O(n)으로 먼저 제거**하면, DP에 들어가는 $n$ 자체가 크게 줄어듭니다.

```swift
// 정차 중 제자리에서 튀는 GPS 좌표 등을 먼저 걷어낸다
var result: [CLLocationCoordinate2D] = [points[0]]
for point in points.dropFirst() where distance(point, result.last!) > tolerance {
    result.append(point)
}
```

- Leaflet / simplify-js 가 실제로 쓰는 방식 (`simplify(points, tolerance, highQuality)` 의 저품질 모드)
- 정차 구간이 많은 GPS 트랙에서는 이 한 번의 패스만으로 절반 이상이 사라지는 경우도 흔함

---

## $\epsilon$ 값은 어떻게 정하나

$\epsilon$ 은 **"화면에서 몇 픽셀까지의 오차를 허용할 것인가"** 로부터 역산하는 것이 가장 실용적입니다.

Web Mercator 기준 줌 레벨 $z$ 에서의 해상도(미터/픽셀)는 다음과 같습니다.

$$\text{resolution} = \frac{156543.03392 \times \cos(\text{latitude})}{2^{z}}$$

```swift
func epsilon(forZoom zoom: Int, latitude: Double, pixelTolerance: Double = 1.0) -> Double {
    let resolution = 156543.03392 * cos(latitude * .pi / 180) / pow(2, Double(zoom))
    return resolution * pixelTolerance
}
```

- 줌 10, 위도 37.5° → 약 **124 m/px** → $\epsilon \approx 124m$ 이어도 화면상 차이는 1px
- 줌 18 → 약 **0.48 m/px** → $\epsilon \approx 0.5m$ 수준이 필요

**전략**

- **줌 레벨별로 미리 여러 벌 만들어 두기**: 서버에서 저/중/고 3~4단계로 만들어 캐싱하고, 클라이언트가 현재 줌에 맞는 것을 요청 (벡터 타일이 쓰는 방식)
- **한 벌만 쓸 거라면 최대 줌 기준**: 확대했을 때 각지게 보이는 것이 가장 눈에 띄므로 보수적으로 잡는 편이 안전
- **원본은 반드시 보관**: 거리·페이스 계산은 단순화 전 데이터로 해야 함 (아래 참고)

---

## 주의할 점

<details>
<summary><b>1. 위경도(degree)를 그대로 넣으면 안 된다</b></summary>

<br>

위도 1도는 어디서나 약 111km지만, **경도 1도는 위도에 따라 줄어듭니다.** (적도 111km → 위도 60°에서 약 55.5km)

degree 좌표로 유클리드 거리를 계산하면 **동서 방향 오차를 실제보다 크게 평가**하게 되어, 고위도로 갈수록 세로선만 과하게 단순화되는 왜곡이 생깁니다.

- **투영 좌표계로 변환 후 계산** (`MKMapPoint`, Web Mercator, UTM 등)
- 특히 지도 렌더링이 목적이라면 Mercator 평면에서 계산하는 것이 **화면 픽셀 오차와 정확히 대응**되므로 오히려 더 정확
- 실제 미터 오차가 중요하면 Haversine 기반 수직 거리를 쓰거나, 로컬 평면(ENU)으로 변환

</details>

<details>
<summary><b>2. 토폴로지(Topology)가 깨질 수 있다</b></summary>

<br>

DP는 각 구간을 **독립적으로** 처리하기 때문에, 단순화 결과가 **자기 자신과 교차(self-intersection)** 하거나 인접한 다른 선과의 관계가 깨질 수 있습니다.

- 되돌아오는 왕복 트랙, 트랙 필드를 도는 트랙 등에서 발생하기 쉬움
- 행정구역 경계처럼 **여러 폴리곤이 변을 공유**하는 데이터에서는 틈(gap)이나 겹침(overlap)이 생김

→ 토폴로지 보존이 필요하면 PostGIS `ST_SimplifyPreserveTopology()` 같은 토폴로지 인식 알고리즘을 사용해야 합니다.

</details>

<details>
<summary><b>3. 단순화된 데이터로 거리를 계산하면 안 된다</b></summary>

<br>

점을 지웠으므로 **총 이동 거리는 항상 원본보다 짧아집니다.** 단순화는 어디까지나 **표시(rendering)와 전송용**이고, 통계 계산은 원본으로 해야 합니다.

- 서버: 원본 저장 + 단순화 버전을 캐시 컬럼/테이블로 별도 보관
- 클라이언트: 거리·페이스·고도 계산 후 → 렌더링 직전에 단순화

</details>

<details>
<summary><b>4. 재귀 깊이 (스택 오버플로)</b></summary>

<br>

최악의 경우 재귀 깊이가 $O(n)$ 까지 갑니다. 트랙 포인트가 수만 개인 상황에서 순수 재귀 구현은 스택 오버플로로 크래시할 수 있습니다.

→ 위 Swift 예제처럼 **명시적 스택(explicit stack)** 으로 바꾸면 해결됩니다.

</details>

<details>
<summary><b>5. GPS 노이즈는 DP로 걸러지지 않는다</b></summary>

<br>

DP는 **"기준선에서 가장 먼 점"을 보존**하는 알고리즘입니다. 즉 튀어나간 GPS 이상치(outlier)는 정확히 "가장 먼 점"이므로 **오히려 우선적으로 살아남습니다.**

→ 노이즈 제거(칼만 필터, 정확도 기반 필터링, 속도 임계값 필터)를 **먼저** 하고, 그 다음에 DP를 적용해야 합니다.

</details>

---

## 다른 단순화 알고리즘과의 비교

| 알고리즘 | 기준 | 특징 |
| :--- | :--- | :--- |
| **Douglas-Peucker** | 기준선까지의 **수직 거리** | 전체 형태와 극점(모서리) 보존에 강함. 사실상의 표준 |
| **Visvalingam-Whyatt** | 연속 3점이 이루는 **삼각형 넓이** | 넓이가 작은 점부터 제거. 결과가 시각적으로 더 자연스럽고, **점 개수를 직접 지정**할 수 있음 |
| **Radial Distance** | 직전 점과의 **거리** | $O(n)$, 매우 빠름. 형태 보존력은 약해 **전처리용**으로 사용 |
| **Reumann-Witkam** | 진행 방향 기준 **띠(corridor)** | $O(n)$ 스트리밍 처리 가능. 실시간 기록 중 적용에 유리 |
| **Topology-preserving** | 위상 관계 | 자기 교차·인접 폴리곤 관계를 보존. 느림 |

> Visvalingam-Whyatt는 **"점을 몇 개까지 줄일지"** 를 직접 정할 수 있어서, 응답 크기 상한이 정해진 API 설계에 유리합니다. 반대로 DP는 **"오차를 얼마까지 허용할지"** 를 정하는 방식이라 정확도 보장이 필요할 때 유리합니다.

---

## 실제 사용처

| 분야 | 사례 |
| :--- | :--- |
| GIS / 지도 | PostGIS `ST_Simplify()`, Mapbox 벡터 타일 생성, Leaflet 폴리라인 렌더링 |
| GPS 트래킹 | Strava, Nike Run Club 등 러닝/사이클 앱의 경로 저장·전송 |
| 컴퓨터 비전 | OpenCV `approxPolyDP()` — 윤곽선(Contour) 다각형 근사, 도형 인식 |
| 벡터 그래픽 | 펜/브러시 스트로크 경로 단순화, SVG path 최적화 |
| 라이브러리 | Turf.js `simplify`, simplify-js, Shapely `simplify()` |

> 트랙 전송량을 더 줄이고 싶다면 DP로 점을 줄인 뒤 **Google Encoded Polyline** 같은 인코딩을 얹는 조합이 일반적입니다. 알고리즘으로 점 개수를 줄이고, 인코딩으로 점당 바이트 수를 줄이는 2단 최적화입니다.

---

## 정리

- 더글라스-포이커 = **기준선에서 가장 먼 점을 남기고 나머지를 버리는** 분할 정복 단순화 알고리즘
- $\epsilon$ 이하로 벗어나는 점만 제거하므로 **오차 상한이 보장**되고, 결과는 항상 **원본의 부분집합**
- 복잡도는 평균 $O(n \log n)$ / 최악 $O(n^2)$, Radial Distance 전처리로 실효 성능을 크게 개선 가능
- 맵 트랙라인에 적용할 때 챙길 것
    - 위경도가 아닌 **투영 좌표계**에서 계산할 것
    - $\epsilon$ 은 **줌 레벨의 미터/픽셀**로부터 역산할 것
    - **노이즈 필터링 → 단순화** 순서를 지킬 것
    - **거리·통계 계산은 원본**으로 할 것
    - 재귀 대신 **명시적 스택**으로 구현할 것

---

### 참고 문헌

[Douglas, D. & Peucker, T. (1973), Algorithms for the reduction of the number of points required to represent a digitized line or its caricature](https://doi.org/10.3138/FM57-6770-U75U-7727)

[Ramer, U. (1972), An iterative procedure for the polygonal approximation of plane curves](https://doi.org/10.1016/S0146-664X(72)80017-0)

[Hershberger, J. & Snoeyink, J. (1992), Speeding Up the Douglas-Peucker Line-Simplification Algorithm](https://dl.acm.org/doi/10.5555/902273)

[Ramer-Douglas-Peucker algorithm - Wikipedia](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)

[simplify-js - A high-performance JS polyline simplification library](https://mourner.github.io/simplify-js/)

[Visvalingam-Whyatt: Line Simplification - Mike Bostock](https://bost.ocks.org/mike/simplify/)

[ST_Simplify - PostGIS Documentation](https://postgis.net/docs/ST_Simplify.html)

[MKMapPoint - Apple Developer Documentation](https://developer.apple.com/documentation/mapkit/mkmappoint)
