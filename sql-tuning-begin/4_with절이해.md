# 4. with 절의 이해


#### ⚠️ with절은 동일한 데이터를 반복적으로 읽어야 하는 상황에서 고민하자

모든 with절이 Global Temporary Table을 생성하는 것이 아니다.

SQL이 실행되는 횟수에 따라서 달라진다

> GLobal Temporary Table 
>
> Session 레벨의 임시데이터를 저장하는 용도

### Inline View 

SQL의 실행횟수가 1회일때 옵티마이저가 이 방식으로 작동한다. 그냥 우리가 아는 inline view로 변형되어서 실행된다.

그렇기에 필터조건을 걸어서 데이터양을 줄일수도 있다.


### Materialize 

WITH절을 Global Temporary Table을 만들어 적재하고, 그곳에서 읽어오는 방식

그렇기에 SQL이 2회 이상 실행될 경우 사용된다. 


## 😁 WITH절이 SQL 성능개선에 도움을 주는 경우

### 1. 데이터 중복 엑세스 제거하기

실제로 with절이 도움이 되는 경우는 with절에서 추출되는 결과가 적으면서,
데이터 추출시 IO가 많은 경우에 도움이 된다. 

데이터를 1회 추출하여 저장하고 읽어서 사용하면 되기 때문이다.

> 만약 with절에 추출되는 데이터가 많으면 GTT를 생성하고 저장한다음, 읽는 비용 역시 만만치 않기에 생각해봐야 되는 문제다.

### 2. View Predicating 성능 문제 제거

뷰 외부저건이 뷰 내부로 침투하지 못해서 뷰에서 테이블 FULL SCAN이 일어나고 그 데이터에 대해 조건이 걸리는 경우가 있다.

이때 optimizer는 sql 성능 개선을 위해서 뷰 외부조건을 뷰 내부로 침투시키려고 하는 데, 이때 성공하면 그게 **View Predicating**이 발생했다고 하는 것.

#### 문제가 되는 쿼리 

```sql
SELECT T1.c1,
       T2.c2,
       T3.c2,
       T3.c3
  FROM T1 T1,
  	   T2 T2,
  	   (
  	   		SELECT c1,c2,sum(c3) c3 
  	   		  FROM T3
  	   		 GROUP BY c1,c2
  	   ) T3 
 WHERE T1.c1 = T2.c1(+)
   AND T1.c1 = T3.c3(+) 
   AND T1.c1 = 'A' 
```
#### 1. T1 조회 부분을 T3 인라인뷰 내부에 추가하여 성능 개선

```sql
SELECT T1.c1,
       T2.c2,
       T3.c2,
       T3.c3
  FROM T1 T1,
  	   T2 T2,
  	   (
  	   		SELECT c1,c2,sum(c3) c3 
  	   		  FROM T3,
  	   		  	   (
  	   		  	    SELECT c1,c2
  	   		  	      FROM T1
  	   		  	     WHERE T1.c1 = 'A'
  	   		  	   ) T1
  	   		 WHERE T1.c1 = T3.c1 
  	   		 GROUP BY c1,c2
  	   ) T3 
 WHERE T1.c1 = T2.c1(+)
   AND T1.c1 = T3.c3(+) 
   AND T1.c1 = 'A' 
```

이렇게 되면 테이블 A를 결국 2번 처리한 것을 볼 수있다.

#### 2. With 절을 선언해서 인라인뷰 내부에 추가

```sql
WITH T1 AS (
			SELECT /*+ materialize */
				   c1,c2 
		      FROM T1
		     WHERE c2 = 'A'
			)
SELECT T1.c1,
       T2.c2,
       T3.c2,
       T3.c3
  FROM T1,
  	   T2 T2,
  	   (
  	   		SELECT c1,c2,sum(c3) c3 
  	   		  FROM T3,
  	   		  	   T1
  	   		 WHERE T1.c1 = T3.c1 
  	   		 GROUP BY c1,c2
  	   ) T3 
 WHERE T1.c1 = T2.c1(+)
   AND T1.c1 = T3.c3(+) 
```
### 3. 계층쿼리의 데이터 처리 최소화 하기 

계층 구조 분석대상이 되는 데이터 추출 부분에 대해서 먼저 그룹핑을 수행하도록 해야된다.

이때 중요한건 무조건 Materialize 힌트를 적는 것 


## 🚩 주의사항

### 1. 동시성 상황에선 Materialize 고려

Materialize 방식은 컨트롤 파일을 읽게된다.

이 과정에서 Oracle Wait Event가 발생하므로 오히려 DB 서버에 약영향을 줄 수 있다.

### 2. 추출건수가 많은 WITH 절 피하기

### 3. WITH절 선언문은 쿼리 가장 앞에 위치하자

호환성문제

### 4. WITH절에힌트를 추가하자

성능 개선을 위해서 사용하는 경우에는 inline, materialize 같은 힌트를 명시적으로 작성해주자.
