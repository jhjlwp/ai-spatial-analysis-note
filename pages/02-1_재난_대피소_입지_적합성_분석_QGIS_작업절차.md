# 재난 대피소 입지 적합성 분석 - QGIS 작업 절차 정리

## 개요

행정동 경계, 기존 대피소 위치, 도로망 데이터를 이용해 500m 격자 후보지를 생성하고, **인구밀도 · 기존 대피소와의 거리 · 도로 접근성** 세 가지 기준을 정규화한 뒤 가중합하여 입지 적합도(suitability) 점수를 계산하는 작업입니다.

세 가지 평가 기준의 방향성은 다음과 같습니다.

- 인구밀도: 값이 높을수록 유리합니다. 많은 인구를 커버할 수 있기 때문입니다.
- 기존 대피소와의 거리: 값이 클수록 유리합니다. 기존 시설과의 중복을 피할 수 있기 때문입니다.
- 도로 접근성: 값이 작을수록(도로에 가까울수록) 유리합니다. 접근성이 좋아지기 때문입니다.

가중치는 예시로 인구밀도 0.4, 기존 대피소 거리 0.3, 도로 거리 0.3을 사용합니다. 실제 값은 팀에서 논의해 확정합니다.

> ⚠️ **주의**: 이 문서와 스크립트에 등장하는 필드명(`pop`, `POP_DENSITY`, `AREA_KM2`, `shelter_distance`, `road_distance` 등)과 레이어명(행정동, 기존 대피소, 도로 등)은 예시일 뿐입니다. 실제 사용하는 데이터(SGIS, 공공데이터포털 등에서 받은 shp·csv 파일)의 속성 테이블을 열어 실제 필드명과 레이어명을 먼저 확인하고, 아래 절차와 스크립트의 해당 부분을 데이터에 맞게 수정해서 사용해야 합니다.

---

## 전체 작업 순서 (QGIS GUI 기준)

### 0단계. 데이터 준비

- 행정동 경계(shp, 인구수 필드 `pop` 포함), 기존 대피소 위치(shp), 도로망(shp)을 다운로드합니다.
- 좌표계를 확인합니다. 위경도(EPSG:4326) 좌표계는 면적·거리 계산이 부정확하므로, 미터 단위 투영좌표계(예: EPSG:5179)로 재투영합니다.

### 1단계. 레이어 불러오기

- `레이어 > 레이어 추가 > 벡터 레이어 추가`로 행정동, 기존 대피소, 도로 shp를 각각 불러옵니다.

### 2단계. 면적 계산

- 필드 계산기로 `AREA_KM2`(Double) 필드를 추가합니다.
- 식: `$area / 1000000` (투영좌표계 기준, ㎢ 단위)

### 3단계. 인구밀도 계산

- 필드 계산기로 `POP_DENSITY`(Double) 필드를 추가합니다.
- 식: `"pop" / "AREA_KM2"`

> 참고: 인구수가 행정동 데이터와 별도의 통계표로 제공되어 코드 기준 조인이 필요한 경우에는, 이 단계 전에 `속성 > 결합(Joins)` 또는 Processing의 **"속성 테이블로 결합(Join attributes by field value)"**으로 행정동 코드를 기준 삼아 인구 통계표를 먼저 조인합니다.

### 4단계. 후보지 격자 생성 (500m 간격)

- Processing 툴박스 → **"격자 생성(Create grid)"**
- 유형: 점(Point) / Extent: 행정동 레이어 범위 / 간격: 500, 500

### 5단계. 행정동 경계 내부만 추출

- Processing 툴박스 → **"위치로 추출(Extract by location)"**
- Input: 격자 / 조건: 포함됨(within) / Intersect: 행정동 레이어

### 6단계. 인구밀도 값 공간 조인

- Processing 툴박스 → **"위치로 속성 결합(Join attributes by location)"**
- Input: 5단계 후보지 / Join: 행정동 레이어 / 조건: within / 결합 필드: `POP_DENSITY`

### 7단계. 기존 대피소까지 최근접 거리

- Processing 툴박스 → **"최근접 항목으로 결합(Join attributes by nearest)"**
- Input: 6단계 결과 / Input 2: 기존 대피소 레이어 / 이웃 개수: 1 / Prefix: `shelter_` → `shelter_distance` 필드 생성

### 8단계. 도로까지 최근접 거리

- 같은 도구를 다시 실행합니다.
- Input: 7단계 결과 / Input 2: 도로 레이어 / Prefix: `road_` → `road_distance` 필드 생성

