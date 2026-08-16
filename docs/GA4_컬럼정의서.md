# GA4 컬럼 정의서

이 문서는 GA4 BigQuery 공개 샘플 데이터에서 어떤 컬럼이 무엇을 의미하는지 쉽게 정리한 문서임.

분석을 할 때는 단순히 컬럼 이름만 보는 것이 아니라, 이 컬럼이 어느 수준의 정보인지 이해해야 함.

- 이벤트 단위 정보인지
- 사용자 단위 정보인지
- 세션 단위 정보인지
- 파라미터(`event_params`)인지
- 상위 구조 컬럼인지

이걸 구분해야 실제로 어떤 데이터를 어떻게 해석할지 정확히 판단할 수 있음.

---

## 1. 데이터 전체 구조 이해하기

GA4 데이터는 크게 이렇게 나눌 수 있음.

1. 이벤트 기본 정보
   - 이벤트가 언제 발생했는지
   - 어떤 이벤트가 발생했는지
   - 어떤 사용자가 어떤 행동을 했는지

2. 사용자/세션 정보
   - 사용자가 누구인지
   - 세션이 어떻게 시작되고 끝나는지

3. 유입 정보
   - 어디서 들어왔는지
   - 어떤 채널로 유입됐는지
   - 캠페인이나 소스 정보

4. 지역/기기 정보
   - 지역별 유입인지
   - 모바일/데스크톱인지
   - 브라우저, OS 정보

5. 상품/거래 정보
   - 구매, 장바구니, 상품, 가격, 환불 관련

6. 파라미터 정보
   - 이벤트마다 추가로 붙는 세부 속성
   - `event_params` 구조

7. 사용자 속성 정보
   - 사용자별로 관리되는 속성
   - `user_properties` 구조

---

## 2. 최상위 컬럼

최상위 컬럼은 이벤트 자체를 설명하는 기본 정보임.

### 대표 컬럼

- `event_date`
  - 이벤트가 발생한 날짜
  - 예: 20201105

- `event_timestamp`
  - 이벤트 발생 시각
  - 마이크로초 단위의 UTC 시간

- `event_name`
  - 이벤트 이름
  - 예: `page_view`, `purchase`, `add_to_cart`, `first_visit`

- `user_pseudo_id`
  - 사용자 식별값
  - 로그인이 안 되어 있어도 동일 사용자를 추적하는 데 사용됨

- `user_id`
  - 로그인한 사용자 ID
  - 비로그인 사용자는 비어 있을 수 있음

- `user_first_touch_timestamp`
  - 사용자가 처음 방문한 시점
  - 신규/재방문 구분에 중요함

- `platform`
  - 이벤트가 발생한 플랫폼
  - 예: WEB, IOS, ANDROID

- `stream_id`
  - 데이터 스트림 식별값

### 핵심 포인트

이런 컬럼은 이벤트의 가장 기본적인 뼈대를 제공함.

즉, "언제, 누구, 어떤 플랫폼에서, 어떤 이벤트가 발생했는지"를 확인하는 기본 축임.

---

## 3. `event_params` 컬럼

`event_params`는 이벤트마다 부가적으로 붙는 세부 속성들을 담는 구조임.

예를 들어 하나의 `page_view` 이벤트에는 다음과 같은 값이 같이 붙을 수 있음.

- 페이지 URL
- 페이지 제목
- 캠페인 이름
- 검색어
- 체류 시간 관련 값

### 구조

`event_params`는 보통 다음처럼 보임.

- `event_params.key`
- `event_params.value.string_value`
- `event_params.value.int_value`
- `event_params.value.double_value`

### 예시

- `key = page_location`
- `value.string_value = https://shop.googlemerchandisestore.com/...`

- `key = campaign`
- `value.string_value = spring_sale`

- `key = search_term`
- `value.string_value = t-shirt`

### 중요한 점

`event_params`는 세부 속성 확인용임.

즉, 이벤트 안에 어떤 값이 들어왔는지 보는 영역이라서, 실제 비즈니스 의미를 해석하려면 상위 컬럼과 함께 봐야 함.

---

## 4. `user_properties` 컬럼

`user_properties`는 사용자 자체에 대한 속성을 저장하는 영역임.

예를 들면 다음과 같은 정보가 여기에 들어갈 수 있음.

- 회원 등급
- 사용자 관심 카테고리
- 가입 여부
- 사용자 속성 값

