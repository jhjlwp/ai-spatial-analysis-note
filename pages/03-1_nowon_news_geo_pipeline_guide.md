# 노원구 뉴스 지오코딩 → Shapefile 파이프라인 정리

> 작성 기준일: 2026-09-30
> 목적: 뉴스에서 **서울시 노원구 관련 내용이 어디에, 얼마나 분포하는지** 공간적으로 파악

**표기 규칙** — ✅ 공식 문서로 확인 · 🔶 2차 출처(블로그·이슈 등)로만 확인 · ❓ 미확인(직접 확인 필요)

---

## 1. 전체 구조

```
[네이버 뉴스 검색 API(HUB)] → [장소 추출] → [카카오 로컬 지오코딩] → [SHP 저장] → QGIS 분석
      기사 수집                사전+정규식      좌표 변환·bbox 검증      포인트/동별 집계
```

| 단계 | 내용 |
|---|---|
| 1. 수집 | 뉴스 검색 API로 "노원구" 쿼리 수집(`sort=date`). 제목 정규화로 중복 제거. 옵션으로 네이버뉴스 본문 수집 |
| 2. 장소 추출 | 도로명주소·지번 정규식 → 장소 사전(동/역/산/하천/대학/병원 등) 순. 본문에 "노원"이 없는 기사 제외 |
| 3. 지오코딩 | 카카오 주소검색/키워드검색, 결과 캐시, 노원구 bbox 밖 결과 폐기 |
| 4. 저장 | 장소별 포인트 SHP(언급 횟수 포함) + (선택) 행정동 집계 SHP + CSV |

### 산출 파일

| 파일 | 설명 |
|---|---|
| `nowon_news_points.shp` | `name`, `type`, `n_art`(기사 수), `n_ment`(언급 횟수), `first_dt`, `last_dt`, `lon`, `lat`, `matched` |
| `nowon_news_by_dong.shp` | (선택) 동 폴리곤별 `n_ment`, `n_place` |
| `articles.csv` / `place_mentions.csv` | 기사·장소별 통계 (검수용) |
| `geocode_cache.json` | 지오코딩 캐시 |

### 제공 스크립트
- `nowon_news_geo_pipeline.py` — 본 파이프라인
- `check_keys.py` — 네이버 HUB·카카오 키 사전 점검

---

## 2. 사전 준비

### A. 네이버 API HUB (뉴스 검색)

개발자센터(`developers.naver.com`) 키는 HUB에서 쓸 수 없고 **NCP에서 새로 발급**해야 합니다. (뉴스 검색은 HUB로 이관되어 계속 제공, 쇼핑·책·전문자료 검색은 대체 없이 종료 🔶)

| 항목 | 기존(개발자센터) | HUB (이 프로젝트) |
|---|---|---|
| 도메인 | `openapi.naver.com` | `naverapihub.apigw.ntruss.com` |
| 뉴스 경로 | `/v1/search/news.json` | `/search/v1/news` |
| ID 헤더 | `X-Naver-Client-Id` | `X-NCP-APIGW-API-KEY-ID` |
| Secret 헤더 | `X-Naver-Client-Secret` | `X-NCP-APIGW-API-KEY` |

**발급 절차** (✅ 공식 이관 가이드, 메뉴명은 화면에서 재확인)
1. 네이버클라우드 플랫폼(ncloud.com) 가입/로그인 — 결제수단 요구 여부 ❓
2. `Services > Application Services > NAVER API HUB > 신청하기 > Subscription > [서비스 이용 신청]`, 약관 동의
3. `Application > [Application 등록]` → 이름 입력(표시용 라벨, 예: `nowon-news-geo`), API는 **뉴스 검색** 선택
4. Application 목록 → `[인증 정보]` → **Client ID / Client Secret** 복사
   - ⚠️ 🔶 NCP 콘솔의 계정 수준 **IAM Access Key(`ncp_iam_…`)로는 HUB가 열리지 않는다**는 보고가 있음. 반드시 **Application의 Client ID/Secret**을 사용
