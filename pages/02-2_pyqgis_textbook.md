# PyQGIS 입문 교과서
### QGIS를 코드로 다루기 위한 최소 기반 지식

---

## 0. 이 교과서의 목표와 핵심 역량 분해

이 교과서의 목표는 두 가지이다.

1. PyQGIS를 구성하는 핵심 원리를 이해한다.
2. 혼자 힘으로 기본적인 공간분석 스크립트를 작성하고 실행한다.

이를 위해 "PyQGIS를 다루는 능력"을 다음 다섯 가지 하위 역량으로 분해한다. 각 장은 이 역량을 하나씩 다룬다.

| 역량 | 내용 | 다루는 위치 |
|---|---|---|
| ① 실행 환경 이해 | 코드가 어디서, 어떤 상태로 실행되는지 이해 | 1장 |
| ② 객체 모델 이해 | 프로젝트·레이어·피처·지오메트리가 어떤 관계인지 이해 | 2장 |
| ③ 레이어 조작 | 레이어를 불러오고, 속성을 읽고 쓰는 능력 | 2장~3장 |
| ④ 공간분석 자동화 | Processing 알고리즘을 코드로 호출하는 능력 | 3장 |
| ⑤ 반복·확장 | 동일한 절차를 값만 바꾸어 반복 실행하는 능력 | 4장 |

이 다섯 가지가 준비되면, GUI로 수행하던 QGIS 작업 대부분을 코드로 재현하고 자동화할 수 있다.

---

# 1장 — 기초: PyQGIS는 무엇이고 어디서 실행되는가

## 1.1 PyQGIS란

PyQGIS는 QGIS 내부 기능을 Python 코드로 호출할 수 있도록 제공하는 API(라이브러리)이다. QGIS 자체는 C++로 작성되어 있으나, 이 기능들을 Python에서도 그대로 사용할 수 있도록 연결해 두었다. 즉 QGIS 메뉴를 클릭해서 수행하던 작업 대부분을 같은 결과를 내는 코드 몇 줄로 대체할 수 있다.

PyQGIS를 배우는 이유는 다음과 같다.

- **반복 작업의 자동화**: 동일한 절차를 여러 데이터셋·여러 값으로 반복해야 할 때, 매번 메뉴를 클릭하는 대신 코드를 실행하면 된다.
- **재현 가능성**: 수행한 절차가 코드에 그대로 기록되므로, 나중에 같은 결과를 다시 만들어낼 수 있다.
- **GUI에 없는 세밀한 제어**: 조건에 따라 분기하거나, 여러 도구의 결과를 즉시 다음 입력으로 넘기는 등의 작업은 코드로 더 명확하게 처리된다.

## 1.2 코드가 실행되는 세 가지 환경

PyQGIS 코드는 상황에 따라 서로 다른 환경에서 실행되며, 이 차이를 처음부터 구분해 두어야 나중에 혼란이 없다.

### ① QGIS Python 콘솔 (가장 먼저 사용하게 될 환경)

QGIS를 실행한 상태에서 `플러그인 → Python 콘솔`(또는 상단 도구모음의 파이썬 아이콘)로 연다. 이 환경에서는 이미 열려 있는 QGIS 프로그램을 곧바로 다룰 수 있다. 따라서 현재 화면에 보이는 레이어나 현재 프로젝트에 곧바로 접근할 수 있다.

```python
# 현재 프로젝트에 열려 있는 레이어 이름을 모두 출력
from qgis.core import QgsProject

for layer in QgsProject.instance().mapLayers().values():
    print(layer.name())
```

콘솔에서만 사용할 수 있는 특별한 객체가 하나 있는데, 바로 `iface`이다. 이는 QGIS 프로그램 자체(메뉴, 지도 캔버스, 레이어 패널 등)를 가리키는 객체이며, 콘솔이나 플러그인 환경에서만 존재한다.

### ② Processing 스크립트 (Script Editor)

