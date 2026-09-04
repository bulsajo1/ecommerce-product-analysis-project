# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 이 저장소의 성격

**GA4 obfuscated sample ecommerce 데이터셋**(Google 머천다이즈 스토어)을 대상으로, BigQuery를 Jupyter 노트북에서 직접 조회하며 진행하는 AARRR 프레임워크 기반 프로덕트 분석 프로젝트임. 빌드할 애플리케이션은 없고, **노트북 안의 분석 서술 자체가 결과물**임. `main.py`는 `uv init` 스캐폴딩 잔재이며 분석과 무관함.

작업 언어는 **한국어**임. 노트북 마크다운, 문서, 커밋 메시지가 모두 한국어로 작성되어 있으니 이 관례를 따를 것.

## 환경 및 명령어

의존성은 `uv`로 관리함 (Python 3.12, `.venv/`는 로컬에 존재하나 gitignore 대상).

```powershell
uv sync                  # 환경 설치/갱신
uv run jupyter lab       # 노트북 실행
uv run python main.py    # 스캐폴딩 엔트리포인트 (hello 출력)
```

테스트 스위트, 린터, 빌드 단계가 없음. 검증은 노트북 셀을 재실행해 출력된 데이터프레임을 확인하는 방식으로 이루어짐.

BigQuery 접근에는 Application Default Credentials가 필요함 (`gcloud auth application-default login`). 과금 프로젝트는 모든 `bigquery.Client(...)` 호출에 `pro-talon-503713-s3`으로 하드코딩되어 있고, 데이터 자체는 공개 데이터셋인 `bigquery-public-data.ga4_obfuscated_sample_ecommerce`임.

## 노트북 구성

`data_analysis/`에 AARRR 단계별로 `A(<stage>).ipynb` 형식의 노트북이 있음.

- `A(acquisition).ipynb` — 완료. 주별 신규 사용자 추이 → 채널 → 디바이스 → 지역 순으로 분해하고 최종 요약까지 포함.
- `A(activation).ipynb` — 진행 중. 이벤트 집계 확인과 퍼널 분석 계획까지 작성됨. 퍼널 쿼리 자체는 아직 구현되지 않음.

`docs/GA4_컬럼정의서.md`는 GA4의 중첩 스키마가 어떤 분석 단위(이벤트 / 사용자 / 세션 / `event_params` / 상위 구조 컬럼)에 대응하는지 정리한 기준 문서임. 익숙하지 않은 컬럼으로 쿼리를 작성하기 전에 먼저 읽을 것.

차트 PNG는 `plt.savefig(...)`에 상대 파일명만 넘겨 저장하므로, 커널의 cwd가 `data_analysis/`일 때만 해당 디렉터리에 생성됨.

## 쿼리 작성 시 지켜야 할 분석 관례

아래는 `A(acquisition).ipynb`에서 확립된 결정들로, 이후 단계에서도 일관성을 위해 그대로 따라야 함.

- **분석 기간은 `20201102`~`20210131`**이며 `_TABLE_SUFFIX BETWEEN`으로 적용함. 2020-11-01은 일요일이라 월요일 시작 주별 집계 기준에 포함되지 않으므로 의도적으로 제외한 것임.
- **주별 버킷**: `DATE_TRUNC(PARSE_DATE('%Y%m%d', event_date), WEEK(MONDAY)) AS week_start`.
- **신규 사용자 = `event_name = 'first_visit'`**, `COUNT(DISTINCT user_pseudo_id)`로 집계. 다만 `first_visit`은 이벤트 발생 건수가 고유 사용자 수보다 약 148건 많아 일부 중복이 존재한다는 점을 감안할 것.
- **채널 그룹핑**은 `traffic_source`에 `CASE`를 적용해 도출하며, `medium`을 기준으로 하되 `source` 예외 하나를 둠. 아래 매핑을 그대로 재사용할 것.
  - `medium = 'organic'` → Organic Search
  - `medium = 'cpc'` → Paid Search
  - `medium = '(none)' AND source = '(direct)'` → Direct
  - `medium = 'referral' AND source = 'shop.googlemerchandisestore.com'` → Direct (자사 도메인 셀프 리퍼럴이라 제3자 소개 조건 미충족)
  - `medium = 'referral'` → Referral
  - 그 외 → Unassigned
