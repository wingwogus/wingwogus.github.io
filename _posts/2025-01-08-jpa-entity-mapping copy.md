---
title: "[JPA] DB와 JPA의 소개팅 시간 - 엔티티 매핑"
excerpt: "JPA에서 엔티티에 대한 DB 설정을 하고 싶다면 어떻게 해야할까?"

categories:
  - JPA
tags:
  - [jpa, db, entity, mapping]

permalink: /jpa/entity-mapping/

toc: true
toc_sticky: true

date: 2025-01-08
last_modified_at: 2025-01-08
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 엔티티란 무엇일까

JPA에서 가장 많이 보고, 사용할 것 바로 엔티티이다. 또한 JPA를 사용하는데 엔티티에 대해 모른다면 JPA를 제대로 알지 못 한다고 할 수 있다. 한 번 알아보자

### @Entity

- @Entity가 붙은 클래스는 JPA가 관리하고 이를 엔티티라고 부른다.
- JPA를 사용해서 테이블과 매핑할 클래스들은 반드시 @Entity를 붙여야 한다.

```java
@Entity
public class Member{}
```

**엔티티를 쓸 때 주의사항**

- 기본 생성자 필수(파라미터가 없는 public 또는 protected 생성자가 필요하다)
- final 클래스, enum, interface, inner 클래스는 엔티티로 지정하지 말아야한다.
- 저장할 필드에 final을 사용하면 안된다.

```java
@Entity
public class Member{
	
	@Id
	private Long id;
	private String name;
	private int age;
}
```

 **@Entity 속성**

`@Entity(name = "엔티티 이름")`

- JPA에서 사용할 엔티티 이름을 지정한다.
- 기본값은 클래스 이름을 그대로 사용한다.
- 같은 클래스 이름이 없으면 가급적 기본값을 사용하는 것이 좋다.

### @Table

@Table은 엔티티와 매핑할 테이블을 지정할 수 있다.

`@Table(속성명 = “value”)`를 사용하여 원하는 속성을 지정할 수 있다.

- name : 기본 값으로는 엔티티 이름과 테이블을 매칭하지만 따로 지정해 줄 수 있다.
- catalog: 데이터베이스 catalog를 매핑시켜 준다.
- schema: 원하는 스키마와 매핑시켜준다.
- uniqueConstraints: DDL(Data Definition Language) 생성 시에 유니크 제약 조건을 생성한다.

## 데이터베이스 스키마 자동 생성

- JPA는 DDL을 애플리케이션 실행 시점에 엔티티와 연관 관계 매핑을 통해 테이블 중심이 아닌 객체 중심으로 DDL을 생성한다.
- 어떤 데이터베이스든 사용할 수 있도록 각 데이터베이스 쿼리에 맞게 생성해준다.
- 하지만 이렇게 만든 DDL은 개발 장비에서만 사용하고 운영서버에서는 사용하지 않거나 적절히 다듬고 추가를 한 뒤에 사용한다.

### 자동 생성 속성

persistence.xml에 hibernate.hbm2ddl.auto라는 속성이 있는데 해당 값을 변경할 수 있다.

**Maven 기반 persistence.xml**

```xml
<properties>
	<property name="hibernate.hbm2ddl.auto" value="create" />
</properties>
```

**Gradle 기반 스프링 설정 파일 application.yml**

```yaml
  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
      # show_sql: true
        format_sql: true
```

- `create`: 기존에 있던 테이블을 삭제 후 다시 생성한다.(DROP + CREATE)
- `create-drop`: create와 같으나 종료 시점에 테이블을 DROP한다.
- `update`: 변경분만 반영된다(운영 시점에서는 사용하면 안된다)
- `validate`: 엔티티와 테이블이 정상적으로 매핑되었는지만 확인한다.
- `none`: 사용하지 않는다. → 아무 단어나 입력해도 됨, 대부분 관례상 none으로 작성
    
    → 운영 단계에서는 validate 혹은 none을 사용한다.
    

### 제약 조건 추가

테이블에 제약 조건을 추가하고 싶다면 필드에 애노테이션을 붙여 원하는 제약 조건을 추가할 수 있다.

**@Column을 사용한 제약 조건 추가**

