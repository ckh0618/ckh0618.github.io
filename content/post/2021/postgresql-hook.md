---
published: true
title: PostgreSQL Hooks   
date: 2018-02-19T11:43:46+09:00
last_modified_at: 2019-03-09T03:30:00+09:00
categories: [데이터베이스]
tags: [postgresql, hook]
--- 


# 들어가며   

PostgreSQL 은 Hook 이라는 아주 멋진 기능을 제공합니다. 본 글은 PostgreSQL 에서 제공하는 Hook 의 기본 개념과 동작방식에 대해서 소개합니다. 

# Hook 

Hook 은 일종의 Function Pointer 로써, 각각의 모듈별로 정의되어 있습니다. PostgreSQL 개발자들은 기능을 확장할 수 있도록 
여러가지 Hook 등을 정의해놓았고, 여기에 내가 만든 코드를 넣어 손쉽게 PostgreSQL 의 동작을 변경하거나, 내가 필요한 코드를 추가로 넣어 기능을 확장 시킬 수 있습니다.

이러한 Hook 들은 PostgreSQL 의 _hook 접미사를 가진 전역변수로 정의되어있는데, 사실 제대로 Document 되어있지는 않습니다. 
버젼이 올라갈 때마다 훅이 추가되고 있는데, https://github.com/AmatanHead/psql-hooks 여기에 비공식적이기는 하지만, 
Hook 들이 정리되고 있습니다.  

contribs 코드를 보면 많은 코드가 hook 을 이용하여 구현한걸 볼 수 있습니다. 


# 동작방식 
Hook 을 사용하는 가장 간단한 예제는 contribs 에 있는 auth_delay 입니다. 80여줄도 안되는 짧은 코드로 
사용자의 인증이 실패했을때 sleep 을 줘서 무작위대입 공격을 막기 위한 기능을 구현했습니다.  

몇줄 안되니까 전체를 보면 

{{< gist ckh0618 bd2fc22ddd00a819f3f40984f853ce82 >}}


_PG_init() 은 모듈이 로딩될떄 호출되는 함수라는 정도만 이해하면, 그리 이해하는데 어려운 코드는 아닐겁니다.  

PostgreSQL 은 각각의 주요 모듈의 헤더파일마다 _hook_type 의 접미사를 가진 Hook 을 정의해놨습니다. 여기서 예를 든 
ClientClientAuthentication_hook_type 과 변수는 auth.h / auth.c 파일에 다음과 같이 정의되어있습니다. 

```
typedef void (*ClientAuthentication_hook_type) (Port *, int);
extern PGDLLIMPORT ClientAuthentication_hook_type ClientAuthentication_hook;
```

이런식으로 PostgreSQL 개발자들은 Hook 을 모두 별도의 타입으로 정의해놨고, 전역변수로 Hook 을 노출 해놨습니다. 
ClientAuthentication_hook 에 사용자가 만든 로직을 넣어주면 사용자가 정의한 함수를 수행하고, 원래 PostgreSQL 에서 의도된 
동작을 수행하게 됩니다. 

위 코드에서 하나 더 주의 깊게 살펴봐야하는 부분은, 기존에 설정된 Hook을 복사하는 과정입니다. 똑같은 Hook 을 사용하는 여러개의 모듈들이 존재할 수 있기 때문에 
혹시라도 설치되어있을 Hook 을 복사해놓고, 이를 나중에 호출해 주는 것입니다. 

제대로 코딩을 하려면 사실 Hook 에 정의된 인자 및 구조체를 따라가봐야하는데, 이는 제공하는 각각의 훅마다 다르므로 정의를 보고 인자들을 연구해봐야합니다. 
제가 주로 사용하는 방식은 gdb 이나 clion 같은 툴을 사용하여 debugging 모드로 따라가면서 인자를 분석합니다.  

# 사용예 
Hook 은 엄청나게 많은 contribs 코드에서 사용되고 있습니다. 

* auth_deply 
* pg_stat_statement 
* auth_explain 

그외 PostgreSQL 을 확장한 Citus의 경우도 이 Hook 을 적극적으로 사용하여 Multi-Node Cluster 방식의 OLAP 엔진을 구현했습니다. 

다음은 이 Hook 을 이용하여 간단하게 만들어본  pg_log 에 로그를 필터링하는 모듈입니다. 
https://github.com/ckh0618/pg_logfilter 자세한 사용법은 README 를 참고하시면 됩니다. 

그럼 Happy Coding ! 