5. 이용 한도·알림 설정 (공식 절차 문서에 "일정 사용량 초과 시 과금되므로 한도 설정 권장"이라는 안내 ✅. 한편 🔶 공지를 인용한 자료는 Search API가 기존 무료 정책과 동일하고 현재 한시적 무료라고 함 → 향후 변경 가능)

**뉴스 API 사양** (✅ 공식): `query` 필수, `display` 1~100(기본 10), `start` 1~1000(기본 1), `sort` = `sim`(기본)|`date`, `format` = json|xml. 오류코드 `SE01`~`SE06`, `SE99`. 한도: 뉴스 검색 API 문서 "하루 25,000회" / HUB 개요 FAQ "검색 카테고리 통합 월 최대 775,000건, 키당 최대 50 RPS".

### B. 카카오 로컬 API (지오코딩)

1. developers.kakao.com 로그인 → `내 애플리케이션 > 애플리케이션 추가`
2. `[앱] > [플랫폼 키] > [REST API 키]` 복사 (다른 종류의 키 사용 금지)
3. **`[카카오맵] > [사용 설정]` 상태 ON** (2024-12-01부터 신규 앱 필수)
4. 허용 IP를 설정했다면 내 공인 IP 등록
5. `[쿼터]`에서 사용량 확인

**호출 방식** (✅ 공식 dev-guide): `Authorization: KakaoAK ${REST_API_KEY}`
- 주소 검색 `GET https://dapi.kakao.com/v2/local/search/address.json` (`size` 최대 30)
- 키워드 검색 `GET https://dapi.kakao.com/v2/local/search/keyword.json` (`size` 최대 15)
- 응답 `x`=경도, `y`=위도. 주소 검색의 `address_type`이 `REGION`이면 지명(동 이름 등) 결과
- 오류 (✅ 공식 레퍼런스): **401**=유효하지 않은 키/토큰, **403**=권한·설정 문제(사용 설정 미활성 등), **429**=쿼터·초당 한도 초과(로컬 API). 일반 예시로 **400 + `code -10`**(허용 요청 횟수 초과)도 문서에 있음

**쿼터/요금** (✅ 공식): 주소→좌표, 키워드 검색 각각 **일 100,000건 무료**, 전체 API 월 3,000,000건. 유료 API 설정 시에만 초과 요금(주소 0.5원/건, 키워드 2원/건). ⚠️ **카카오맵 API는 개발자 계정 기준 "첫 번째로 활성화한 앱"에만 무료 쿼터가 제공**됨 (이미 다른 앱에서 카카오맵을 활성화했다면 이 앱은 무료 쿼터 대상이 아닐 수 있으니 이용 정책 확인)

---

## 3. Miniconda 환경 설정

환경 이름: **`giscourse`** (이미 있는 환경이면 `conda create`는 재실행하지 않음 — 같은 이름으로 만들면 덮어써질 수 있음)

```bash
conda env list
conda activate giscourse
conda list | findstr /i "geopandas pandas requests beautifulsoup4 shapely"     # PowerShell: Select-String
# 없는 패키지만 설치 (변경 목록이 과하면 n으로 중단하고 새 환경 권장)
conda install -n giscourse -c conda-forge geopandas pandas requests beautifulsoup4
```

새 환경 예: `conda create -n nowon_news -c conda-forge python=3.11 geopandas pandas requests beautifulsoup4 -y`
(`-n` 환경 이름, `-c conda-forge` 채널 지정, `-y` 자동 승인. geopandas는 GDAL/PROJ 등 지리공간 라이브러리에 의존해 conda-forge로 설치하는 편이 의존성 관리에 수월)

### 키 등록 (환경변수)

```bash
conda env config vars set NAVER_CLIENT_ID=HUB_ID NAVER_CLIENT_SECRET=HUB_SECRET KAKAO_REST_KEY=카카오키 -n giscourse
conda deactivate
conda activate giscourse
```

