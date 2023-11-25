# Hint

# Hint 이란?

- Optimizer에게 query의 실행 계획을 어떻게 수립할지 알려줄 수 있는 방법
- MySQL Hint에은 Index Hint, Optimizer Hint가 있음

## Index Hint

- Index Hint는 `SELECT`, `UPDATE` 에서만 사용 가능
- Index Hint 보다는 Optimizer Hint를 사용하는 것이 추천

### STRAIGHT_JOIN

- `STRAIGHT_JOIN`은 Hint인 동시에 Join keyword임
- `SELECT`, `UPDATE`, `DELETE` query에서 여러 개의 table이 JOIN되는 경우 join 순서를 고정하는 역할
- `FROM` 절에 명시된 순서대로 JOIN을 수행
    - example.
        ``` sql
        SELECT STRAIGHT_JOIN 
            a.col1, a.col2, b.col3 
        FROM t1 a, t2 b, t3 c 
        WHERE a.col1=b.col1 AND b.col2 = c.col2;
        ```
        - table1, table2, table3 순서대로 JOIN을 수행
- 아래와 기준에 맞게 JOIN 순서가 결정되지 않을 경우에 사용하는 것이 좋음 (**아래에서 말하는 record 수는 WHERE 절에 명시된 조건에 의해 결정되는 record 수임**)
    - Temp Table과 일반 Table의 JOIN
        - 일반적으로 Temp Table을 Driving Table로 사용하는 것이 좋음
        - 일반 Table의 Join column에 index가 없는 경우에는 record 건수가 작은 쪽은 Driving Table로 사용하는 것이 좋음
    - Temp Table과 Temp Table의 JOIN
        - 둘다 index가 없기 때문에 크기가 작은 Temp Table을 Driving Table로 사용하는 것이 좋음
    - 일반 Table과 일반 Table의 JOIN
        - 모두 index가 있거나 없는 경우에는 record 건수가 작은 쪽을 Driving Table로 사용하는 것이 좋음
        - index가 없는 table을 Driving Table로 사용하는 것이 좋음
- `STRAIGHT_JOIN`와 비슷한 역할을 하는 Optimizer Hint는 아래의 것들이 있음 (아래에서 설명)
    - `JOIN_FIXED_ORDER`, `JOIN_ORDER`, `JOIN_PREFIX`, `JOIN_SUFFIX`

### USE INDEX / FORCE INDEX / IGNORE INDEX

- INDEX 사용법이나 좋은 실행 계획이 어떤 것인지 파단하기 힘든 상황이라면 HINT을 사용하는것은 좋지 않음
- 최적의 실행 계획은 Data의 성격에 따라서 시시각각 변하므로, 지금은 PK을 사용하는 것이 좋은 실행 계획이여도 실행 시점에 따라서 변할 수 있음
- 가장 휼룡한 최적화는 그 query를 service에서 없애 버리거나 튜닝할 필요가 없게 data을 최소화 하는 것
- USE INDEX
    - 가장 자주 사용되는 INDEX Hint
    - Optimizer에게 특정 Table의 Index을 사용하도록 권장 (**<span style="color:red">사용할지는 Optimizer가 판단!</span>**)
- FORCE INDEX
    - USE INDEX와 비슷한 역할을 수행
    - USE INDEX보다 Optimizer에게 보다 더 강하게 영향을 미치게하는 Hint
    - 필자 경험상, USE INDEX을 사용했는데 해당 INDEX을 사용하지 않는 경우라면, FORCE INDEX를 사용해도 해당 INDEX을 사용하지 않음
- IGNORE INDEX
    - 특정 INDEX을 사용하지 않도록 설정하는 Hint
    - 때때로 Optimizer가 Table Full Scan을 사용하도록 유도하기 위해 해당 Hint를 사용하기도 함
