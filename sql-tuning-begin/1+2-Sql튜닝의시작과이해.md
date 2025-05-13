# 1. SQL 튜닝의 시작점 

sql 튜닝의 시작은 해당 sql의 의미를 파악하는 것 부터 시작된다.

해당 sql이 어떤 의미인지 모르면 잘못된 튜닝을 할 수 있다. 단순히 IO를 줄이고 수행시간을 단축하기 위해 힌트를 남발하면 안된다.

sql은 결국 명령어기에 의미를 파악해야한다.

#### 🚩 어떤 행동을 수행하는 가?

```oracle
SELECT 
       *
  FROM (
            SELECT /*+ INDEX_DESC(A IDX_MBOX_SENDDATE) */
                   A.*
                 , ROWNUM AS RNUM
              FROM TBS_MBOX A 
             WHERE USERID = :B1
               AND STATUS = :B2
               AND ROWNUM <= :B3
       )
 WHERE RNUM >= :B4;
```

여기서 만약 인덱스가 

IDX_NORMAL : USERID, STATUS 
IDX_MBOX_SENDDATE : USERID, SENDDATE 

이런식으로 걸려있다.

만약 이 SQL의 의미를 분석하지 않았다면 그냥 단순하게 "IDX_NORMAL로 바꿔주면 되는거 아닌가"로 생각하기 쉽다.

하지만 이건 INDEX_MBOX_SENDDATE를 왜썼는지 의미파악을 하지 않은 것이다.

이는 ORDER BY USERID,SENDDATE를 대체하고자 사용하였으며 최신에 보내진 메일중 B3보다 작은 메일중 B4보다 큰 메일을 추출하는 쿼리라는 것을 알 수있다.

그러므로 이걸 만약 인덱스를 바꿔버리면 문제가 생기는 것이다.
이럴 때는 차라리 기존 인덱스에 STATUS를 추가하는 방식이 맞다는 것이다.

> Ⓜ️ COUNT STOPKEY 
>
> 👉 ROWNUM 혹은 FETCH FIRST N ROWS 같은 제한 조건이 걸린 경우,
> 👉 전체 데이터를 다 읽지 않고 중간에 멈추고 결과를 반환한다는 것을 의미합니다.

> 즉,“필요한 개수만큼 데이터를 읽은 후, 더 이상 읽지 않고 멈춤(STOPKEY)”

# 2. 서브쿼리와 성능 문제 이해하기

## 서브쿼리란?

### 1. 서브쿼리

**WHERE절에 비교조건으로 사용되는 SELECT 쿼리**

### ⚠️ 주의

- 서브쿼리를 사용해야 의도된 결과 값을 가져올 수 있는 경우
- 서브쿼리를 이용할 경우 성능이 좋아지는 경우

>위 2가지의 경우를 제외하곤 서브쿼리보단 조인을 사용하는 게 옵티마이저의 정확한 판단을 도운다.

### 기본적인 SQL 지식 

SELECT 절의 실행순서

**FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY**

## 서브쿼리를 활용한 SQL 성능 개선

### 1. 비효율적인 MINUS 대신 NOT_EXISTS를 사용하자

MINUS는 A와 B간의 차집합으로 원하는 결과를 얻고 싶다고 할 경우, A와B 모두 SORTING을 수행하게 된다.

SORTING을 함으로써 부하가 발생하고, 만약 비교 대상 테이블에 조회조건이 적절히 들어가지 않는다면 FULL SCAN에 대한 컬럼들에 대해 
소팅이 들어가서 더욱 성능을 떨어뜨린다.

### 주의사항

하지만 무조건 NOT_EXISTS로 바꾸면 안된다. 왜냐하면 MINUS작업은 중복을 제거하지만, NOT_EXISTS는 중복을 제거하지 않는다.

예를 들어서 1,2,2가 A에서 SELECT되고 1이 B에서 SELECT 된 상태에서 MINUS 작업시 2만 나오지만 , NOT_EXISTS는 2,2 로 중복된 값이 나오게 된다.

#### 그러면 그냥 DISTINCT쓰면 되잖아

참고로 DISTINCT역시 정렬을 수행한다. 그렇기에 항상 SELECT 할때 NOT_EXISTS로 바꿀려고 하는경우

**SELECT 하는 컬럼들이 UNIQUE한지 확인하자 ! PK가 포함되어 있는지**


### 2. 조인대신 서브쿼리를 활용하자.

#### 확인자 테이블

FROM절에 나열된 테이블 중 SELECT 절에 추출되는 컬럼은 없고 단순히 where절에 조인조건이나 filter조건으로 사용되는 테이블을 말한다.

테이블 A,B,C가 있다.만약 테이블 A와 C가 서로 조인 조건이 없어서 A->B->C 순서로 조인을 했다.

하지만 A와 B 두 테이블 간 조인을 수행하면서 중복데이터가 증가했다.하지만 조인순서를 바꿀수도 없으니 문제다.

이럴경우 어처피 B테이블이 확인자 테이블로 쓰인다면, 데이터 정합성에 문제가없다. 이럴때는 조인보다 서브쿼리를 사용하면된다.

#### 기존
```oracle
  SELECT T2.A, T2.B, T3.A
    FROM T1,T2,T3 
   WHERE T2.B = :B1
     AND T1.A = T2.B
     AND T1.C = T3.D
```

#### 변경 (서브쿼리)
```oracle
  SELECT T2.A, T2.B, T3.A
    FROM T2, T3 
   WHERE T2.B = :B1
     AND EXISTS (
          SELECT 
                 'X'
            FROM T1
           WHERE T1.A = T2.B
             AND T1.C = T3.D 
     )
```
### 3. WHERE절의 서브쿼리를 조인으로 변경하자 