QGIS 콘솔의 코드 편집기, 또는 `프로세싱 → 도구상자 → 스크립트로 새 스크립트 만들기`에서 `.py` 파일로 저장해 실행하는 방식이다. 콘솔과 마찬가지로 QGIS 내부에서 실행되므로 `iface`를 사용할 수 있으나, 이후 QGIS를 다시 열 때 자동 실행되지는 않는다.

### ③ 독립 실행형(Standalone) 스크립트

QGIS 프로그램을 켜지 않은 상태에서, 터미널이나 별도의 Python 환경에서 QGIS 라이브러리만 불러와 실행하는 방식이다. 이 경우 QGIS 프로그램 자체가 없으므로 `iface`가 존재하지 않으며, 대신 다음과 같이 QGIS 환경을 직접 초기화해야 한다.

```python
from qgis.core import QgsApplication

# QGIS 설치 경로는 시스템마다 다르므로, 실제 경로를 확인하여 지정한다
QgsApplication.setPrefixPath("/path/to/qgis", True)
qgs = QgsApplication([], False)
qgs.initQgis()

# --- 이 사이에 실제 작업 코드를 작성한다 ---

qgs.exitQgis()
```

**처음 배우는 단계에서는 ①(콘솔)만 사용하는 것을 권장한다.** 독립 실행형 스크립트는 설치 경로·환경변수 설정이 추가로 필요해 난이도가 높으므로, 콘솔에 충분히 익숙해진 뒤에 다루는 것이 안전하다.

## 1.3 확인 사항

- [ ] QGIS Python 콘솔을 열고, 위 예시 코드로 현재 프로젝트의 레이어 이름을 출력해 보았다
- [ ] 콘솔 환경과 독립 실행형 환경의 차이(`iface` 사용 가능 여부)를 설명할 수 있다

---

# 2장 — 핵심 개념: PyQGIS의 객체 모델

PyQGIS 코드를 읽고 쓰기 위해서는 몇 가지 핵심 객체와 그 관계를 알아야 한다. 이 관계를 먼저 정리하면 이후 코드가 훨씬 쉽게 읽힌다.

```
QgsProject (프로젝트 전체)
  └─ QgsVectorLayer / QgsRasterLayer (레이어 하나)
        └─ QgsFeature (레이어 안의 피처 하나 — 표의 한 행에 해당)
              ├─ 속성값 (표의 각 칸의 값)
              └─ QgsGeometry (그 피처의 공간적 위치·모양)
```

## 2.1 QgsProject — 프로젝트 전체를 가리키는 객체

현재 열려 있는(또는 코드로 구성한) 프로젝트 전체를 가리킨다. 싱글턴(단일 인스턴스)이므로 `QgsProject.instance()`로 항상 동일한 객체를 참조한다.

```python
project = QgsProject.instance()
project.mapLayers()          # 프로젝트에 포함된 모든 레이어 (딕셔너리 형태)
project.crs()                 # 프로젝트의 좌표계
```

## 2.2 QgsVectorLayer — 레이어 하나를 불러오고 다루기

벡터 레이어(점·선·면 데이터)를 코드로 불러올 때 사용한다.

```python
from qgis.core import QgsVectorLayer

# 문법: QgsVectorLayer(경로, 화면에 표시할 이름, 데이터 제공자)
layer = QgsVectorLayer("/data/shelters.shp", "기존대피소", "ogr")

if not layer.isValid():
    print("레이어를 불러오지 못했습니다. 경로를 확인하세요.")
else:
    QgsProject.instance().addMapLayer(layer)
```

- 세 번째 인자("ogr")는 데이터 제공자(provider)이다. Shapefile, GeoPackage 등 파일 기반 벡터 데이터는 `"ogr"`을 사용한다.
- CSV처럼 위경도 컬럼만 있고 지오메트리가 없는 파일은 URI 형식으로 지정해야 한다.

```python
uri = "file:///data/shelters.csv?type=csv&xField=경도&yField=위도&crs=EPSG:4326"
layer = QgsVectorLayer(uri, "기존대피소_CSV", "delimitedtext")
```

`xField`, `yField`에는 실제 CSV의 컬럼명을 그대로 입력해야 한다.

