
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