- example.
  ``` sql
  SELECT * FROM t1 WHERE col1 = 10001;
  SELECT * FROM t1 USE INDEX(primary) WHERE col1 = 10001;
  SELECT * FROM t1 FORCE INDEX(primary) WHERE col1 = 10001;
  SELECT * FROM t1 IGNORE INDEX(primary) WHERE col1 = 10001;
  SELECT * FROM t1 FORCE INDEX(col2_index_name) WHERE col1 = 10001;
  ```
    - 첫 번째부터 세 번째까지의 query는 모두 t1 table의 PK을 이용하여 동일한 실행 계획으로 처리함
    - 네 번째 query는 primary index을 사용하지 않고, table full scan을 수행함
    - 다섯 번째 query는 query와 관계없는 index을 Hint로 주었기 때문에, table full scan을 수행함

## Optimizer Hint

- Optimizer Hint는 영향 범위에 따라 4개의 그룹으로 나누어 볼 수 있음
    - INDEX : 특정 INDEX의 이름을 사용할 수 있는 Hint
        - INDEX, NO_INDEX
            - 기존의 Index Hint을 대체하는 용도로 사용
                - USE INDEX -> INDEX
                - USER INDEX FOR GROUP BY -> GROUP_INDEX
                - USE INDEX FOR ORDER BY -> ORDER_INDEX
                - IGNORE INDEX -> NO_INDEX
                - IGNORE INDEX FOR GROUP BY -> NO_GROUP_INDEX
                - IGNORE INDEX FOR ORDER BY -> NO_ORDER_INDEX
            - 기존의 Index Hint는 특정 table뒤에 사용했기 때문에 별도로 table을 명시하지 않았지만, Optimizer Hint는 table명을 명시해야 함
            - example.
              ``` sql
              // Index Hint 사용
              SELECT * FROM t1 USE INDEX(t1_col2) WHERE col2 = 'lee wongi';
              // Optimizer INDEX Hint 사용
              SELECT /** INDEX(t1 t1_col2) */ * FROM t1 WHERE col2 = 'lee wongi';
              ```
        - INDEX_MERGE, NO_INDEX_MERGE
            - INDEX_MERGE Hint를 사용하면, 해당 table에 대해 여러 개의 Index을 사용하여 처리할 수 있음
            - Index merge 실행 계획은 때로는 성능 향상에 도움이 되지만 항상 그렇지는 않음
            - example.
              ``` sql
              SELECT /*+ INDEX_MERGE(t1, t1_idx1, t1_idx2) */ * FROM t1 WHERE col1 = 10001;
              ```
        - NO_ICP (Index Condition Pushdown)
            - Index Condition Pushdown 최적화는 사용 가느하다면 항상 성능 향상에 도움이 되므로, Optimizer는 최대한 해당 기능을 사용하는 방향으로 실행 계획을
              수립함 (CIP Hint가 따로 없는 이유)
            - NO_ICP Hint를 사용하면, Index Condition Pushdown을 사용하지 않도록 설정할 수 있음
        - SKIP_SCAN, NO_SKIP_SCAN
            - 선행 column이 가지는 유니크 갯수가 많아서 index skip 성능이 오히려 떨어지는 경우 NO_SKIP_SCAN Hint를 사용하여 index skip을 사용하지 않도록
              설정할 수 있음
            - example.
              ``` sql
              SELECT /*+ NO_SKIP_SCAN(t1 index_name) */ col2, col3 FROM t1 WHERE col3 >= '2022-01-01';
              ```
    - TABLE : 특정 TABLE의 이름을 사용할 수 있는 Hint
        - MERGE, NO_MERGE
            - FROM 절의 Subquery을 외부 Query와 병합하여 처리할지, Temp Table을 사용하여 처리할지 제어
            - example.
              ``` sql
              SELECT /*+ MERGE(sub) */ * FROM (SELECT * FROM t1 WHERE col2='test') sub LIMIT 100
              SELECT /*+ NO_MERGE(sub) */ * FROM (SELECT * FROM t1 WHERE col2='test') sub LIMIT 100
              ```
        - INDEX_MERGE, NO_INDEX_MERGE (위를 참고)
        - NO_ICP (위를 참고)
        - SKIP_SCAN, NO_SKIP_SCAN (위를 참고)
    - QUERY BLOCK : 특정 QUERY BLOCK에 사용할 수 있는 Hint (해당 QUERY BLOCK에만 영향을 미침)
        - SEMIJOIN, NO_SEMIJOIN
            - SEMIJOIN Hint는 SEMIJOIN 세부 전략 중 어떠한 것을 사용할지를 제어하는데 사용
            - NO_SEMIJOIN는 특정 전략을 사용하지 않게 하게 제어하는 사용
            - example.
              ``` sql
              SELECT * FROM t1 WHERE col1 IN (SELECT /*+ SEMIJOIN(MATERIALIZATION) */ col1 FROM t2)
              SELECT /*+ SEMIJOIN(@subq1 MATERIALIZATION) */ * FROM /** QB_NAME(subq1) */ t1 WHERE col1 IN (SELECT col1 FROM t2)
              SELECT * FROM t1 WHERE col1 IN (SELECT /*+ NO_SEMIJOIN(DUPSWEEDOUT, FIRSTMATCH) */ col1 FROM t2)
              ```
        - SUBQUERY
            - SEMIJOIN 최적화가 사용되지 못할 때 사용하는 최적화 방법
            - `SUBQUERY(INTOEXISTS)`, `SUBQUERY(MATERIALIZATION)` 2가지 형태로 최적화할 수 있음
            - SEMIJOIN 최적화와 비슷하게 Subquery에 Hint를 사용하거나 query block name을 지정하여 사용 가능
        - BNL, NO_BNL
            - Block Nested Loop Join을 사용할지를 제어하는데 사용
            - MySQL 8.0.20 버전부터는 BNL Hint을 사용해도 Hash Join을 사용하도록 변경됨
            - example
              ``` sql
              SELECT /*+ BNL(t1, t2) */ * FROM t1 JOIN t2 ON t1.col1 = t2.col1
              ```
        - HASH_JOIN, NO_HASH_JOIN
            - MySQL 8.0.18 버전에서만 유효
            - MySQL 8.0.20 버전부터는 Hash Join 사용/비사용을 유도하려면 위의 BNL Hint를 사용해야 함
        - JOIN_FIX_ORDER, JOIN_ORDER, JOIN_PREFIX, JOIN_SUFFIX
            - JOIN 순서를 결정하기 위한 Hint (전통적으로는 위의 `STRAIGHT_JOIN`을 사용해왔음)
                - `JOIN_FIXED_ORDER` : `STRAIGHT_JOIN` 와 동일하게 FROM 절에 명시된 순서대로 JOIN을 수행하도록 제어
                - `JOIN_ORDER` : Hint에 명시한 table 순서대로 JOIN을 수행하도록 제어
                - `JOIN_PREFIX` : Driving table만 강제하는 Hint
                - `JOIN_SUFFIX` : Driven table만 강제하는 Hint (가장 마지막에 Join돼야 할 table들)
    - GLOBAL : 전체 QUERY에 대해서 영향을 미치는 Hint
        - MAX_EXECUTION_TIME
            - 실행 계획에 영향을 미치지 않는 Hint. 단순히 query의 실행 시간을 제한하는 역할을 함 (단위 : ms)
            - example
            ``` sql
            SELECT /*+ MAX_EXECUTION_TIME(1000) */ * FROM t1 LIMIT 100;
            ```
        - SET_VAR
            - MySQL의 System Variable을 변경하는 Hint
            - [Server System Variables](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html)
              에서 `SET_VAR Hint Applies`가 Yes 인 변수만 설정 가능
            - 잘못된 값이나 변경하지 못하는 System Variable을 변경하려고 하면, Warning이 발생하고 해당 Hint는 무시됨
                - warning message
                  example. `Message: Variable 'collation_server' cannot be set using SET_VAR hint`
            - example
            ``` sql
            SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ * FROM t1 LIMIT 100;
            ```
        - BNL, NO_BNL (위를 참고)
        - HASH_JOIN, NO_HASH_JOIN (위를 참고)
