
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

### 3.1.4 플러그인 스토리지 엔진 모델

![image](https://user-images.githubusercontent.com/28394879/138054290-5636540f-f6d0-485f-88a7-ad994183e624.png)
- MySQL의 독특한 구조 중 대표적인 것이 바로 플러그인 모델이다.
  - 플러그인해서 사용할 수 있는 것이 스토리지 엔진만 가능한 것은 아니다.
  - MySQL 5.1부터는 전문 검색 엔진을 위한 검색어 파서(인덱싱할 키워드를 분리해내는 작업)도 플러그인 형태로 개발해서 사용할 수 있다.
  - MySQL은 이미 기본적으로 많은 스토리지 엔진을 가지고 있다.
  - 하지만 이 세상의 수 많은 사용자의 요구조건을 만족시키기 위해 기본적으로 제공되는 스토리지 엔진 이외에 부가적인 기능을 더 제공하는 스토리지 엔진이 필요할 수 있으며, 이러한 요건을 기초로 다른 전문 개발 회사 또는 우리가 직접 스토리지 엔진을 제작하는 것도 가능하다.
- MySQL에서 쿼리가 실행되는 과정을 크게 아래 그림과 같이 나눈다면 거의 대부분의 작업이 MySQL엔진에서 처리되고, 마지막 "데이터 읽기/쓰기" 작업만 스토리지 엔진에 의해 처리된다(만약 우리가 아주 새로운 용도의 스토리지 엔진을 만든다 하더라도 DBMS의 전체 기능이 아닌 일부분의 기능만 수행하는 엔진을 작성하게 된다는 의미다)
  - ![image](https://user-images.githubusercontent.com/28394879/138054999-4ecbbfa7-6460-4fb7-bfbb-a5548c1bd666.png)
- 위의 그림의 각 처리 영역에서 "데이터 읽기/쓰기" 작업은 거의 대부분 1건의 레코드 단위로 처리된다
  - 예) 특정 인덱스의 레코드 1건 읽기 또는 마지막 읽었던 레코드의 다음 또는 이전 레코드 읽기
  - 그리고 MySQL을 사용하다 보면 "핸들러(Handler)"라는 단어를 자주 접하게 될 것이다.
  - 핸들러라는 단어는 MySQL 서버의 소스코드로부터 넘어온 표현인데, 이는 우리가 매일 타고 다니는 자동차로 비유해 보면 쉽게 이해할 수 있따.
  - 사람이 핸들(운전대)을 이용해 자동차를 운전하듯이, 프로그래밍 언어에서는 어떤 기능을 호출하기 위해 사용하는 운전대와 같은 역할을 하는 객체를 핸들러(또는 핸들러 객체)라고 표현한다.
  - MySQL 서버에서는 MySQL 엔진은 사람 역할을 하고, 각 스토리지 엔진은 자동차 역할을 하게 되는데, MySQL 엔진이 스토리지 엔진을 조정하기 위해 핸들러라는 것을 사용하게 된다.
- MySQL에서 핸들러라는 것은 개념적인 내용이라서 완전히 이해하지 못하더라도 크게 문제되진 않지만 최소한 MySQL 엔진이 각 스토리지 엔진에게 데이터를 읽어오거나 저장하도록 명령하려면 핸들러를 꼭 통해야 한다는 점만 기억하자.
  - 나중에 MySQL 서버의 상태 변수라는 것을 배우게 될 텐데, 이러한 상태 변수 가운데 "Handler_"로 시작하는 것(대표적으로 Handler_read_md_next 같은)이 많다는 사실을 알게 될 것이다.
  - 그러면 "Handler_"로 시작하는 상태 변수는 "MySQL 엔진이 각 스토리지 엔진에게 보낸 명령의 횟수를 의미하는 변수"라고 이해하면 된다.
  - MySQL에서 MyISAM이나 InnoDB와 같이 다른 스토리지 엔진을 사용하는 테이블에 대해 쿼리를 실행하더라도 MySQL의 처리 내용은 대부분 동일하며, 단순히 위의 그림의 마지막 단계인 "데이터 읽기/쓰기" 영역의 처리만 차이가 있을 뿐이다.
  - 실질적인 GROUP BY나 ORDER BY 등 많은 복잡한 처리는 스토리지 엔진 영역이 아니라 MySQL 엔진의 처리 영역인 "쿼리 실행기"에서 처리된다.
- 그렇다면 MyISAM이나 InnoDB 스토리지 엔진 가운데 뭘 사용하든 별 차이가 없는 것 아닌가, 라고 생각할 수 있지만 "그렇진 않다."
  - 여기서 설명한 내용은 아주 간략하게 언급한 것일 뿐이고, 단순히 보이는 "데이터 읽기/쓰기" 작업 처리 방식이 얼마나 달라질 수 있는가를  뒷 챕터들을 통해 느끼게 될 것이다.
  - 여기서 중요한 내용은 '하나의 쿼리 작업은 여러 하위 작업으로 나뉘는데, 각 하위 작업이 MySQL 엔진 영역에서 처리되는지 아니면 스토리지 엔진 영역에서 처리되는지 구분할 줄 알아야 한다.'는 점이다.
  - 사실 여기서는 스토리지 엔진의 개념을 설명하기 위한 것도 있지만 각 단위 작업을 누가 처리하고 "MySQL 엔진 영역"과 "스토리지 엔진 영역"의 차이를 설명하는데 목적이 있다.
- 이제 설치돼 있는 MySQL 서버(mysqld)에서 지원되는 스토리지 엔진이 어떤 것이 있는지 확인해보자.
  ```
  mysql> show ENGINES;
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    | Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    | InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
    | MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
    | MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
    | BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
    | MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
    | CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
    | ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
    | PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
    | FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
    +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
    9 rows in set (0.00 sec)
  ``` 
- Support 칼럼에 표시될 수 있는 값은 아래 4가지다.
  - YES: MySQL 서버(mysqld)에 해당 스토리지 엔진이 포함돼 있고, 사용 가능으로 활성화된 상태임
  - DEFAULT: "YES"와 동일한 상태이지만 필수 스토리지 엔진임을 의미함(즉 이 스토리지 엔진이 없으면 MySQL이 시작되지 않을수도 있음을 의미한다)
  - NO: 현재 MySQL 서버(mysqld)에 포함되지 않았음을 의미함
  - DISABLED: 현재 MySQL 서버(mysqld)에는 포함됐지만 파라미터에 의해 비활성화된 상태임
- MySQL 서버(mysqld)에 포함되지 않은 스토리지 엔진(Support 칼럼이 NO로 표시되는)을 사용하려면 MySQL 서버를 다시 빌드(컴파일)해야 한다.
  - 하지만 우리의 MySQL 서버가 적절히 준비만 돼 있다면 플러그인 형태로 빌드된 스토리지 엔진 라이브러리를 내려받아 끼워 넣기만 하면 사용할 수 있다.
  - 또한 플러그인 형태의 스토리지 엔진은 손쉽게 업그레이드할 수 있다.
  - 스토리지 엔진뿐 아니라 모든 플러그인의 내용은 다음과 같이 확인할 수 있다.
  - 이 명령으로 스토리지 엔진뿐 아니라 전문 검색용 파서와 같은 플러그인도 (만약 설치돼 있다면) 확인할 수 있다.
  ```
  mysql> show plugins;
    +----------------------------+----------+--------------------+---------+---------+
    | Name                       | Status   | Type               | Library | License |
    +----------------------------+----------+--------------------+---------+---------+
    | binlog                     | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
    | sha256_password            | ACTIVE   | AUTHENTICATION     | NULL    | GPL     |
    | CSV                        | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | MEMORY                     | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | InnoDB                     | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | INNODB_TRX                 | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_LOCKS               | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_LOCK_WAITS          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP                 | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_RESET           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMPMEM              | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMPMEM_RESET        | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_PER_INDEX       | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_CMP_PER_INDEX_RESET | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_PAGE         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_PAGE_LRU     | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_BUFFER_POOL_STATS   | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_TEMP_TABLE_INFO     | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_METRICS             | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_DEFAULT_STOPWORD | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_DELETED          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_BEING_DELETED    | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_CONFIG           | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_INDEX_CACHE      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_FT_INDEX_TABLE      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_TABLES          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_TABLESTATS      | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_INDEXES         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_COLUMNS         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_FIELDS          | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_FOREIGN         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_FOREIGN_COLS    | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_TABLESPACES     | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_DATAFILES       | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | INNODB_SYS_VIRTUAL         | ACTIVE   | INFORMATION SCHEMA | NULL    | GPL     |
    | MyISAM                     | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | MRG_MYISAM                 | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | PERFORMANCE_SCHEMA         | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | ARCHIVE                    | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | BLACKHOLE                  | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | FEDERATED                  | DISABLED | STORAGE ENGINE     | NULL    | GPL     |
    | partition                  | ACTIVE   | STORAGE ENGINE     | NULL    | GPL     |
    | ngram                      | ACTIVE   | FTPARSER           | NULL    | GPL     |
    +----------------------------+----------+--------------------+---------+---------+
    44 rows in set (0.00 sec)
  ``` 

### 3.1.5 쿼리 실행 구조 
![image](https://user-images.githubusercontent.com/28394879/138060457-b34a9a39-ea7a-40f1-ba10-4c0715edd43b.png)

- 위의 그림은 쿼리를 실행하는 관점에서 MySQL의 구조를 간략하게 그림으로 표현한 것이며, 다음과 같이 기능별로 나눠서 볼 수 있다.
- 파서
  - 파서는 사용자 요청으로 들어온 쿼리 문장을 토큰(MySQL이 인식할 수 있는 최소 단위의 어휘나 기호)으로 분리해 트리 형태의 구조로 만들어 내는 작업을 의미한다.
  - 쿼리 문장의 기본 문법 오류는 이 과정에서 발견되며 사용자에게 오류 메시지를 전달하게 된다.
- 전처리기
  - 파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인한다.
  - 각 토큰을 테이블 이름이나 칼럼 이름 또는 내장 함수와 같은 개체를 매핑해 해당 객체의 존재 여부와 객체의 접근권한 등을 확인하는 과정을 이 단계에서 수행한다.
  - 실제 존재하지 않거나 권한상 사용할 수 없는 개체의 토큰은 이 단계에서 걸러진다.
- 옵티마이저
  - 옵티마이저란 사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할지 결정하는 역할을 담당하는데, DBMS의 두뇌에 해당한다고 볼 수 있다.
  - 여기에서 이야기하고자 하는 내용은 대부분 옵티마이저가 선택하는 내용을 설명하는 것이며, 어떻게 하면 옵티마이저가 더 나은 선택을 할 수 있게 유도하는가를 알려주는 것이라고 생각해도 될 정도로 옵티마이저의 역할은 중요하고 영향 범위 또한 아주 넓다.
- 실행 엔진
  - 옵티마이저가 두뇌라면 실행 엔진과 핸들러는 손과 발에 비유할 수 있다(좀더 재미있게 회사로 비유한다면 옵티마이저는 회사의 경영진, 실행 엔진은 중간 관리자, 핸들러는 각 업무의 실무자로 비유해 볼 수 있다).
  - 실행 엔진이 하는 일을 더 쉽게 이해할 수 있게 간단하게 예를 들어서 살펴보자.
  - 옵티마이저가 GROUP BY를 처리하기 위해 임시 테이블을 사용하기로 결정했다고 해보자.
    1. 실행 엔진은 핸들러에게 임시 테이블을 만들라고 요청
    2. 다시 실행 엔진은 WHERE 절에 일치하는 레코드를 읽어오라고 핸들러에게 요청
    3. 읽어온 레코드들을 1번에서 준비한 임시 테이블로 저장하라고 다시 핸들러에게 요청
    4. 데이터가 준비된 임시 테이블에서 필요한 방식으로 데이터를 읽어 오라고 핸들러에게 다시 요청
    5. 최종적으로 실행 엔진은 결과를 사용자나 다른 모듈로 넘김
  - 즉, 실행 엔진은 만들어진 계획대로 각 핸들러에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행한다.
- 핸들러(스토리지 엔진)
  - 위에서 잠깐 언급한 것처럼 핸들러는 MySQL 서버의 가장 밑단에서 MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 디스크로부터 읽어 오는 역할을 담당한다.
  - 핸들러는 결국 스토리지 엔진을 의미하며 MyISAM 테이블을 조작하는 경우에는 핸들러가 MyISAM 스토리지 엔진이 되고, InnoDB 테이블을 조작하는 경우에는 핸들러가 InnoDB 스토리지 엔진이 된다.     

### 3.1.6 복제(Replication)

![image](https://user-images.githubusercontent.com/28394879/138198111-c7bf2fa7-de87-4c36-9fe0-0118bdd4f726.png)

- 데이터베이스의 데이터가 갈수록 대용량화돼 가는 것을 생각하면 확장성(Scalability)은 DBMS에서 아주 중요한 요소다.
  - MySQL은 확장성을 위한 다양한 기술을 제공하는데 그중에서 가장 일반적인 방법이 복제(Replication)다.
  - MySQL의 복제는 거의 2000년도부터 제공됐기 때문에 타 DBMS의 복제보다 훨씬 이전부터 제공된 기능이며 또한 그만큼 안정적이다.
- MySQL의 복제는 레플리케이션(Replication)이라고도 하는데, 복제는 2대 이상의 MySQL 서버가 동일한 데이터를 담도록 실시간으로 동기화하는 기술이다.
  - 일반적으로 MySQL의 복제에는 INSERT나 UPDATE와 같은 쿼리를 이용해 데이터를 변경할 수 있는 MySQL 서버와 SELECT 쿼리로 데이터를 읽기만 할 수 있는 MySQL 서버로 나뉜다.
  - MySQL에서는 쓰기와 읽기의 역활로 구분해, 전자를 마스터(Master)라고 하고 후자를 슬레이브(Slave)라고 하는데(이러한 역할별 명칭은 DBMS 종류별로 조금씩 차이가 있다)
  - 일반적으로 MySQL 서버의 복제에서는 마스터는 반드시 1개이며 슬레이브는 1개이상으로 구성될 수 있다.
- 하나의 MySQL이 일반적으로는 마스터 또는 슬레이브 가운데 하나의 역할만을 수행하지만 때로는 MySQL 서버 하나가 마스터이면서 슬레이브 역할까지 수행하도록 설정하는 것도 가능하다.
  - 또한 마스터용 MySQL 프로그램과 슬레이브용 MySQL 프로그램이 정해져 있는 것은 더더욱 아니다.
  - 마스터나 슬레이브라는 것은 단지 그 서버의 역할 모델을 지칭하는 것일 뿐이다.

- **마스터(Master)**
  - 기술적으로는 MySQL 의 바이러니 로그가 활성화되면 어떤 MySQL 서버든 마스터가 될 수 있다.
  - 애플리케이션의 입장에서 본다면 마스터 장비는 주로 데이터가 생성 및 변경,삭제되는 주체(시작점)라고 볼 수 있다.
  - 일반적으로 MySQL 복제를 구성하는 경우 복제에 참여하는 여러 서버 가운데 변경이 허용되는 서버는 마스터로 한정될 때가 많다.
  - 그렇지 않은 경우 복제되는 데이터의 일관성을 보장하기 어렵기 때문이다.
  - 마스터 서버에서 실행되는 DML(데이터를 조작하는 문장)과 DDL(스키마를 변경하는 문장) 가운데 데이터의 구조나 내용을 변경하는 모든 쿼리 문장은 바니어리 로그에 기록한다.
  - 슬레이브 서버에서 변경 내역을 요청하면 마스터 장비는 그 바이너리 로그를 읽어 슬레이브로 넘긴다.
  - 마스터 장비의 프로세스 가운데 "Binlog dump"라는 스레드가 이 일을 전담하는 스레드다.
  - 만약 하나의 마스터 서버에 10개의 슬레이브가 연결돼 있다면 "Binlog dump" 스레드는 10개가 표시될 것이다.
- **슬레이브(Slave)**
  - 데이터(바이너리 로그)를 받아 올 마스터 장비의 정보(IP주소와 포트 정보 및 접속 계정)를 가지고 있는 경우 슬레이브가 된다(마스터나 슬레이브라고 해서 별도의 빌드 옵션이 필요하거나 프로그램을 별도로 설치해야 하는것은 아니다)
  - 마스터 서버가 바이너리 로그를 가지고 있다면 슬레이브 서버는 릴레이 로그를 가지고 있다
  - 일반적으로 마스터와 슬레이브의 데이터를 동일한 상태로 유지하기 위해 슬레이브 서버는 읽기 전용으로 설정할 때가 많다.
  - 슬레이브 서버의 I/O 스레드는 마스터 서버에 접속해 변경 내역을 요청하고, 받아 온 변경 내역을 릴레이 로그에 기록한다.
  - 그리고 슬레이브 서버의 SQL 스레드가 릴레이 로그에 기록된 변경 내역을 재실행(Replay)함으로써 슬레이브의 데이터를 마스터와 동일한 상태로 유지한다.
  - I/O 스레드(Slave_IO_Thread)와 SQL 스레드(Slave_SQL_Thread)는 마스터 MySQL에서는 기동되지 않으며, 복제가 설정된 슬레이브 MySQL 서버에서 자동으로 기동되는 스레드다.


복제를 사용할 경우 주로 잘못 생각하거나 간과하는 부분이 있는데 최소한 다음 사항에 대해서는 주의 해야 한다.

- **슬레이브는 하나의 마스터만 설정 가능**
  - MySQL의 복제에서 하나의 슬레이브는 하나의 마스터만 가질 수 있다.
  - 이 제약사항만 피할 수 있다면 상당히 다양한 형태의 구성으로 데이터를 복제할 수 있다.
  - 하나의 마스터에 N개의 슬레이브가 일반적인 형태이며 그 밖에 링(Ring)형태나 트리(Tree) 형태의 구성도 가능하다.
  - 그리고 많이 사용하진 않지만 마스터-마스터 형태의 복제도 사용된다.
  - 마스터-마스터 형태에는 사실 2개의 MySQL 서버 모두 마스터이면서 슬레이브가 되는 형태로 구성되는 것이다.
  - 더 자세한 복제의 구성 형태에 대해서는 16.9.1 "MySQl 복제의 형태"를 참조하자.
- **마스터와 슬레이브의 데이터 동기화를 위해 슬레이브는 읽기 전용으로 설정**
  - 마스터와 슬레이브로 복제가 구성된 상태에서 데이터는 마스터로 접속해서 변경해야 하는데, 사용자 실수나 애플리케이션 오류로 인해 슬레이브로 접속해서 실행하는 경우가 가끔 발생한다.
  - 만약 운 나쁘게도 일부 변경 작업은 마스터에서 실행되고 일부는 슬레이브에서 실행되고 있엇다면 데이터 동기화에 상당항 노력이 필요할지도 모른다.
  - 이러한 사용자 실수를 막기 위해 슬레이브는 읽기 전용(read_only 설정 파라미터)으로 설정하는 것이 일반적이다.
- **슬레이브 서버용 장비는 마스터와 동일한 사양이 적합**
  - 많은 사용자가 착각하는 부분이기도 한데, 슬레이브 서버용 장비는 마스터 서버용 장비보다 한 단게 낮은 장비로 선택하려고 할 때가 있다.
  - 하지만 마스터 서버에서 수많은 동시 사용자가 실행한 데이터 변경 쿼리 문장이 슬레이브 서버에서는 하나의 스레드로 모두 처리돼야 한다(이 부분은 지금의 구조상 피해 갈 방법이 없다) 
  - 그래서 변경이 매우 잦은 MySQL 서버일수록 마스터 서버의 사양보다 슬레이브 서버의 사양이 더 좋아야 마스터에서 동시에 여러 개의 스레드로 실행된 쿼리가 슬레이브에서 지연되지 않고 하나의 스레드로 처리될 수 있다.
  - 하지만 데이터 변경은 데이터 조회보다는 1/10 이하 수준으로 유지되는 것이 일반적으로 마스터 서버와 슬레이브 서버를 같은 사양으로 유지할 때가 많다.
  - 또한, 슬레이브 서버는 마스터 서버가 다운된 경우 그에 대한 복구 대안으로 사용될 때도 많기 때문에 사양을 동일하게 맞추는 경우가 대부분이다.
- **복제가 불필요한 경우에는 바이너리 로그 중지**
  - 바이너리 로그를 작성하기 위해 MySQL이 얼마나 많은 자원을 소모하고 성능이 저하되는지 잘 모르는 사용자가 많다.
  - 바이너리 로그를 안정적으로 기록하기 위해 갭 락(Gap lock)을 유지하고, 매번 트랜잭션이 커밋될 때마다 데이터를 변경시킨 쿼리 문장을 바이너리 로그에 기록해야 한다(어떤 경우에는 바이너리 로그에 정확히 기록되고 나서야 사용자가 요청한 쿼리 문장이 완료될 때도 있다)
  - 바이너리 로그를 기록하는 작업을 AutoCommit이 활성화된 MySQL 서버에서 더 심각한 부하로 나타날 때가 많다.
  - 특히 트랜잭션을 지원하지 않는 MyISAM 테이블은 항상 AutoCommit 모드로 작동하기 때문에 InnoDB 테이블보다 바이너리 로그를 기록하는 데 더 많은 자원을 사용하게 된다.
  - 바이너리 로그가 성능에 끼치는 영향은 16.11.2절 "운영체제의 파일 시스템 선정"을 참고하자.
- **바이너리 로그와 트랜잭션 격리 수준(Isolation level)**
  - 바이너리 로그 파일은 어떤 내용이 기록되느냐에 따라 STATEMENT 포맷 방식과 ROW 포맷 방식이 있다.
  - STATEMENT 방식은 바이너리 로그 파일에 마스터에서 실행되는 쿼리 문장을 기록하는 방식이며, ROW 포맷은 마스터에서 실행된 쿼리에 의해 변경된 레코드 값을 기록하는 방식이다.
  - MySQL 5.0 이하 버전까지는 STATEMENT 방식만 제공됐었는데, 이 방식에서는 마스터와 슬레이브의 데이터 일치를 위해 REPEATABLE READ 격리 수준만 사용 가능하다.
  - "MySQL 5.0 + STATEMENT 포맷의 바이너리 로그 + REPEATABLE READ 격리 수준" 환경에서는 "INSERT INTO ... SELECT ... FROM *" 형태의 쿼리 문장을 사용할 때 주의해야 한다. 자세한 내용은 12.2.2 절 "INSERT 쿼리의 잠금"을 참고하자.

> MySQL의 복제는 마스터에서 처리된 내용이 바이너리 로그로 기록되고, 그 내용이 슬레이브 MySQL 서버로 전달되어 재실행되는 방식으로 처리된다.
> MySQL 5.0까지는 바이너리 로그에는 마스터 MySQL 서버에서 실행된 SQL 문장이 그대로 기록되고, 슬레이브 MySQL 서버에서 똑같은 SQL 문장이 재실행되는 방식만 지원됐다.
> MySQL 5.1부터는 MySQL 5.0과 같이 SQL 문장을 기록하는 방법과 마스터에서 변경된 데이터 레코드를 기록하는 두 가지 방법을 제공한다.
> 바이너리 로그 파일에 SQL 문장을 기록하는 방식을 문장 기반 복제(Statement based replication)라고 하며, 변경된 레코드를 바이너리 로그에 기록하는 방식을 레코드 기반의 복제(Row based replication)라고 한다.
> SQL 기반의 복제는 아무리 데이터의 변경을 많이 유발하는 쿼리라 하더라도 SQL 문장 하나만 슬레이브로 전달되므로 네트워크 트래픽을 많이 유발하지는 않는다.
> 하지만 SQL 기반의 복제가 정상적으로 작동하려면 REPEATABLEREAD 이상의 트랜잭션 격리 수준을 사용해야 하며, 그로 인해 InnoDB 테이블에서는 레코드 간의 간격을 잠그는 갭락이나 넥스트 키 락이 필요해진다.
> 반면 레코드 기반의 복제는 마스터와 슬레이브 MySQL 서버 간의 네트워크 트래픽을 많이 발생시킬 수 있지만 READ-COMMITTED 트랜잭션 격리 수준에서도 작동할 수 있으며 InnoDB 테이블에서 잠금의 경합은 줄어든다.

### 3.1.7 쿼리 캐시

![image](https://user-images.githubusercontent.com/28394879/138218201-8ff79c93-de86-4b50-abb1-df587bd0d0d5.png)

- 쿼리 캐시(Query Cache)는 타 DBMS에는 없는 MySQL의 독특한 기능 중 하나로서 적절히 설정만 해두면 상당한 성능 향상 효과를 얻을 수 있다.
  - 여러 가지 복잡한 처리 절차와 꽤 큰 비용을 들여 실행된 결과를 쿼리 캐시에 담아 두고, 동일한 쿼리 요청이 왔을 때 간단하게 쿼리 캐시에서 찾아서 바로 결과를 내려 줄 수 있기 때문에 기대 이상의 효과를 거둘 수 있다.
  - 하지만 항상 그렇듯이 이 쿼리 캐시에 장단점이 있으므로 적절한 조율이 중요하다.
  - 쿼리 캐시는 단어의 의미와는 달리 SQL 문장을 캐시하는 것이 아니라 쿼리의 결과를 메모리에 캐시해 두는 기능이다.
  - 쿼리 캐시의 구조는 간단한 키와 값의 쌍으로 관리되는 맵(Map)과 같은 데이터 구조로 구현돼 있다.
  - 여기서 키를 구성하는 요소 가운데 가장 중요한 것은 쿼리 문장 자체일 것이며, 값은 해당 쿼리의 실행 결과가 될 것이다.
  - 위의 그림에서 쿼리 캐시 안의 작은 표와 같이 생각해 볼 수 있다.
- 하지만 데이터베이스에서 쿼리를 처리 할 떄는 상당히 많은 부분의 처리 절차가 있다.
  - 이를 전부 무시하고 동일한 쿼리 문장이 요청됐다고 그냥 캐시된 결과를 보내서는 안 된다.
  - 쿼리 캐시의 결과를 내려 보내주기 전에 반드시 다음과 같은 확인 절차를 거쳐야 한다.
    1. 요청된 쿼리 문장이 쿼리 캐시에 존재하는가?
    2. 해당 사용자가 그 결과를 볼 수 있는 권한을 가지고 있는가?
    3. 트랜잭션 내에서 실행된 쿼리인 경우, 그 결과가 가시 범위 내의 트랜잭션에서 만들어진 결과인가? (InnoDB의 경우)
    4. 쿼리에 사용된 기능(내장 함수나 저장 함수 등)이 캐시돼도 동일한 결과를 보장할 수 있는가?
       1. CURRENT_DATE, SYSDATE, RAND 등과 같이 호출 시점에 따라 결과가 달라지는 요소가 있는가?
       2. 프리페어 스테이트먼트의 경우 변수가 결과에 영향을 미치지 않는가?
    5. 캐시가 만들어지고 난 이후 해당 데이터가 다른 사용자에 의해 변경되지 않았는가?
    6. 쿼리에 의해 만들어진 결과가 캐시하기에 너무 크지 않은가?
    7. 그 밖에 쿼리 캐시를 사용하지 못하게 만드는 요소가 사용됐는가?

물론 이 밖에도 더 세세한 쿼리 캐시 비교 검색 과정이 있지만, 생략하고 우선 이 내용을 조금 더 자세히 살펴보자.

- **요청된 쿼리 문장이 쿼리 캐시에 존재하는가?**
  - 위의 그림에서도 알 수 있듯이, 쿼리 캐시는 MySQL의 어떠한 처리보다 앞 단에 위치하며, 캐시된 결과를 찾기 위해 쿼리 문장을 분석해서 복잡한 비교 과정을 거치는 것이 아니기 때문에 아주 간단하고 빠르게 진행 된다.
  - 비교 방식은 그냥 요청된 쿼리 문장 자체가 동일한지 여부를 비교하는 것이다.
  - 여기서 비교하는 대상으로는 공백이나 탭과 같은 문자까지 모두 포함되며, 대소문자까지 완전히 동일해야 같은 쿼리로 인식한다.
  - 결론적으로 애플리케이션의 전체 쿼리 가운데 동일하거나 비슷한 작업을 하는 쿼리는 하나의 쿼리로 통일해 문자열로 관리하는 것이 좋다.
  - 그렇다고 무리하게 쿼리를 통합하라는 것은 아니며, 적절히 추가 비용이 없이 변경이 가능한 것들은 통합하라는 것이다.
  - 동일한 쿼리가 여러곳에서 정의되어 사용되면 어느 순간에 각 쿼리가 달라지고(공백이나 개행 문자 하나라도) 그렇게 되면 쿼리 캐시를 공유하지 못하게 된다.
- **해당 사용자가 그 결과를 볼 수 있는 권한을 가지고 있는가?**
  - 어떤 사용자가 요청한 쿼리에 대해 동일한 쿼리 결과가 쿼리 캐시에 저장돼 있더라도 이 사용자가 해당 테이블의 읽기 권한이 없다면 쿼리 캐시의 결과를 보여줘서는 안 되기 때문에 이런 확인 작업은 당연한 것이다.
- **트랜잭션 내에서 실행된 쿼리의 경우 가시 범위 내에 있는 결과인가?**
  - InnoDB의 모든 트랜잭션은 각 트랜잭션 ID를 갖게 된다.
  - 트랜잭션 ID는 트랜잭션이 시작된 시점을 기준으로 순차적으로 증가하는 6바이트 숫자 값이어서 트랜잭션 ID 값을 비교해 보면 어느 쪽이 먼저 시작된 트낼잭션인지 구분할 수 있다.
  - InnoDB에서는 트랜잭션 격리 수준을 준수하기 위해 각 트랜잭션은 자신의 ID보다 ID 값이 큰 트랜잭션에서 변경한 작업 내역이나 쿼리 결과는 참조할 수 없다.
  - 이를 트랜잭션의 가시 범위라고 한다. 
  - 쿼리 캐시도 그 결과를 만들어낸 트랜잭션의 ID가 가시 범위 내에 있을 때만 사용할 수 있는 것이다.
- **CURRENT_DATE(), SYSDATE(), RAND() 등과 같이 호출 시점에 따라 결과가 달라지는 요소가 있는가?**
  - SYSDATE()나 RAND() 같은 함수는 동일 사용자가 동일 쿼리를 실행하더라도 호출하는 시간에 따라 결과가 달라진다.
  - 또한 이런 내장 함수뿐 아니라 NOT DETERMINISTIC 옵션으로 정의된 스토어드 루틴이 사용된 쿼리도 시점에 따라 결과가 달라질 가능성이 있다.
  - 스토어드 루틴의 NOT DETERMINITIC 옵션과 관련해서는 11.3.3절 "DETERMINISTIC과 NOT DETERMINISTIC 옵션"을 참고하자.
  - 그래서 호출될 때마다 결과 값이 달라지는 CURRENT_DATE()나 SYSDATE(), 그리고 RAND()와 같은 내장 함수뿐 아니라 NOT DETERMINISTIC으로 정의된 스토어드 함수 등은 사용하지 않는 편이 쿼리 캐시의 효율을 높이는 데 도움된다.
- **프리페어 스테이트먼트의 경우 변수가 결과에 영향을 미치지 않는가?**
  - 프리페어 스테이트먼트(바인드 변수가 사용된 쿼리)의 경우에는 쿼리 문장 자체에 변수("?")가 사용되기 때문에 쿼리 문장 자체로 쿼리 캐시를 찾을 수가 없다.
  - 여기서 한 가지 주의해야 할 사항은 프로그램 코드에서는 프리페어 스테이트먼트를 사용했다 하더라도 실제 MySQL 서버에서는 프리페어 스테이트먼트 형태로 실행되지 않는다는 점이다. 
  - 진정한 프리페어 스테이트먼트를 사용하려면 프로그램의 소스코드에서 데이터베이스 커넥션을 생성할 때 특별한 옵션을 사용해야만 한다.
  - 이를 서버 사이드(Server side) 프리페어 스테이트먼트라고 하는데, MySQL 5.0까지는 프리페어 스테이트먼트로 실행된 쿼리는 쿼리 캐시를 사용할 수 없었지만 MySQL 5.1부터는 이러한 제약이 없어졌다.
  - 서버 사이드 프리페어 스테이트먼트에 대해서는 13.1.2절 "MySQL Connector/J를 이용한 개발"의 "프리페어 스테이트먼트 의종류"를 참고하자.
- **캐시가 만들어지고 난 이후 해당 데이터가 다른 사용자에 의해 변경되지 않았는가?**



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