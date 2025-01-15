---
title: "[JPA] 내가 왜 persist인지 알려줄게 - 영속성 관리"
excerpt: "JPA를 한다면 영속성에 대해선 마스터해야 인지상정"

categories:
  - JPA
tags:
  - [jpa, db, context, persist]

permalink: /jpa/persist/

toc: true
toc_sticky: true

date: 2025-01-07
last_modified_at: 2025-01-07
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 영속성 컨텍스트

쉽게 말하면 엔티티를 영구 저장하는 환경이다. 지금으로썬 EntityManager = 영속성 컨테스트라고 생각해도 된다.

- `EntityManager.persist(entity)`를 호출하면 entity는 영속성
- 영속성 컨텍스트는 논리적인 개념으로 실제로 코드에 보이진 않는다.
- 엔티티 매니저를 통해 영속성 컨텍스트에 접근한다.

### 엔티티의 생명주기

- 비영속(new/transient)
    - 영속성 컨테스트와 전혀 관계가 없는 새로운 상태이다.
    - 엔티티를 새로 만들었을 때의 상태라고 생각하면 된다.
    
    ```java
    Member member = new Member();
    member.setId("member1");
    member.setUsername("회원");
    ```
    
- 영속(managed)
    - 영속성 컨텍스트에 관리되는 상태이다.
    - `em.persist(entity)`를 호출한 상태라고 보면 된다.
    - 엔티티에 변경이 일어나면 감지한다.
    
    ```java
    Member member = new Member();
    member.setId("member1");
    member.setUsername("회원");
    
    EntityManager em = emf.createEntityManager();
    em.getTransaction.begin();
    
    //객체를 저장한 상태
    em.persist(member);
    
    //영속 상태에선 변경을 감지하여 update 쿼리를 날림
    member.setUsername("회원2");
    ```
    
- 준영속(detached)
    - 영속성 컨텍스트에 저장되었다가 분리된 상태이다.
    - `em.detach(entity)`를 호출하여 영속성 컨텍스트로부터 분리한다.
    - 해당 상태에선 엔티티에 변경이 일어나도 감지하지 않는다.
    
    ```java
    //회원 엔티티를 영속성 컨텍스트에서 분리, 준영속 상태
    em.detach(member);
    
    //준영속 상태에선 엔티티 속성을 변경해도 아무일도 일어나지 않음
    member.setUsername("회원3");
    ```
    
    - `em.detach(entity)`: 특정 엔티티만 준영속 상태로 전환한다.
    - `em.clear()`: 영속성 컨텍스트를 완전히 초기화시킨다.
    - `em.close()`: 영속성 컨텍스트를 종료시킨다.
- 삭제(removed)
    - 말 그대로 삭제된 상태이다.
    
    ```java
    //객체를 삭제한 상태
    em.remove(member);
    ```
    

## 영속성 컨텍스트로 인한 이점

영속성 컨텍스트가 존재함으로써 얻는 이점들이 많기 때문에 JPA사용하는 것이다. 어떤 것들이 있는지 알아보자

### 1차 캐시

영속성 컨텍스트(EntityManager)는 1차 캐시를 갖고 엔티티를 persist(영속) 하면 즉시 DB에 저장되는 것이 아닌 1차 캐시에 저장이 된다.

```java
Member member = new Member();
member.setId("member1");
member.setUsername("회원");

//DB가 아닌 1차 캐시에 저장된다.
em.persist(member);
```

- 엔티티를 조회할 시 우선적으로 1차 캐시에서 조회한 후 엔티티가 없다면 DB를 조회한다.

```java
//1차 캐시에서 조회한다.
Member findMember1 = em.find(Member.class, "member1");

//1차 캐시에 없다면 DB에서 조회한다.
//member2는 이미 DB에 저장된 상태
Member findMember2 = em.find(Member.class, "member2");
```

### 영속 엔티티의 동일성 보장

같은 Id를 가진 엔티티라면 동일성을 보장해준다

```java
Member a = em.find(Member.class, "member1");
Member b = em.find(Member.class, "member2");

System.out.println(a == b); //true
```

