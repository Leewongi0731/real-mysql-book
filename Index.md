# Index

## B-TREE INDEX 
### 1. Multi column index (= Concatenated Index)
- Multi column index란 2개 이상의 column으로 구성된 index
- n번째 column은 n-1번째 column에 의존하여 정렬 되기 떄문에, index 내에서 각 column의 순서가 매우 중요

### 2. B-Tree Index 정렬 및 스캔 방향
- B-Tree Index는 각 index의 정렬 규칙에 따라서 index key을 항상 오름차순이거나 내림차순으로만 정렬
  - 8.0 버전부터 column 단위로 정렬 순서를 혼합할 수 있도록 지원 (5.7 버전까지는 정렬 순서를 혼합하여 생성할 수 없었음.) 
- 거꾸로 끝에서부터 읽기가 가능함 (오름차순 정렬이여도, 내림차순으로 읽을 수 있음)
  - 어느 방향으로 읽을지는 옵티마이저가 실시간으로 만들어내는 실행 계획에 따라 결정
  - innodb storage engine에서 정순/역순 스캔은 페이지(블록) 간의 dobule linkded list을 통해 Forward/Backward 하냐의 차이
  - **<span style="color:red">하지만 실제 내부적으로는 역순 스캔이 정순 스캔에 비해 느림!!</span>**
    - 페이지 잠금이 Forward index scan에 적합한 구조
    - 단일 페이지 내에서는 index가 단방향으로만 연결된 구조 
- 용어
  - Ascending index : 작은 값의 index 키가 B-Tree 왼쪽으로 정렬
  - Descending index : 큰 값의 index 키가 B-Tree 왼쪽으로 정렬
  - Forward index scan : 리프 노드의 왼쪽 페이지부터 오른쪽으로 스캔
  - Backward index scan  리프 노드의 오른쪽 페이지부터 왼쪽으로 스캔


### 3. B-Tree Index 가용성과 효율성
- Multi column index에서 각 column의 순서와 쿼리 조건(=, >, <)에 따라 활용/효율이 달라짐
  - case 1. INDEX (a, b) 
    ``` sql
    SELECT * FROM table_name WHERE a='test' AND b>=10;
    ```
    - **a='test' AND b>=10** 인 레코드를 찾고, a가 'test'가 아닐때까지 읽음 (효율적)
  - case 2. INDEX (b, a)
    ``` sql
    SELECT * FROM table_name WHERE a='test' AND b>=10;
    ```
    - **b>=10 ANDa='test'** 인 레코드를 찾고, 그 이후 모든 레코드가 a='test'인지 비교 (비효율적)
- B-Tree Index는 왼쪽 값에 기준하여 오른쪽 값이 정렬되어 있는 형태라 아래와 같은 한계점이 있음
  - case 1. INDEX (a)
    ``` sql
    SELECT * FROM table_name WHERE a LIKE '*test';
    ```
    - '*test' 인 경우, 왼쪽 부분이 고정되지 않았기 떄문에 index를 사용할 수 없음
  - case 2. INDEX (a, b)
    ``` sql
    SELECT * FROM table_name WHERE b>=10;
    ```
    - 선행 column a에 대한 조건이 없기 때문에, index를 효율적으로 사용할 수 없음
- **참고) 인덱스 매칭도**
  - 작업 범위 결정 조건(드라이빙 조건)과 체크 조건 판단 예시
    - index: (컬럼1, 컬럼2, 컬럼3, 컬럼4) (매칭도가 높을수록 효율적인 index)

    | 인덱스 컬럼 | 조건 1 | 조건 2 | 조건 3 | 조건 4 | 조건 5 |
    | --- | --- | --- | --- | --- | --- |
    | 컬럼1 | **BETWEEN** | = | = | = | IN |
    | 컬럼2 | = | = | **>=** | IN | = |
    | 컬럼3 | = | **LIKE** | = | **<** | = |
    | 컬럼4 | LIKE | = | BETWEEN | = | **>** |
    | 매칭도 | 1 | 3 | 2 | 3 | 4 |

- 아래의 경우 B-Tree index를 제대로 사용할 수 없음을 주의
  1. NOT-EQUAL의 비교 (<>, NOT IN, NOT BETWEEN, IS NOT NULL)
  2. LIKE '%문자열'의 비교
  3. 스튜어드 함수나 다른 연산자로 index column이 형된 후 사용하는 경우 (SUBSTRING, DAYOFMONTH 등..)
  4. NOT-DETERMINISTIC 속성의 스튜어드 함수가 비교 조건에 사용 된 경우
  5. 데이터 타입이 서로 다른 비교
  6. 문자열 데이터 타입의 콜레이션이 다른 경우
  7. Multi column index에서 선행 column에 대한 조건이 없거나 index를 탈 수 없는 경우
    
## R-TREE INDEX (Rpatial Index)
### 매커니즘은 B-Tree와 비슷하지만, B-Tree는 1차원 스칼라 값을 사용하고, R-Tree는 2차원 공간 개념값을 사용하여 Index을 구성

- R-Tree index은 MBR(Minimum Bounding Rectangle)의 포함 관계를 B-Tree 형태로 구현한 index
  - MBR : 공간 데이터(_POINT, LINE, POLYGON, GEOMETRY_)를 포함하는 최소 크기의 사각형
  
