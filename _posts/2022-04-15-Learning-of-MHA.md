---
layout: post
title: 우당탕쿵탕 MHA 학습기
comments: true
categories: [MySQL]
tag: [MySQL]
---

## Purpose of Study
MHA에 대해 학습해보고자 하였습니다. 

## 우당탕쿵탕 MHA 학습기
 - [Purpose of Study](#purpose-of-study)
 - [What is this?](#what-is-this)
 - [Why Choose this?](#why-choose-this)
 - [Let's Get Started](#lets-get-started)
 - [Completion](#전부-적용이-된-모습)

## What is this?
MHA는 Master High Availability의 약자로 Master DB의 고가용성을 위해 Yoshinori Matsunobu에 의해 개발되었으며 GPL v2 라이센스 기반의 오픈소스입니다.   
고가용성이란, 장애 발생 시, 최대한 단축하는 것을 의미하는데 Master/Slave 구조의 Replication 환경에서 Master DB의 장애 발생 시 수동으로 Master 변경할 경우, 서비스 중단 시간이 길어지므로 자동화 할 수 있는 기능을 만든 것입니다. 
  

장애 발생 시, MHA에 의해 자동으로 Fail-Over를 지원하며 Slave 중 가장 최신의 SlaveDB를 MasterDB로 승격, 고가용성을 유지시키는 역활을 하게 됩니다. 
또한 SlaveDB 간의 binlog 적용의 차이를 해결 하는 기능을 제공합니다. 
  

MHA는 아래와 같이 여러 기능에 대해 지원하고 있습니다.  
 1. MasterDB의 네트워크 단절 시, Fail-Over
 2. MasterDB의 mysqld 프로세스의 비정상 종료로 인하여 DB 접속이 불가할 경우, Fail-Over
 3. Fail-Over 시 기존 MasterDB와 새로운 MasterDB 간의 최대한의 데이터 정합성 보장
  -> show slave status: Second behind master 값을 확인, Slave에서 동기화가 아직 완료되지 않았다면 MasterDB의 Binlog를 읽어, 새로운 MasterDB에 반영함
  단, 네트워크의 단절 및 서버의 전원 장애든, 기존 Master에서 데이터를 읽을 수 없는 예외 상황을 제외. 최근까지 작성된 binlog 중 읽을 수 있다면 읽어서 새롭게 MasterDB가 될 Slave에 적용하여 정합성을 최대한 유지하기 위한 기능 제공 
 4. SlaveDB 간의 적용된 Binlog 차이를 확인하여 복제 정합성 기능 제공
 5. MasterDB와 SlaveDB 간의 수동 Online Switch Over 및 데이터 정합성 유지
 6. 복제에 대한 모니터링
 7. MySQL 5.6 GTID 및 Multi-Threaded Slave 기능 지원(MHA 0.56부터)
  
  
  
![](../asset/MHA/images/MHA%EC%9D%98%EC%9E%A5%EC%95%A0%EB%B0%9C%EC%83%9D%EC%8B%9CFailOver.png)  
  
1대의 Master와 N대의 Slave 구조로 구성되어있기 때문에 Fail-Over 시에는 어느 Slave가 Master로 승격될 지 구분해야하며, 최신의 Slave가 Master로 승격된 이후에 다른 Slave에서 아직 모든 바이너리 로그 이벤트를 수신하지 않았을 가능성이 있습니다.  
이러한 일관성 문제를 방지하려면 새 Master 에서 복제를 시작하기 전에 손실된 바이너리 로그 이벤트를 식별 후, 각 Slave에 차례로 적용해야 합니다.  

  
![](../asset/MHA/images/%EC%9E%A5%EC%95%A0%EC%8B%9C%EC%88%98%ED%96%89%EB%90%98%EB%8A%94FailOver%EC%A0%88%EC%B0%A8.png)  



## Let's Get Started
1. 환경 구축하기<br/>
 i. AWS 인스턴스 생성<br/>

![](../asset/MHA/images/AWS%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%83%9D%EC%84%B1.png)  


![](../asset/MHA/images/AWS%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4%EC%83%9D%EC%84%B12.png)  


먼저 사용할 EC2 인스턴스를 생성합니다. <br/>
OS: **Ubuntu 18.04**<br/>
<br/>
보안 그룹 허용 아이피는 일단 제 IP로 설정합시다. <br/>
인바운드 규칙은 아래와 같이 설정하였습니다. <br/>

<br/>

#### 인바운드 규칙
 - SSH 접속을 위해 특정 아이피 22 포트 허용
 - 내부망 끼리의 3306 포트 허용(리플리케이션을 위해서)
 - Workbench를 사용하여 작업할 수 있게 특정 아이피 3306 포트 허용 
 - 테스트 용으로 특정 아이피  80 포트 허용
<br/>
<br/>

 ii. EC2 인스턴스 접속 설정 <br/>

SSH 접속 툴: mRemoteNG<br/>

현재 업무용으로 사용하고 있어, 익숙하고 Putty보다 사용하기 편하여 선택함. <br/>
다른 걸 사용해도 문제없을 것으로 생각됨. <br/>

.pem 파일을 ppk 파일로 변환 해당 키값은 향후 집에서 작업할 일을 위하여 클라우드에 저장<br/>
<br/>
mRemoteNG 에서 AWS EC2 인스턴스 접속 참고 글: <br/>
[https://victorydntmd.tistory.com/62](https://victorydntmd.tistory.com/62)  
[https://growingsaja.tistory.com/690](https://growingsaja.tistory.com/690)  
[https://baengsu.tistory.com/1](https://baengsu.tistory.com/1)  


putty 세션 저장한 후, mRemoteNG 새연결을 꼭 아래와 같이 설정할 것



첫 접속 시엔 ubuntu user명으로 접속해야 함.  
이후, 접속 시에 root 를 사용하고 싶으면 관련 설정이 필요함.  
그러나 굳이? 그럴 필요 있나 싶어서?   
ubuntu로 접속한 후에 root로 로그인 하여 작업하기로 함.  
접속 시엔 ubuntu   
실질적인 root 권한은 root   

Elastic 설정을 해야, 장비 IP가 고정된다고 함. 지금 당장은 할 필요가 없어서 하지 않음.  


 iii. MYSQL 설치  
참고한 글: [https://awakening95.tistory.com/2](https://awakening95.tistory.com/2)  
  
죽이 되든 밥이 되든 최신 사양으로 설치  

```
sudo wget https://dev.mysql.com/get/mysql-apt-config_0.8.22-1_all.deb
```

패키지 다운로드  

```
sudo dpkg -i mysql-apt-config_0.8.22-1_all.deb
```

최신화

```
sudo apt-get update
```

mysql server 설치 
```
sudo apt-get install mysql-server
```
  
보안 관련 설정은 생략함  
당장 필요할 거 같지 않음  

 iv. Slave DB 설정
참고한 글 : [https://ltlkodae.tistory.com/31](https://ltlkodae.tistory.com/31)  
  
위에서 만든 mysql만 설치된 인스턴스를 복제  
db 장비의 기초 환경으로 생각  
단순 복제라서 별도 기술 없이 넘어감  


2. MHA 적용하기
이제 MHA를 적용해보자. 
참고한 글: [https://blog.naver.com/theswice/222489176002](https://blog.naver.com/theswice/222489176002)  
  
master 장비의 my.cnf 수정   
  
```shell
[mysqld]
max_allowed_packet=100M
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
max_binlog_size=512M
sync_binlog = 1
expire-logs-days = 7
binlog_do_db = test
log-bin-trust-function-creators = 1
report-host = acs
# -> host name of DB
relay-log = relay_log
relay-log = relay_log
relay-log-index = relay_log.index
relay_log_purge = off
log_slave_updates = ON
```  



참고한 글

[https://hoing.io/archives/9812](https://hoing.io/archives/9812)

해당 글에서는 MYSQL 5.7을 기반으로 테스트 하였으나 

8.0으로 하고자 함