| 질문 | 답 |
|---|---|
| 성격 | 파이썬 전역변수가 아니라 **환경변수**. 스크립트는 `os.getenv()`로 읽음 |
| 유효 범위 | `giscourse`를 활성화한 터미널의 프로그램. 다른 환경/비활성 CMD에서는 안 보임. Windows 시스템 환경변수는 아님 |
| 껐다 켜도 유지? | 유지됨 (deactivate는 해제만, 저장값은 남음. 재부팅해도 유지) |
| 저장 위치 | `<miniconda경로>\envs\giscourse\conda-meta\state` (JSON 평문) ❓ conda 일반 동작 기준 |
| 사라지는 경우 | `unset`, 환경 삭제, 같은 이름으로 환경 재생성 |

```bash
conda env config vars list -n giscourse
conda env config vars unset KAKAO_REST_KEY -n giscourse
```
⚠️ 키는 **평문 저장** → 환경 폴더 공유/업로드 금지, 노출 의심 시 재발급.

---

## 4. AI에게 파이프라인 코드 생성 요청하기

> 이미 만들어 둔 `nowon_news_geo_pipeline.py`를 그대로 쓴다면 이 단계는 건너뛰고 5번으로 가도 됩니다.
> 코드를 새로 만들거나 수정·확장(사전 추가, 다른 지역 적용, 다른 뉴스 소스 등)할 때 이 절차를 사용하세요.

### 4-1. 요청 전 준비 (AI에게 알려줄 정보)

AI 모델은 학습 시점에 따라 2026년에 바뀐 **NAVER API HUB**를 모르고 예전 방식(`openapi.naver.com`)으로 코드를 만들 수 있습니다(웹 검색 기능이 없는 경우 특히). 아래 정보를 프롬프트에 **명시**하세요.

| 알려줄 정보 | 이 프로젝트의 값 |
|---|---|
| 목표·결과물 | 노원구 뉴스의 공간 분포 파악 → 장소별 언급 횟수 포인트 SHP(+동별 집계 SHP) |
| 실행 환경 | **본인 PC의 실제 값**: OS(예: Windows), Miniconda 환경 `giscourse`의 python 버전과 설치 패키지 버전 (`python --version`, `conda list geopandas pandas requests beautifulsoup4`로 확인해 기입) |
| 수집 API 사양 | NAVER API HUB 뉴스 검색 (도메인·경로·헤더·파라미터, 2번 A절 참조) |
| 지오코딩 API 사양 | 카카오 로컬 주소/키워드 검색 (엔드포인트·헤더·응답 필드, 2번 B절 참조) |
| 키 전달 방식 | 코드에 쓰지 않고 환경변수(`NAVER_CLIENT_ID`, `NAVER_CLIENT_SECRET`, `KAKAO_REST_KEY`)로 읽기 |
| 검증 기준 | 노원구 범위 밖 좌표 폐기, 캐시, 오류 처리 방식 |

> 📝 `giscourse`의 파이썬 버전·설치 패키지는 이 문서 작성 과정에서 확인되지 않았습니다. 추측하지 말고 위 명령의 실제 출력값을 넣으세요.
> ⚠️ **실제 API 키, Client Secret은 프롬프트에 붙여넣지 마세요.** 환경변수 이름만 알려주면 됩니다.
> 💡 공식 문서 내용(요청 파라미터 표, 응답 예시)을 그대로 복사해 붙여넣으면 AI가 옛 사양으로 착각하는 것을 막을 수 있습니다.

### 4-2. 프롬프트 템플릿 (복사해서 사용)

