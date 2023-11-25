# Optimizer

## Optimizer 이란

- 요청된 쿼리를 최적의 비용으로 수행하기 위한 실행 계획을 수립하는 기능을 담당
- MySQL에서는 EPLAIN 명령어로를 통해 실행 계획을 확인할 수 있음
- 실행 계획을 이해하는 것은 상당히 어렵지만, 이를 이해하여야 불합리한 부분을 찾아내고 더 최적화된 방법으로 수립을 유도할 수 있음
- 종류
    - Rule-based Optimizer (RBO)
        - 초기 버전의 Oracle DBMS에서 많이 사용
        - 테이블의 레코드 건수, 선택도 등을 고려하지 않고 Optimizer에 미리 정의된 규칙에 따라 실행 계획을 수립
        - 같은 쿼리에 대해서는 거의 항상 같은 실행 계획을 만들어냄
    - **Cost-based Optimizer (CBO)**
        - 현재 대부분의 DBMS는 해당 방법을 사용
        - 각 단위 작업의 비용 정보와 대상 테이블의 예측된 통계 정보를 이용하여 실행 계획을 수립

## 쿼리 실행 절차

1. SQL 파싱 (SQL 파서에서 처리)
    - 요청된 SQL 문장을 잘게 쪼개서 MySQL 서버가 이해할 수 있는 수준으로 분리
2. 최적화 및 실행 계획 수립 (옵티마이저에서 처리)
    - 분리된 SQL 문장에서 어떤 Table, Index가 사용되는지 선택
    - 불필요한 조건 제거 및 복잡한 연산의 단순화
    - 여러 테이블의 조인이 있는 경우 어떤 순서로 테이블을 읽을지 결정
    - 각 테이블에 사용된 조건과 인덱스 통계 정보를 이용하여 사용할 인덱스 결정
    - 가져온 레코드들을 임시 테이블에 넣고 다시 한번 가공
3. (2)에서 결정된 테이블의 읽기 순서나 선택된 인덱스를 이용하여 Storage engine 에서 데이터를 읽음

## MySQL에서 기본 데이터 처리

1. Full Table Scan
    - Index을 사용하지 않고, 테이블의 데이터를 처음부터 끝까지 읽어서 작업을 처리
    - 아래의 조건이 일치하면 주로 Full Table Scan을 선택함
        - 테이블의 레코드 수가 매우 적어서, 인덱스를 읽는 것보다 Full table scan이 빠를 경우
        - 적절한 인덱스를 이용할 수 없는 경우
        - **Index Range Scan이 가능한 쿼리라도 조건 일치 레코드 건수가 너무 많은 경우 (Index의 B-Tree을 샘플링하여 조사한 통계 정보를 기준으로 함)**
    - 데이터 Load
        - InnoDB : 연속된 데이ㅓ 페이지는 Backgroud Thread가 읽어서 Buffer Pool에 적재
        - MyISAM : Disk로 부터 페이지를 하나씩 읽어옴
2. Full Index Scan
    - Index을 처음부터 끝까지 읽어서 작업을 처리
    - `SELECT COUNT(*) FROM table` 와 같은 쿼리는 비교적 Table보다 크기가 작은 Index을 Full sacn 할 확률이 큼
3. 병렬 처리
    - 여기서의 병렬처리는 하나의 쿼리를 여러 thread가 작업을 나누어 처리하는 것을 의미
    - MySQL 8.0에서는 병렬 처리를 지원함
    - `innodb_parallel_read_threads` 시스템 변수를 이용하여 하나의 쿼리를 최대 몇 개의 thread을 이용하여 처리할지 설정 가능
        - 0으로 설정하면 병렬 처리를 사용하지 않음
        - 1로 설정하면 병렬 처리를 사용하지 않음
        - 2 이상으로 설정하면 병렬 처리를 사용함 (기본값은 4임)
