---
published: true
title: MemoryContext concepts in PostgreSQL 
date: 2018-05-19T01:42:46+09:00
last_modified_at: 2019-03-09T03:30:00+09:00
categories: [database]
tags: [postgresql, memory management]
--- 

## 배경 

메모리 관리는 여러운 작업중에 하나입니다. 메모리를 할당하고 사용하는 것은 좋은데 할당받은 메모리 포인터를 적시에 반환하지 않거나 놓쳐버리는 경우 흔히 말하는 메모리 릭이 발생하게 됩니다. 이러한 문제는 간단한 커멘드라인 유틸리티등과 같이 짧은 시간 동작하고 종료되는 경우는 그리 큰 문제가 되지 않습니다. OS 가 알아서 프로세스가 종료될때 할당받은 자원을 정리하여 주거든요. 그러나 데이터베이스 서버와 같이 오랜시간 데몬 형태로 동작하여야하는 경우는 메모리를 다 잡아먹고 시스템을 서비스 불능 상태로 만들어버리는 등의 심각한 문제를 일으킬수 있습니다. 


## MemoryContext 

PostgreSQL 에서는 이러한 메모리 관리 이슈를 단순화 하기 위하여 MemoryContext 라는 추상화된 개념을 사용합니다. MemoryContext 를 한마디로 간단히 표현하면 계층화된 Memory Pool Manager 라고 할 수 있는데요. 다음과 같은 동작을 지원합니다. 

* Create 
* Allocate 
* Destroy 
* Reset 

하나씩 살펴보면 다음과 같습니다. 

### Create 

새로운 MemoryContext 를 생성합니다. 이때 부모 Context 를 지정하는데, 최상위 부모는 TopMemoryContext 입니다. TopMemoryContext 하위에 각 모듈별로 세부적인 MemoryContext 가 존재합니다. 개발하는 모듈에 따라 만들고자 하는 MemoryContext 가 어느 위치에 존재하여야할지는 개발자가 판단하여야 합니다. 예를 들어 현재의 Transaction 으로 라이프사이클이 정해진  메모리를 할당하기 위해서는 CurTransactionContext 하단에 위치하는 MemoryContext 를 만들어서 사용하는 것이 유리합니다. 만약 내가 메모리 해제를 잊었더라도 CurTransactionContext 가 Destory 되면서 내가 할당한 MemoryContext 안의 메모리를 같이 정리해줄꺼니까요. 

### Allocate

현재 내가 사용하는 MemoryContext 를 통해서 메모리를 할당합니다. palloc() 을 하면 현재 내가 세팅한 MemoryContext 에서 관리되는 메모리를 리턴해줍니다. 이 내용은 아래에서 좀 더 자세히 살펴봅니다. 

### Destroy 

현재 Context 를 삭제합니다. 계층화 되어서 좋은 이유가 여기서 나오는데, Destroy 는 현재 삭제되는 MemoryContext 하단의 MemoryContext 도 같이 같이 Destory 해줍니다. SQL옵션중에 CASCADE 와 비슷합니다. 그렇기 때문에 설령 하부 MemoryContext 에서 Destroy 하지 않더라도 상위에서 한번에 정리할 수 있습니다. 

### Reset 

현재 Memory Context 가 관리하는 메모리를 Reset 합니다 free() 와 동일한데, 해당 MemoryContext에서 할당된 메모리를 한번에 free() 한다고 생각하면 될것 같습니다. 

## MemoryContext 개념의 장점 

이러한 MemoryContext 라는 개념을 도입하면서 개발자는 다음과 같은 장점을 가집니다. 

* 모듈 A 에서 관리되는 메모리는 모듈 A에서 만들어진 Context 에서 모두 관리됩니다. 만약 A 모듈의 하위 모듈인 A' 모듈에서 새로운 Context 를 부모 Context 를 A 로 지정하여 만들어 관리 주체를 세분화 시킬수도 있습니다. 즉 이러한 세부적인 메모리 관리는 여러사람들이 코드를 고칠때 매우 높은 수준의 메모리 안정성을 제공할 수 있는 토대가 됩니다. 