```text
너는 GIS/파이썬 데이터 엔지니어야. 아래 조건으로
"뉴스 → 장소 추출 → 지오코딩 → Shapefile" 파이프라인을 파이썬 스크립트 하나로 작성해줘.

[목표]
- 뉴스에서 서울시 노원구 관련 내용이 어디에, 얼마나 분포하는지 공간적으로 파악
- 결과: 장소별 언급 횟수가 들어간 포인트 SHP, (선택) 행정동 폴리곤별 집계 SHP

[실행 환경]
- OS: (예: Windows 11)                          <- 실제 값으로 교체
- Miniconda 환경 giscourse, python 버전: (예: 3.11.x)   <- python --version 결과로 교체
- 설치 패키지: geopandas (버전), pandas (버전), requests (버전), beautifulsoup4 (버전)   <- conda list 결과로 교체
- 위 목록에 없는 패키지는 쓰지 말 것 (꼭 필요하면 conda install 명령을 따로 안내)
- API 키는 코드에 쓰지 말고 환경변수에서 읽기: NAVER_CLIENT_ID, NAVER_CLIENT_SECRET, KAKAO_REST_KEY

[1단계 수집: 네이버 뉴스 검색 API - NAVER API HUB]
(주의: 예전 openapi.naver.com 개발자센터 방식이 아니라 HUB 방식이다)
- GET https://naverapihub.apigw.ntruss.com/search/v1/news
- 헤더: X-NCP-APIGW-API-KEY-ID, X-NCP-APIGW-API-KEY
- 파라미터: query(필수), display 1~100, start 1~1000, sort=sim|date, format=json
- 응답 items: title, originallink, link, description, pubDate (title/description에 <b> 태그 포함 → 제거)
- 최근 N일만 수집, sort=date, 기간 시작점에 도달하면 페이징 중단,
  start 상한(1000)에 도달해 기간을 다 못 채우면 경고, 제목 정규화로 중복 제거
- 검색어는 기본 "노원구" 하나. 동/역 이름 쿼리는 --place-queries 옵션을 줄 때만 추가
  (검색어 자체가 해당 장소의 언급 빈도를 부풀리는 편향이 있으므로 기본은 끔)
- HTTP 오류를 구분해 원인과 조치를 안내: 400(SE01~SE04, SE06 등 파라미터 오류), 401/403(인증·권한),
  404(SE05), 429(호출 제한), 500(SE99). HUB는 API Gateway 기반이라 오류 응답 형식이 기존과 다를 수 있으므로
  errorCode 한 가지 형식만 가정하지 말고 HTTP 상태 코드를 먼저 볼 것

[2단계 장소 추출]
- 제목+요약(+선택: 본문)에 "노원"이 없는 기사는 제외
- 도로명주소/지번 정규식 → 장소 사전(동, 지하철역, 산·하천, 대학, 병원 등) 순으로 추출,
  먼저 추출된 구간은 마스킹해서 이중 카운트 방지
- 장소별 기사 수(n_art), 언급 횟수(n_ment), 최초/최종 날짜(first_dt, last_dt)

[3단계 지오코딩: 카카오 로컬 API]
- 주소: GET https://dapi.kakao.com/v2/local/search/address.json
- 키워드: GET https://dapi.kakao.com/v2/local/search/keyword.json
- 헤더 Authorization: KakaoAK {REST_API_KEY}, 응답 x=경도(lon), y=위도(lat)
- 결과는 JSON 파일로 캐시. 일시 오류(네트워크, 429 속도 제한 등)는 캐시하지 말 것.
- 429는 잠시 대기 후 재시도하고, 계속되면(또는 400 + code -10 쿼터 초과) 원인 안내 후 종료
- 401/403은 원인(REST API 키, 카카오맵 사용 설정, 허용 IP, 권한) 안내 후 종료 (종료 전에 캐시는 저장)
- 노원구 범위 밖 좌표는 폐기 (경도 127.035~127.118, 위도 37.605~37.700)

[4단계 저장]
- geopandas로 포인트 SHP 저장 (필드명 10자 이하, UTF-8 + .cpg, --crs 옵션)
- 동 경계 SHP가 주어지면 sjoin으로 동별 n_ment 집계 SHP 저장
- 검수용 CSV(articles.csv, place_mentions.csv)도 함께 저장

[코드 요구사항]
- argparse 옵션: --days, --max-per-query, --place-queries, --out-dir, --crs, --fetch-body, --dong-shp, --dong-col, --dong-encoding
- 함수 단위로 분리, 주석은 한국어, 오류 메시지는 원인/조치를 알려줄 것
- 네트워크 없이도 돌려볼 수 있도록 장소 추출 함수는 샘플 문장으로 테스트해줘

[출력 형식]
(1) 전체 코드 (2) 설치/실행 명령 (3) 확인하지 못한 가정과 한계 목록
```