4. ORDER BY 처리
    - 구현 방법 (아래로 갈수록 느림)
        - Index을 이용
            - 장점
                - 이미 순서대로 정렬되어 있어서 조회가 빠름
                - 스트리밍 형태의 처리가 가능함 (조건에 일치하는 레코드가 검색될때마다 바로바로 Client에게 전달)
            - 단점
                - INSERT, UPDATE, DELETE 작업 시 부가적인 작업이 필요하기 때문에 느림
                - 부가적인 DISK 공간이 필요함
        - File Sort을 이용 (장단점은 Index을 이용하는 것과 반대)
            - 실행 계획의 Extra 항목에 Using filesort가 표시되면 File Sort을 사용함
        - 조인된 결과를 임시 테이블에 저장 후 정렬
            - 실행 계획의 Extra 항목에 Using temporary, Using filesort가 표시
    - Sort Buffer
        - 정렬을 수행하기 위한 별도의 가변적인 메모리 공간
        - `sort_buffer_size` 시스템 변수로 최대 크기를 설정 가능
        - 정렬해야 할 레코드의 건수가 Sort Buffer의 크기보다 크면 임시 파일을 생성하여 정렬을 수행함
            - 작업을 쪼개서, 임시 파일에 merge하는 작업들이 필요함
            - Sort Buffer를 키운다고 속도가 모든 경우에서 빨라지지는 않음 (56KB ~ 1MB을 추천)
    - Sort Algorithm
        - MySQL 서버의 정렬방식은 아래의 3가지가 존재
            - <sort_key, rowid> : 정렬 키와 레코드의 row id를 저장하는 방식
            - <sort_key, additional_fields> : 정렬 키와 레코드 전체를 저장 (레코드의 칼럼은 고정 사이즈)
            - <sort_key, pack_additional_fields> : 정렬 키와 레코드 전체를 저장 (레코드의 칼럼은 가변 사이즈)
        - Single pass (SELECT 되는 칼럼 모두 Sort Buffer에 저장) (공식 명칭 X)
            - 정렬이 완료되면 Sort Buffer의 내용을 그대로 Client에게 return
            - 최신 버전에서는 일반적으로 해당 방법을 사용
        - Two pass (정렬 대상 칼럼, PK만 Sort Buffer에 저장) (공식 명칭 X)
            - 정렬이 완료되면 Sort Buffer의 PK로 다시 조회하여 Client에게 return
            - 레코드의 크기가 크거나, BLOB이나 TEXT 타입의 칼럼이 있을 경우 사용
    - Sort 관련 상태 변수
        - `SHOW STATUS LIKE 'sort%'`와 같은 명령으로 정렬관련 작업들의 실행 횟수를 확인 가능함
            - Sort_scan : Full Table Scan을 수행한 횟수
            - Sort_merge_passes : multi merge 처리 횟수
            - Sort_range : Range Scan을 수행한 횟수
            - Sort_rows : 정렬한 레코드의 건수
            - example. (Sort_merge_passes : 13, Sort_range : 0, Sort_scan : 1, Sort_rows : 3000)
                - Full Table Scan 결과를 1번 정렬
                - 단위 정렬 작업의 결과를 13번 병합 처리
                - 전체 정렬된 레코드 건수는 3000개