* 기존의 전통적인 C 프로그램에서는 개발자가 메모리의 포인터를 다 관리하고 있어야 적절한 시점에 삭제하고 재사용할수 있도록 OS 에게 돌려줄수 있습니다. 그런데 이러한 MemoryContext 방식은 개발자가 단편화된 메모리를 관리할 필요가 없습니다. 필요할때마다 Allocation 을 통해 메모리를 할당받아서 사용하다가 해당 모듈이 종료되고 더 이상 해당 Memory Context 가 필요하지 않을때 한번에 삭제할수 있습니다. 

* Memory 관리를 Userspace에서 수행하는 것이기 때문에 훨씬 더 빠르고 OS 에 덜 의존적입니다.  


## Memory Context  구현

PostgreSQL 은 현재 사용 중인 MemoryContext 를 저장하는 전역변수를 가지고 있습니다. palloc.h 를 보면 다음과 같은 정의를 볼 수 있습니다.  

```c
/*
 * CurrentMemoryContext is the default allocation context for palloc().
 * Avoid accessing it directly!  Instead, use MemoryContextSwitchTo()
 * to change the setting.
 */
extern PGDLLIMPORT MemoryContext CurrentMemoryContext;
```

이 전역변수를 통해서 현재 사용중인 MemoryContext 포인터를 일일이 전달하지 않고 malloc() / free() 와 동일하게 palloc() / pfree() 을 사용할 수 있습니다. palloc() 의 정의를 보면 다음과 같습니다. 

```c
void *
palloc(Size size)
{
    /* duplicates MemoryContextAlloc to avoid increased overhead */
    void       *ret;
    MemoryContext context = CurrentMemoryContext;

    AssertArg(MemoryContextIsValid(context));
    AssertNotInCriticalSection(context);
    ...
}
```

전역변수인 CurrentMemoryContext 에 세팅된 값을 얻어와서 설정하는 것을 볼 수 있습니다. 

## MemoryContext 사용

CurrentMemoryContext 라는 전역변수가 있으니 직접 컨텍스트 할당하면 될것 같지만, 해당 변수의 선언에 달린 주석에서 경고한것 처럼 그렇게 하면 안됩니다. 대신 PostgreSQL 은 MemoryContextSwitchTo() 라는 함수를 정의했고 사용자는 이걸 사용하면 됩니다. 아래 코드와 같이 매우 간단합니다. 

```c
static inline MemoryContext
MemoryContextSwitchTo(MemoryContext context)
{
    MemoryContext old = CurrentMemoryContext;

    CurrentMemoryContext = context;
    return old;
}
```

MemoryContext 를 전환할때 old 값을 저장하라는 의미로 위 함수를 만들어놓은 것 같습니다. 

보통 MemoryContext 를 새롭게 생성하고 이를 사용하는 방식은 다음과 같이 정형화 되어있습니다. 

```c
MemoryContext old, new; 

// 새로운 MemoryContext 를 만듭니다. 부모는 TopMemoryContext 입니다. 
new = AllocSetContextCreate (
						TopMemoryContext, 
						"mytest context", 
#if PG_VERSION_NUM >= 90600
						ALLOCSET_DEFAULT_SIZES
#else
						ALLOCSET_DEFAULT_MINSIZE,
						ALLOCSET_DEFAULT_INITSIZE,
						ALLOCSET_DEFAULT_MAXSIZE
#endif
						) ;

//  새로운 MemoryContext 로 전환. 이제부터 메모리 할당은 새로운 MemoryContext 에서 관리됩니다. 
old = MemoryContextSwitchTo ( new ); 

/* do somthing.... */ 

// 다 썼으면 MemoryContext 를 이전값으로 옮겨줍니다. 
MemoryContextSwitchTo ( old ); 

```

## 결론  

MemoryContext 는 많은 사람들이 여러가지 모듈을 동시에 변경하는 PostgreSQL 에서 필수적으로 사용되는 개념입니다. 개발자는 자신의 모듈에 대해서만 메모리 관리를 신경쓰면 되고, 혹시라도 실수로 메모리 해제를 잊었더라도 상위 MemoryContext 가 삭제되면 같이 삭제되기 때문에 안전합니다.  

## Reference 
* https://blog.pgaddict.com/posts/introduction-to-memory-contexts
* https://github.com/postgres/postgres/tree/master/src/backend/utils/mmgr