### 4-3. 단계별로 나눠 요청·검증하기 (권장)

한 번에 전부 만들게 하면 오류 위치를 찾기 어렵습니다. 단계마다 실행해 보고 다음으로 넘어가세요.

| 순서 | AI에게 요청 | 내 쪽 확인 |
|---|---|---|
| 1 | 위 템플릿에서 **1단계(수집)만** 작성 요청 (수집 결과를 `articles.csv`로 저장하도록 포함) | `check_keys.py`로 키 확인 후 실행, 기사 수·날짜 범위 확인 |
| 2 | 수집 결과(`articles.csv` 앞 20행)를 붙여넣고 **2단계(장소 추출)** 요청 | 오탐/누락 장소 확인 |
| 3 | 추출된 장소 목록을 붙여넣고 **3단계(지오코딩)** 요청 | `matched` 컬럼으로 잘못 찍힌 장소 확인 |
| 4 | **4단계(SHP 저장)** 요청 | QGIS에서 열어 좌표계·한글 인코딩·속성 확인 |
| 5 | 4개 단계를 **하나의 스크립트로 통합** 요청 | 시험 실행(`--days 30`) 후 본 실행 |

> 기사 제목·요약은 언론사 저작물이므로 외부 AI에 붙여넣는 양은 꼭 필요한 최소(소수 행, 제목 위주)로 하고, 사용하는 AI 서비스의 데이터 이용 정책도 확인하세요.

### 4-4. 수정·보완 요청 예시

| 상황 | 요청 방법 |
|---|---|
| 오류 발생 | 실행 명령, **에러 메시지 전문**, `python --version`, geopandas 버전을 함께 붙여넣기 (키 값은 가리기) |
| 장소 오탐·누락 | `place_mentions.csv` 상위 30행을 붙여넣고 "오탐 후보와 사전에 추가할 장소를 제안해줘" |
| 좌표가 엉뚱함 | 문제 장소명과 `matched` 값을 붙여넣고 "지오코딩 질의어/검증 로직을 개선해줘" |
| 기능 추가 | "월별 집계 컬럼 추가", "기사 링크 목록을 별도 CSV로 저장" 등 구체적으로 |
| 다른 지역 적용 | 장소 사전과 bbox만 교체하도록 설정 분리를 요청 |

### 4-5. AI가 만든 코드 검토 체크리스트

- [ ] 네이버 호출이 **HUB**(`naverapihub.apigw.ntruss.com`, `X-NCP-...` 헤더)인가? (예전 `openapi.naver.com`이면 수정 요청)
- [ ] 키가 코드에 **하드코딩**되어 있지 않고 환경변수에서 읽는가?
- [ ] 카카오 좌표에서 **x=경도, y=위도** 순서를 지켰는가? (뒤바뀌면 위도가 127°처럼 유효 범위(±90°)를 벗어나 점이 지도에 나타나지 않거나 엉뚱한 곳에 찍힘)
- [ ] `start`/`display` 범위(1~1000 / 1~100)와 호출 한도를 고려했는가?
- [ ] 오류(400/401/403/429/타임아웃) 처리와 캐시 정책이 있는가? (일시 오류를 캐시하면 재시도 불가)
- [ ] SHP 필드명이 10자 이하이고 인코딩(UTF-8/.cpg)이 지정되어 있는가?
- [ ] "확인하지 못한 가정" 목록이 있고, 그 항목을 실제 실행으로 검증했는가?
- [ ] 크롤링(본문 수집)을 쓴다면 이용약관·robots 정책을 확인했는가?

---

## 5. 실행 순서

