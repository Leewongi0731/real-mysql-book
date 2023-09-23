# Transaction and lock

## Transaction vs Lock
- Transaction : 데이터의 정합성을 보장하기 위한 기능
- Lock : 동시성을 제어하기 위한 기능

## Transaction in MySQL
- 하나의 논리적인 작업 셋 자체가 100% 적용되거나 아무것도 적용되지 않아야 함을 보장해 주는 것. (작업 셋에 쿼리가 한개이거나 여러개인것은 관계없다)

- MyISAM vs INNODB 
  - Transaction 지원 여부
    - MyISAM은 지원하지 않음.
    - INNODB은 지원함
  - bulk write query 시나리오 (각 table이 [(3)]에서 다음의 쿼리를 실행. 
    ``` sql
    INSERT INTO table name (pk) VALUES (1), (2), (3);
    ```
    - MyISAM, INNODB table모두 dumplication key error가 발생하지만, 실제 DB는 다른 형태를 띄게됨
    - MyISAM table 은 [(1), (2), (3)] 이됨
    - INNODB은 table 은 [(3)] 이됨

## Lock in MySQL
- MuSQL의 Lock은 MySQL 엔진 레벨, 스토리지 엔진 레벨로 나눌 수 있음
  - MySQL 엔진 레벨 Lock
    - 모든 스토리지 엔진에 영향을 미침
    - 테이블 데이터 동기화를 위한 Table Lock, 테이블 구조를 잠그는 Metadata Lock, 사용자 필요에 맞게 사용할 수 있는 Named Lock을 제공
    - Global Lock
      - MySQL에서 제공하는 Lock중에 가장 범위가 큼 (영향 범위 : MySQL 서버 전체)
      - 하나의 세션에서 Globacl Lock을 사용하면 다른 세션에서 SELECT을 제외한 쿼리들이 실행되지 못하고 대기 상태가 됨
      - 작업 대상 테이블/데이터베이스가 다르더라도 영향을 받음
      - Backup Lock
        - Global Lock을 조금 더 가볍게 사용하기 위해 나옴
        - Lock이 걸려도 일반적인 테이블의 데이터 변경은 허용됨
        - 주로 레프리카 서버에서 실행 됨.
    - Table Lock
      - 개별 Table 단위로 설정되는 Lock
      - 명시적, 묵시적으로 Lock을 사용 가능
        - 명시적 : LOCK TABLES table_name [READ | WRITE]로 발생 가능, UNLOCK TABLES 의 명령어로 해제 가능 (일반적으로 사용 X)
        - 묵시적 : MyISAM, MEMORY 테이블에 데이터를 변경하는 쿼리를 사용하면 발생 (**InnoDB는 스토리지 레벨에서 Lock을 사용하기 때문에 묵시적 Table Lock을 사용하지 않음**)
    - Lamed Lock
      - GET_LOCK() 함수로 임의의 문자열에 대해 Lock을 발생 가능
      - 여러 Client가 상호 동기화를 처리해야 할 때나 많은 레코드에 대해 복잡한 요건으로 레코드를 변경하는 경우에 유용하게 사용 (Ex. Batch job)
    - Metadata Lock
      - 데이터베이스 객체(ex. table, view ..)의 이름이나 구조를 변경하는 경우에 사용되는 Lock

  - 스토리지 엔진 레벨 Lock (= InnoDB 스토리지 엔진 Lock)
    - 다른 스토리지 엔진 간 상호 영향을 미치지 않음
    - InnoDB 스토리지 엔진 Lock
      - 레코드 단위의 Lock 기능을 제공
      - InnoDB Lock monitoring
        - 옛 버전의 MySQL : lock_monitor 도구를 이용하여 lock 정보를 dump 하거나 SHOW ENGINE INNODB STATUS 명령어를 이용하여 어셈블리 코드를 확인
        - 신 버전의 MySQL : infomation_schema에 존재하는 INNODB_TRX, INNODB_LCOKS, INNODB_LOCK_WAITS 라는 테이블에서 확인 및 종료 가능
      - Lock의 범위가 커지는 경우는 없음 (락 에스컬레이션 : ~~record lock -> page lock -> table lock~~)
      - Record Lock
        - 다른 DBMS의 Record Lock과 역할은 동일함. 하지만 InnoDB는 **레코드 자체가 아니라 인덱스의 레코드를 잠금**
          - 시나리오. (A field에 대한 index가 존재)
          ``` sql
          SELECT COUNT(*) FROM table_name WHERE A='test';
          > 200
          SELECT COUNT(*) FROM table_name WHERE A='test' AND B='test';
          > 1
          UPDATE table_name SET c='new value' WHERE A='test' AND B='test';
          ```
          **<span style="color:red">UPDATE query에서 record(index) lock은 1개가 아니라 200개가 걸림!!</span>**
      - Gap Lock
        - Record 자체가 아니라, Record 사이의 간격(Gap)을 잠그는 것을 의미
          - Gap은 Record와 Record사이에 데이터가 생성되는 것을 제어 함
      - Next key Lock
        - Record Lock + Gap Lock의 형태
        - innodb_locks_unsafe_for_binlog 시스템 별수가 비활성화 되면 변경을 위해 검색하는 record에는 Next key Lock이 걸림
      - Auto increment Lock
        - insert, replace 쿼리에서 발생
        - Transaction과 관계없이 AUTH_INCREMENT 값을 가져오는 순간만 걸렸다가 즉시 해제됨 (아주 짧은 시간동안 걸렸다가 해제됨. 5.0이후 버전에서는 innodb_autoinc_lock_mode 시스템별수를 변경하여 설정 가능)


## Isolation level
- 여러 Transaction이 동시에 처리될 때 특정 Transaction이 다른 Transaction에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것

||Dirty read |Non-repeatable read|Phantom read|
|---|---|---|---|
|Read uncommitted| 발생 | 발생 | 발생 |
|Read committed| X | 발생 | 발생 |
|Repeatable read| X | X | 발생(innoDB는 없음) |
|Serializable| X | X | X |

1. Read uncommitted
    - 변경 내용이 commit이나 rollback 여부와 상관없이 다른 Transaction에서 조회
2. Read committed
    - Oracle DBMS에서 Default Isolation level
    - commit된 데이터만 조회하여 보여줌
3. Repeatable read
    - Mysql의 InnoDB에서 Default Isolation level
    - 각 Transaction마다 발급되는 Transaction에서 id을 이용힘
      - undo 영역에서 해당 Transaction에서 id보다 table의 trx-id가 크면 undo 영역의 데이터을 읽음
4. Serializable
