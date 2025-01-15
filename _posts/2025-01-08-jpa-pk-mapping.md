---
title: "[JPA] 기본키, 너의 이름은… - 기본키 매핑"
excerpt: "엔티티를 식별하는 기본키, 어떤 방식으로 설정해야 할까?"

categories:
  - JPA
tags:
  - [jpa, db, entity, mapping, pk]

permalink: /jpa/pk-mapping/

toc: true
toc_sticky: true

date: 2025-01-08
last_modified_at: 2025-01-08
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 기본키 매핑 방법

JPA는 어떻게 기본키의 값을 할당할까?

- 직접 할당: @Id만 사용
- 자동 생성: @GeneratedValue
    - IDENTITY: 데이터베이스에 위임한다.
    - SEQUENCE: 데이터베이스 시퀀스 오브젝트를 사용한다.
    → @SequenceGenerator 필요
    - TABLE: 키 생성용 테이블을 사용
    → @TableGenerator 필요
    - AUTO: 데이터베이스에 따라 자동 지정 → 기본값이다.

지금부터 하나하나 자세히 살펴보자

### IDENTITY 전략

```java
@Entity
public class Member {
	@Id @GeneratedValue(strategy = GeneerationType.IDENTITY)
	private Long id;
}
```

- 기본 키 생성을 데이터베이스에 위임한다.
- 주로 MySQL, PostrgreSQL, SQL Server, DB2에서 사용한다.
→ AUTO_INCREAMENT
- JPA는 보통 트랜잭션 커밋 시점에 INSERT SQL을 실행하기 때문에 데이터베이스에 INSERT SQL을 실행한 후 ID 값을 알 수 있다.
- 따라서 IDENTITY 전략은 persist() 시점에 즉시 INSERT SQL을 실행하고 DB에서 식별자를 조회한다.

### SEQUENCE 전략

```java
@Entity
@SequenceGenerator(
		name = "MEMBER_SEQ_GENERATOR".
		sequenceName = "MEMBER_SEQ",
		initialValue = 1, allocationSize = 1)
public class Member {
	@Id @GeneratedValue(strategy = GenerationType.SEQUENCE,
											generator = "MEMBER_SEQ_GENERATOR")
	private Long id;
}
```

- DB 시퀀스는 유일한 값을 순서대로 생성하는 특별한 DB 오브젝트이다.
- Oracle, PostgreSQL, DB2, H2 데이터베이스에서 사용

**@SequenceGenerator 속성**

- name: 식별자 생성기 이름
- sequenceName: 데이터베이스에 등록되어 있는 시퀀스 이름
- initialValue: DDL 생성 시에만 사용되고 시퀀스 DDL을 생성할 때 처음 시작하는 수를 지정한다.
- allcationSize: 시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용된다)
→ DB에 시퀀스 값이 하나씩 증가하도록 설정되어 있으면 이 값을 반드시 1로 설정해야 함
- catalog, schema: 데이터베이스 catalog, schema 이름

### TABLE 전략

```java
@Entity
@TableGenerator(
		name = "MEMBER_SEQ_GENERATOR",
		table = "MY_SEQUENCE",
		pkColumnValue = "MEMBER_SEQ", allocationSize = 1)
public class Member {
	@Id @GeneratedValue(strategy = GenerationType.TABLE, 
											generator = "MEMBER_SEQ_GENERATOR")
	private Long id;
}
```

**시퀀스를 흉내낸 테이블**

```sql
create table MY_SEQUENCE (
	sequence_name varchar(255) not null,
	next_val bigint,
	primary key (sequence_name)
)
```

- 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략
- 시퀀스 전략과 유사하지만 모든 데이터베이스 적용이 가능하다.

**@TableGenerator 속성**

- name: 식별자 생성기 이름
- table: 키생성 테이블명
- pkColumnName: 시퀀스 컬럼명
- valueColumnNa: 시퀀스 값 컬럼명
- pkColumnValue: 키로 사용할 값 이름
- initialValue: 초기 값, 마지막으로 생성된 값이 기준
- allocationSize: 시퀀스 한 번 호출에 증가하는 수
- catalog, schema: 데이터베이스 catalog, schema 이름
- uniqueConstraints: 유니크 제약 조건 지정 가능

### 영한님이 권장하는 식별자 전략

- 기본 키 제약 조건에 따라 기본키는 변하면 안된다.
- 초기에 올바르게 기본키를 설정을 하여도 미래까지 이를 만족하는 자연키는 많지 않다.
→ 과거에는 주민번호를 PK로 사용했지만 정책 변경으로 인해 PK로 사용하지 못하게 됨
→ 수많은 데이터베이스 변경 발생
- 따라서 비즈니스 의미가 없는 대체 키를 사용하는 것이 좋다.
- Long형 + 대체키 + 키 생성전략