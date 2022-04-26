---
layout: post
title: Xtrabackup이란?
comments: true
categories: [MySQL]
tag: [MySQL]
---

참고한 글: 

xtarbackup 2.4 버전 소개 및 사용방법: https://myinfrabox.tistory.com/219

tar 명령어 사용법: https://moonuibee.tistory.com/4





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