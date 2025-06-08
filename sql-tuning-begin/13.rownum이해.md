# 13. ROWNUM에 대한 이해

ROWNUM은 SELECT 절에 사용시 추출하는 데이터에 순번을 부여하고, WHERE 절에 사용시에는 추출하는 데이터 중 일부만 가져올때 사용된다.

## ROWNUM 데이터를 먼저 추출한 후 조회하자

rownum을 쓸때 1을 기준으로 써야한다. 만약에 rownum > 6을 시작해서 쓴다면, 사용이 안된다. 의도와 다르다는 것. 

그렇기에 사용하려면 select절에 rownum을 주고 써야된다.

## ORDER BY와 ROWNUM을 같은 위치에 두지 말자

order by와 rownum을 같이 쓰는 이유는 정렬된 결과에 대해서 rownum을 사용해 특정 갯수 만큼 데이터를 가져오기 위함이다.

하지만 의도한 바와 다르게 rownum이 order by보다 sql 실행순서상 먼저 실행되므로 , 실제로는 rownum을 사용해 특정개수 만큼 가져온 데이터를 가지고 order by를 하게 됨으로 원하는 바와 다르게 얻게되는 것이다.

> 인라인뷰를 통해서 order by 된 결과를 가지고 rownum을 쓰자 

아래 부분은 내용적으로 크게 당연한 부분이라고 생각한다..

## ROWNUM=1은 ROWNUM <= 1로 사용하자

## INDEX_DESC 와 ROWNUM <= 1 은 같이 사용하지마라

## ROWNUM은 항상 빠르지 않다

## 인라인뷰에 ROWNUM추가시 주의하자
