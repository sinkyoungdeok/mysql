
- 전체 17개의 장으로 구성돼 있고, MySQL 서버의 안쪽부터 바깥쪽으로 살펴보는 순서대로 구성돼 있다.
  - MySQL을 관리해야 하는 DBA가 목표면 순서대로 읽으면서 공부하면 좋을 것이다.
  - MySQL을 이용하는 애플리케이션을 개발하는 개발자면 관심 대상이나 업무 역할에 따라 공부할 순서나 중요도를 적절히 조절할 필요가 있다.
- 아래의 내용은 대략적으로 공부하는 순서를 초, 중, 고급으로 구분해서 나열한 것인데, MySQL을 체계적으로 살펴보고자 한다면 아래의 순서가 도움 될 것이다.

**필수 내용**
- 6장 실행 계획
- 7장 쿼리 작성 및 최적화
- 8장 확장 검색
- 9장 사용자 정의 변수
- 11장 스토어드 루틴
- 13장 프로그램 연동
- 14장 데이터 모델링
- 15장 데이터 타입
- 16장 베스트 프랙티스 

**중급 내용**
- 5장 인덱스
- 10장 파티션
- 12장 쿼리 종류별 잠금

**고급 내용**
- 2장 설치와 설정
- 3장 아키텍처
- 4장 트랜잭션과 격리 수준
- 17장 응급처치


