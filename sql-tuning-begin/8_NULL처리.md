# 8. NULL 처리

## NULL 처리 함수 이해하기

NVL(EXPR1,EXPR2)

NULL을 다른 값으로 대체한다.

EXPR1이 NULL 이면 EXPR2를 리턴한다 여기서 데이터 타입은 EXPR1을 기준으로 return 된다.

## NVL 활용

NVL과 DECODE는 실행계획이 NULL 여부에 따라 분기된다. 이점을 생각하자.

## 그룹함수(count,sum 등)과 NVL 

count를 제외한 집계함수들은 NULL 데이터를 NULL로 리턴한다.

이점을 주의하자 sum( NULL  +  1 ) = NULL이다.

이런한 연산이나 NULL이 포함된 작업을 주의하여 NVL처리를 잘하자.

## NULLABLE 컬럼 사용에 의한 비효율 COUNT 함수처리

COUNT 함수 사용시 NOT NULL 제약조건이 있는 컬럼인 경우에는 인덱스만 엑세스하지만, NOT NULL 없이 Nullable 컬럼인 경우에는 테이블엑세스가 발생한다.

## IS NULL 조회에 대한 개선방법

IS NULL는 인덱스를 사용하지 못한다.

한가지 경우가 그나마 가능할것으로 보여서 이 경우만 생각하자.

### 컬럼 추가 및 인덱스 생성후 where 절 변경 

결합인덱스의 경우 선두인덱스가 NOT NULL이면 Nullable 성격으이 인덱스 컬럼의 NULL데이터까지 저장하게 된다. NULL데이터도 인덱스 존재하제하기에 IS NULL도 인덱스 사용이 가능해지는 것이다.

기존 컬럼들 중에서도 가능하다. 그리고 또한 그냥 에초에 설계시 디폴트값을 부여하고, NOT NULL을 준다면 IS NULL을 써도 인덱스를 탄다.


## IS NOT NULL 조회에 대한 개선방법 

IS NOT NULL을 데이터 타입에 따라서 다른 조건을 줌으로써 성능 개선이 가능하다.
그 요지는 **그 데이터 타입이 가질수있는 가장 작은 값보다 큰값일경우**로 조건을 변경하는 것. 

### CHAR(VARCHAR2)타입인 경우 

where c1 > CHR(0) 으로 변경한다.
### DATE타입

where c1 > TO_DATE('19000101','yyyymmdd')
### NUMBER타입

where c1 >= 0 OR c1 < 0

## ''(blank)와 NULL 데이터 처리하기

Blank 데이터는 NOT NULL데이터이다. 빈칸은 CHR함수로 표현하면 CHR(0)이다.