### 9단계. 정규화 및 가중합 계산

**일반적인 Min-Max 정규화 공식**은 다음과 같습니다.

$$x_{norm} = \frac{x - x_{min}}{x_{max} - x_{min}}$$

이 공식은 값을 0~1 사이로 변환합니다. 클수록 좋은 지표(인구밀도, 기존 대피소와의 거리)는 공식을 그대로 적용하고, 작을수록 좋은 지표(도로 거리)는 $1 - x_{norm}$으로 반전해서 적용합니다.

속성 테이블에서 **필드 계산기**로 `suitability`(Double) 필드를 추가한 뒤, 아래 식을 입력합니다.

```
0.4 * ( ("POP_DENSITY" - minimum("POP_DENSITY")) / (maximum("POP_DENSITY") - minimum("POP_DENSITY")) )
+ 0.3 * ( ("shelter_distance" - minimum("shelter_distance")) / (maximum("shelter_distance") - minimum("shelter_distance")) )
+ 0.3 * ( 1 - ("road_distance" - minimum("road_distance")) / (maximum("road_distance") - minimum("road_distance")) )
```

- `maximum()`, `minimum()` 함수를 사용하면 min·max 값을 미리 확인하지 않아도 식 안에서 자동으로 계산됩니다.
- NULL 값 처리가 필요하면 각 필드를 `coalesce("필드", minimum("필드"))`로 감쌉니다.
- 참고로 이 함수들은 실행될 때마다 레이어 전체를 스캔하므로, 후보지 수가 매우 많으면 계산이 느려질 수 있습니다. 속도가 문제라면 `벡터 > 분석 도구 > 기초 통계`로 값을 미리 확인해 숫자로 고정하는 방식이 더 빠릅니다.

### 10단계. 시각화

- 결과 레이어 속성 → **심볼로지 > 단계 구분(Graduated)**
- 값: `suitability` / 팔레트: 순차형(Sequential) / 등급: Natural Breaks 5~7단계 추천

### 11단계. 결과 저장

- 레이어 우클릭 → **내보내기 > 다른 이름으로 객체 저장** → GeoPackage로 저장합니다.

---

## PyQGIS 스크립트로 자동화

> 📌 **참고**: 이 섹션은 다음 PyQGIS 실습 이후에 참조하시기 바랍니다.

위 절차는 GUI로 매번 반복하기 번거로우므로 PyQGIS 스크립트로 자동화할 수 있습니다. QGIS Python 콘솔(Plugins > Python Console) 또는 Processing > 스크립트 편집기에서 실행합니다.

### 스크립트 버전 비교

| 항목 | v1 (원본) | v2 (수정본) |
|---|---|---|
| 인구밀도 필드 | 이미 조인되어 있다고 가정 | 스크립트 안에서 직접 계산 |
| 인구수 데이터 | 별도 처리 없음 | 행정동 레이어의 `pop` 필드 사용 |
| 면적 계산 | 없음 | `geometry().area()`로 `AREA_KM2` 계산 |
| 인구밀도 계산 | 없음 | `pop / AREA_KM2`로 `POP_DENSITY` 계산 |

### v2 스크립트 처리 순서

1. 레이어 불러오기 (행정동 / 기존 대피소 / 도로)
2. 면적 및 인구밀도 계산 (`pop` 필드 → `AREA_KM2`, `POP_DENSITY`)
3. 500m 격자 후보지 생성
4. 인구밀도 공간 조인
5. 기존 대피소 최근접 거리 조인 (`shelter_distance`)
6. 도로 최근접 거리 조인 (`road_distance`)
7. 정규화 및 가중합 (`suitability`) 계산
8. GeoPackage로 결과 저장

### 유의 사항

- 행정동 레이어가 위경도(EPSG:4326)라면 면적 계산이 부정확하므로, 스크립트 안의 재투영 코드(`native:reprojectlayer`, 예: EPSG:5179)를 먼저 활성화해서 사용해야 합니다.
- 인구수 필드명(`pop`), 가중치(`W_POP`, `W_SHELTER`, `W_ROAD`), 파일 경로(`PATH_*`)는 실제 데이터에 맞게 스크립트 상단에서 수정합니다.
- 반복 작업이 많다면 Processing > **그래픽 모델러**로 만들어두면 버튼 한 번으로 재실행할 수 있습니다.

---

## 첨부 파일

- `pyqgis_suitability_script_v2.py`: 인구밀도 계산이 포함된 최종 PyQGIS 스크립트