- **합계 처리**: 집계 쿼리는 읽는 사람이 직접 더하게 두지 않고 `'Total Sum'`(또는 헬퍼 사용 시 `'Total'`) 라벨의 합계 행을 덧붙임. 정렬은 `ORDER BY <col> = 'Total Sum', ...`으로 합계 행을 마지막에 배치함.
- **차트**: Windows에서 한글 라벨 출력을 위해 `plt.rcParams['font.family'] = 'Malgun Gothic'`과 `axes.unicode_minus = False`가 필요함. 누적 막대는 고정된 Google 팔레트(`#4285F4`, `#34A853`, `#FBBC05`, `#EA4335`, `#9AA0A6`)를 쓰고 그 위에 전체 합계를 검은색 라인으로 겹쳐 그림.

## `get_grouped_counts()` 헬퍼

`A(acquisition).ipynb` 안에서 정의되고, 이후 버그 수정본으로 재정의된 함수임. `group_col` 문자열을 SQL에 그대로 보간하고, `AS` 절 또는 마지막 점(dot) 세그먼트에서 결과 별칭을 도출한 뒤 합계 행을 붙임. 이 함수를 수정할 때 반드시 유지해야 할 두 가지:

1. `col == alias` 체크가 `is_numeric_dtype`보다 **먼저** 와야 함. 값이 전부 NULL인 그룹 컬럼은 pandas가 float64로 캐스팅하므로, 순서가 바뀌면 숫자 분기가 실행되어 `"Total"` 라벨이 `0`으로 덮어써짐.
2. 노트북 로컬 함수이므로, 다른 노트북에서 쓰려면 정의를 복사해야 함. 공유 모듈은 없음.

## 이미 검증 후 제외된 컬럼

아래 컬럼들로는 분석을 구성하지 말 것. 각각 노트북에 제외 근거가 서술되어 있음.

- `device.mobile_brand_name`, `device.mobile_model_name` — GA4가 User-Agent로 추론하는 값이라 desktop 트래픽과 브라우저명(Chrome, Safari, Edge, Firefox)이 섞여 들어옴.
- `device.mobile_marketing_name`, `device.vendor_id`, `device.advertising_id`, `device.time_zone_offset_seconds` — 전량 `<Other>` 또는 NULL.
- `device.is_limited_ad_tracking` — 값이 고정된 상수 필드.
- `device.operating_system` — 약 58%가 불특정 값인 `Web`.
- `device.operating_system_version`, `device.web_info.browser_version` — 동일 버전이 표기 형식 차이로 여러 값으로 분열됨.
- `device.language` — NaN 약 29%, 표기 형식 불일치.
- `geo.metro`(100% `(not set)`), `geo.city`(41.6%), `geo.region`(9.4%).
- `traffic_source.name`(캠페인) — 실제 UTM 캠페인명이 단 1건도 없고 GA4 자동 placeholder뿐이라, 캠페인 단위 분석과 CAC 산출이 이 데이터셋에서는 불가능함.

**대신 채택된 컬럼**: `device.category`, `device.web_info.browser`, `geo.continent`, `geo.country`.

## 노트북 작업 방식

각 분석 단계는 목적과 확인 포인트를 서술한 마크다운 셀 뒤에 코드 셀이 오는 구조임. 분석 결과, 가설, 가설 검증 결론(기각/채택), 데이터 품질 이슈, 버그 기록이 모두 코드 옆 마크다운 셀에 함께 남아 있음. 분석을 변경할 때는 주변 서술도 함께 갱신할 것 — 그 서술이 실질적인 결과물이기 때문임.
