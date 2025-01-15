---
title: "[JPA] 각자의 연애는 다른 법이야 - 다양한 연관관계 매핑"
excerpt: "엔티티는 몇 개의 엔티티들과 관계를 맺을 수 있을까?"

categories:
  - JPA
tags:
  - [jpa, db, entity, relation]

permalink: /jpa/various-relationship/

toc: true
toc_sticky: true

date: 2025-01-11
last_modified_at: 2025-01-11
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 연관관계 매핑 용어

연관관계를 매핑할 때 고려해야 할 사항들이 몇 가지 있다. 용어와 같이 알아보자

### 다중성

JPA는 다양한 연관관계 매핑을 지원하는데 4가지로 분류할 수 있다.

- **다대일(N:1) - @ManyToOne**
- **일대다(1:N) - @OneToMany**
- **일대일(1:1) - @OneToOne**
- **다대다(N:M) - @ManyToMany**

### 단방향, 양방향

- 데이터베이스 테이블은 외래키 하나로 양쪽에 조인이 가능하기 때문에 사실상 방향이라는 개념이 존재하지 않는다.
- 하지만 객체는 참조용 필드가 있는 쪽으로만 참조가 가능하기 때문에 한쪽만 참조를 한다면 단방향, 양쪽이 서로 참조하면 양방향이다.

### 연관관계의 주인

- 테이블은 외래키 하나로 두 테이블이 연관관계를 맺기 떄문에 연관관계의 주인이 따로 없다.
- 객체 양방향 관계는 양쪽에서 서로 참조를 하기 떄문에 실제 테이블에 저장될 외래키를 관리할 곳을 지정해야 한다.
- 이때 외래 키를 관리하는 참조를 연관관계의 주인이라고 한다.
- 주인의 반대편은 외래 키에 영향을 주지 않으며 실제 테이블에도 생성되지 않고 단순 조회만 가능하다

## 다대일(N:1)

다대일에도 단방향 양방향이 존재한다.

### 다대일 단방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@ManyToOne
	@JoinColumn(name = "team_id")
	private Team team;
}
```

```java
@Entity
public class Team {
	
	@Id @GeneratedValue
	@Column(name = "team_id")
	private Long id;
	
	private String name;
}
```

- 가장 많이 사용하는 연관관계이다.
- Member에서는 Team을 참조할 수 있지만 역으로 Team에서는 Member를 참조하지 못한다.

### 다대일 양방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@ManyToOne
	@JoinColumn(name = "team_id")
	private Team team;
}
```

```java
@Entity
public class Team {
	
	@Id @GeneratedValue
	@Column(name = "team_id")
	private Long id;
	
	private String name;
	
	@OneToMany(mappedBy="region")
	private List<Member> members = new ArrayList<>();
}
```

- 외래 키가 있는 곳이 연관관계의 주인으로 양쪽이 서로를 참조하도록 개발하여야 한다.

## 일대다(1:N)

역으로 일(1)에서 외래키를 관리할 수도 있다.

### 일대다 단방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
}
```

```java
@Entity
public class Team {
	
	@Id @GeneratedValue
	@Column(name = "team_id")
	private Long id;
	
	private String name;
	
	@OneToMany
	@JoinColumn(name = "team_id")
	private List<Member> members = new ArrayList<>();
}
```

- 일대다 단방향은 일대다에서 일(1)이 연관관계의 주인이다.
- 테이블 일대다 관계는 항상 다쪽에 외래키가 있는데 객체와 테이블의 차이 떄문에 반대편 테이블의 외래키를 관리하게 되는 특이한 구조이다.
- @JoinColumn을 사용하지 않으면 조인 테이블 방식을 사용하기 때문에 반드시 사용해야 한다.

**하지만 실무에선 사용하지 말자**

- 엔티티가 관리하는 외래키가 다른 테이블에 있기 떄문에 객체지향적이지 못하고
- 연관관계 관리를 위해 실제로 외래키가 있는 다른 테이블에도 UPDATE SQL이 실행되기 때문에 비효율적이다.
- 따라서 일대다 단방향 매핑보다는 다대일 양방향 매핑을 사용하자.

### 일대다 양방향

- 이런 매핑은 공식적으로 존재하지 않는다.
- @JoinColumn(insertable=false, updatable=false)를 사용해 읽기 전용 필드를 생성하여 마치 양방향인것처럼 사용하는 것이다.
- 하지만 mappedBy를 사용한 다대일 양방향이 훨씬 객체지향적이기 때문에 다대일을 사용하자

## 일대일(1:1)

일대일 관계는 그 반대도 일대일 관계이며 주 테이블이나 대상 테이블 중 외래키를 둘 곳을 선택할 수 있다. 또한 외래 키에 데이터베이스 유니크 제약 조건을 추가한다.

### 주 테이블에 외래키 단방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@OneToOne
	@JoinColumn(name = "locker_id")
	private Locker locker;
}
```

