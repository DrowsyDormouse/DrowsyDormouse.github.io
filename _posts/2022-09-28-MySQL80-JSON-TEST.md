---
layout: post
title: MySQL 8.0에서 JSON 사용하기
comments: true
update: 2022-09-28
categories: [MySQL 8.0, JSON]
tag: [MySQL 8.0, JSON]
---

최근 필요하여 R&D 한 내용을 정리.  

<br>
<br>
<br>

기존에도 MySQL에서 JSON 타입의 데이터를 저장하거나 사용할 수 있었으나,  
해당 기능이 8.0에 들어서 많이 업그레이드 되었음.  

<br>

특히 대량의 데이터 BULK INSERT가 가능해졌다는 점이 좋은 것 같습니다.  

<br>

```sql

CREATE
    DEFINER = ssh22@`%` PROCEDURE PUBLIC_DEV.SP_INSERT_AVATAR_FASHION_JSON_TEST(IN IN_CHAR_ID BIGINT,
                                                                                IN IN_ACCOUNT_ID BIGINT,
                                                                                IN IN_CHAR_INFO_LIST JSON)
BEGIN
    INSERT INTO CHAR_INFO (CHAR_ID, ACCOUNT_ID, CHAR_TYPE, JOB_TYPE, HP, MP, SP, SPEED, X, Y, `STR`, DEX, `INT`, LUCK)
    SELECT IN_CHAR_ID
         , IN_ACCOUNT_ID
         , CHAR_TYPE
         , JOB_TYPE
         , HP
         , MP
         , SP
         , SPEED
         , X
         , Y
         , `STR`
         , DEX
         , `INT`
         , LUCK
    FROM JSON_TABLE(IN_CHAR_INFO_LIST, '$[*]'
        COLUMNS (
              CHAR_TYPE INT path '$.CHAR_TYPE'
            , JOB_TYPE  INT path '$.JOB_TYPE'
            , HP        INT path '$.HP'
            , MP        INT path '$.MP'
            , SP        INT path '$.SP'
            , SPEED     INT path '$.SPEED'
            , X         INT path '$.X'
            , Y         INT path '$.Y'
            , `STR`     INT path '$.STR'
            , DEX       INT path '$.DEX'
            , `INT`     INT path '$.INT'
            , LUCK      INT path '$.LUCK'
        )) jst;
end;
```

위 SP 처럼 JSON 인자를 PARAM으로 받았을 경우, JSON_TABLE 함수를 사용하면 반복문 사용없이 INSERT가 가능합니다. 
단일건 INSERT가 아닌, BULK INSERT이므로 속도 또한 개선됩니다. 
~~이거 없던 시절엔 어케 JSON 타입 썼는지 몰라...~~  

물론 RDBMS에서 JSON을 쓰는 것보단 그에 적합한 다른 DBMS를 사용하는 것이 좋겠으나, 늘 최적의 상황에서 개발할 수 있는 것이 아니니...ㅠㅜㅠ  