```java
@Entity
public class Member{
	
	@Id
	private Long id;
	
	//회원 이름은 필수이고 10자를 초과하면 안된다.
	@Column(nullalbe = false, length = 10)
	private String name;
	private int age;
}
```

**@Table을 사용한 유니크 제약 조건 추가**

```java
@Entity
@Table(uniqueConstraints = {@UniqueConstraint(name = "NAME_AGE_UNIQUE", 
																							columnNames = {"Name", "AGE"})})
public class Member{
	@Id
	private Long id;
	
	//회원 이름은 필수이고 10자를 초과하면 안된다.
	@Column(nullalbe = false, length = 10)
	private String username;
	private int age;
}
```

참고로 JPA의 DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고 JPA의 실행 로직에는 영향을 주지 않는다.

## 필드와 컬럼 매핑

요구 사항이 추가로 있다고 가정하고 다시 코드를 수정해보자

**요구사항**

- 회원은 일반 회원과 관리자로 구분해야 한다.
- 회원 가입일과 수정일이 있어야 한다.
- 회원을 설명할 수 있는 필드가 있어야 한다. 이 필드는 길이 제한이 없다.

**요구사항을 충족한 코드**

```java
@Column
public class Member{
	@Id
	private Long id;
	
	@Column(name = "name")
	private String username;
	
	private Integer age;
	
	@Enumerated(EnumType.STRING)
	private RoleType roleType;
	
	private LocalDate createdDate;
	
	private LocalDate lastModifiedDate;
	
	@Lob
	private String description;
}
```

- **@Column: 컬럼과 필드를 매핑하고 속성을 통해 제약 조건 부여 가능**
    - name: 필드와 매핑할 테이블의 컬럼 이름
    - insertable, updatable: 등록, 변경 가능 여부
    - nullable(DDL): null 값의 허용 여부를 설정
    - unique(DDL): @Table의 uniqueConstraints와 같지만 한 컬럼에만 간단히 유니크 제약 조건을 걸 때 사용
    - columnDefinition: 데이터베이스의 컬럼 정보를 직접 줄 수 있다.
    
    ```java
    @Column(columnDefinition = "varchar(100) default 'EMPTY'")
    private String address;
    //DDL -> address varchar(100) default 'EMPTY'
    ```
    
    - length(DDL): 문자 길이 제약 조건, String 타입에만 사용, 기본값은 255
    - prescision, scale(DDL): BigDecimal, BigInteger 타입에서 사용
    prescision은 소수점을 포함한 전체 자릿수 scale은 소수의 자릿수이다.
- **@Enumerated: enum 타입을 매핑할 때 사용**
    - value: 데이터베이스에 저장할 값 유형
        - EnumType.ORDINAL: enum 순서를 데이터베이스에 저장
        - EnumType.STRING: enum 이름을데이터베이스에 저장
        
        → 절대 ORDINAL을 사용하면 안된다. 중간에 enum 순서가 바뀐다면 데이터베이스를 재구성해야하는 대참사가 벌어질 수 있다.
        
- **@Temporal: 날짜 타입(Date, Calender)을 매핑할 때 사용 → LocalDate류에서는 생략 가능**
- **@Lob: 데이터베이스 BLOB, CLOB 타입과 매핑**
    - 지정할 수 있는 속성이 없다.
    - CLOB: String, char[], java.sql.CLOB → 매핑하는 필드 타입이 문자
    - BLOB: byte[], java.sql.BLOB → 나머지
- **@Transient: 메모리상에서만 사용하고 싶을 떄 사용**
    - 필드 매핑, 데이터베이스에 저장과 조회 모두 되지 않는다.
    - 잠시 메모리에 어떤 값을 보관하고 싶을 때 사용한다.
    
    ```java
    @Column
    public class Member{
    	@Id
    	private Long id;
    	
    	@Column(name = "name")
    	private String username;
    	
    	private Integer age;
    	
    	@Enumerated(EnumType.STRING)
    	private RoleType roleType;
    	
    	private LocalDate createdDate;
    	
    	private LocalDate lastModifiedDate;
    	
    	@Lob
    	private String description;
    	
    	@Transient
    	private Integer temp;
    }
    ```