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
- server-id
    MySQL 고유 식별값 

- datadir
    데이터 파일이 저장되는 디렉토리

- tmpdir
    MySQL 에서는 정렬, 그룹핑 시, 내부적으로 임시테이블을 생성하는데 해당 임시테이블의 저장 위치  

- default-storage-engine
    MySQL 내에서 사용할 스토리지 엔진 설정  

- skip-name-resolve
    MySQL은 외부로부터 접속 요청을 받을 경우, 인증을 위해 IP 주소를 호스트 네임으로 바꾸는 과정(hostname lookup)을 수행합니다. 이는 접속 시에 불필요한 부하를 일으킵니다. 해당 변수를 설정 후, IP 기반으로 접속을 하게 되면 hostname lookup 과정을 생략하게 되어, 빠르게 접속이 가능합니다. 

- max_connecion
    접속 가능한 최대 클라이언트 수.  

- max_allowed_packet  
    1153 Error 발생 시, 확인해봐야하는 변수. 서버로 질의하거나 받게되는 패킷의 최대 길이를 나타냄. 
    기본 값은 67108864이며 최대 값은 1GB.  

- sort_buffer_size	
    MySQL에서는 데이터를 정렬하기 위해 별도의 메모리 공간을 할당함. 이 때 사용되는 메모리가 Sort buffer임. 
    해당 메모리는 정렬이 필요한 경우에만 할당되며 쿼리 실행이 완료되면 시스템으로 즉시 반납됨. 
    이 Sort Buffer의 크기를 조정할 수 있는 시스템 변수임. 단위는 byte.  

- query_cache_limit
    쿼리 캐싱 메모리 제한 설정
- query_cache_size
    쿼리 캐싱 메모리 사이즈 설정
- query_cache_type
    MySQL Query Cache 설정이 켜져있고 Query Cache를 사용하는 쿼리일 경우, MySQL은 들어온 쿼리에 대해 먼저 Query Cache를 조회합니다. Query Cache에 존재하는 경우라면 바로 캐시에 담겨져 있는 데이터를 반환함.
    MySQL 8.0 버전부터는 제거되었음. 




- connect_timeout
    MySQL이 클라이언트로부터 접속 요청을 받을 경우, 대기하는 시간. 기본값은 5초.  
    

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