```java
@Entity
public class Locker {
	@Id @GeneratedValue
	@Column(name = "locker_id")
	private Long id;
	
	private String number;
}
```

- 다대일 단방향 매핑과 유사하다.
- 유니크 제약 조건은 따로 DB에서 제작하는 것이 좋다.

### 주 테이블에 외래키 양방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@OneToOne
	@JoinColumn(name = "locker_id")
	private Locker locker;
}
```

```java
@Entity
public class Locker {
	@Id @GeneratedValue
	@Column(name = "locker_id")
	private Long id;
	
	private String number;
	
	@OneToOne(mappedBy="locker")
	private Member member;
}
```

- 다대일 양방향 매핑처럼 외래키가 있는 곳을 연관관계의 주인으로 선정한다.
- 주인이 아닌 반대편은 mappedBy를 적용시켜 조회만 가능하도록 한다.

### 대상 테이블에 외래키 단방향

- 주 테이블이 아닌 대상 테이블에도 외래키를 놓을 수 있지만 단방향은 JPA가 지원하지 않는다.

### 대상 테이블에 외래키 양방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@OneToOne(mappedBy = "locker")
	private Locker locker;
}
```

```java
@Entity
public class Locker {
	@Id @GeneratedValue
	@Column(name = "locker_id")
	private Long id;
	
	private String number;
	
	@OneToOne
	@JoinColumn(name = "member_id")
	private Member member;
}
```

- 주 테이블에 외래키 양방향과 매핑 방법은 같다
- 단순히 외래키를 대상 테이블로 옮긴 것 뿐이다.

### 주 테이블 외래키 VS 대상 테이블 외래키

그렇다면 도대체 외래키를 어디에 두는 것이 좋은 것일까? 각 방법의 특징을 알아보자

**주 테이블 외래키 특징**

- 주 객체가 대상 객체의 참조를 가지는 것처럼 주 테이블에 외래키를 두고 대상 테이블을 찾는다.
- JPA 매핑이 편하기 때문에 객체지향 개발자가 선호한다.
- 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인이 가능하다.
- 하지만 값이 없다면 외래키에 null을 허용하게 된다.

**대상 테이블 외래키 특징**

- 전통적인 데이터베이스 개발자가 선호한다.
- 만약 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경한다면 단순히 @ManyToOne으로 변경해주면 되기 때문에 편리하다.
- 프록시 기능의 한계로 지연 로딩으로 설정하여도 항상 주 테이블이 같이 로딩된다.

→ 따라서 각 방법에 대한 특징을 숙지하고 해당 상황에 맞게 선택을 하면 될 것 같다.

## 다대다(N:M)

관계형 데이터베이스는 정규화된 테이블 2개의 다대다 관계를 표현할 수 없기 때문에 중간 테이블을 추가해서 일대다, 다대일 관계로 변환해야한다.

반면에 객체는 컬렉션을 사용할 수 있기 때문에 다대다 매핑이 가능하다.

### 다대다 단방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@ManyToMany
	@JoinTable(name = "MEMBER_PRODUCT")
	private List<Product> products = new ArrayList<>();
}
```

```java
@Entity
public class Product {

	@Id @GeneratedValue
	@Column(name = "product_id")
	private Long id;
	
	private String name;
}
```

### 다대다 양방향

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@ManyToMany
	@JoinTable(name = "MEMBER_PRODUCT")
	private List<Product> products = new ArrayList<>();
}
```

```java
@Entity
public class Product {

	@Id @GeneratedValue
	@Column(name = "product_id")
	private Long id;
	
	private String name;
	
	@ManyToMany(mappedBy = "products")
	private List<Member> members = new ArrayList<>();
}
```

### 다대다 매핑의 한계

- 편리해 보이지만 실무에서는 사용하면 안된다.
- 연결 테이블이 단순히 두 엔티티를 연결해주는 것뿐 아니라 주문시간, 수량 같은 데이터가 들어올 수 있기 때문이다.
- 따라서 연결 테이블용 엔티티를 하나 추가해준다.

```java
@Entity
public class Order {
	
	@Id @GeneratedValue
	@Column(name = "order_id")
	private Long id;
	
	@ManyToOne
	@JoinColumn(name = "member_id")
	private Member member;
	
	@ManyToOne
	@JoinColumn(name = "product_id")
	private Product product;
	
	private int orderAmount;
	
	private LocalDateTime orderDate;
}
```

**연결 테이블과 엔티티 연결관계 설정**

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	private String name;
	
	@ManyToOne(mappedBy = "order_id")
	private List<Order> orders = new ArrayList<>();
}
```

```java
@Entity
public class Product {

	@Id @GeneratedValue
	@Column(name = "product_id")
	private Long id;
	
	private String name;
	
	@ManyToOne(mappedBy = "order_id")
	private List<Order> orders = new ArrayList<>();
}
```