---
layout: post
title: Xtrabackup이란?
comments: true
categories: [MySQL]
tag: [MySQL]
---

## Purpose of Study
Xtrabackup에 대해 알아보고자 하였습니다.  
아래 글을 참고하였습니다.  

 - xtarbackup 2.4 버전 소개 및 사용방법: [https://myinfrabox.tistory.com/219](https://myinfrabox.tistory.com/219)  
 - tar 명령어 사용법: [https://moonuibee.tistory.com/4](https://moonuibee.tistory.com/4)  
  

사실 저희 서비스는 제가 처음 맡을 때만 해도 모든 테이블이 MYISAM 엔진으로 이루어져 있었습니다.  
그래서 저는 지금 회사를 다니면서 InnoDB 엔진의 테이블을 저희 서비스에서 사용할만한 기회를 호시탐탐 노리고 있었죠.  

기존에 있는 MYISAM 테이블을 InnoDB로 변경하자니...  

> *캐릭터 정보 테이블 안에 버프 정보랑 장비 아이템 정보가 다 있는 건 좀...*  
*테이블도 나누고 그 참에 InnoDB로 바꿔버리죠!!*

> **아... 저희는 단일 스레드여서... 그렇게 바꾸면 한 번에 쿼리를 여러번 치게 될텐데... I/O가 많아져서 부담스러워요.**  

> *엣...* 1  


그렇다고 신규 테이블을 InnoDB로 넣자니,  

> *이번에 신규로 들어가는 강화 시스템, 그거 InnoDB로 해보면 어떨까요? 그렇게 하면 붙는 강화 옵션 갯수에 제한을 두지 않아도 될 거 같은데!!*

> **음... 좋네요. 아... 근데 저희는 단일 스레드 서버라서 그렇게 스키마를 구성하시면 강화 옵션을 변경할 때마다 옵션 갯수만큼 UPDATE를 쳐야되겠네요...**  
**그건 좀 힘들 거 같은데요...**

> *엣...* 2  

위와 같은 사유로 InnoDB 엔진의 테이블을 적용하는 일은 번번히 실패로 돌아가곤 했습니다.  
그러나 기회는 기다리는 자에게 오는 법 ! ! !  
  

> **이번에 신규로 들어가는 시스템. 기존 시스템으로는 어려울 거 같고 새로 테이블 빼야겠는데요.**  
**이전처럼 슬롯 형식으로 하죠. 기획 쪽에서도 슬롯 형으로 할거니까 갯수 제한을...**

> *엥? 아뇨. 아뇨. 이거 InnoDB로 하고 로그 형태로 남기죠??*

> **네??**

마침내 저한테 기회가 왔습니다. 새로 들어가는 테이블을 로그 형식으로 하고 기타등등 **뚝딱뚝딱** 해서 InnoDB로 변경했습니다.  
기분 좋게 InnoDB로 변경하고 다음 날, 청천벽력 같은 소리를 듣게 됩니다.

> **세희님, 운영툴 로그가 안보여요.**

> *엥???*

왜 안보였을까요?  
저희 운영툴은 BackUp DB 장비 내의 데이터를 기반으로 합니다.  
또한 저희 BackUp 시스템은 MySQL의 기능 중에 mysqlhotcopy를 사용하여 진행됩니다.  
그리고... mysqlhotcopy는 InnoDB의 백업 및 리스토어를 지원하지 않습니다... ㅠㅠ  

자세히 알아보지 않고 진행한, 후폭풍을 맞게 된 것이죠...  
급하게 SlaveDB의 테이블을 MYISAM 엔진으로 복구하고... 다른 Backup 시스템을 알아보았더니 mysqldump와 Xtrabackup에 대해서 알게 되었습니다. 



## MySQLHotcopy를 XtraBackup으로 바꿔보기 !!!
 - [Purpose of Study](#purpose-of-study)
 - [What is this?](#what-is-this)
 - [Why Choose this?](#why-choose-this)
 - [Let's Get Started](#lets-get-started)
 - [Completion](#최종-결과)


## What is this?
 - [mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)
 - [XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup)
mysqldump: 이고르 로마넨코(Igor Romaneko)가 작성한 백업 프로그램. MySQL 설치 시, 함께 번들로 설치됨. 테이블 생성, 데이터 쿼리, 등에 대한 SQL 생성문을 백업(논리적 백업). sql 파일로 백업됨. 

Xtrabackup: Percona에서 개발된 오픈소스 백업 툴, 엔진 데이터를 그대로 복사하는 물리적 백업 방식. 풀백업, 증분백업, 암호화 백업, 압축백업 지원.  


## Why Choose this?
mysqldump의 경우, master-data 옵션 사용 시, backup 시점의 binary event의 position 기록을 위해 global lock 또는 table lock을 획득하고 백업이 종료될 때까지 해제되지 않습니다. single-transaction 옵션을 같이 사용할 경우, binlog position 추출한 뒤, repeatable read 설정 후, start transation 후 lock을 해제할 수 있습니다. 이 과정은 매우 빠르게 진행되지만 global lock을 갖고 있는 동안에는 ddl이 실패할 수 있습니다. 또한 global lock을 획득할 때도 장시간 실행중인 transaction이 있으면 계속 대기하게 됩니다. 
추가로 저희 서비스에서는 MYISAM과 Inno를 함께 사용하는데 이 경우, mysqldump를 두번 실행하여 MYISAM 테이블 백업 후, Inno 테이블 백업해야 합니다. (순서는 반대여도 괜찮으나, **두 번 실행** 인 점이 중요)



## Let's Get Started  

내부 망에 설치해야 하므로 소스 설치 방법 채용

https://www.percona.com/downloads/Percona-XtraBackup-2.4/LATEST



percona-xtrabackup-2.4.24-Linux-x86_64.glibc2.12.tar.gz

다운로드 후, winscp 사용하여 장비에 업로드 



업로드 후에 /usr/local/src 에 압축 해제해서 넣기.. 

```
tar -zxvf percona-xtrabackup-2.4.24-Linux-x86_64.glibc2.12.tar.gz -C /usr/local/src
```

xtrabackup의 타겟 디렉토리 지정  

```
[xtrabackup]
target_dir = /usr/local/mysql55/data/
```

dev 장비의 DB 재시작

별도의 백업 테스트 유저 생성


백업 시,
```
./xtrabackup --defaults-file=/etc/my.cnf --user=유저 --host=localhost --port=3306 --password=패스워드 --socket=/usr/local/mysql55/tmp/mysql.sock --databases=백업DB --backup --target-dir=/usr/local/mysql55/tmep_backup
```
  
실행 결과, 개발 서버의 저장소 하나 백업하는데에 1분 33초 정도 걸림.  
또한 MYISAM 테이블과 Inno 테이블 모두 한 번에 문제 없이 백업되었음.  

복구 시,
```
./xtrabackup --copy-back --target-dir=/usr/local/mysql55/tmep_backup
chown -R mysql:mysql data
```

복구도 별다른 문제는 없이 되었음. 


## Completion
직접 테스트 해본 결과
실 서비스에 적용하기 어려운 점이 발견됨. 
 - restore 하는 동안, Slave DB의 서비스를 종료해야 함. 
 - 현재 운영툴은 매 시 정각마다 백업한 데이터저장소를 기반으로 작동하고 있음. 그러나 Xtrabackup을 사용하여 restore할 경우, 기본으로 원본 저장소에 데이터가 복원됨.  


위와 같은 문제를 해결하기 전까지는 실적용은 어려울 것으로 보임. 
따라서, 일단은 Slave DB에서만 해당 테이블을 MYISAM으로 변경,
점검 작업 시에 slave reset 할 때, 해당 테이블들을 MYISAM으로 변경하도록 쉘 스크립트를 생성함.  
  
<br/>
  
결론: 잘 알아보고 바꾸자!!!!!!!! ㅠㅠㅠㅠ  
~~개인 공부해서 다행인가...~~