## 2.3 QgsFeature — 레이어의 한 행

레이어에 포함된 각 개체(예: 대피소 하나, 격자 한 칸)를 피처(feature)라 한다. 레이어 전체를 순회할 때 사용하는 기본 패턴은 다음과 같다.

```python
for feature in layer.getFeatures():
    print(feature.id())              # 피처 고유 ID
    print(feature.attributes())      # 모든 속성값 리스트
    print(feature["필드명"])          # 특정 필드 값 하나
```

특정 조건을 만족하는 피처만 가져오려면 `QgsFeatureRequest`를 사용한다.

```python
from qgis.core import QgsFeatureRequest

request = QgsFeatureRequest().setFilterExpression('"인구밀도" > 100')
for feature in layer.getFeatures(request):
    print(feature["인구밀도"])
```

## 2.4 QgsGeometry — 피처의 공간 정보

피처의 위치와 모양을 다루는 객체이다.

```python
geom = feature.geometry()
geom.asPoint()          # 점 지오메트리인 경우 좌표 반환
geom.area()             # 면적 (레이어의 좌표계 단위 기준)
geom.distance(other_geom)   # 다른 지오메트리와의 거리
```

**주의:** `area()`, `distance()`의 결과 단위는 레이어의 좌표계에 따라 달라진다. 좌표계가 위경도(EPSG:4326)라면 단위가 "도(degree)"가 되어 실제 거리·면적과 무관한 값이 나온다. 미터 단위 결과가 필요하면 반드시 투영좌표계(예: EPSG:5179)로 변환한 뒤 계산해야 한다.

## 2.5 좌표계 확인과 변환

```python
print(layer.crs().authid())   # 예: 'EPSG:4326'

# 좌표계 변환이 필요한 경우 (예: 4326 → 5179)
from qgis.core import QgsCoordinateReferenceSystem, QgsCoordinateTransform

source_crs = layer.crs()
target_crs = QgsCoordinateReferenceSystem("EPSG:5179")
transform = QgsCoordinateTransform(source_crs, target_crs, QgsProject.instance())

geom = feature.geometry()
geom.transform(transform)   # geom 자체가 변환된 좌표로 바뀜
```

레이어 전체의 좌표계 자체를 영구히 바꾸려면(재투영), 뒤에서 다룰 Processing 알고리즘 `"native:reprojectlayer"`를 사용하는 편이 더 간단하다.

## 2.6 속성 편집 — 값을 수정하거나 새 필드를 추가하기

레이어의 속성을 수정하려면 **편집 모드**를 열고 값을 수정한 다음, 반드시 변경 사항을 저장(커밋)해야 한다.

```python
layer.startEditing()

# 새 필드 추가
from qgis.core import QgsField
from qgis.PyQt.QtCore import QVariant

layer.dataProvider().addAttributes([QgsField("정규화값", QVariant.Double)])
layer.updateFields()

# 각 피처의 값 계산 및 저장
idx = layer.fields().indexOf("정규화값")
for feature in layer.getFeatures():
    원래값 = feature["인구밀도"]
    layer.changeAttributeValue(feature.id(), idx, 원래값 * 0.4)

layer.commitChanges()   # 실제 저장은 이 시점에 이루어진다
```

`commitChanges()`를 호출하지 않으면 변경 사항이 저장되지 않는다. 코드 실행 중 오류가 발생했다면 `layer.rollBack()`으로 편집을 취소할 수 있다.

## 2.7 확인 사항

- [ ] 콘솔에서 shp 파일 하나를 `QgsVectorLayer`로 불러오고 화면에 추가하였다
- [ ] `getFeatures()`로 피처를 순회하며 속성값을 출력하였다
- [ ] 레이어의 좌표계를 `crs().authid()`로 확인하였다
- [ ] 편집 모드로 새 필드를 추가하고 값을 계산하여 저장하였다

---

# 3장 — 실습: 스크립트로 공간분석 절차 재현하기

