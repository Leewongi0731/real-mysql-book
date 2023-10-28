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
                - 전체 정렬된 레코드이ㅡ 건수는 3000개
