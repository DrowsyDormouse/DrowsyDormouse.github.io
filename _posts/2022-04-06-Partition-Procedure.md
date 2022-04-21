---
layout: post
title: log 테이블의 파티션 관리 프로시저에 대해
comments: true
categories: [MySQL]
tag: [MySQL]
---

## Purpose of Study
MySQL 파티션 시스템을 편리하게 관리하는 프로시저를 학습하고 만들어보고자 하였습니다.   
로그 파티션 관리 테이블의 값에 따라, 테이블마다 설정된 기간으로 파티션을 생성하고 추가하는 프로시저를 만들고자 하였습니다.  

<br/>

여러분은 책을 자주 읽으시나요?   
저는 특별한 이슈가 없다면 아침, 점심, 저녁 가리지 않고 자리에 앉아, 책을 펼치고 밀려오는 졸음과 씨름하며 읽는 편입니다.   
회사에서 읽는 자기 개발 서적이 가장 머리에 쏙쏙 들어오는 편이죠. ~합법 월루 너무 좋아요~   
그 날도 평소와 같이 자리에서 들어오는 이슈를 다 치고 월급 루팡 겸 자습을 하고 있었습니다.   

> **세희님, 잠깐 시간 괜찮으세요?**

> *네?*

![](../asset/Partition%20Procedure/images/joker_bear_1.jpg)   

대체 무슨 일이지. 방금 졸면서 머리 헤드 벵잉 하는 거 보셨나... 

> **세희님, 혹시 바쁘세요?**

> *앗... 아뇨....*

> **그럼 공부할 겸 해서 이거 한번 해보실래요?**

> ***헉 뭔데요? 완전 좋아용~***


<br/>
<br/>


