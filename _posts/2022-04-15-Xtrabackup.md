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
 - 
mysqldump: 

Xtrabackup


## Why Choose this?


## Let's Get Started


## Completion






설치 사유: 

1. innoDB 스토리지 엔진을 사용하는 테이블은 mysqlhotcopy로 백업이 되지 않음.

2. mysqldump를 사용할 경우, innoDB 테이블 백업 작업과 myisam 테이블 백업 작업을 따로 해야함.

3. 내 개인적인 공부 욕



내부 망에 설치해야 하므로 소스 설치 방법 채용

https://www.percona.com/downloads/Percona-XtraBackup-2.4/LATEST



percona-xtrabackup-2.4.24-Linux-x86_64.glibc2.12.tar.gz

다운로드 후, winscp 사용하여 장비에 업로드 



업로드 후에 /usr/local/src 에 압축 해제해서 넣기.. 

tar -zxvf percona-xtrabackup-2.4.24-Linux-x86_64.glibc2.12.tar.gz -C /usr/local/src



글에서는 컴파일할 때, cmake를 사용했으나 우리 장비에는 그런거 없.. 이라서 다른 방법을 추구함. 

gcc와 make를 사용,

gcc -v -I/usr/local/src/percona-xtrabackup-2.4.24/include -DDEBUG -Wall -W -O2 -L/usr/local/src/percona-xtrabackup-2.4.24/lib -o percona-xtrabackup main.c -lm


글쓴이가 글을 헷갈리게 적어뒀음

해당 링크에서 다운 받는 것도 소스 코드가 아님

걍 아래로 



dev 장비의 DB 재시작이 필요로 해보임 



아~ 멈춰!!!

mysqlhotcopy 사용 시, ibd 파일로 저장되고 있긴함.

inno엔진이어도. 

~~그럼 얘네를 살리는 방안으로 가는 게 더 나을 거 같아서 일단 멈춤. ~~  
살릴 수 없습니다..  
mysqldump 또는 XtraBackup을 이용하여 정상적인 방식을 통해 restore 하는 과정이 필요함.  


차후 다시 연구... 