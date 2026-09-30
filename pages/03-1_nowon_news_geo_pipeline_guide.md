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

## 4. 실행 순서

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
| 카카오 401/403 | REST API 키, `[카카오맵]` 사용 설정 ON, 허용 IP, 쿼터 확인 |

---

## 5. 결과 해석 시 주의 (분석 타당성)

- **표본의 성격**: 네이버 검색에 잡힌 "노원구" 기사 기준이며 실제 사건·관심의 분포와 같지 않음. 구청 보도자료·지역지 재게재가 언급 횟수를 키울 수 있음(제목 기준 중복 제거는 하지만 완전하지 않음).
- **기간 편향**: `sort=date`로 최근순 최대 1,000건까지만 조회. 경고가 뜨면 기간을 나눠(예: 월별) 실행해야 함.
- **쿼리 유도 편향**: `--place-queries`를 켜면 검색어에 쓴 동/역이 과대 집계됨.
- **공간 해상도**: 동·산·하천 등 지명형은 카카오가 반환하는 대표점에 찍히므로 "동 단위 분포"로 해석(대표점 산정 방식은 문서에 없음 ❓). 정밀 위치는 도로명·지번이 나온 기사에 한함.
- **오탐**: 지번 정규식은 드물게 오탐 가능. bbox는 사각형이라 인접 지역 일부를 통과시킬 수 있으므로 동 경계 폴리곤으로 최종 검증 권장.
- **사전 한계**: `GAZETTEER`에 없는 장소는 잡히지 않음. 개별 항목은 검증하지 않았으니 `matched` 컬럼으로 점검.

## 6. QGIS 활용
- 포인트: `n_ment` 가중 **히트맵** / 비례 심볼, 동별 SHP: **단계구분도**
- 한글 깨짐: 레이어 소스의 인코딩을 UTF-8로 (ArcGIS는 `encoding="cp949"`로 변경) ❓

## 7. 참고 링크 (열람 확인한 것)
- NAVER API HUB 이관 가이드: https://guide.ncloud-docs.com/docs/apihub-migration
- NAVER API HUB 개요(한도 FAQ): https://guide.ncloud-docs.com/docs/apihub-overview
- HUB 뉴스 검색 API: https://api.ncloud-docs.com/docs/naver-api-hub-search-news
- 카카오 로컬 API 개발 가이드: https://developers.kakao.com/docs/ko/local/dev-guide
- 카카오 쿼터: https://developers.kakao.com/docs/ko/getting-started/quota
- 카카오 로컬 이해하기(사용 설정): https://developers.kakao.com/docs/ko/local/common
- 노원구청 지역특성(극점 좌표): https://www.nowon.kr/www/intrcn/intrcn1/intrcn1_13.jsp
