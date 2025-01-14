---
title: "[JSP] 모든 게 객체인 세상 - JPA 개요"
excerpt: "JSP는 어떻게 알아서 데이터베이스를 구성할까?"

categories:
  - JSP
tags:
  - [jsp, db, entity]

permalink: /jsp/about-jsp/

toc: true
toc_sticky: true

date: 2025-01-06
last_modified_at: 2025-01-06
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## JPA, 첫 걸음을 디뎌보자

처음 JPA를 사용하기 위해서는 우선 설정 정보가 필요하다. 스프링 부트를 사용하면 모든 걸 알아서 해주지만 JPA에 대한 심도 높은 이해를 위해 이번엔 일반 자바 환경에서 구성해보자.

### 설정 파일 - persistence.xml

JPA를 실행하기 위한 설정 파일로써 데이터베이스 종류, 하이버네이트 설정 등을 지원한다.

**persistence.xml설정 코드**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.2" xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">
    <persistence-unit name="hello">
        <properties>
            <!-- 필수 속성 -->
            <property name="jakarta.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="jakarta.persistence.jdbc.user" value="sa"/>
            <property name="jakarta.persistence.jdbc.password" value=""/>
            <property name="jakarta.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>

            <!-- 옵션 -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments"  value="true"/>
<!--            <property name="hibernate.hbm2ddl.auto" value="create" />-->
        </properties>
    </persistence-unit>

</persistence>
```

- persistence-unit name으로 이름을 지정한다.
- jakarta.persistence로 시작하는 속성: JPA 표준 속성
- hibernate로 시작하는 속성: 하이버네이트 전용 속성

### 어떤 데이터베이스든 Chill한 JPA

- JPA는 특정 데이터베이스에 종속되지 않는다.
- 데이터베이스마다 제공하는 SQL 문법과 함수는 조금씩 다른데 JPA는 데이터베이스 종류를 설정해주면 알아서 쿼리문을 작성해준다.

**어떻게 이런 일이 가능한걸까?**

- JPA는 Dialect라는 인터페이스를 사용하여 SQL을 생성한다.
- Dialect라는 클래스를 각 데이터베이스에 맞게 상속을 받아 [MySQL, Oracle, H2]Dialect 등을 생성 후 사용해 SQL을 생성한다.
- hibernate.dialect 속성에 지정한다.
`<property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>`
- 하이버네이트는 40가지 이상의 데이터베이스를 지원한다.

### 구동 방식

1. META-INF/persistence.xml 설정 정보를 조회하여 기본 설정을 완료한다.
2. EntityManagerFactory(emf)를 생성한다. → 애플리케이션에서 하나만 생성하여 공유한다.
3. emf를 사용하여 EntityManager(em)을 생성한다. → 쓰레드 간에 공유를 하면 안된다.
→ JPA의 모든 데이터 변경은 Transacion 안에서 실행해야만 한다.

### JPA 기본 동작

**엔티티**

```java
@Entity
public class Member {
	@Id
	private Long id;
	private String name;
}
```

- @Entity: JPA가 관리할 객체를 뜻한다.
- @Id: 데이터베이스 PK(기본키)와 매핑하는 필드이다.

**엔티티 조회**

- `EntityManager.find(”id”)` 를 사용하여 조회한다.
- 조회한 member를 사용하여 `member.getId( )`, `member.getName( )` 등을 사용해 객체 그래프를 탐색한다.

## JPQL

JPA를 사용하면 엔티티 객체를 중심으로 개발을 하게 되는데 엔티티를 검색할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색한다. 하지만 모든 DB 데이터를 객체로 변환하기엔 무리가 있기 떄문에 결국 SQL이 필요하게 된다. 그래서 등장한 것이 JPQL이다.

```java
em.createQuery("select m from Member m", Member.class);
```

### 왜 쓰는거야

- 위에서 말했듯이 상세한 검색을 위해서는 결국 SQL이 필요하다.
- 이를 위해 JPA는 SQL 추상화한 JPQL이라는 객체 지향 쿼리 언어를 제공한다.
- SQL을 추상화하기 때문에 특정 데이터베이스 SQL에 의존하지 않는다.
- 테이블을 대상으로 쿼리를 만드는 SQL과 문법이 유사하지만 JPQL은 엔티티 객체를 대상으로 쿼리를 만든다.