5. GROUP BY 처리
    - GROUP BY 결과에 대한 필터링을 하고 싶을 경우 HAVING을 사용
    - 처리 방법
        - INDEX SCAN을 이용
            - **Extra 칼럼에 `Using index for group-by`, `Using temporary. Using filesort`가 표시되지 않음**
        - LOOSE INDEX SCAN을 이용
            - INDEX record을 건너뛰면서 필요한 부분만 읽어서 가져오는 것을 의미
            - Extra 칼럼에 `Using index for group-by`이 표시됨
            - 단일 table에 대해 수행되는 GROUP BY 처리에만 사용할 수 있음
            - 인덱스의 유니크한 값의 수(_그룹키_)가 적을수록 성능이 향상됨 (분포도가 좋지 않은 인덱스일수록 더 빠른 결과를 만듬)
            - 임시 테이블이 필요하지 않음
            - LOOSE INDEX SCAN을 사용할 수 없는 쿼리 패턴
              ``` sql
              // MIN(), MAX() 이외의 집합 함수가 사용되었을때.
              SELECT c1, SUM(c2) FROM table_name GROUP BY c1;
                
              // GROUP BY 절에 사용된 칼럼이 인덱스의 첫 번째 칼럼이 아닐때.
              SELECT c1, c2 FROM table_name GROUP BY c2, c3;
                
              // SELECT 절에 칼럼이 GROUP BY와 일치하지 않을 때
              SELECT c1, c3 FROM table_name GROUP BY c1, c2;
              ``` 
            - 사용되는 Example
              ``` sql
              EXPLAIN SELECT no FROM table WHERE data='target' GROUP BY no;
              ```
                - 상황. table은 (no, data) 인덱스가 존재함
                - 쿼리 결과의 Extra에 `Using where; Using index for group-by`이 표시됨
                - 쿼리의 동작 순서
                    1. (no, data) 인덱스를 차례대로 스캔하면서 no의 첫 번째 그룹키 "10001"을 찾아냄 (가정)
                    2. (no, data) 에서 no가 "10001"인 것 중에서 data 값이 `target`인 레코드만 가져옴
                    3. (no, data) 인덱스에서 no의 다음 그룹키를 가져옴
                    4. (3)번 단계에서 결과가 더 없으면 처리를 종료하고, 결과가 있으면 (2)으로 돌아가서 계속 수행
        - 임시 테이블을 이용
            - INDEX을 전혀 사용하지 못할때는 해당 방식을 이용
            - **<span style="color:orange">동일한 table, 동일한 query의 실행계획이 version마다 다를 수 있음</span>**
                - 8.0 이전에는 GROUP BY가 사용된 쿼리는 GROUPING되는 칼럼을 기준으로 묵시적인 정렬까지 수행함 => Using temporary, Using
                  filesort가 표시됨
                - 8.0 이후부터는 묵시적인 정렬을 수행하지 않음 (GROUP BY 칼럼들로 구성된 유니크 인덱스를 가진 임시 테이블을 만들어서 연산을 수행) => Using
                  temporary만 표시됨
6. DISTINCT 처리
    - 특정 칼럼의 유니크한 값만 조회하기 위해 사용
    - 처리 방법
        - 집합 함수 없이 DISTINCT만 사용
            - 단순히 SELECT되는 레코드 중 유니크한 레코드만을 가져오고자 할 때 사용
            - GROUP BY와 동일한 방식으로 처리 됨
            - DISTINCT는 함수가 아니기 때문에 뒤에 괄호는 의미가 없음
            - **<span style="color:red">`SELECT DISTINCT c1, c2 FROM table_name`
              와 `SELECT DISTINCT(c1), c2 FROM table_name` 은 완전히 동일한 쿼리임</span>**
            - `DISTINCT c1, c2` 은 c1만 유니크하게 조회하는 것이 아니라, (c1, c2) tuple이 유니한 것
        - 집합 함수와 함께 사용된 DISTINCT
            - 집합 함수 (COUNT(), SUM(), AVG()) 내에서 사용된 DISTINCT는 그 집합 함수의 인자로 전달된 칼럼값이 유니크한 것들을 가져옴
    - 아래의 쿼리의 차이점을 잘 기억해두는게 좋음
      ``` sql
      SELECT DISTINCT c1, c2 FROM table_name WHERE no BETWEEN 10001 AND 10200;
      
      SELECT COUNT(DISTINCT c1), COUNT(DISTINCT c2) FROM table_name WHERE no BETWEEN 10001 AND 10200;
      
      SELECT COUNT(DISTINCT c1, c2) FROM table_name WHERE no BETWEEN 10001 AND 10200;
      ```