## 사용하는 예제 데이터베이스의 ERD와 각 테이블 구조
![image](https://user-images.githubusercontent.com/28394879/137692687-f7159e93-21aa-4b24-81ee-9a7ac636e48b.png)

```mysql
CREATE TABLE 'departments' (
    'dept_no' char(4) NOT NULL,
    'dept_name' varchar(40) NOT NULL,
    PRIMARY KEY ('dept_no')
    KEY ux_deptname' ('dept_name')
) ENGINE=InnoDB;

CREATE TABLE 'employees' (
    'emp_no' int(11) NOT NULL,
    'birth_date' date NOT NULL,
    'first_name' varchar(14) NOT NULL,
    'last_name' varchar(16) NOT NULL,
    'gender' enum('M', 'F') NOT NULL,
    'hire_date' date NOT NULL,
    PRIMARY KEY ('emp_no'),
    KEY 'ix_firstname' ('first_name'),
    KEY 'ix_hiredate' ('hire_date')
) ENGINE=InnoDB;

CREATE TABLE 'salaries' (
    'emp_no' int(11) NOT NULL,
    'salary' int(11) NOT NULL,
    'from_date' date NOT NULL,
    'to_date' date NOT NULL,
    PRIMARY KEY ('emp_no', 'from_date')
    KEY 'ix_salary' ('salary')
) ENGINE=InnoDB;

CREATE TABLE 'dept_emp' (
    'emp_no' int(11) NOT NULL,
    'dept_no' char(4) NOT NULL,
    'from_date' date NOT NULL,
    'to_date' date NOT NULL,
    PRIMARY KEY ('dept_no','emp_no'),
    KEY 'ix_fromdate' ('from_date'),
    KEY 'ix_empno_fromdate' ('emp_no','from_date')
) ENGINE=InnoDB;

CREATE TABLE 'dept_manager' (
    'dept_no' char(4) NOT NULL,
    'emp_no' int(11) NOT NULL,
    'from_date' date NOT NULL,
    'to_date' date NOT NULL,
    PRIMARY KEY ('dept_no','emp_no')
) ENGINE=InnoDB;

CREATE TABLE 'titles' (
    'emp_no' int(11) NOT NULL,
    'title' varchar(50) NOT NULL,
    'from_date' date NOT NULL,
    'to_date' date DEFAULT NULL,
    PRIMARY KEY ('emp_no','from_date','title'),
    KEY 'ix_todate' ('todate')
) ENGINE=InnoDB;

CREATE TABLE 'employee_name' (
    'emp_no' int(11) NOT NULL,
    'first_name' varchar(14) NOT NULL,
    'last_name' varchar(16) NOT NULL,
    PRIMARY KEY ('emp_no'),
    FULLTEXT KEY 'fx_name' ('firstname', 'last_name')
) ENGINE=InnoDB;

CREATE TABLE 'tb_dual' (
    'fd1' tinyint(4) NOT NULL,
    PRIMARY KEY ('fd1')
) ENGINE=InnoDB;
```

# 2. 설치와 설정

<details> <summary> 7. 예제 데이터 적재</summary>

## 7. 예제 데이터 적재

- docker를 활용한 mysql 적재
```
$ docker run --platform linux/amd64 -d -p 3306:3306 \
-e MYSQL_ALLOW_EMPTY_PASSWORD=true \
--name mysql \
-v /Users/singyeongdeog/Documents/mysql:/var/lib/mysql \
mysql:5.7 --character-set-server=utf8 --collation-server=utf8_unicode_ci

$ docker exec -it mysql bin/bash

$ apt-get update
$ apt-get install wget
$ apt-get install bzip2
$ wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2
$ tar jxvf employees_db-full-1.0.6.tar.bz2
$ cd employees_db
$ mysql
$ SOURCE employees.sql
```


</details>

# 3. 아키텍처

<details> <summary> 1. MySQL 아키텍처 </summary>

## 3.1 MySQL 아키텍처

- 이 장의 목적은 MySQL의 쿼리를 작성하고 튜닝할 때 필요한 기본적인 MySQL의 구조를 훑어 보는데 있다.
  - MySQL은 다른 DBMS에 비해 구조가 상당히 독특하다.
  - 사용자 입장에서 보면 거의 차이가 느껴지지 않지만 이런 독특한 구조 때문에 다른 DBMS에서는 가질 수 없는 혜택을 누릴 수도 있으며, 반대로 다른 DBMS에서는 문제되지 않을 것들이 가끔 문제가 되기도 한다.


### 3.1.1 MySQL의 전체 구조 
![image](https://user-images.githubusercontent.com/28394879/138015478-1f63e917-968f-41e9-9338-1c21e98e66ef.png)

- MySQL은 일반 상용 RDBMS에서 제공하는 대부분의 접근법을 모두 지원한다.
  - MySQL 고유의 C API 부터 시작해 JDBC나 ODBC, 그리고 .NET의 표준 드라이버를 제공하며, 이러한 드라이버를 이용해 C/C++, PHP, 자바, 펄, 파이썬, 루비나 .NET 및 코볼까지 모든 언어를 이용해 MySQL 서버에서 쿼리를 사용할 수 있게 지원한다.
- MySQL 서버는 크게 MySQL 엔진과 스토리지 엔진으로 구분해서 볼 수 있다.
  - 여기에서는 MySQL의 쿼리 파서나 옵티마이저 등과 같은 기능을 스토리지 엔진과 구분하고자 위의 그림에서는 "MySQL 엔진"과 "스토리지 엔진"으로 구분했다.
  - 그리고 이 둘을 모두 합쳐서 그냥 MySQL 또는 MySQL 서버라고 표현하겠다.

**MySQL 엔진**

- MySQL 엔진은 클라이언트로부터의 접속 및 쿼리 요청을 처리하는 커넥션 핸들러와 SQL 파서 및 전처리기, 그리고 쿼리의 최적화된 실행을 위한 옵티마이저가 중심을 이룬다.
  - 그리고 성능 향상을 위해 MyISAM의 키 캐시나 InnoDB의 버퍼 풀과 같은 보조 저장소 기능이 포함돼 있다.
  - 또한, MySQL은 표준 SQL(ANSI SQL -92) 문법을 지원하기 때문에 표준 문법에 따라 작성된 쿼리는 타 DBMS와 호환되어 실행될 수 있다.
  
**스토리지 엔진**

- MySQL 엔진은 요청된 SQL 문장을 분석하거나 최적화하는 등 DBMS의 두뇌에 해당하는 처리를 수행하고, 실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어오는 부분은 스토리지 엔진이 전담한다.
  - MySQL 서버에서 MySQL 엔진은 하나지만 스토리지 엔진은 여러 개를 동시에 사용할 수 있다.
  - 다음 예제와 같이 테이블이 사용할 스토리지 엔진을 저장하면 이후 해당 테이블의 모든 읽기 작업이나 변경 작업은 정의된 스토리지 엔진이 처리한다.
  ```
  mysql> CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB;
  ```
  - 위 예제에서는 test_table은 InnoDB 스토리지 엔진을 사용하도록 정의했다.
  - 이제 test_table에 대해 INSERT, UPDATE, DELETE, SELECT, ... 등의 작업이 발생하면 InnoDB 스토리지 엔진이 그러한 처리를 담당하게 된다.


**핸들러 API**

- MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리지 엔진에게 쓰기 또는 읽기를 요청하는데, 이러한 요청을 핸들러(Handler) 요청이라고 하고, 여기서 사용되는 API를 핸들러 API라고 한다.
  - InnoDB 스토리지 엔진 또한 이 핸들러 API를 이용해 MySQL 엔진과 데이터를 주고받는다.
  - 이 핸들러 API를 통해 얼마나 많은 데이터(레코드) 작업이 있었는지는 "SHOW GLOBAL STATUS LIKE 'Handler%';" 명령으로 확인할 수 있다.


### 3.1.2 MySQL 스레딩 구조
![image](https://user-images.githubusercontent.com/28394879/138018864-85b789f3-7677-4672-a687-07eec6c00ece.png)

- MySQL 서버는 프로세스 기반이 아니라 스레드 기반으로 작동하며, 크게 포그라운드(Foreground) 스레드와 백그라운드(Background) 스레드로 구분할 수 있다.

- **포그라운드 스레드(클라이언트 스레드)**
  - 포그라운드 스레드는 최소한 MySQL 서버에 접속된 클라이언트의 수만큼 존재하며, 주로 각 클라이언트 사용자가 요청하는 쿼리 문장을 처리하는 것이 임무다.
    - 클라이언트 사용자가 작업을 마치고 커넥션을 종료하면 해당 커넥션을 담당하던 스레드는 다시 스레드 캐시(Thread pool)로 되돌아간다.
    - 이때 이미 스레드 캐시에 일정 개수 이상의 대기 중인 스레드가 있으면 스레드 캐시에 넣지 않고 스레드를 종료시켜 일정 개수의 스레드만 스레드 캐시에 존재하게 된다.
    - 이렇게 스레드의 개수를 일정하게 유지하게 만들어주는 파라미터가 thread_cache_size다.
  - 포그라운드 스레드는 데이터를 MySQL의 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우에는 직접 디스크의 데이터나 인덱스 파일로부터 데이터를 읽어와서 작업을 처리한다.
    - MyISAM 테이블은 디스크 쓰기 작업까지 포그라운드 스레드가 처리하지만(MyISAM도 지연된 쓰기가 있지만 일반적인 방식은 아님) InnoDB 테이블은 데이터나 버퍼나 캐시까지만 포그라운드 스레드가 처리하고, 나머지 버퍼로부터 디스크까지 기록하는 작업은 백그라운드 스레드가 처리한다.


- **백그라운드 스레드**
  - MyISAM의 경우에는 별로 해당 사항이 없는 부분이지만 InnoDB는 여러 가지 작업이 백그라운드로 처리된다.
    - 대표적으로 인서트 버퍼(Insert Buffer)를 병합하는 스레드, 로그를 디스크로 기록하는 스레드, InnoDB 버퍼 풀의 데이터를 디스크에 기록하는 스레드, 데이터를 버퍼로 읽어들이는 스레드, 그리고 여러 가지 잠금이나 데드락을 모니터링하는 스레드가 있다.
    - 이러한 모든 스레드를 총괄하는 메인 스레드로 있다.
  - 모두 중요한 역할을 하지만 그중에서도 가장 중요한 것은 로그 스레드(Log thread)와 버퍼의 데이터를 디스크로 내려쓰는 작업을 처리하는 쓰기 스레드(Write thread)일 것이다.
    - 쓰기 스레드는 윈도우용 MySQL 5.0에서부터 1개 이상을 설정할 수 있었지만 리눅스나 유닉스 계열 MySQL에서는 5.1 버전부터 쓰기 스레드의 개수를 1개 이상으로 지정할 수 있게 됐다.
    - 이 쓰기 스레드의 개수를 지정하는 파라미터는 innodb_read_io_threads 다.
    - InnoDB에서도 데이터를 읽는 작업은 주로 클라이언트 스레드에서 처리되기 때문에 읽기 쓰레드는 많이 설정할 필요가 없지만, 쓰기 스레드는 아주 많은 작업을 백그라운드로 처리하기 때문에 일반적인 내장 디스크를 사용할 때는 2~4 정도, DAS나 SAN과 같은 스토리지를 사용할 때는 4개 이상으로 충분히 설정해 해당 스토리지 장비가 충분히 활용될 수 있게 하는 것이 좋다.

- SQL 처리 도중 데이터의 쓰기 작업은 지연(버퍼링)되어 처리될 수 있지만 데이터의 읽기 작업은 절대 지연될 수 없다.(사용자가 SELECT 쿼리를 실행하는데, "요청된 SELECT는 10분 뒤에 결과를 돌려주겠다"라고 응답을 보내는 DBMS는 없다).
  - 그래서 일반적인 상용 DBMS에는 대부분 쓰기 작업을 버퍼링해서 일괄 처리하는 기능이 탑재돼 있으며 InnoDB 또한 이러한 방식으로 처리한다.
  - 하지만 MyISAM은 그렇지 않고 사용자 스레드가 쓰기 작업까지 함께 처리하도록 설계돼 있다.
  - 이러한 이유로 InnoDB에서는 INSERT와 UPDATE 그리고 DELETE 쿼리로 데이터가 변경되는 경우, 데이터가 디스크의 데이터 파일로 완전히 저장될 때까지 기다리지 않아도 된다.
  - 하지만 MyISAM에서 일반적인 쿼리는 쓰기 버퍼링 기능을 사용할 수 없다.


> MySQL에서 사용자 스레드와 포그라운드 스레드는 똑같은 의미로 사용된다.
> 클라이언트가 MySQL 서버에 접속하게 되면 MySQL 서버는 그 클라이언트의 요청을 처리해 줄 스레드를 생성해 그 클라이언트에게 할당해 준다.
> 이 스레드는 DBMS의 앞단에서 사용자(클라이언트)와 통신하기 떄문에 포그라운드 스레드라고 하며, 또한 사용자가 요청한 작업을 처리하기 때문에 사용자 스레드라고도 한다.

### 3.1.3 메모리 할당 및 사용구조

![image](https://user-images.githubusercontent.com/28394879/138021759-f3421f96-7d21-46de-9c5b-3bdff5319bdf.png)

- MySQL에서 사용되는 메모리 공간은 크게 글로벌 메모리 영역과 로컬 메모리 영역으로 구분할 수 있다.
  - 글로벌 메모리 영역의 모든 메모리 공간은 MySQL 서버가 시작되면서 무조건 운영체제로부터 할당된다.
  - 운영체제의 종류에 따라 다르겠지만 요청된 메모리 공간을 100% 할당해줄 수도 있고, 그 공간만큼 예약해두고 필요할 떄 조금씩 할당해주는 경우도 있다.
  - 각 운영체제의 메모리 할당 방식은 상당히 복잡하며, MySQL 서버가 사용하고 있는 정확한 메모리 양을 측정하는 것 또한 쉽지 않다.
  - 그냥 단순하게 MySQL의 파라미터로 설정해 둔 만큼 운영체제로부터 메모리를 할당받는다고 생각하는 것이 좋을 듯하다.
- 글로벌 메모리 영역과 로컬 메모리 영역의 차이는 MySQL 서버 내에 존재하는 많은 스레드가 공유해서 사용하는 공간인지 아닌지에 따라 구분되며 각각 다음과 같은 특성이 있다. 
- **글로벌 메모리 영역**
  - 일반적으로 클라이언트 스레드의 수와 무관하게 일반적으로 하나의 메모리 공간만 할당된다.
  - 단, 필요에 따라 2개 이상의 메모리 공간을 할당받을 수도 있지만 클라이언트의 스레드 수와는 무관하며, 생성된 글로벌 영역이 N개라 하더라도 모든 스레드에 의해 공유된다.
- **로컬 메모리 영역**
  - 세션 메모리 영역이라고도 표현하며, MySQL 서버에 존재하는 클라이언트 스레드가 쿼리를 처리하는 데 사용하는 메모리 영역이다.
    - 대표적으로 위의 그림에서의 커넥션 버퍼와 정렬(소트) 버퍼 등이 있다.
    - 클라이언트가 MySQL서버에 접속하면 MySQL 서버에서는 클라이언트 커넥션으로부터의 요청을 처리하기 위해 스레드를 하나씩 할당하게 되는데, 클라이언트 스레드가 사용하는 메모리 공간이라고 해서 클라이언트 메모리 영역이라고도 한다. 
    - 클라이언트와 MySQL 서버와의 커넥션을 세션이라고 하기 떄문에 로컬 메모리 영역을 세션 메모리 영역이라고도 표현한다.
  - 로컬 메모리는 각 클라이언트 스레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않는다는 특징이 있다.
    - 일반적으로 글로벌 메모리 영역의 크기는 주의해서 설정하지만 소트 버퍼와 같은 로컬 메모리 영역은 크게 신경 쓰지 않고 설정하는데, 최악의 경우(가능성이 희박하지만)에는 MySQL 서버가 메모리 부족으로 멈춰 버릴 수도 있으므로 적절한 메모리 공간을 설정하는 것이 중요하다.
  - 로컬 메모리 공간의 또 한 가지 중요한 특징은 각 쿼리의 용도별로 필요할 때만 공간이 할당되고 필요하지 않은 경우에는 MySQL이 메모리 공간을 할당조차도 하지 않을 수도 있다는 점이다.
    - 대표적으로 소트 버퍼나 조인 버퍼와 같은 공간이 그러하다.
  - 그리고 로컬 메모리 공간은 커넥션이 열려 있는 동안 계속 할당된 상태로 남아 있는 공간도 있고(커넥션 버퍼나 결과 버퍼) 그렇지 않고 쿼리를 실행하는 순간에만 할당했다가 다시 해제하는 공간(소트 버퍼나 조인 버퍼)도 있다.

</details>


<details> <summary> 2. InnoDB 스토리지 엔진 아키텍처 </summary>

</details>


<details> <summary> 3. MyISAM 스토리지 엔진 아키텍처 </summary>

</details>


<details> <summary> 4. MEMORY 스토리지 엔진 아키텍처 </summary>

</details>


<details> <summary> 5. NDB 클러스터 스토리지 엔진 </summary>

</details>


<details> <summary> 6. TOKUDB 스토리지 엔진 </summary>

</details>


<details> <summary> 7. 전문 검색 엔진 </summary>

</details>


<details> <summary> 8. MySQL 로그 파일 </summary>

</details>