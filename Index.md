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

### 1. 구조 및 특징
- R-TREE INDEX은 MBR(Minimum Bounding Rectangle)의 포함 관계를 B-Tree 형태로 구현한 index
  - MBR : 공간 데이터(_POINT, LINE, POLYGON, GEOMETRY_)를 포함하는 최소 크기의 사각형