## log 테이블의 파티션 관리 프로시저에 대해
 - [Purpose of Study](#purpose-of-study)
 - [What is this?](#what-is-this)
    + 파티션이란?
 - [Why Choose this?](#why-choose-this)
 - [Let's Get Started](#lets-get-started)
 - [Completion](#최종-프로시저-모습)

<br/>

![](../asset/Partition%20Procedure/images/2022-04-05%20171027.png)   

다음 블로그 글을 참고하였습니다.   
 - [[MySQL] 사용자 로그 테이블 - (1) Primary Key가 필요한가?](https://purumae.tistory.com/209?category=652830)
 - [[MySQL] 사용자 로그 테이블 - (2) 파티션 관리](https://purumae.tistory.com/210?category=652830)
 - [[MySQL] 사용자 로그 테이블 - (3) 자동화된 파티션 관리](https://purumae.tistory.com/211?category=652830)

<br/>

일단 무엇을 만들어야하는지 정확히 알아봅시다.  
고객의 needs는 아래와 같습니다.  

 - 매개변수로 일, 주, 월을 받아서 해당 주기만큼 파티션을 나눠줘야 함.  
 - 이미 파티션이 있을 경우에는 추가해주고 없을 경우에는 새로 생성해줘야 함.  
 - 주 단위는 일요일 또는 월요일부터 시작하는 걸로  
 - 프로시저로 만들어줬으면 좋겠음.
  
<br/>
  
> *좋아! 이제 만들어보자! 엥? 근데 파티션이 뭐지?*  
  
<br/>
  
먼저 MySQL 파티션의 개념을 알아봅시다.  
  
<br/>

## What is this?

### 파티션이란  
데이터를 별도의 테이블로 분리해서 저장하나, 사용자 입장에서 하나의 테이블로 읽기와 쓰기를 할 수 있게 해주는 기능  
Range, List, Hash, Key, 이렇게 4가지 방식이 있음.
 - Range: 범위 기반, 가장 메이저한 방식
 - List: 코드나 카테고리 등 특정 값 기반
 - Hash: 설정한 HASH 함수 기반
 - Key: MD5() 함수를 이용한 HASH 값 기반


## Why Choose this?
그렇다면 왜 파티션을 이용해야할까요?  
파티션을 이용할 경우, 아래와 같은 장점이 있습니다.  

 1. INSERT와 범위 SELECT의 빠른 처리
 2. 주기적으로 삭제 등의 작업이 이루어지는 이력성 데이터의 효율적인 관리
 3. 데이터의 물리적인 저장소를 분리 


## Let's Get Started  
백문이 불여일견!  
파티션 테이블을 직접 사용해봅시다.  
임시로 가상의 로그 테이블 sample_log 을 생성합니다.  
<br/>

````sql
CREATE TABLE sample_log{
    log_date datetime NOT NULL,
    log_id bigint(20) UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (log_date, log_id)
};
````


```javascript
/* Some pointless Javascript */
var rawr = ["r", "a", "w", "r"];
```

임시 데이터를 넣어봅시다. 저는 아래와 같이 넣었습니다.  

<br/>

<div>
    <table>
        <th>log_date </th>
        <th>log_id </th>
        <tr>
            <td>'2021-12-01 00:00:00'</td>
            <td>1</td>
        </tr>
        <tr>
            <td>'2021-12-02 00:00:00'</td>
            <td>2</td>
        </tr>
        <tr>
            <td>'2021-12-03 00:00:00'</td>
            <td>3</td>
        </tr>
        <tr>
            <td>...</td>
            <td>...</td>
        </tr>
        <tr>
            <td>'2022-03-31 00:00:00'</td>
            <td>121</td>
        </tr>
    </table>
</div>
<br/>

이제 파티션을 생성해봅시다.  

```sql
ALTER TABLE sample_log
    PARTITION BY RANGE(log_date)
    (
        PARTITION p_20211201 VALUES LESS THAN ('2022-01-01') ENGINE=InnoDB,
        PARTITION p_MAX VALUES LESS THAN MAXVALUE ENGINE=InnoDB
    );
```  

위와 같이 파티션을 생성해보면 22년 1월 1일 이전 데이터로 구성된 p_20211201 파티션과 나머지로 구성된 M_MAX 파티션이 생성됩니다.
  

#### 파티션 조회 시, 결과 예시  

```sql
SELECT * FROM sample_log PARTITION (p_20220101);
```  
<div>
    <table>
        <th>log_date </th>
        <th>log_id </th>
        <tr>
            <td>'2021-12-01 00:00:00'</td>
            <td>1</td>
        </tr>
        <tr>
            <td>'2021-12-02 00:00:00'</td>
            <td>2</td>
        </tr>
        <tr>
            <td>'2021-12-03 00:00:00'</td>
            <td>3</td>
        </tr>
        <tr>
            <td>...</td>
            <td>...</td>
        </tr>
        <tr>
            <td>'2021-12-31 00:00:00'</td>
            <td>31</td>
        </tr>
    </table>
</div>
<br/>

```sql
SELECT * FROM sample_log PARTITION (p_MAX);
```  

<div>
    <table>
        <th>log_date </th>
        <th>log_id </th>
        <tr>
            <td>'2022-01-01 00:00:00'</td>
            <td>32</td>
        </tr>
        <tr>
            <td>'2022-01-02 00:00:00'</td>
            <td>33</td>
        </tr>
        <tr>
            <td>'2022-01-03 00:00:00'</td>
            <td>34</td>
        </tr>
        <tr>
            <td>...</td>
            <td>...</td>
        </tr>
        <tr>
            <td>'2022-03-31 00:00:00'</td>
            <td>121</td>
        </tr>
    </table>
</div>
<br/>

실제로 파티션을 나눈 뒤, 쿼리를 쳐보면 위 예시와 동일한 결과가 나옵니다.  
파티션을 생성할 때, 사용하는 쿼리를 다시 한 번 자세히 봅시다.  

```sql
ALTER TABLE sample_log -- 파티션이 추가될 테이블명
    PARTITION BY RANGE(log_date) -- 파티션의 구분 값으로 사용될 컬럼명
    (
        PARTITION p_20211201 -- 생성될 파티션 이름
          VALUES LESS THAN ('2022-01-01') -- 생성될 파티션 기준 값
          ENGINE=InnoDB,
        PARTITION p_MAX VALUES LESS THAN MAXVALUE ENGINE=InnoDB
    );
```  

보시다시피, 파티션이 추가될 테이블명과 파티션의 구분 값으로 사용될 컬럼명, 이렇게 두 개가 필수 값인 거 같네요.

그럼 이제 파티션을 관리할 테이블을 만들어 봅시다. 
```sql
CREATE TABLE log_retention (
  log_table_name varchar(64) NOT NULL COMMENT '로그 테이블 이름',
  log_date_column_name varchar(64) NOT NULL COMMENT '로그 일자 컬럼 이름',
  retention_period smallint(5) UNSIGNED NOT  NULL COMMENT '로그 보관 기간',
  log_read_cycle tinyint(4) DEFAULT NULL COMMENT '0: Year, 1: Month, 2: Week, 3: Day',
  PRIMARY KEY (log_table_name)
);
```  
위 테이블은 앞으로 로그 테이블을 관리할 테이블입니다.  



## 최종 프로시저 모습  

### 이미 파티션이 있는 테이블에 추가로 생성(Update)
```sql
-- `pUpd_Sample_Partition`
PROCEDURE pUpd_Sample_Partition (OUT V_RET int, IN IN_PUT_NUM int)
BEGIN
  /*=======================================================    
    @create by Se-Hee Song - 220112(KST)
    @ 이미 파티셔닝 된 로그 테이블에 추가로 파티션 생성
    @
  =========================================================*/

  DECLARE v_vch_log_table_name varchar(64);
  DECLARE v_vch_log_date_column_name varchar(64);
  DECLARE v_vch_retention_period smallint;
  DECLARE v_vch_log_read_cycle tinyint;
  DECLARE v_vch_partition_name varchar(64);

  DECLARE v_now_dw tinyint(4) DEFAULT WEEKDAY(NOW());
  DECLARE v_now_date datetime DEFAULT NOW();
  DECLARE v_now_year_date datetime DEFAULT DATE_FORMAT(NOW(), '%Y-01-01');
  DECLARE v_now_month_date datetime DEFAULT DATE_FORMAT(NOW(), '%Y-%m-01'); -- 이번달 1일
  DECLARE v_now_monday datetime DEFAULT DATE_ADD(NOW(), INTERVAL -v_now_dw DAY); -- 이번주 월요일

  DECLARE v_int_retention_period int UNSIGNED;
  DECLARE v_int_partition_count int UNSIGNED;
  DECLARE v_dat_start_date date;

  DECLARE v_int_i int UNSIGNED;
  DECLARE v_int_j int UNSIGNED;

  DECLARE EXIT HANDLER FOR SQLEXCEPTION SET V_RET = -1;

  SET SESSION group_concat_max_len = 1000000;

  DROP TABLE IF EXISTS tmp_reorganize;

  CREATE TEMPORARY TABLE tmp_reorganize (
    seq int UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    log_table_name varchar(64) NOT NULL,
    log_retention_period smallint NOT NULL,
    log_read_cycle tinyint(4) NOT NULL, -- 0: Year, 1: Month, 2: Week, 3: Day
    start_date date NOT NULL,
    partition_name varchar(64) NOT NULL
  );

  INSERT tmp_reorganize (log_table_name, log_retention_period, log_read_cycle, start_date, partition_name)
    SELECT STRAIGHT_JOIN
      LPM.log_table_name,
      LPM.retention_period,
      LPM.log_read_cycle,
      DATE_FORMAT(RIGHT(P.PARTITION_NAME, 8), '%Y-%m-%d'),
      P.PARTITION_NAME
    FROM log_retention LPM
      INNER JOIN information_schema.PARTITIONS P
        ON P.TABLE_SCHEMA = DATABASE()
        AND P.TABLE_NAME = LPM.log_table_name
    WHERE P.PARTITION_DESCRIPTION = 'MAXVALUE'
    AND RIGHT(P.PARTITION_NAME, 8) < RIGHT(CONCAT('p_', DATE(TIMESTAMPADD(year, IN_PUT_NUM + 1, v_now_year_date)) + 0, '_', DATE(TIMESTAMPADD(year, IN_PUT_NUM + 2, v_now_year_date)) + 0), 8)
    AND LPM.log_read_cycle = 0; -- Year

  INSERT tmp_reorganize (log_table_name, log_retention_period, log_read_cycle, start_date, partition_name)
    SELECT STRAIGHT_JOIN
      LPM.log_table_name,
      LPM.retention_period,
      LPM.log_read_cycle,
      DATE_FORMAT(RIGHT(P.PARTITION_NAME, 8), '%Y-%m-%d'),
      P.PARTITION_NAME
    FROM log_retention LPM
      INNER JOIN information_schema.PARTITIONS P
        ON P.TABLE_SCHEMA = DATABASE()
        AND P.TABLE_NAME = LPM.log_table_name
    WHERE P.PARTITION_DESCRIPTION = 'MAXVALUE'
    AND RIGHT(P.PARTITION_NAME, 8) < RIGHT(CONCAT('p_', DATE(TIMESTAMPADD(MONTH, IN_PUT_NUM + 1, v_now_month_date)) + 0, '_', DATE(TIMESTAMPADD(MONTH, IN_PUT_NUM + 2, v_now_month_date)) + 0), 8)
    AND LPM.log_read_cycle = 1; -- Month

  INSERT tmp_reorganize (log_table_name, log_retention_period, log_read_cycle, start_date, partition_name)
    SELECT STRAIGHT_JOIN
      LPM.log_table_name,
      LPM.retention_period,
      LPM.log_read_cycle,
      DATE_FORMAT(RIGHT(P.PARTITION_NAME, 8), '%Y-%m-%d'),
      P.PARTITION_NAME
    FROM log_retention LPM
      INNER JOIN information_schema.PARTITIONS P
        ON P.TABLE_SCHEMA = DATABASE()
        AND P.TABLE_NAME = LPM.log_table_name
    WHERE P.PARTITION_DESCRIPTION = 'MAXVALUE'
    AND RIGHT(P.PARTITION_NAME, 8) < RIGHT(CONCAT('p_', DATE(TIMESTAMPADD(WEEK, IN_PUT_NUM + 1, v_now_monday)) + 0, '_', DATE(TIMESTAMPADD(WEEK, IN_PUT_NUM + 2, v_now_monday)) + 0), 8)
    AND LPM.log_read_cycle = 2; -- Week

  INSERT tmp_reorganize (log_table_name, log_retention_period, log_read_cycle, start_date, partition_name)
    SELECT STRAIGHT_JOIN
      LPM.log_table_name,
      LPM.retention_period,
      LPM.log_read_cycle,
      DATE_FORMAT(RIGHT(P.PARTITION_NAME, 8), '%Y-%m-%d'),
      P.PARTITION_NAME
    FROM log_retention LPM
      INNER JOIN information_schema.PARTITIONS P
        ON P.TABLE_SCHEMA = DATABASE()
        AND P.TABLE_NAME = LPM.log_table_name
    WHERE P.PARTITION_DESCRIPTION = 'MAXVALUE'
    AND RIGHT(P.PARTITION_NAME, 8) < RIGHT(CONCAT('p_', DATE(TIMESTAMPADD(DAY, IN_PUT_NUM + 1, v_now_date)) + 0), 8)
    AND LPM.log_read_cycle = 3; -- Day

  SELECT
    1,
    COUNT(*) INTO v_int_i, v_int_j
  FROM tmp_reorganize;

  WHILE v_int_i <= v_int_j DO
    SELECT
      log_table_name,
      log_retention_period,
      log_read_cycle,
      start_date,
      partition_name INTO v_vch_log_table_name, v_vch_retention_period, v_vch_log_read_cycle, v_dat_start_date, v_vch_partition_name
    FROM tmp_reorganize FORCE INDEX FOR JOIN (PRIMARY)
    WHERE seq = v_int_i;

    IF (v_vch_log_read_cycle = 0) THEN -- Year
      SET v_dat_start_date = DATE_FORMAT(v_dat_start_date, '%Y-01-01');
      SET v_int_partition_count = TIMESTAMPDIFF(year, v_dat_start_date, DATE(TIMESTAMPADD(year, IN_PUT_NUM, v_now_year_date)));

      -- ALTER TABLE .. REORGANIZE PARTITION .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` REORGANIZE PARTITION ', v_vch_partition_name, ' INTO (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(year, (num - 1), v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(DAY, -1, TIMESTAMPADD(year, num, v_dat_start_date))) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(year, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM join_num FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

    ELSEIF (v_vch_log_read_cycle = 1) THEN -- Month
      SET v_dat_start_date = DATE_FORMAT(v_dat_start_date, '%Y-%m-01');
      SET v_int_partition_count = TIMESTAMPDIFF(MONTH, v_dat_start_date, DATE(TIMESTAMPADD(MONTH, IN_PUT_NUM, v_now_month_date)));

      -- ALTER TABLE .. REORGANIZE PARTITION .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` REORGANIZE PARTITION ', v_vch_partition_name, ' INTO (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(MONTH, (num - 1), v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(DAY, -1, TIMESTAMPADD(MONTH, num, v_dat_start_date))) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(MONTH, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM join_num FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

    ELSEIF (v_vch_log_read_cycle = 2) THEN -- Week
      -- 추가할 파티션의 개수
      -- SET v_int_partition_count = TIMESTAMPDIFF(DAY, v_dat_start_date, DATE(TIMESTAMPADD(DAY, pi_int_partition_buffer + 1, v_dt5_now)));
      SET v_int_partition_count = TIMESTAMPDIFF(WEEK, v_dat_start_date, DATE(TIMESTAMPADD(WEEK, IN_PUT_NUM + 1, v_now_monday)));

      -- ALTER TABLE .. REORGANIZE PARTITION .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` REORGANIZE PARTITION ', v_vch_partition_name, ' INTO (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(WEEK, num - 1, v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(WEEK, num, v_dat_start_date)) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(WEEK, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM join_num FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

    ELSEIF (v_vch_log_read_cycle = 3) THEN -- Day
      SET v_int_partition_count = TIMESTAMPDIFF(DAY, v_dat_start_date, DATE(TIMESTAMPADD(DAY, IN_PUT_NUM, v_now_date)));

      -- ALTER TABLE .. REORGANIZE PARTITION .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` REORGANIZE PARTITION ', v_vch_partition_name, ' INTO (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(DAY, num, v_dat_start_date)) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(DAY, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM join_num FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

    END IF;

    IF @vch_stmt IS NOT NULL THEN
      PREPARE stmt FROM @vch_stmt;
      EXECUTE stmt;
      DEALLOCATE PREPARE stmt;
    END IF;
    SET v_int_i = v_int_i + 1;
  END WHILE;

  DROP TABLE tmp_reorganize;
END
```

### 파티션이 없는 테이블에 새로이 생성(Create)
```sql
-- pIns_Sample_Partition
PROCEDURE pIns_Sample_Partition (OUT V_RET int, IN IN_PUT_NUM int)
BEGIN
  /*=======================================================    
    @create by Se-Hee Song - 220112(KST)
    @ 아직 파티셔닝 되지 않은 로그 테이블에 최초로 파티션 생성
    @
  =========================================================*/

  DECLARE v_vch_log_table_name varchar(64);
  DECLARE v_vch_log_date_column_name varchar(64);
  DECLARE v_vch_retention_period smallint;
  DECLARE v_vch_log_read_cycle tinyint;

  DECLARE v_now_dw tinyint(4);
  DECLARE v_oldest_dw tinyint(4);
  DECLARE v_now_date datetime;
  DECLARE v_oldest_date date;
  DECLARE v_dat_start_date date;
  DECLARE v_int_partition_count int UNSIGNED;

  DECLARE v_int_i int UNSIGNED;
  DECLARE v_int_j int UNSIGNED;


  DECLARE EXIT HANDLER FOR SQLEXCEPTION SET V_RET = -1;

  SET SESSION group_concat_max_len = 1000000;

  DROP TEMPORARY TABLE IF EXISTS tmp_create;
  CREATE TEMPORARY TABLE tmp_create (
    seq int UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    log_table_name varchar(64) NOT NULL,
    log_date_column_name varchar(64) NOT NULL,
    log_retention_period smallint NOT NULL,
    log_read_cycle tinyint(4) NOT NULL -- 0: Year, 1: Month, 2: Week, 3: Day
  );

  INSERT tmp_create (log_table_name, log_date_column_name, log_retention_period, log_read_cycle)
    SELECT STRAIGHT_JOIN
      LPM.log_table_name,
      LPM.log_date_column_name,
      LPM.retention_period,
      LPM.log_read_cycle
    FROM log_retention LPM
      INNER JOIN information_schema.PARTITIONS P
        ON P.TABLE_SCHEMA = DATABASE()
        AND P.TABLE_NAME = LPM.log_table_name
    WHERE P.PARTITION_NAME IS NULL;

  SELECT
    1,
    COUNT(*) INTO v_int_i, v_int_j
  FROM tmp_create;
  -- 1.2 아직 파티셔닝 되지 않은 로그 테이블에 최초로 파티션 생성
  --  ㄴ 임시 테이블 `tmp_create`의 Rows 수 만큼 LOOP
  --    ㄴ 파티션 생성이 필요한 로그 테이블의 수 만큼 LOOP
  WHILE v_int_i <= v_int_j DO
    -- 테이블에 기록된 "가장 오래된 날짜"를 담을 세션 변수 초기화
    --  ㄴ 동적 쿼리를 사용해 조회한 결과를 외부로 추출해야 하기 때문에 로컬 변수를 사용할 수 없음
    SET @dt5_oldest_log_date = NULL;

    -- 이번 LOOP에서 파티션을 생성할 테이블에 대해, 파티션 생성에 필요한 정보를 추출
    --  ㄴ log_table_name, log_date_column_name은 tmp_create 테이블에서 직접 추출
    --  ㄴ oldest_log_date는 동적 쿼리를 만들어 추출
    SELECT
      CONCAT(
      'SELECT ', log_date_column_name, '
          INTO @dt5_oldest_log_date
          FROM ', log_table_name, '
          FORCE INDEX FOR ORDER BY (PRIMARY)
          ORDER BY ', log_date_column_name, '
          LIMIT 1;'
      ) AS dynamic_query,
      log_table_name,
      log_date_column_name,
      log_retention_period,
      log_read_cycle INTO @vch_stmt, v_vch_log_table_name, v_vch_log_date_column_name, v_vch_retention_period, v_vch_log_read_cycle
    FROM tmp_create FORCE INDEX FOR JOIN (PRIMARY)
    WHERE seq = v_int_i;

    -- oldest_log_date를 구하기 위해 동적 쿼리 @vch_stmt 실행
    PREPARE stmt FROM @vch_stmt;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt; -- 삭제 

    -- log_read_cycle 에 따라 파티션 주기가 달라짐. 
    -- 각 주기에 맞춰서 프로시저 호출 
    IF (v_vch_log_read_cycle = 0) THEN -- Year
      SET v_now_date = DATE_FORMAT(NOW(), '%Y-01-01');
      SET v_oldest_date = DATE_FORMAT(@dt5_oldest_log_date, '%Y-01-01'); -- 기록된 가장 오래된 년도의 1월 1일
      SET v_dat_start_date = DATE(IFNULL(v_oldest_date, v_now_date)); -- 첫 파티션 일자 : 테이블에 기록된 가장 오래된 년도의 1월 1일
      -- 만드는 파티션의 총 개수 = 파티션 시작 일자 ~ 오늘까지 기간 + 버퍼 삼아 미리 만들 파티션의 수
      SET v_int_partition_count = TIMESTAMPDIFF(year, v_dat_start_date, DATE(TIMESTAMPADD(year, IN_PUT_NUM, v_now_date)));
      -- ALTER TABLE .. PARTITION BY RANGE COLUMN .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` PARTITION BY RANGE COLUMNS (`', v_vch_log_date_column_name, '`) (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(year, (num - 1), v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(DAY, -1, TIMESTAMPADD(year, num, v_dat_start_date))) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(year, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM JOIN_NUM FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

      PREPARE stmt FROM @vch_stmt;
      EXECUTE stmt;
      DEALLOCATE PREPARE stmt;
    ELSEIF (v_vch_log_read_cycle = 1) THEN -- Month
      SET v_now_date = DATE_FORMAT(NOW(), '%Y-01-01'); -- 이번달 1일
      SET v_oldest_date = DATE_FORMAT(@dt5_oldest_log_date, '%Y-%m-01'); -- 기록된 가장 오래된 달의 1일
      SET v_dat_start_date = DATE(IFNULL(v_oldest_date, v_now_date)); -- 첫 파티션 일자 : 테이블에 기록된 가장 오래된 달의 1일
      -- 만드는 파티션의 총 개수 = 파티션 시작 일자 ~ 오늘까지 기간 + 버퍼 삼아 미리 만들 파티션의 수
      SET v_int_partition_count = TIMESTAMPDIFF(MONTH, v_dat_start_date, DATE(TIMESTAMPADD(MONTH, IN_PUT_NUM, v_now_date)));
      -- ALTER TABLE .. PARTITION BY RANGE COLUMN .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` PARTITION BY RANGE COLUMNS (`', v_vch_log_date_column_name, '`) (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(MONTH, (num - 1), v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(DAY, -1, TIMESTAMPADD(MONTH, num, v_dat_start_date))) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(MONTH, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM JOIN_NUM FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

      PREPARE stmt FROM @vch_stmt;
      EXECUTE stmt;
      DEALLOCATE PREPARE stmt;
    ELSEIF (v_vch_log_read_cycle = 2) THEN -- Week
      SET v_now_dw = WEEKDAY(NOW());
      SET v_now_date = DATE_ADD(NOW(), INTERVAL -v_now_dw DAY); -- 이번주 월요일
      SET v_oldest_dw = WEEKDAY(@dt5_oldest_log_date);
      SET v_oldest_date = DATE_ADD(@dt5_oldest_log_date, INTERVAL -v_oldest_dw DAY); -- 기록된 가장 오래된 날짜의 주의 월요일 
      SET v_dat_start_date = DATE(IFNULL(v_oldest_date, v_now_date)); -- 첫 파티션 일자 : 테이블에 기록된 가장 오래된 날짜의 주의 월요일. 테이블이 비어있다면 이번주 월요일

      -- 만드는 파티션의 총 개수 = 파티션 시작 일자 ~ 오늘까지 기간 + 버퍼 삼아 미리 만들 파티션의 수
      -- IN_PUT_NUM은 만들 파티션이 몇주치인가 이므로 몇주치를 일자로 변경해야함. 
      -- 첫 로그 월요일 부터 일요일 -> 1개 
      -- 1 ~ + IN_PUT_NUM 
      SET v_int_partition_count = TIMESTAMPDIFF(WEEK, v_dat_start_date, DATE(TIMESTAMPADD(WEEK, IN_PUT_NUM + 1, v_oldest_date)));

      -- ALTER TABLE .. PARTITION BY RANGE COLUMN .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` PARTITION BY RANGE COLUMNS (`', v_vch_log_date_column_name, '`) (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(WEEK, (num - 1), v_dat_start_date)) + 0, '_', DATE(TIMESTAMPADD(DAY, -1, TIMESTAMPADD(WEEK, num, v_dat_start_date))) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(WEEK, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM JOIN_NUM FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

      PREPARE stmt FROM @vch_stmt;
      EXECUTE stmt;
      DEALLOCATE PREPARE stmt;
    ELSEIF (v_vch_log_read_cycle = 3) THEN -- Day
      SET v_dat_start_date = DATE(IFNULL(@dt5_oldest_log_date, NOW())); -- 첫 파티션 일자 : 테이블에 기록된 가장 오래된 날

      -- 만드는 파티션의 총 개수 = 파티션 시작 일자 ~ 오늘까지 기간 + 버퍼 삼아 미리 만들 파티션의 수
      SET v_int_partition_count = TIMESTAMPDIFF(DAY, v_dat_start_date, DATE(TIMESTAMPADD(DAY, IN_PUT_NUM + 1, NOW())));

      -- ALTER TABLE .. PARTITION BY RANGE COLUMN .. 문을 동적 쿼리로 생성하여 실행
      SELECT
        CONCAT('ALTER TABLE `', v_vch_log_table_name,
        '` PARTITION BY RANGE COLUMNS (`', v_vch_log_date_column_name, '`) (',
        GROUP_CONCAT('  PARTITION p_', DATE(TIMESTAMPADD(DAY, num, v_dat_start_date)) + 0,
        ' VALUES LESS THAN ', IF(num = v_int_partition_count, 'MAXVALUE', CONCAT('(''', DATE(TIMESTAMPADD(DAY, num, v_dat_start_date)), ''')')), ' ENGINE = InnoDB' SEPARATOR ','), ');'
        ) INTO @vch_stmt
      FROM JOIN_NUM FORCE INDEX FOR JOIN (PRIMARY)
      WHERE num <= v_int_partition_count;

      PREPARE stmt FROM @vch_stmt;
      EXECUTE stmt;
      DEALLOCATE PREPARE stmt;
    END IF;

    SET v_int_i = v_int_i + 1;
  END WHILE;
  DROP TABLE tmp_create;
  SET V_RET = 0;
END
```