이 장에서는 "행정동·대피소·도로" 세 레이어를 예시로, GUI에서 수행하던 절차를 코드로 재현한다. 실제 파일 경로와 필드명은 각자의 데이터에 맞게 수정해야 한다.

## 3.1 Processing 알고리즘을 코드로 호출하기

QGIS의 Processing Toolbox(도구상자)에 있는 대부분의 도구는 `processing.run()` 함수로 코드에서 그대로 호출할 수 있다. GUI에서 여러 단계에 걸쳐 클릭하며 입력하던 값들을 파라미터 딕셔너리 하나로 전달하는 방식이다.

```python
import processing
```

각 알고리즘에는 고유한 **알고리즘 ID**가 정해져 있다. ID가 정확히 기억나지 않을 때는 다음 방법으로 확인한다.

```python
# 이름에 "grid"가 포함된 알고리즘을 검색
for alg in QgsApplication.processingRegistry().algorithms():
    if "grid" in alg.id():
        print(alg.id(), "-", alg.displayName())
```

또는 콘솔에서 `processing.algorithmHelp("알고리즘ID")`를 입력하면 해당 알고리즘이 요구하는 파라미터 이름과 형식을 그대로 확인할 수 있다. **파라미터 이름은 QGIS 버전에 따라 달라질 수 있으므로, 코드를 작성하기 전에 이 명령으로 반드시 먼저 확인하는 습관을 들인다.**

### 격자 생성 예시

```python
result = processing.run("native:creategrid", {
    'TYPE': 2,                       # 2 = 사각형(Rectangle)
    'EXTENT': layer_boundary.extent(),
    'HSPACING': 500,
    'VSPACING': 500,
    'CRS': QgsCoordinateReferenceSystem("EPSG:5179"),
    'OUTPUT': 'memory:'               # 메모리에만 생성 (디스크에 저장하지 않음)
})

grid_layer = result['OUTPUT']
QgsProject.instance().addMapLayer(grid_layer)
```

`'OUTPUT': 'memory:'`로 지정하면 결과가 임시 메모리 레이어로 생성된다. 파일로 저장하려면 실제 경로(예: `'/data/grid.gpkg'`)를 지정한다.

### 경계로 자르기(Clip)

```python
clipped = processing.run("native:clip", {
    'INPUT': grid_layer,
    'OVERLAY': boundary_layer,
    'OUTPUT': 'memory:'
})['OUTPUT']
```

### 위치 기반 속성 결합 (인구밀도를 격자에 조인)

```python
joined = processing.run("native:joinattributesbylocation", {
    'INPUT': clipped,
    'JOIN': dong_layer,          # 인구밀도 필드를 가진 행정동 레이어
    'PREDICATE': [0],             # 0 = intersects
    'JOIN_FIELDS': ['인구밀도'],
    'METHOD': 0,
    'DISCARD_NONMATCHING': False,
    'OUTPUT': 'memory:'
})['OUTPUT']
```

`PREDICATE`, `METHOD` 등 숫자로 지정하는 옵션은 버전에 따라 의미가 달라질 수 있으므로, 실행 전 `processing.algorithmHelp("native:joinattributesbylocation")`로 각 숫자가 어떤 옵션을 뜻하는지 확인한다.

### 최근접 대상까지의 거리 계산

가장 가까운 대피소·도로까지의 거리를 구하는 알고리즘은 QGIS 버전에 따라 이름이 다르게 등록되어 있을 수 있다(`distance to nearest hub` 계열). 정확한 ID는 아래와 같이 직접 검색하여 사용한다.

```python
for alg in QgsApplication.processingRegistry().algorithms():
    if "nearest" in alg.id() or "hub" in alg.id():
        print(alg.id(), "-", alg.displayName())
```

검색된 ID를 확인한 뒤 `processing.algorithmHelp()`로 파라미터를 확인하고 동일한 방식(`processing.run(알고리즘ID, {파라미터: 값, ...})`)으로 호출한다.

## 3.2 필드 계산기 대응 코드 — 정규화와 가중합