## Full text Index (전문 검색 Index)
- B-Tree index는 실제 칼럼의 값이 1MB이상 이더라도 1MB, 3072Byte로 잘라서 사용함
- Full text Index는 단어의 어근 분석, n-gram 분석을 통해 구현 가능함
  - 단어 어근 분석 
    - 불용어(Stop Work)를 제거하고, 단어의 어근을 추출하는 작업
    - 어근분석은 검색어로 선정된 단어의 뿌리인 원형을 찾는 작업
      - 각 국가의 언어가 서로 문법이 다르고 다른 형태로 발전해봐왔기 땨문에 언어별로 모두 다른 어근 분석 라이브러리가 필요
  - n-gram 알고리즘
    - 키워드를 검색하기 위한 인덱싱 알고리즘
    - 본문을 무조건 몇 글짜식 짤라ㅣ 인덱싱하는 방법
    - 인데스의 크기가 상당히 큰 편임
    - n-gram 알고리즘은 각 단어를 토큰 라이즈하고, 불용어 처리을 하기 떄문에, 불 필요한 불용어들은 무시하거나 대체하여 사용하는 것이 좋음
      - 토큰 라이즈는 각 글자를 중첩해서 토큰으로 만듬. (2-gram ex. that -> th, ha, at)
  - MySQL 불용어 설정
    - MySQL 내장된 불용어는 `informainon_scheam_innodb_ft_default_stopword` 테이블에 저장되어 있음
    - **불용어 처리 자체를 무시하거나 서버에 내장된 불용어 대신 사용자가 직접 불용어를 등록하는 방법을 권장함**
    - 불용어 처리 설정
      - 불용어가 담긴 `fy_stopword_file` 시스템 변수 수정 
        - 불용어를 사용하지 않으려면, 빈 문자열로 설정 `ft_stopword_file = ''`
        - 사용자 지정 불용어를 사용하려면, 불용어가 담긴 파일의 경로를 설정 `ft_stopword_file = '/usr/local/mysql/stopwords.txt'`
        - 해당 시스템 변수는 서버가 시작될 때만 인지하기 때문에, 서버 시작시에 설정해줘야 함
      - `informainon_scheam_innodb_ft_default_stopword` 시스템 변수를 수정
        - 불용어를 사용하지 않으려면, OFF로 설정 `SET GLOBAL innodb_ft_server_stopword_table = 'OFF';`
        - 사용자 지정 불용어를 사용하려면, 아래와 같이 불용어 테이블을 생성하고 설정
          ``` sql
          CREATE TABLE my_stopword(value VARCHAR(30)) ENGINE=InnoDB;
          INSERT INTO my_stopword VALUES('a'),('an'),('the'),('is'),('of'),('and'),('or'),('not');
          
          SET GLOBAL innodb_ft_server_stopword_table = 'my_stopword';
          ```
        - 해당 시스템 변수는 서버가 실행 중인 상태에서도 변경 가능 함
- Full text index을 사용하려면 아래의 두 조건을 만족해야 함
  - 쿼리 문장이 전문 검색을 위한 문법(Match, Against)을 사용해야 함
  - 테이블이 전문 검색 대상 칼럼에 대해서 전문 인덱스를 가지고 있어야 함

## 함수 기반 인덱스
- 칼럼의 값을 변형해서 만들어진 값에 대해 인덱스를 구축
- MySQL 8.0 버전부터 함수 기반 인덱스를 지원함
- 인덱스의 내부적인 구조 및 유지관리는 B-Tree와 동일함
- 구현 방법
  - 가상 칼럼을 이용한 인덱스
    ``` sql
    ALTER TABLE test_table
    ADD fullname VARCHAR(20) AS (CONCAT(firstname, ' ', lastname)) VIRTUAL,
    ADD INDEX idx_fullname(fullname);
    ```
  - 함수를 이용한 인덱스
    ``` sql
    CREATE TABLE test_table(
      id INT NOT NULL AUTO_INCREMENT,
      firstname VARCHAR(10),
      lastname VARCHAR(10),
      PRIMARY KEY(id),
      INDEX idx_fullname((CONCAT(firstname, ' ', lastname)))
    );
    ```
## Multi Value Index
- 하나의 데이터 레코드가 여러 개의 키 값을 가질 수 있는 형태의 index
- 주로 JSON 데이터 타입 레코드의 배열 타입 필드에 저장된 원소들에 대한 인덱스를 지원하기 위함
- MySQL 8.0 버전부터 지원함

## Clustering Index (= Clustering Key, Clustering Table)
- Clustering Key가 비슷한 것들끼리 묶어서 저장하는 형태의 inedx
  - Clustering Key 선발 기준
    - PK가 있으면 PK을 Clustering Key로 사용
    - NOT NULL option의 unique index 중에서 첫 번째 index을 Clustering Key로 사용
    - 자동으로 유니크한 값을 가지도록 증가되는 칼럼을 내부적으로 추가한 후 Clustering Key로 사용 
- InnoDB Storage Engine에서만 지원 (InnoDB에서는 PK값이 레코드의 물리적인 위치를 결정하는 특징을 이용)
- 구조 자체는 B-Tree와 비슷하지만, **Clustering Index의 leaf node에는 레코드의 모든 칼럼이 같이 저장되어 있음**
- 장점
  - Clustering Key의 검색이 매우 빠름
  - table의 모든 secondary index는 Clustering Key을 포함하고 있기 때문에 인덱스만으로 처리될 수 있는 경우가 많음
- 단점
  - Clustering Key의 크기가 클 경우 모든 전체적으로 index 크기가 커짐
  - secondary index를 통해 검색할 때 Clustering Key로 다시 한 번 검색해야 하므로 처리 성능이 느림
  - 물리적 저장 위치가 Clustering Key에 의해 결정되기 떄문에 INSERT, DELETE, Clustering Key 변경등의 장업이 느림 