- 1차 캐시에서 조회 가능한 등급의 트랜잭션 격리 수준을 DB가 아닌 애플리케이션 차원에서 제공한다.

### 트랜잭션을 지원하는 쓰기 지연 - 엔티티 등록

EntityManager는 데이터 변경 시 반드시 트랜잭션을 시작해야 한다.

```java
EntityTransaction tx = em.getTransaction();
//트랜잭션 시작
tx.begin();

//persist 호출 시에는 INSERT SQL을 DB에 보내지 않음
//1차 캐시에 저장하고, INSERT SQL을 생성하여 쓰기 지연 SQL 저장소에 저장한다.
em.persist(memberA);
em.persist(memberB);

//커밋하는 순간 쓰기 지연 저장소에 있는 INSERT SQL을 DB에 보낸다
tx.commit();
//1. tx.commit() -> 2. flush -> 3. commit
```

- persistence.xml에 `<property name="hibernate.jdbc.batch_size" value="10"/>` 같은 속성을 추가하여 쿼리를 몇 개씩 모아보낼 것인지 설정할 수 있다.

### 엔티티 수정 - 변경 감지(Dirty Checking)

일반적이라면 영속 엔티티를 변경하고 사용자가 직접 `em.update()`같은 메서드를 호출해야 하지 않을까?

**그런 거 필요없다.** JPA는 스스로 엔티티의 변경을 감지하고 update 쿼리문까지 알아서 날려준다.

```java
EntityTransaction tx = em.getTransaction()
tx.begin();

//영속 엔티티 조회
Member member = em.find(Member.class, "memberA");

//영속 엔티티 데이터 수정 -> 변경 감지
memberA.setUsername("hi");
memberA.setAge(10);

//커밋
tx.commit();
```

영속성 컨텍스트가 변경을 감지하고 DB에 UPDATE SQL을 보내는 과정을 알아보자

1. 변경 감지하고 `flush()` 호출
2. 엔티티와 스냅샷 비교를 하여 변경 사항 체크
→ 1차 캐시에는 초기 엔티티의 상태를 보관하는 스냅샷이 존재한다.
3. UPDATE SQL 생성하여 쓰기 지연 SQL 저장소에 저장
4. flush하여 DB에 SQL 날림
5. commit 

### 엔티티 삭제

```java
//삭제 대상 엔티티 조회
Member memberA = em.find(Member.class, "memberA");

//엔티티 삭제
em.remove(memberA); 
```

## 플러시 - flush

영속성 컨텍스트의 변경 내용을 데이터베이스에 반영한다.

### 발생 시점

- Dirty Checking 발생
- 수정된 엔티티 쓰기 지연 SQL 저장소에 등록한다.
- 쓰기 지연 SQL 저장소의 쿼리를 데이터베이스에 전송한다. (등록, 수정, 삭제)

### 플러시 방법

- em.flush() → 직접 호출
- 트랜잭션 커밋 - > 플러시 자동 호출
- JPQL 쿼리 실행  → 플러시 자동 호출
    
    ```java
    em.persist(memberA);
    em.persist(memberB);
    em.persist(memberC);
    
    //중간에 JPQL 실행 -> 만약 플러시가 되지 않는다면 아무것도 가져오지 못하게 됨
    query = em.createQuerey("select m from Member m", Member.class);
    List<Member> members = query.getResultList();
    ```
    

실제로 플러시를 직접 호출하는 일은 잘 없다고 한다. → 테스트 코드에서 DB 저장 확인을 위해 사용

### 플러시 옵션

```java
em.setFlushMode(FlushModeType.COMMIT);
```

- `FlushModeType.AUTO`: 커밋이나 쿼리를 실행할 때 플러시 → 기본값
- `FlushModeType.COMMIT`: 커밋할 때만 플러시

### 플러시 특징

- 영속성 컨텍스트를 비우지 않는다. → 변경 내용을 데이터베이스에 동기화
- 트랜잭션이라는 작업 단위가 매우 중요하기 떄문에 커밋 직전에만 동기화 하면 된다.