### 예시

- `membership_level = gold`
- `favorite_category = apparel`
- `visit_count = 12`

### 의미

이 값은 이벤트 자체보다 사용자 중심으로 분석할 때 중요함.

- 신규 사용자 vs 기존 사용자
- 회원/비회원 구분
- 사용자 세그먼트별 분석

이런 분석에 쓰임.

---

## 5. 지역/기기/환경 정보 컬럼

이 영역은 사용자가 어디서 왔는지, 어떤 환경에서 접속했는지 확인하는 데 사용함.

### 지역 정보

- `geo.country`
- `geo.region`
- `geo.city`

### 기기 정보

- `device.category`
- `device.operating_system`
- `device.browser`
- `device.language`

### 의미

- 지역별 유입 비중 확인
- 모바일/데스크톱별 전환율 비교
- 브라우저 또는 OS별 사용자 특성 파악
- 신규 유입의 지역 분포 분석

---

## 6. 유입 관련 컬럼

### 대표 컬럼

- `traffic_source.name`
- `traffic_source.medium`
- `traffic_source.source`
- `source`
- `medium`
- `campaign`

### 의미

이 값들은 어디서 유입됐는지를 보여줌.

예를 들면 다음과 같음.

- `google / cpc / black_friday_campaign`
- `facebook / paid_social / spring_sale`
- `instagram / referral / influencer`
- `(direct) / (none) / direct`

이 값들은 유입 경로를 이해하는 데 중요하지만, 실제 데이터 구조에서는 상위 컬럼과 파라미터가 함께 존재할 수 있음.

---

## 7. 상품/거래 정보 컬럼

이 영역은 구매, 장바구니, 상품 관련 정보를 담음.

### 대표 컬럼

- `ecommerce.total_item_quantity`
- `ecommerce.purchase_revenue`
- `ecommerce.transaction_id`
- `items.item_id`
- `items.item_name`
- `items.item_category`
- `items.price`
- `items.quantity`

### 의미

- 장바구니에 담긴 상품 수 확인
- 구매 금액 확인
- 어떤 상품이 많이 팔렸는지 확인
- 어떤 카테고리의 상품이 많이 판매되는지 확인

---

## 8. 컬럼을 이해할 때 체크해야 할 기준

분석할 때는 아래 기준으로 컬럼을 보자.

1. 이벤트 단위 컬럼인지
   - 예: `event_name`, `event_timestamp`

2. 사용자 단위 컬럼인지
   - 예: `user_pseudo_id`, `user_id`

3. 세션 단위 컬럼인지
   - 예: `session_id`

4. 파라미터 구조인지
   - 예: `event_params.key`, `event_params.value.string_value`

5. 상위 구조 컬럼인지
   - 예: `traffic_source.source`, `geo.country`, `device.category`

6. 실제 비즈니스 의미가 무엇인지
   - 예: 유입 경로인지, 지역인지, 구매인지

이 기준을 이해하면 데이터 구조를 훨씬 쉽게 해석할 수 있음.

---

## 9. 컬럼을 확인할 때의 핵심 기준

GA4 컬럼 정의서를 이해하는 핵심은 다음 한 문장임.

"어떤 정보가 이벤트 자체의 기본 정보인지, 어떤 정보가 사용자/세션 정보인지, 어떤 정보가 파라미터인지, 어떤 정보가 상위 분류 컬럼인지 구분하는 것임."

이걸 이해하면 데이터 전체 구조를 훨씬 쉽게 해석할 수 있음.

- `event_params`는 세부 속성 확인용임
- 상위 컬럼은 분류 정보 확인용임
- 파라미터와 상위 컬럼을 함께 보아야 데이터 의미를 정확하게 해석할 수 있음

---

## 10. 마지막 정리

이 문서는 GA4 데이터 전체를 이해하는 기준점이 되는 문서임.

분석을 시작할 때는 다음 순서로 보는 것이 좋음.

1. 어떤 이벤트인지 확인
2. 누가 발생시켰는지 확인
3. 언제 발생했는지 확인
4. 어떤 플랫폼/기기/지역인지 확인
5. 어떤 파라미터가 붙어 있는지 확인
6. 상위 컬럼에서 추가 분류 정보를 확인
7. 상품, 거래, 사용자 속성까지 함께 확인

이렇게 구조적으로 보면 GA4 BigQuery 데이터를 훨씬 안정적으로 해석할 수 있음.