7. Internal Temporary Table 활용
    - MySQL engine이 Storage engine으로 부터 받아온 record을 정렬하거나 grouping 할 때 사용
    - `CREATE TEMPORARY TABLE`로 만드는 임시 테이블과는 다른 것임
    - 다른 session, query에서는 해당 table을 사용할 수 없음
    - 쿼리의 처리가 완료되면 자동으로 삭제됨
    - Extra 칼럼에 `Using temporary`이 표시됨
    - 기본적으로는 메모리에 생성되었다가, 크기가 커지면(`1GB`) 디스크로 옮겨짐 (특정 케이스에는 디스크에 바로 만들어지기도 함)
        - 버전/상황에 따라 사용하는 Storage engine이 다름

          |  | 메모리를 사용 | 디스크를 사용 |
                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | --- | --- | --- |
          | 8.0 미만 | MEMORY Storage engine (가변 길이 타입을 지원하지 않음) | MyISAM Storage engine (트랜잭션을 지원하지 못함) |
          | 8.0 이상 | TempleTable Storage engine | InnoDB Storage engine |

        - 디스크에 생성되는 경우
            - UNION, UNION ALL에서 SELECT되는 칼럼 중에서 길이가 512Byte 이상의 칼럼이 있을 때
            - GROUP BY, DISTINCT 칼럼에서 512Byte 이상의 칼럼이 있을 때
            - Internal Temporary Table 크기가 tmp_table_size or max_heap_table_size 보다 큰 경우 (MEMORY Storage engine
              case)
            - Internal Temporary Table 크기가 temptable_size 보다 큰 경우 (TempleTable Storage engine case)
        - `SHOW SESSION STATUS LIKE 'Created_tmp%'` 을 통해 메모리 / 디스크 중 무엇을 사용했는지 확인 가능
    - 사용되는 쿼리
        - 인덱스를 사용하지 못할 때 일반적으로 사용
        - ORDER BY, GROUP BY에 명시된 칼럼이 다른 쿼리
        - ORDER BY, GROUP BY에 명시된 칼럼이 JOIN 순서상 첫 번째 테이블이 아닌 쿼리
        - DISTINCT와 ORDER BY가 동시에 사용된 쿼리
        - UNION이 사용된 쿼리
        - 쿼리의 실행 계획에서 select_type이 DERIVED인 쿼리

## 최적화 (OPTIMIZATION)