Processing 결과 레이어에 값을 계산해 넣을 때는 앞서 2.6에서 다룬 편집 패턴을 그대로 사용한다.

```python
layer = joined  # 이전 단계 결과 레이어

# 1) 최솟값·최댓값 확인
values = [f["인구밀도"] for f in layer.getFeatures() if f["인구밀도"] is not None]
최솟값, 최댓값 = min(values), max(values)

# 2) 정규화 필드 추가
layer.startEditing()
layer.dataProvider().addAttributes([QgsField("인구밀도_정규화", QVariant.Double)])
layer.updateFields()
idx = layer.fields().indexOf("인구밀도_정규화")

for feature in layer.getFeatures():
    v = feature["인구밀도"]
    if v is None:
        continue
    정규화값 = (v - 최솟값) / (최댓값 - 최솟값)
    layer.changeAttributeValue(feature.id(), idx, 정규화값)

layer.commitChanges()
```

거리 기준처럼 "값이 작을수록 유리한" 항목은 정규화 후 `1 - 정규화값`으로 반전하여 저장한다. 세 항목의 정규화 필드가 모두 준비되면, 같은 방식으로 `적합도점수` 필드를 추가하고 가중합 공식을 계산하여 저장한다.

## 3.3 스타일링(단계 구분) 자동 적용

```python
from qgis.core import (
    QgsGraduatedSymbolRenderer, QgsClassificationJenks,
    QgsStyle
)

renderer = QgsGraduatedSymbolRenderer()
renderer.setClassAttribute("적합도점수")
renderer.setClassificationMethod(QgsClassificationJenks())
renderer.updateClasses(layer, 5)   # 5단계로 분류

색상램 = QgsStyle().defaultStyle().colorRamp("Oranges")
renderer.updateColorRamp(색상램)

layer.setRenderer(renderer)
layer.triggerRepaint()
```

## 3.4 인쇄 레이아웃(지도) 자동 내보내기

인쇄 레이아웃을 GUI에서 미리 하나 만들어 두었다면, 이를 코드로 불러와 이미지로 내보낼 수 있다.

```python
from qgis.core import QgsLayoutExporter

project = QgsProject.instance()
layout_manager = project.layoutManager()
layout = layout_manager.layoutByName("노원구_적합도지도")   # 미리 만들어 둔 레이아웃 이름

exporter = QgsLayoutExporter(layout)
exporter.exportToImage("/data/output_map.png", QgsLayoutExporter.ImageExportSettings())
```

레이아웃 자체(지도, 범례, 제목 등 요소 배치)를 완전히 코드로 새로 만들려면 다루어야 할 요소가 많아 난이도가 높다. 따라서 처음에는 "GUI로 레이아웃 틀을 만들고, 내보내기만 코드로 자동화하는" 방식을 권장한다.

## 3.5 확인 사항

- [ ] `processing.run()`으로 격자 생성 알고리즘을 호출하였다
- [ ] 알고리즘 ID를 모를 때 `processingRegistry()`와 `algorithmHelp()`로 직접 찾는 방법을 연습하였다
- [ ] 코드로 정규화·가중합 필드를 계산하여 저장하였다
- [ ] 저장된 결과 레이어에 단계 구분 스타일을 코드로 적용하였다

---

# 4장 — 응용: 반복과 확장

지금까지 작성한 코드는 한 번 실행하면 끝나는 절차였다. 이 장에서는 같은 절차를 값만 바꾸어 여러 번 실행하고, 그 결과를 비교하는 방법을 다룬다.

## 4.1 절차를 함수로 묶기

앞서 작성한 절차(격자 생성 → 자르기 → 조인 → 정규화 → 가중합)를 하나의 함수로 묶으면, 파라미터만 바꾸어 반복 실행할 수 있다. 이는 격자 간격이나 가중치를 바꾸어가며 결과를 비교하는 민감도 분석에 바로 활용할 수 있다.

