---
layout: post
title:  "oracle user expired & lock"
date:   2019-12-14 21:03:36 +0530
categories: Oracle Sec
---

### oracle user expired & lock

11g 부터 user가 기본으로 180일 제한이 걸려있다.. 
오라클형들은 자꾸 기본 보안을 높이넹 
일단  dba_profiles 테이블을 보자면 다음과 같이 되어있음을 확인할수 있다.

```
10:14:24 SYS@RAC1> desc dba_profiles;
 Name                                                  Null?    Type
 ----------------------------------------------------- -------- ------------------------------------
 PROFILE                                               NOT NULL VARCHAR2(30)
 RESOURCE_NAME                                         NOT NULL VARCHAR2(32)
 RESOURCE_TYPE                                                  VARCHAR2(8)
 LIMIT                                                          VARCHAR2(40)
```

다음과 같은 명령으로 lock혹은 expired 되어있는 user와 profile을 확인한다

```
15:04:19 SYS@RAC1> select USERNAME, ACCOUNT_STATUS,PROFILE
15:06:44   2  from dba_users;
USERNAME                       ACCOUNT_STATUS                   PROFILE
------------------------------ -------------------------------- ------------------------------
BI                             OPEN                             DEFAULT
SCOTT                          OPEN                             DEFAULT
OGGT                           OPEN                             DEFAULT
```

Lock 되어있는 유저는 다음과 같은 명령으로 해당 사용자를 unlock 할수 있다.

```
10:49:40 SYS@RAC1> alter user bi account unlock;
User altered.
Elapsed: 00:00:00.28
11:17:00 SYS@RAC1> select USERNAME, ACCOUNT_STATUS,PROFILE from dba_users where USERNAME='BI';
USERNAME               ACCOUNT_STATUS                   PROFILE
------------------------- -------------------------------- ------------------------------
BI                             EXPIRED                          DEFAULT
1 row selected.
```

Lock 상태는 사라지고 Expired 상태로 된 것을 확인

```
13:46:48 SYS@RAC1> select name, password from user$ where name='BI';
NAME                           PASSWORD
------------------------------ ------------------------------
BI                             EB32A21961929D0C
```

해당 명령어로 user의 password hash값 확인

```
13:48:17 SYS@RAC1> alter user bi identified by values 'EB32A21961929D0C';
User altered.
13:48:51 SYS@RAC1> select USERNAME, ACCOUNT_STATUS,PROFILE from dba_users where USERNAME='BI';
USERNAME                ACCOUNT_STATUS                   PROFILE
----------------------- -------------------------------- ------------------------------
BI                             OPEN                             DEFAULT
1 row selected.
```


계정이 Open 상태로 된 것을 확인한다. 
이렇게만 하면 일시적으로 expired 되거나 unlock된 계정을 풀수는 있지만... 
장기간 운영해하는 입장에서는 여간 귀찮은게 아니다... 
default profile을 다음과 같이 수정해서 귀차니즘에서 탈피~!

```
10:14:12 SYS@RAC1> col limit for a20
select * from dba_profiles where resource_type='PASSWORD';
PROFILE                        RESOURCE_NAME                    RESOURCE LIMIT
------------------------------ -------------------------------- -------- --------------------
MONITORING_PROFILE             FAILED_LOGIN_ATTEMPTS            PASSWORD UNLIMITED
DEFAULT                        FAILED_LOGIN_ATTEMPTS            PASSWORD 10
MONITORING_PROFILE             PASSWORD_LIFE_TIME               PASSWORD DEFAULT
DEFAULT                        PASSWORD_LIFE_TIME               PASSWORD 180
MONITORING_PROFILE             PASSWORD_REUSE_TIME              PASSWORD DEFAULT
DEFAULT                        PASSWORD_REUSE_TIME              PASSWORD UNLIMITED
MONITORING_PROFILE             PASSWORD_REUSE_MAX               PASSWORD DEFAULT
DEFAULT                        PASSWORD_REUSE_MAX               PASSWORD UNLIMITED
MONITORING_PROFILE             PASSWORD_VERIFY_FUNCTION         PASSWORD DEFAULT
DEFAULT                        PASSWORD_VERIFY_FUNCTION         PASSWORD NULL
MONITORING_PROFILE             PASSWORD_LOCK_TIME               PASSWORD DEFAULT
DEFAULT                        PASSWORD_LOCK_TIME               PASSWORD 1
MONITORING_PROFILE             PASSWORD_GRACE_TIME              PASSWORD DEFAULT
DEFAULT                        PASSWORD_GRACE_TIME              PASSWORD 7
SQL>alter profile default limit FAILED_LOGIN_ATTEMPTS unlimited
PASSWORD_LOCK_TIME unlimited
PASSWORD_GRACE_TIME unlimited
PASSWORD_LIFE_TIME unlimited;
```

- 패스워드 제한설정 
   - password_history    : 이전 암호를 기억해 놓았다가 다음에 변경시 동일한 암호사용을 금지함
   - password_reuse_time : 동일한 password를 적용한 기간동안 사용금지 
   - password_reuse_max : 입력된 value값만큼만 사용가능한 횟수를 제한 
     (ex: 3이라고 입력하면 3번만 사용가능)
   - password_life_time     : password 생명주기 ex : 30 -> 30일마다 변경해야함 
   - password_grace_time : password 변경 만료 알림을 value일 전부터 알림 
   - failed_login_attempts   :password 입력실패시 재시도 가능횟수 최종 실패시 계정 lock걸림 
   - password_lock_time   : lock걸렸을때 value값만큼 잠겨있음 1일단위임 1/24는 1시간 1/1440은 1분 
   - password complexity verification(패스워드 복잡성) password설정시 제약조건
