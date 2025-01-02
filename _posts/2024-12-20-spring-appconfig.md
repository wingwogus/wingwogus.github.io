---
title: "[Spring] 구현체, 내가 어디까지 해줘야 해? (1) - AppConfig"
excerpt: "AppConfig의 등장"

categories:
  - Spring
tags:
  - [spring, OCP, DI]

permalink: /spring/appconfig/

toc: true
toc_sticky: true

date: 2024-12-20
last_modified_at: 2024-12-20
---

**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 현재 코드들이 맡은 역할

### MemberApp 코드

- 우선 임의의 사이트의 회원가입 기능을 구현한 코드를 보자
    - MemberService: 회원 가입, 조회를 담당하는 인터페이스
    - MemberServiceImpl: 위 인터페이스를 구현한 구현체
    - MemberRepository: 회원 저장소 인터페이스
    - MemoryMemberRepository: 현재 어떤 데이터베이스를 쓸지 정해지지 않아 메모리를 사용

```java
public class MemberServiceImpl implements MemberService{

    private final MemberRepository memberRepository = new MemoryMemberRepository;

    @Override
    public void join(Member member) {
        memberRepository.save(member);
    }

    @Override
    public Member findMember(Long memberId) {
        return memberRepository.findById(memberId);
    }
}
```

```java
public class MemberApp {

    public static void main(String[] args) {
        Memberservice memberService = new MemberServiceImpl();

        Member memberA = new Member(1L, "memberA", Grade.VIP);
        memberService.join(memberA);

        Member findMember = memberService.findMember(1L);
        System.out.println("new member = " + memberA.getName());
        System.out.println("findMember = " + findMember.getName());
    }
}
```

### MemberApp 코드의 문제점

- 현재 코드의 설계상 문제점은 무엇일까?
    - 현재 MemberApp의 의존관계가 인터페이스가 아닌 구현체에까지 의존하고 있다.
    **MemberApp → MemborServiceImpl**
    - MemberServiceImpl 또한 구현체에 의존 중이다.
    **MemberServiceImpl → MemoryMemberRepository**
        
        → **DIP 위반**
        
- 만약 데이터베이스를 정하고 데이터베이스를 바꿔야 한다면 MemberServiceImpl 코드를 변경해야만 한다. → **OCP 위반**
- 즉, 자신의 업무도 해야 하는데 다른 코드들의 총괄 업무까지 맡아야 하는 상황이다.
- 따라서 DIP를 위반하지 않도록 인터페이스에만 의존하도록 의존관계를 변경해야 한다.

### AppConfig, 관리자의 등장

- 현재 구현체들은 자신의 역할에만 집중해야 하기 때문에 대신 구현체들을 생성하고 연결하는 무언가가 필요하다
- 이때 필요한 것이 AppConfig이다

```java
public class AppConfig {

    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }

    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
}
```

**생성자를 주입한 MemberServiceImpl**

```java
public class MemberServiceImpl implements MemberService{

    private final MemberRepository memberRepository;

    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    public void join(Member member) {
        memberRepository.save(member);
    }
    
    public Member findMember(Long memberId) {
        return memberRepository.findById(memberId);
    }
}
```

- MemberServiceImpl에 생성자를 주입함으로써 MemberRepository 인터페이스에만 의존한다
- **어떤 구현체가 들어올지는 AppConfig가 결정한다**

**변경된 MemberApp**

```java
public class MemberApp {

    public static void main(String[] args) {
        AppConfig appConfig = new AppConfig();
        MemberService memberService = appConfig.memberService();

        Member memberA = new Member(1L, "memberA", Grade.VIP);
        memberService.join(memberA);

        Member findMember = memberService.findMember(1L);
        System.out.println("new member = " + memberA.getName());
        System.out.println("findMember = " + findMember.getName());
    }
}

```

- 객체의 생성과 연결은 AppConfig가 담당
- DIP 완성: 클래스들이 각 추상에만 의존한다
- 역할 분리: 객체를 생성하고 연결하는 역할과 실행하는 역할이 분리되었다

### AppConfig로 인한 효과

