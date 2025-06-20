# 데이터 모델링 실습

## 개요
- 6개월치의 카드 사용 금액 데이터셋을 사용하여 스타 스키마 구조를 생성하고 카드 사용금액 대시보드를 생성하였습니다.

## 데이터셋
- [AI 허브 사이트](https://www.aihub.or.kr/aihubdata/data/view.do?currMenu=115&topMenu=100&&srchDataRealmCode=REALM015&aihubDataSe=data&dataSetSn=71792)에서 제공하는 금융 합성 데이터를 활용하였습니다. 
- 총 12개의 압축 파일로 구성되어 있으며, 이 중 '**01.카드 회원정보**'와 '**03.카드 승인매출정보**' 데이터를 사용하였습니다.
- 각 파일들은  **2018년 7월부터 2018년 12월까지**의 6개의 CSV 파일들로 구성되어 있으며, 각 CSV 파일들은 **300만 건의 레코드**를 포함하고 있습니다.

## ERD 다이어그램
![ERD](./img/erd.png)
## 아키텍처
![ERD](./img/dw_architect.png)

## 진행과정
  ### 1. 데이터 병합
  - '**01.카드 회원정보**'와 '**03.카드 승인매출정보**'에서 필요한 컬럼들만 추출
  - **1800만 건의 레코드**를 포함하는 2개의 원본 테이블 생성
  
  ### 2. 원본 테이블    
  - #### `member_info` - 카드회원 정보 (2018/07 ~ 2018/12)
    - 79개의 컬럼 중 6개의 컬럼 추출
    - 추출한 컬럼 : **`기준년월`**, **`발급회원번호`**, **`남녀구분코드`**, **`연령`**, **`거주시도명`**, **`직장시도명`**

  - #### `payment_amt` - 승인매출 정보 (2018/07 ~ 2018/12)
    - 430개의 컬럼 중 13개 컬럼 추출
    - 추출한 컬럼 : **`기준년월`**, **`발급회원번호`**, **`이용금액_온라인_B0M`**, **`이용금액_오프라인_B0M`**, **`이용금액_페이_온라인_B0M`**, **`이용금액_페이_오프라인_B0M`**, **`이용금액_당사페이_B0M`**, **`이용금액_A페이_B0M`**, **`이용금액_B페이_B0M`**, **`이용금액_C페이_B0M`**, **`이용금액_D페이_B0M`**, **`이용금액_간편결제_B0M`**

  
   
  ### 3. 차원 테이블 및 팩트 테이블
  - 팩트 테이블
    - #### `fact_monthly_amt` - 월별 결제 금액
        | 컬럼명 | 설명 | 타입 |
        | --- | --- | --- |
        | `date_key` | 날짜 Key (**기준년월**) | INT (**FK**) |
        | `member_key` | 고객 Key | VARCHAR (**FK**) |
        | `payment_key` | 결제 수단 Key | INT (**FK**) |
        | `channel_key` | 결제 채널 Key | INT (**FK**) |
        | `total_amt` | 결제 금액 | INT |
    
      <details>
      <summary>쿼리문</summary>
        
        - 병합한 2개의 테이블에서 월별 고객별 지불 수단별 총합의 결과를 unpivot을 사용하여 생성하였습니다. 
  
      ```sql
      CREATE TABLE fact.fact_total_amt AS
      WITH unpivoted AS 
      (
          SELECT 기준년월 AS date_key, 발급회원번호 AS member_key, '이용금액_온라인_B0M' AS col_name, 이용금액_온라인_B0M AS total_amt FROM card_usage.payment_amt WHERE 이용금액_온라인_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_오프라인_B0M', 이용금액_오프라인_B0M FROM card_usage.payment_amt WHERE 이용금액_오프라인_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_페이_온라인_B0M', 이용금액_페이_온라인_B0M FROM card_usage.payment_amt WHERE 이용금액_페이_온라인_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_페이_오프라인_B0M', 이용금액_페이_오프라인_B0M FROM card_usage.payment_amt WHERE 이용금액_페이_오프라인_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_당사페이_B0M', 이용금액_당사페이_B0M FROM card_usage.payment_amt WHERE 이용금액_당사페이_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_A페이_B0M', 이용금액_A페이_B0M FROM card_usage.payment_amt WHERE 이용금액_A페이_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_B페이_B0M', 이용금액_B페이_B0M FROM card_usage.payment_amt WHERE 이용금액_B페이_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_C페이_B0M', 이용금액_C페이_B0M FROM card_usage.payment_amt WHERE 이용금액_C페이_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_D페이_B0M', 이용금액_D페이_B0M FROM card_usage.payment_amt WHERE 이용금액_D페이_B0M IS NOT NULL
          UNION ALL
          SELECT 기준년월, 발급회원번호, '이용금액_간편결제_B0M', 이용금액_간편결제_B0M FROM card_usage.payment_amt WHERE 이용금액_간편결제_B0M IS NOT NULL
      ), with_keys AS (
          SELECT
              date_key,
              member_key,
              CASE col_name
                  WHEN '이용금액_온라인_B0M' THEN 1
                  WHEN '이용금액_오프라인_B0M' THEN 1
                  WHEN '이용금액_페이_온라인_B0M' THEN 2
                  WHEN '이용금액_페이_오프라인_B0M' THEN 2
                  WHEN '이용금액_당사페이_B0M' THEN 3
                  WHEN '이용금액_A페이_B0M' THEN 4
                  WHEN '이용금액_B페이_B0M' THEN 5
                  WHEN '이용금액_C페이_B0M' THEN 6
                  WHEN '이용금액_D페이_B0M' THEN 7
                  WHEN '이용금액_간편결제_B0M' THEN 8
              END AS payment_key,
              CASE col_name
                  WHEN '이용금액_온라인_B0M' THEN 1
                  WHEN '이용금액_오프라인_B0M' THEN 2
                  WHEN '이용금액_페이_온라인_B0M' THEN 1
                  WHEN '이용금액_페이_오프라인_B0M' THEN 2
                  ELSE -1
              END AS channel_key,
              total_amt
          FROM unpivoted
      )
      SELECT
          date_key,
          member_key,
          payment_key,
          channel_key,
          SUM(total_amt) AS total_amt
      FROM with_keys
      GROUP BY date_key, member_key, payment_key, channel_key
      ```
  
      </details>
      
    <details>
    <summary>차원 테이블</summary>
  
    - #### `dim_member` - 회원
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    | `member_key` | 고객 Key  | VARCHAR (**PK**) |
    | `gender` | 성별 (**M/F**) | VARCHAR |
    | `age_group` | 연령대 | VARCHAR |
  
    - #### `dim_date` - 날짜
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    | `date_key` | 날짜 Key (**YYYYMM**)  | INT (**PK**) |
    | `used_date`  | 날짜 (**YYYY-MM-01**) | DATE |
    | `year` | 연도 | INT |
    | `month` | 월 | INT |
    | `quarter` | 분기 | VARCHAR |
  
    - #### `dim_payment`- 결제 수단
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    | `payment_key` | 결제 방식 Key | INT (**PK**) |
    | `payment_name` | 결제 방식 이름 | VARCHAR |
    
      <payment_name 컬럼 값에 따른 표기>
      1 : card
      2 : pay
      3 : a_pay
      4 : b_pay
      5 : c_pay
      6 : d_pay
      7 : simple_pay
      8 : our_pay (당사페이) 
  
    - #### `dim_channel` - 결제 채널
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    | `channel_key` | 결제 채널 Key | INT (**PK**) |
    | `channel_name` | 결제 채널 이름 | VARCHAR |
      <channel_name 컬럼 값에 따른 표기>
      1 : online
      2 : offline
      -1 : unknown
    </details>

### 4. 데이터 마트 
  <details>
    <summary>테이블 목록</summary>

  - #### `dm_monthly_amt` - 전체 월별 총액

    |컬럼명|설명|타입|
    |------|---|---|
    |`used_date`|날짜 (**YYYY-MM-01**) |DATE|
    |`monthly_sum`|총 금액|INT|
  
  - #### `dm_quarter_amt` - 분기별 총액
      
    |컬럼명|설명|타입|
    |------|---|---|
    |`year`|년|INT|
    |`quarter`|분기|VARCHAR|
    |`quarter_amt`|분기별 총액|INT|
  
  - #### `dm_monthly_payment_amt` - 결제 수단별 월별 총액
      
    |컬럼명|설명|타입|
    |------|---|---|
    |`used_date`|날짜 (**YYYY-MM-01**) |DATE|
    |`payment_name`|결제 수단|VARCHAR|
    |`monthly_payment_amt`|결제 수단별 월별 총액|INT|
  
  - #### `dm_monthly_channel_amt` - 유입 채널별 월별 총액
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    |`used_date`| 날짜 (**YYYY-MM-01**)  | DATE |
    |`channel_name`| 유입 채널명 | VARCHAR |
    |`monthly_channel_amt`| 채널별 월별 총액 | INT |
  
  - #### `dm_monthly_gender_amt` - 성별 월별 총액
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    |`used_date`| 날짜 (**YYYY-MM-01**)  | DATE |
    |`gender`| 성별 (**M/F**) | VARCHAR |
    |`monthly_gender_amt`| 성별 월별 총액 | INT |
  
  - #### `dm_monthly_age_amt` - 연령대별 월별 총액
  
    | 컬럼명 | 설명 | 타입 |
    | --- | --- | --- |
    |`used_date` 날짜 (**YYYY-MM-01**)  | DATE |
    |`age_group` 연령대 | VARCHAR |
    |`monthly_age_amt` 연령대별 월별 총액 | INT |
  
  
    
  </details>


    


### 5. 데이터 정합성 & 무결성 체크
- 팩트 테이블에서 key 컬럼들을 FK로 지정 후 데이터 삽입하는 과정에서 20분이 넘어도 삽입이 완료되지 않는 문제가 발생하였습니다.
- 따라서, **`CREATE TABLE ~ AS SELECT`** 로 쿼리 결과를 팩트 테이블로 만든 후 수동으로 데이터 정합성과 무결성을 체크하였습니다.
  
- **원본 테이블**
  - 회원 중 신규 회원 혹은 탈퇴 회원이 존재하는지 **HAVING**절과 **DISTINCT**로 **데이터 정합성** 검사

  ```sql
  select 
    발급회원번호, 
    count(발급회원번호) as cnt
  from card_member.member_info
  group by 1
  having cnt < 6;
  ```


- **팩트 테이블**
  - **`fact_monthly_amt`** 테이블의 외래 키가 참조하는 차원 테이블에 실제로 존재하는 값인지 확인
  - **`NOT EXISTS`** 와 **`IS NULL`** 조건으로 **참조 무결성 & 데이터 정합성** 검사

    <details>
      <summary>쿼리문</summary>
  
      ```sql
      -- date_key
      SELECT
          f.*
      FROM
          fact.fact_monthly_amt AS f
      WHERE
          f.date_key IS NULL
          OR NOT EXISTS (
              SELECT
                  1
              FROM
                  dim.dim_date AS d
              WHERE
                  f.date_key = d.date_key
          );

      -- member_key
      SELECT
          f.*
      FROM
          fact.fact_monthly_amt AS f
      WHERE
          f.member_key IS NULL
          OR NOT EXISTS (
              SELECT
                  1
              FROM
                  dim.dim_member AS d
              WHERE
                  f.member_key = d.member_key
          );

    -- payment_key
    SELECT
        f.*
    FROM
        fact.fact_monthly_amt AS f
    WHERE
        f.payment_key IS NULL
        OR NOT EXISTS (
            SELECT
                1
            FROM
                dim.dim_payment AS d
            WHERE
                f.payment_key = d.payment_key
        );

    -- channel_key
    SELECT
        f.*
    FROM
        fact.fact_monthly_amt AS f
    WHERE
        f.channel_key IS NULL
        OR NOT EXISTS (
            SELECT
                1
            FROM
                dim.dim_channel AS d
            WHERE
                f.channel_key = d.channel_key
        );
      
      ```
    </details>


## 📊 Tableau 시각화
- 대시보드
  <details>
    <summary>시각화</summary>
        <img src="./img/dashboard.png"width="850" height="680"/>
  </details>
    

- 대시보드에 포함 안된 데이터 마트 시각화
  <details>
    <summary>연령대별 월별 카드 사용 금액</summary>
        <img src="./img/age_monthly.png"width="400" height="450"/>
  </details>

  <details>
    <summary>분기별 카드 사용 금액</summary>
        <img src="./img/quarter.png"width="200" height="550"/>
  </details>

## ❗ 알게된 점과 보완할 점
- 데이터 정합성 및 무결성 검사
    - 외래 키(Foreign Key) 제약 조건을 설정하면 무결성과 정합성은 확보되지만, 데이터 양이 많아질수록 삽입 속도가 느려질 수 있다는 점을 알게 되었습니다.
    - 이에 따라, 초기 데이터 적재 시 외래 키 제약 조건을 적용하지 않고, 별도로 정합성 및 무결성 여부를 확인하는 쿼리를 작성하여 수동으로 검사하는 방식으로 진행하였습니다.
- 시간에 따라 변하는 데이터 처리
    - 카드 회원 정보에 **`거주시도명`** 과 **`직장시도명`** 컬럼이 포함되어 있었고, 일부 회원의 경우 해당 값이 **`기준월`** 컬럼에 따라 달라지는 것을 확인했습니다.
    - 당시에는 이러한 변동 데이터를 어떻게 처리해야 할지 몰라 해당 컬럼들을 분석 대상에서 제외하고 프로젝트를 진행하였습니다.
    - 이후 조사 결과, 시간에 따라 값이 변하는 차원 데이터는 SCD(Slowly Changing Dimension)로 관리해야 하며, 분석 목적에 따라 Type 1, Type 2 등 적절한 SCD 유형을 적용해야 한다는 것을 알게 되었습니다.
    - 향후에는 **`거주시도명`** 과 **`직장시도명`** 컬럼을 포함한 차원 테이블을 SCD 방식으로 재구성한 뒤, 대시보드에 포함할 예정입니다.


