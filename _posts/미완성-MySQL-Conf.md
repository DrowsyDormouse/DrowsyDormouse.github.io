---
layout: post
title: 자주 쓰는 my.conf 시스템 변수를 공부해보자.
comments: true
categories: [MySQL]
tag: [MySQL]
---

my.cnf 에서 사용되는 시스템 변수를 공부할 겸 정리해보도록 합시다.  
~~워낙 자주 까먹는 나를 위해서~~  
<br/>  

**Server&System**
- connect_timeout
    MySQL이 클라이언트로부터 접속 요청을 받을 경우, 대기하는 시간. 기본값은 5초.  
    
- max_allowed_packet  
    1153 Error 발생 시, 확인해봐야하는 변수. 서버로 질의하거나 받게되는 패킷의 최대 길이를 나타냄. 
    기본 값은 67108864이며 최대 값은 1GB.

- net_buffer_length  
    max_allowed_packet 와 함께 확인해야하는 변수. 일반적으로는 max_allowed_packet만 정해놓은 경우가 많으나, 해당 변수를 설정해 두면 그 용량을 넘기는 메시지를 전달해야할 경우, 자동으로 이 값을 늘리도록 되어있음. 그러므로 효율을 위해서는 해당 변수의 값은 어플리케이션이 사용하는 일반적인 쿼리에서 전송되는 바이트 값의 평균 정도로 충분히 낮은 값을 설정해주고 max_allowed_packet은 최대로 높은 값을 설정하는 것이 좋음. 
    기본 값은 16384이며 최대 값은 1MB.


**InnoDB**
- innodb_flush_log_at_trx_commit
    commit 시, redo log flush 동작 방식을 설정.  
    transaction commit(쓰기 작업) 요청이 시작되면 데이터 변경 내용이 log buffer(메모리 영역)와 os cache(메모리 영역)를 거쳐 fsync 디스크 동기화를 통해 redo log file(디스크 영역)에 기록됨.  
    1로 설정되면 위 과정을 모두 처리한 후에 완료가 반환되게 됨.  
    0으로 설정되면 log_buffer까지의 기록 후 완료 반환하며 1초마다 디스크 동기화 발생.  
    2로 설정하면 os cashe까지의 기록 후 완료 반환하며 1초마다 디스크 동기화 발생.  
    기본 값은 1임.  

