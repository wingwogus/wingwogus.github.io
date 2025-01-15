---
title: "[JPA] 우리 연관됐어요 - 연관관계 매핑"
excerpt: "엔티티들 사이 연관관계를 설정할 때 주의해야 할 점에 대해 알아보자"

categories:
  - JPA
tags:
  - [jpa, db, entity, relation]

permalink: /jpa/relationship-basic/

toc: true
toc_sticky: true

date: 2025-01-10
last_modified_at: 2025-01-10
---
**본 게시글은 김영한님의 인프런 [자바 ORM 표준 JPA 프로그래밍 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 연관관계가 왜 필요한데?

우선 하나의 시나리오를 예로 들어보자. 회원과 팀이 있고 회원은 한 팀에만 속할 수 있으며 회원과 팀은 다대일 관계이다.

**객체를 테이블에 맞추어 모델링 했을 경우**

```java
@Entity
public class Member {

	@Id @GeneratedValue
	@Column(name = "member_id")
	private Long id;
	
	@Column(name = "USERNAME")
	private String name;
			
	@Column(name = "TEAM_ID")
	private Long teamId;
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

- 위와 같이 teamId, 즉 외래키를 필드로 넣고 모델링 했을 경우 동작과정을 보자

**팀에 멤버를 저장**

```java
//팀 저장
Team team = new Team();
team.setName("TeamA");
em.persist(team);

//회원 저장
member member = new Member();
member.setName("member1");
member.setTeamId(team.getId());
em.persist(member)
```

**멤버가 속한 팀을 찾는 경우**

```java
Member findMember = em.find(Member.class, member.getId());
Team findTeam = em.find(Team.class, findMember.getTeamId());
```

- 테이블은 외래 키로 조인을 해서 연관관계를 찾지만 객체는 참조를 통해 연관관계를 찾는다
- findMember에서 바로 team을 불러오는 것이 아닌 teamId 필드를 불러온 후 다시 EntityManager를 호출해야 한다.
→ 객체지향적이지 않음

### 단방향 연관관계

그럼 이제 객체지향 세계에 맞게 객체지향적으로 코드를 리팩토링 해보자

**Team을 필드로 갖는 Member**

```java
@Entity
public class Member {

	@Id @GeneratedValue
	private Long id;
	
	@Column(name = "USERNAME")
	private String name;
			
	@ManyToOne
	@JoinColumn(name = "team_id")
	private Team team;
}
```

**객체 참조를 사용한 Team 조회**

```java
//팀 저장
Team team = new Team();
team.setName("TeamA");
em.persist(team);
//회원 저장
Member member = new Member();
member.setName("member1");
member.setTeam(team); //단방향 연관관계 설정, 참조 저장

em.persist(member);

//조회
Member findMember = em.find(Member.class, member.getId());
//참조를 사용해서 연관관계 조회
Team findTeam = findMember.getTeam();
```

- 멤버를 찾은 후 소속 팀을 찾기 위해서 다시 EntityManager를 부를 필요가 없어졌다!
- 객체 그래프를 통해 멤버에서 바로 팀으로 이동할 수 있게 되었다.

### 양방향 연관관계

이제 Member에서 Team으로는 조회가 가능하다. 그렇다면 특정 팀에 속한 Member를 조회하기 위헤서는 어떻게 해야할까? Team에서도 Member를 참조하도록 변경하면 된다.

Member를 참조하는 Team 코드

```java
@Entity
public class Team {

 @Id @GeneratedValue
 @Column(name = "team_id")
 private Long id;
 
 private String name;
 
 @OneToMany(mappedBy = "team")
 List<Member> members = new ArrayList<Member>();
}
```

- Memeber LIst를 통해 팀에 속한 멤버를 확인할 수 있게 되었다.
- 그런데 mappedBy가 무엇일까? 이때 중요한 것이 연관관계의 주인이다.

### 연관관계의 주인

**양방향 매핑 규칙**

- 객체의 두 관계중 하나를 연관관계의 주인으로 지정’
- 연관관계의 주인만이 외래 키를 관리(등록, 수정)
- 주인이 아닌쪽은 읽기만 가능
- 주인은 mappedBy 속성 사용X
- 주인이 아니면 mappedBy 속성으로 주인 지정

**주인을 정하는 기준**

- 연관관계의 주인은 반드시 외래 키가 있는 곳으로

### 연관관계 편의 메소드

- 연관관계를 설정하는 중 가장 많이 하는 실수는 한 곳에만 값을 넣는 것이다.

```java
Team team = new Team();
team.setName("TeamA");
 
em.persist(team);
 
Member member = new Member();
member.setName("member1");
 
//역방향(주인이 아닌 방향)만 연관관계 설정
team.getMembers().add(member);
em.persist(member);
```

- 위 코드를 실행하게 되면 member의 팀은 설정되지 않고 null이 들어가게 된다
- 따라서 양방향으로 연관관계를 설정하도록 해야한다.

**연관관계 편의 메소드를 추가한 Member**

```java
@Entity
public class Member {
	
	@Id @GeneratedValue
	private Long id;
	
	@Column(name = "USERNAME")
	private String name;
		
	@ManyToOne
	@JoinColumn(name = "team_id")
	private Team team;
	
	//연관관계 편의 메소드
	public void changeTeam(Team team) {
		this.team = team;
		team.getMembers().add(this);
	}
}
```

- 해당 로직을 설정하면 member의 team만 변경하여도 team에 member가 추가된다.