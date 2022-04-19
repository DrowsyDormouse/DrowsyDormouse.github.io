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

```SQL
CREATE TABLE sample_log{
    log_date datetime NOT NULL,
    log_id bigint(20) UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (log_date, log_id)
};
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

```SQL
ALTER TABLE sample_log
    PARTITION BY RANGE(log_date)
    (
        PARTITION p_20211201 VALUES LESS THAN ('2022-01-01') ENGINE=InnoDB,
        PARTITION p_MAX VALUES LESS THAN MAXVALUE ENGINE=InnoDB
    );
```  

위와 같이 파티션을 생성해보면 22년 1월 1일 이전 데이터로 구성된 p_20211201 파티션과 나머지로 구성된 M_MAX 파티션이 생성됩니다.
  

#### 파티션 조회 시, 결과 예시  

```SQL
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

```SQL
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

```SQL
ALTER TABLE sample_log -- 파티션이 추가될 테이블명
    PARTITION BY RANGE(log_date) -- 파티션의 구분 값으로 사용될 컬럼명
    (
        PARTITION p_20211201 -- 생성될 파티션 이름
          VALUES LESS THAN ('2022-01-01') -- 생성될 파티션 기준 값
          ENGINE=InnoDB,
        PARTITION p_MAX VALUES LESS THAN MAXVALUE ENGINE=InnoDB
    );
```  
