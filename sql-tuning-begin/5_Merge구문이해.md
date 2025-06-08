# 5. Merge구문의 이해

```sql
MERGE INTO (TARGET_TABLE)
USING (SELECT 문이 들어간다 - SOURCE TABLE)
ON (TARGET_TABLE과 SOURCE TABLE간 조인 -> 유니크해야한다. SELECT 문에 포함되지 않는
컬럼이 들어가야됨. 아니면 ROWNUM써야된다)
WHEN MATCHED THEN 
    UPDATE SET 
    DELETE WHERE
    INSERT () VALUES ()
WHEN NOT MATCHED THEN 
    INSERT () VALUES ()
    WHERE 

COMMIT; -- MERGE구문은 트랜잭션 반영위해서 COMMIT 수행한다.
```

머지를 쓰는 이유는 CURSOR에서 결국 행을 패치해서 가져오는 과정을 한번에 할 수 있는 것

USING을 통해서 가져오고, 아래 분기에 따라서 처리가 가능하다는 것이다.

또한 명시적으로 CURSOR를 오픈해서 사용하는 경우에도 PL/SQL과 SQL간의 엔진 변화로 인해서 컨텍스트 스위칭이 발생하여 성능을 저하시키기도 한다.