- 객체의 생성과 연결은 AppConfig가 담당
- 애플리케이션이 크게 사용 영역, 객체를 생성하고 구성하는 영역으로 분리되었다
- 이제 저장소를 변경해도 구성 역할을 담당하는 AppConfig만 변경하면 된다
→ 사용 영역의 어떤 코드도 변경할 필요가 없다.

---

## 좋은 객체 지향 설계의 5가지 원칙(SOLID)

**우리는 AppConfig로 코드들에게 역할을 분담하여 5가지 원칙 중 3가지를 만족했다**

### SRP(Single Responsibility Principle): 단일 책임 원칙

**한 클래스는 하나의 책임만 가져야 한다**

- 해당 원칙을 따라 코드들의 역할을 분리
- 구현 객체를 생성하고 연결하는 책임(역할)은 구성 영역의 AppConfig가 담당
- 클라이언트 객체는 실행하는 책임(역할)만 담당

### DIP(Dependenct Inversion Principle): 의존 역전 원칙

**구현체에 의존하지 말고 추상화에 의존해야 한다**

- 클라이언트 코드가 MemberRepository에만 의존하도록 코드 변경
- AppConfig가 객체 인스턴스를 클라이언트 코드 대신 생성하여 클라이언트 코드에 의존관계를 주입했다

### OCP(Open-Closed Principle): 개방 폐쇠 원칙

**소프트웨어 요소는 확장에는 열려있으나 변경에는 닫혀 있어야 한다**

- 애플리케이션을 사용 영역과 구성 영역으로 나눔
- AppConfig가 의존관계를 클라이언트에 주입을 해주므로 클라이언트 코드는 변경하지 않아도 된다

---

## IoC, DI, 컨테이너

### IoC(Inversion of Control: 제어의 역전)

- 기존 프로그램은 클라이언트 구현 객체가 스스로 생성, 연결, 실행 등 **모든 제어 흐름을 스스로 조종**했다
- 하지만 AppConfig가 등장한 이후에는 구현 객체들은 **자신의 로직을 실행하는 역할만 담당(관심사의 분리)**한다
- 프로그램에 대한 제어 흐름 권한은 모두 AppConfig가 가지고 있다
- 이렇듯 **프로그램의 제어 흐름을 구현체가 직접 제어하는 것이 아니라 외부에서 관리하는 것**을 IoC라고 한다

### 프레임워크와 라이브러리

**이쯤에서 흔히들 헷갈리는 프레임워크와 라이브러리가 무엇이 다른 건지 살펴보자**

- 프레임워크: 원하는 기능 구현에 집중하여 개발할 수 있도록 **일정한 형태와 기능을 갖추고 있는 골격, 뼈대**를 의미한다
    - 애플리케이션 개발 시 필수적인 코드, 알고리즘, DB연동과 같은 기능을 위해 어느 정도 구조(뼈대)를 제공하고, 이러한 구조 속에서 코드를 작성하여 애플리케이션을 개발한다
    - → **제어의 주도권이 내가 작성한 코드가 아닌 프레임워크에게 있다(IoC)**
- 라이브러리: 단순히 활용 가능한 도구들(클래스, 메소드)의 집합이다
    - → **내가 작성한 코드가 직접 제어의 흐름을 담당한다**

### DI(Dependency Injection: 의존관계 주입)

- 애플리케이션 **실행 시점(런타임)에 외부에서 실제 구현 객체를 생성**하고 **클라이언트에 전달해서 클라이언트와 서버의 실제 의존관계가 연결되는 것을 DI**라고 한다
- 객체 인스턴스를 생성하고 그 **참조값을 전달해서 연결**한다
- DI를 사용하면 정적인 클래스 의존관계를 변경하지 않고(코드를 변경하지 않고), 동적인 객체 인스턴스 의존관계를 쉽게 변경할 수 있다.(AppConfig만 변경)

### IoC 컨테이너, DI 컨테이너

- AppConfig와 같이 객체를 생성하고 연결해주는 것을 IoC 컨테이너 혹은 DI 컨테이너라 칭한다.
- Assembler, Object Factory 등으로 불리기도 한다