- 옵티마이저 옵션
    - 조인 관련된 옵티마이저 옵션
        - MySQL 서버 초기 버전부터 제공된 옵션
    - 옵티마이저 스위치 옵션
        - MySQL 5.5버전부터 지원
        - 고급 최적화 기능들의 활성화 여부를 제어하는 용도
        - 각 switch들은 `default`, `on`, `off` 가 존재
        - `SET GLOBAL optimizer_switch = 'switch_name=on,..';`과 같이 사용 가능 (GLOBAL 말고 SESSION으로도 설정 가능
        - 종류 ([docs](https://dev.mysql.com/doc/refman/8.0/en/switchable-optimizations.html))
            - MRR (Multi-Range Read) & batched_key_access
                - MySQL 서버는 기본적으로 driving table에서 읽어온 record를 하나씩 읽어서 조인을 수행함 (_**Nested Loop Join**_) 건별로
                  처리하다보니, 데이터를 읽어오는 Storage engine은 아무런 최적화를 수행할 수 없음
                - 위와같은 단점을 보안하기위해 driving table record을 join buffer에 저장해두고, join buffer에 저장된 record들을 한 번에 읽어서
                  조인을 수행하는 방식을 지원 (이러한 읽기 방법은 _**MRR**_ 이라고 하고, 이를 응용한 JOIN 방식이 _**Batched Key Access JOIN**_ 임)
                - BKA JOIN은 상황에 따라 성능이 안좋을수도 있어서, 해당 flag는 기본적으로 `off` 임
            - block_nested_loop
                - Block Nested Loop Join은 Nested Loop Join 방식에 join buffer를 적용한 것
                - 처리 순서 (example)
                  ``` sql
                  SELECT * FROM table1, table2 WHERE table1.a > 10 AND table2.b < 1000;
                  ```
                    1. table1의 a 칼럼을 기준으로 10보다 큰 record을 검색
                    2. 조인에 필요한 나머지 칼럼을 table1 테이블로부터 읽어서 `join buffer`에 저장
                    3. table2의 b 칼럼을 기준으로 1000보다 작은 record을 검색
                    4. **(3)<span style="color:red">에</span> (2)번의 캐시된 record를 조인**
                        - **<span style="color:red">Driven table record에 join buffer을 join하는 것이기 때문에, Join 결과의
                          정렬 순서가 흐트러질 수 있음</span>**
                - 해당 flag는 기본적으로 `on` 임
            - index_condition_pushdown
                - MySQL 5.6 버전부터 Index Condition Pushdown 기능을 도입
                - example
                    ``` sql
                    SELECT * FROM table WHERE a='Lee' AND b LIKE '%gi';
                    ```
                    - 상황. table 테이블에는 (a, b) 인덱스가 존재. a가 `Lee`인 record는 1000건, 두 조건을 모두 만족하는 record는 1건
                    - flag가 `off`라면
                        1. Storage engine은 인덱스에 매칭되는 index들을 찾음
                        2. Storage engine이 해당 index의 record 1000개를 MySQL 엔진에 올려보냄
                        4. MySQL 엔진은 `b LIKE '%gi'` 조건을 검사해서 999건을 버림
                    - flag가 `on`이라면
                        1. Storage engine은 인덱스에 매칭되는 index들을 찾음
                        2. Storage engine이 `b LIKE '%gi'` 필터링 조건을 보고 매칭되는 index을 줄임
                        3. Storage engine이 해당 index의 record 1개를 MySQL 엔진에 올려보냄
                - 해당 flag는 기본적으로 `on` 임
            - use_index_extensions
                - innoDB Storage engine을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 PK를 활용할 수 있게 해주는 옵션
                    - ex. PK : a, b, index : c => c를 검색할 때 a, b를 활용할 수 있게 해줌 (c, a, b 인 index와 흡사하게 동작)
                - 해당 flag는 기본적으로 `on` 임
            - index_merge
                - 하나의 테이블에 대해 2개 이상의 INDEX을 이용해 쿼리를 처리할 수 있게 해주는 옵션
                - 쿼리에 사용된 각각의 조건이 다른 INDEX을 사용할 수 있고, 조건을 만족하는 record 건수가 많을 것으로 예상되면 index merge 실행 계획을 선택함
                    - 하나의 INDEX만으로 작업 범위를 충분히 줄일 수 있는 경우면, 하나의 INDEX만 활용함
                - 결과를 어떤 방식으로 merge할 것인지에 따라 3가지로 구분됨
                    - index_merge_intersection
                        - 각각의 INDEX를 이용해 검색된 record들을 교집합으로 처리
                            - ex. `SELECT * FROM table WHERE a=1 AND b=2;`
                        - 쿼리 결과의 Extra에 `Using intersect`이 표시됨
                        - GLOBAL, SESSION, CURRENT QUERY 단위로 on, off 조절이 가능
                    - index_merge_union
                        - 각각의 INDEX를 이용해 검색된 record들을 합집합으로 처리
                            - ex. `SELECT * FROM table WHERE a=1 OR b=2;`
                        - 각 INDEX 조건에 만족하느 record들을 PK로 정렬되어 있으므로, Priority Queue를 이용해 정렬을 수행함 (PK값을 비교하면서 하나씩
                          merge table에 insert)
                        - 쿼리 결과의 Extra에 `Using union({index name}, {index name})`이 표시됨
                    - index_merge_sort_union
                        - 각각의 INDEX를 이용해 검색된 record들을 합집합으로 처리하고, 정렬까지 수행
                        - 쿼리 결과의 Extra에 `Using sort_union({index name}, {index name});`이 표시됨
                - 해당 flag들(4개)은 기본적으로 `on` 임
            - semijoin
                - 테이블간의 실제 조인을 수행하지 않고, 다른 테이블에서 조건에 일치하는 record가 존재하는지만 확인하는 쿼리
                - `SET SESSION optimizer_switch='semijoin=off';` 로 off 가능
                - 해당 flag는 기본적으로 `on` 임
                - semijoin 최적화는 아래의 전략들을 모아서 부르는 말임
                    - Table Pull Out
                        - subquery에 사용된 테이블을 outer query로 끌어온 후에 query을 join query으로 변경함
                            - subquery 최적화가 도입되기 이전에 수동으로 query을 튜닝하던 대표적인 방법
                            - ex. `IN (subquery)` 형태의 쿼리을 `JOIN (subquery)` 형태로 튜닝한 것과 비슷하게 실행
                        - 쿼리 결과의 Extra에 별도로 표기되는 문구는 없음
                            - EXPLAIN 후에 SHOW WARNINGS를 통해 확인했을때, optiomizer가 re write한 query을 보고 최적화 여부를 판단 가능
                        - Table Pull Out 최적화는 아래에 경우에만 사용 가능
                            - semijoin subquery에서만 사용 가능
                            - subquery 부분이 UNIQUE index나 PK lookup으로 결과가 1건인 경우에만 사용 가능
                    - First Match
                        - `IN (subquery)` 형태의 쿼리을 `EXISTS (subquery)` 형태로 튜닝한 것과 비슷하게 실행
                        - 쿼리 결과의 Extra에 `FirstMatch({table name})`이 표시됨
                        - MySQL 5.5에서 수행하던 `IN-to-EXISTS` 변환과 유사하게 처리되지만, First Match 전략을 다음과 같은 장점을 가짐
                            - 여러 테이블이 JOIN되는 경우 원래 쿼리에는 없던 동등 조건을 Optimizer가 추가하기도 함
                            - Subquery의 모든 Table에 대해 First Match 최적화를 수행할지 아니면 일부 Table에만 수행할지 취사선택할 수 있음
                        - First Match 최적화의 특징
                            - subquery는 subquery가 참조하는 모든 outer table이 먼저 조회된 이후에 실행됨
                            - 지집합 함수(GROUP BY, DISTINCT ..)가 사용된 경우에는 사용할 수 없음
                    - Loose Scan
                        - `LOOSE INDEX SCAN`와 비슷한 읽기 방식을 사용함
                        - 쿼리 결과의 Extra에 `LooseScan`이 표시됨
                        - Loose Scan 최적화의 특징
                            - `LOOSE INDEX SCAN`으로 subquery table을 읽고, 그 다음으로 outer table에 JOIN을 수행
                    - Materialization
                        - subquery을 통째로 구체화해서 query을 최적화함
                            - 구체화란 내부의 Temp Table을 생성한다는 의미
                        - 쿼리 결과의 Extra의 select_type에 `MATERIALIZED`가 표시됨
                        - Materialization 최적화의 특징
                            - IN (subquery) 에서 subquery는 상관 서브쿼리 (Correlated subquery)가 아니어야 함
                                - Correlated subquery : subquery에서 외부테이블의 값을 참조할 때 사용
                            - **subquery는 GROUP BY나 집합 합수들이 사용되도 최적화를 사용 가능함**
                            - 내부 Temp Table을 사용함
                    - Duplicate Weedout
                        - semijoin subquery을 일반적인 INNER JOIN query로 바꿔서 실행하고 마지막에 중복된 record을 제거하는 방법으로 처리
                            - 원본 query을 INNER JOIN + GROUP BY 절로 바꿔서 실행됨
                        - 쿼리 결과의 Extra에 `Start temporary`, `End temporary`이 표시됨 (두 문구 구간이 subquery의 Duplicate
                          Weedout 최적화의 실행 구간임)
                        - Duplicate Weedout 최적화의 특징
                            - subquery가 상관 서브쿼리 (Correlated subquery)이여도 사용 가능
                            - subquery는 GROUP BY나 집합 합수들이 있으면 사용 불가
            - condition_fanout_filter
                - 조건절에 사용된 조건을 만족하는 record의 비율을 계산하여 사용
                    - 두 조건을 만족하면 조건절을 만족하는 record의 비율을 계산할 수 있다.
                        - WHERE 조건절에 사용된 column에 대해 INDEX가 있는 경우
                        - WHERE 조건절에 사용된 column에 대해 히스토그램이 존재하는 경우
                    - 다음과 같은 우선순위로 두고 사용 가능한 방식을 사용함
                        - Range optimizer을 이용하여 예측 (실제 INDEX의 데이터를 살펴보고 record 건수를 예측, 소량의 데이터를 먼저 읽어봄)
                        - 히스토그램을 이용하여 예측
                        - INDEX 통계을 이용하여 예측
                        - 추측에 기반한 예층 (Guesstimates)
                    - 예측치를 계산하여 어떻게 사용하는거지??
                - condition_fanout_filter 최적화를 활성화하면 optimizer는 더 정교한 계산을 걸쳐서 실행 계획을 수립하기 때문에 더 많은 시간과 자원을 사용함
                - 해당 flag는 기본적으로 `on` 임
            - derived_merge
                - MySQL 5.7 도입되었으며, Temp table을 만드는 subquery을 외부 query와 병합해서 subquery 부분을 제거하는 최적화
                    - 과거에는 FROM 절의 subquery을 temp table로 만든 후, 그 결과가지고 외부 query을 처리하였지만 아래와 같은 문제들이 있었음
                        - temp table을 만들고, 그 이후 다시 읽는 과정에서 오버헤드가 추가됨
                        - temp table은 초기에는 Memory에 적재되지만, 크기가 커지면 Disk로 옮겨가는데 이 과정에서도 오버헤드가 추가됨
                - derived_merge 최적화를 사용하면, 쿼리가 다음과 같이 변경됨
                    ``` sql
                    -- AS-IS
                    SELECT *  
                    FROM (SELECT * FROM table_name WHERE c1 = 1) derived_table 
                    WHERE derived_table.c2 = 2;
                      
                    -- TO-BE
                    SELECT *
                    FROM table_name 
                    WHERE ((c1 = 1)) AND ((c2 = 2));
                    ```
                - subquery가 다음을 하나라도 만족할 경우, optimizer가 자동으로 derived_merge 최적화를 수행할 수 없음
                    - WINDOW FUNCTION, 집계 합수(SUM, MIN, MAX ..), GROUP BY, HAVING, DISTINCT, LIMIT, UNION, UNION
                      ALL 을 하나라도 포함한 경우
                    - 외부 query의 SELECT 절에 사용된 경우
                    - 값이 변경되는 사용자 변수를 사용
                - 해당 flag는 기본적으로 `on` 임
            - use_invisible_indexes
                - MySQL 8.0 도입되었으며, index을 삭제하지 않고도 optimizer가 해당 index를 사용하지 않도록 할 수 있음
                - 아래와 같이 사용 가능/불가능하게 설정할 수 있음
                  ``` sql
                  ALTER TABLE table_name ALTER INDEX index_name VISIBLE; -- 사용 가능하게 설정
                  ALTER TABLE table_name ALTER INDEX index_name INVISIBLE; -- 사용 불가능하게 설정
                  ```
                - 해당 flag는 기본적으로 `off` 임
            - skip_scan
                - MySQL 8.0 도입되었으며, 제한적이지만 (A, B, C) index를 B, C 조건에서도 사용할 수 있게 할 수 있는 기법
                - 실제 사용되지 않은 선행 컬럼이 매우 다양한 값을 가지는 경우에는 skip_scan 최적화가 매우 비효율적임
                - 해당 flag는 기본적으로 `on` 임
            - hash_join
                - MySQL 8.0.18 부터 hash join을 지원하기 시작함
                    - 8.0.17 까지는 `Block Nested Loop Join`을 사용하였지만, block의 크기가 join buffer보다 작으면 driving table을
                      여러번 scan 해야하는 문제가 있었음
                - nested loop join vs hash join
                    - nested loop join은 첫 번째 record을 찾는건 빠르지만, 마지막 record을 찾는데 오래 걸림
                        - Best Response time 전략에 적합
                        - 일반적인 웹 서비스에 적합함
                    - hash join은 첫 번째 record을 찾는건 오래걸리지만, 마지막 record을 찾는데 빠름
                        - Best Throughput 전략에 적합
                        - 일반적인 데이터 분석과 같은 서비스에 적합함
                    - RDBMS인 MySQL은 일반적으로 데이터 분석보다는 웹 서비스에 사용함으로, hash join은 자선책 기능으로 생각하는게 좋음
                - hash join은 `Build phase`, `Probe phase`로 나누어서 처리됨
                    - `Build phase` : Join 대상 table 중 record가 적어 hash table로 만들기 용이한 table을 찾아서, hash table을 만드는
                      작업
                    - `Probe phase` : 나머지 table의 record을 읽어서 hash table의 일치 record을 찾는 과정.
                    - hash table 크기는 `join_buffer_size` 으로 조정 가능함 (default : 256KB)
                    - buffer size가 부족하면, hash table (build table), probe table을 chunk 단위로 분리한 다음 disk로 옮겨가면서 처리함
                - 해당 flag는 기본적으로 `on` 임
            - prefer_ordering_index
                - MySQL 8.0.21 도입되었으며, ORDER BY을 위한 index에 너무 가중치를 부여하지 않도록 설정 가능
                - MySQL 8.0.20 까지의 optimizer는 아래의 상황에서 order by index에 높은 가중치를 두고 잘못된 판단을 내릴때가 가끔 있었음
                    - example.
                        ``` sql
                        SELECT *  
                        FROM table_name 
                        WHERE c1 BETWWEN '1985-01-01' AND '1985-12-31'
                        ORDER BY pk;
                        ```
                        1. (c1) index을 이용하여 WHERE 절에 일치하는 record을 찾은 다음 pk로 정렬하여 결과를 반환
                        2. pk키를 순서대로 읽으면서 c1 column의 조건에 일치하는지 비교 후 결과를 반환

                        - 1번의 매우 효과적이였을 상황에서도, 2번으로 실행하여 pk을 full scan하는 상황이 가끔 있었음.
                - 해당 flag는 기본적으로 `on` 임
    - JOIN 최적 알고리즘
        - Exhaustive 검색 알고리즘
            - MySQL 5.0 까지 사용되던 JOIN 최적화 기법
            - FROM젛에 명시된 모든 table의 조합에 대해 실행 계획의 비용을 계산해서 최적의 조합 1개를 찾는 방식
            - JOIN하는 table이 20개라면 20! = 3,628,800 개의 모든 case의 비용을 계산 -> table이 조금만 많아져도 매우 오래 걸림
        - Greedy 검색 알고리즘
            - MySQL 5.0에 도입된 JOIN 최적화 기법
            - 아래의 순서대로 진행됨
                1. 전체 N개의 table 중에서 `optimizer_search_depth` 변수에 정의된 개수의 table로 가능한 JOIN 조합을 생성
                2. (1)에서 생성된 조합 중 최소 비용의 실행 계획 하나를 확정
                3. (2)에서 선정된 실행 계획의 첫 번째 테이블을 "**부분 실행 계획**"의 첫 번째 테이블로 선정
                4. 전체 N-1개의 (_**(3)에서 선정된 table 제외**_) table 중에서 `optimizer_search_depth` 변수에 정의된 개수의 table로 가능한
                   JOIN 조합을 생성
                5. (4)에서 생성된 조합 중 하나씩 (3)번에서 생성된 "**부분 실행 계획**"에 대입해 실행 비용을 계산
                6. (5)번의 비용 계산 결과, 최적의 실행 계획에서 두 번째 table을 "**부분 실행 계획**"의 두 번째 table로 선정
                7. 모든 테이블이 없어질 떄까지 (4)번부터 (6)번까지 반복
                8. 최종적으로 "**부분 실행 계획**"이 table의 JOIN 순서로 결정
            - JOIN 최적화를 위한 시스템변수
                - optimizer_search_depth
                    - `optimizer_search_depth`는 0부터 62까지의 정숫값을 설정할 수 있음 (default : 62)
                        - 0으로 설정시 Exhaustive 검색 알고리즘을 사용함
                        - 1~62로 설정시 Greedy 검색 알고리즘을 사용함
                    - `optimizer_search_depth`의 변수 값에 따라 JOIN 최적화 비용이 차이가 남
                        - JOIN table 수가 7 ~ 10 개 미만이라면 큰 문제가 되지 않음
                        - JOIN table 수가 12개 이면서 `optimizer_search_depth`가 12 이상이라면, 며칠이 걸릴 수 있음.
                          이떄 `optimizer_search_depth`을 3~4로만 낮춰도 1분이내로 단축시킬 수
                          있음 ([docs](https://dev.mysql.com/doc/refman/8.0/en/controlling-query-plan-evaluation.html))
                        - 필자는 4~5 정도로 설정하는 것을 추천함
                    - RUN TIME중에 변경 가능
                    - JOIN될 table의 개수가 해당 설정 값보다 크다면, 해당 설정 값만큼은 Exhaustive 검색 알고리즘을 사용하고 나머지 table은 Greedy 검색
                      알고리즘을 사용함
                - optimizer_prune_level
                    - `optimizer_prune_level는 0, 1 둘중 하나로 설정할 수 있음 (default : 1)
                        - 0으로 설정시 Heuristic 검색 알고리즘을 사용하지 않음
                        - 1로 설정시 Heuristic 검색 알고리즘을 사용함
                    - Heuristic 검색 알고리즘
                        - Heuristic 알고리즘은 비용을 계산하던 도중 앞서 계산했던 최적의 JOIN 순서의 비용보다 큰 경우 계산 도중에 포기함