```bash
conda activate giscourse
cd C:\작업폴더
python check_keys.py                                   # 모두 OK여야 진행

python nowon_news_geo_pipeline.py --days 30 --out-dir out_test      # 시험 실행
# out_test\place_mentions.csv 의 matched 컬럼으로 지오코딩 결과 검수

python nowon_news_geo_pipeline.py --days 180 --fetch-body --crs EPSG:5179 --out-dir out
python nowon_news_geo_pipeline.py --days 180 --dong-shp C:\gis\bnd\emd.shp --dong-col EMD_KOR_NM --dong-encoding cp949
```

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `--days` | 180 | 최근 N일 |
| `--max-per-query` | **1000** | 쿼리당 최대 수집(HUB `start` 상한 1000). 기간 전체를 못 채우면 경고 출력 |
| `--place-queries` | 꺼짐 | 동/역 이름 쿼리 추가. 커버리지↑, 단 **해당 장소 빈도가 부풀려지는 편향** |
| `--fetch-body` | 꺼짐 | 네이버뉴스 본문 수집(느림, 크롤링 약관 확인 ❓) |
| `--crs` | EPSG:4326 | 5179=UTM-K, 5186=중부원점 TM |
| `--dong-shp` `--dong-col` `--dong-encoding` | - / `EMD_KOR_NM` / `cp949` | 동별 집계 |
| 환경변수 `NAVER_API_MODE` | `hub` | `legacy`면 구방식 |

### 점검 결과 해석

| 증상 | 조치 |
|---|---|
| 네이버 401 | HUB Application의 Client ID/Secret인지 확인 (개발자센터 키·IAM 키 불가) |
| 네이버 403 | Application에 **뉴스 검색** API 등록 여부 확인 |
| 네이버 429 | 호출 한도 초과 |
| 카카오 401/403 | REST API 키인지, `[카카오맵]` 사용 설정 ON, 허용 IP, 권한 확인 |
| 카카오 429 (또는 400 + code -10) | 쿼터·초당 한도 초과 → `[쿼터]`에서 사용량 확인, 잠시 후 재시도 |

---

## 6. 결과 해석 시 주의 (분석 타당성)

- **표본의 성격**: 네이버 검색에 잡힌 "노원구" 기사 기준이며 실제 사건·관심의 분포와 같지 않음. 구청 보도자료·지역지 재게재가 언급 횟수를 키울 수 있음(제목 기준 중복 제거는 하지만 완전하지 않음).
- **기간 편향**: `sort=date`로 최근순 최대 1,000건까지만 조회. 경고가 뜨면 기간을 나눠(예: 월별) 실행해야 함.
- **쿼리 유도 편향**: `--place-queries`를 켜면 검색어에 쓴 동/역이 과대 집계됨.
- **공간 해상도**: 동·산·하천 등 지명형은 카카오가 반환하는 대표점에 찍히므로 "동 단위 분포"로 해석(대표점 산정 방식은 문서에 없음 ❓). 정밀 위치는 도로명·지번이 나온 기사에 한함.
- **오탐**: 지번 정규식은 드물게 오탐 가능. bbox는 사각형이라 인접 지역 일부를 통과시킬 수 있으므로 동 경계 폴리곤으로 최종 검증 권장.
- **사전 한계**: `GAZETTEER`에 없는 장소는 잡히지 않음. 개별 항목은 검증하지 않았으니 `matched` 컬럼으로 점검.

## 7. QGIS 활용
- 포인트: `n_ment` 가중 **히트맵** / 비례 심볼, 동별 SHP: **단계구분도**
- 한글 깨짐: 레이어 소스의 인코딩을 UTF-8로 (ArcGIS는 `encoding="cp949"`로 변경) ❓

## 8. 참고 링크 (열람 확인한 것)
- NAVER API HUB 이관 가이드: https://guide.ncloud-docs.com/docs/apihub-migration
- NAVER API HUB 개요(한도 FAQ): https://guide.ncloud-docs.com/docs/apihub-overview
- HUB 뉴스 검색 API: https://api.ncloud-docs.com/docs/naver-api-hub-search-news
- 카카오 로컬 API 개발 가이드: https://developers.kakao.com/docs/ko/local/dev-guide
- 카카오 쿼터: https://developers.kakao.com/docs/ko/getting-started/quota
- 카카오 로컬 이해하기(사용 설정): https://developers.kakao.com/docs/ko/local/common
- 노원구청 지역특성(극점 좌표): https://www.nowon.kr/www/intrcn/intrcn1/intrcn1_13.jsp