```python
def 적합도_분석_실행(격자간격, 가중치_인구, 가중치_대피소거리, 가중치_도로거리):
    """
    지정한 격자 간격과 가중치로 전체 분석을 실행하고
    결과 레이어를 반환한다.
    """
    grid = processing.run("native:creategrid", {
        'TYPE': 2,
        'EXTENT': boundary_layer.extent(),
        'HSPACING': 격자간격,
        'VSPACING': 격자간격,
        'CRS': QgsCoordinateReferenceSystem("EPSG:5179"),
        'OUTPUT': 'memory:'
    })['OUTPUT']

    clipped = processing.run("native:clip", {
        'INPUT': grid,
        'OVERLAY': boundary_layer,
        'OUTPUT': 'memory:'
    })['OUTPUT']

    # (조인, 정규화, 가중합 단계는 3장의 코드를 그대로 재사용한다)
    # ...

    return clipped


# 격자 간격을 바꾸어가며 반복 실행
결과들 = {}
for 간격 in [250, 500, 750]:
    결과들[간격] = 적합도_분석_실행(간격, 0.4, 0.3, 0.3)
    print(f"{간격}m 간격 분석 완료 — 격자 수: {결과들[간격].featureCount()}")
```

## 4.2 상위 후보지 비교 자동화

각 결과에서 적합도 상위 N개를 추출하여 비교하는 절차도 함수화할 수 있다.

```python
def 상위_N개_추출(layer, 필드명="적합도점수", N=10):
    features = list(layer.getFeatures())
    features.sort(key=lambda f: f[필드명], reverse=True)
    return features[:N]

상위_250 = {f.id() for f in 상위_N개_추출(결과들[250])}
상위_750 = {f.id() for f in 상위_N개_추출(결과들[750])}

공통 = 상위_250 & 상위_750
print(f"두 격자 간격에서 공통으로 상위권에 포함된 개수: {len(공통)}")
```

**주의:** 격자 간격이 다르면 피처 ID 자체가 서로 다른 격자를 가리키므로, 위 코드처럼 ID만으로 비교하는 것은 정확하지 않다. 실제로는 격자의 중심 좌표나 원래 행정동·지역 단위로 매칭하는 절차가 추가로 필요하며, 이는 격자 기반 분석에서 반드시 고려해야 하는 지점이다. 간단한 비교가 필요할 때는 두 결과를 지도에 겹쳐 놓고 육안으로 확인하는 방법도 함께 사용한다.

## 4.3 다음 단계로 나아가기

이 교과서에서 다룬 범위를 넘어서는 주제는 다음과 같다. 기본기가 익숙해진 이후 필요에 따라 학습한다.

- **PyQGIS 플러그인 개발**: 지금까지 작성한 스크립트를 QGIS 메뉴에 버튼으로 등록하는 방법
- **커스텀 Processing 알고리즘 작성**: `QgsProcessingAlgorithm`을 상속하여, 직접 만든 절차를 Processing Toolbox의 표준 도구처럼 등록하는 방법
- **독립 실행형 스크립트의 완전한 자동화**: QGIS 프로그램 없이 서버나 배치 작업으로 정기 실행하는 구성

## 4.4 확인 사항

- [ ] 분석 절차를 함수로 작성하여 파라미터만 바꾸어 재실행하였다
- [ ] 반복 실행 결과를 비교할 때 발생하는 함정(피처 ID 불일치 등)을 인지하였다

---

## 부록 — 자주 참고하게 되는 명령

| 목적 | 코드 |
|---|---|
| 레이어 불러오기 | `QgsVectorLayer(경로, 이름, "ogr")` |
| 프로젝트에 추가 | `QgsProject.instance().addMapLayer(layer)` |
| 피처 순회 | `for f in layer.getFeatures():` |
| 좌표계 확인 | `layer.crs().authid()` |
| 알고리즘 ID 검색 | `QgsApplication.processingRegistry().algorithms()` |
| 알고리즘 파라미터 확인 | `processing.algorithmHelp("알고리즘ID")` |
| 알고리즘 실행 | `processing.run("알고리즘ID", {파라미터})` |
| 편집 시작/저장 | `layer.startEditing()` / `layer.commitChanges()` |
