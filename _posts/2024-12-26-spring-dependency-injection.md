---
title: "[Spring] 롬복의 은혜 끝이 없네 - 의존관계 주입"
excerpt: "빈들은 어떻게 서로 의존관계를 설정하는 걸까?"

categories:
  - Spring
tags:
  - [spring, di]

permalink: /spring/dependency-injection/

toc: true
toc_sticky: true

date: 2024-12-26
last_modified_at: 2024-12-26
---
**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 4가지의 의존관계 주입 방법

### 생성자 주입

- 생성자를 통해서 의존 관계를 주입 받는 방법이다.
- 이전의 모든 코드들이 바로 생성자 주입 방법을 사용했다.
- 특징
    - 생성자 호출 시점에 1번만 호출되는 것이 보장된다.
    - **불변, 필수 의존관계에 사용한다 → final 상수 사용**

```java
@Component
public class OrderServiceImpl implements OrderService{

    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;

    @Autowired
    public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
        this.memberRepository = memberRepository;
        this.discountPolicy = discountPolicy;
    }
}
```

- 이때 생성자가 딱 1개만 있다면 @Autowired를 생략해도 된다. (스프링 빈에만 해당)

### 수정자 주입 - setter 주입

- 수정자 메서드를 사용해서 의존관계를 주입하는 방법이다.
- 특징
    - 선택, 변경 가능성이 있는 의존관계에 사용
    - 자바빈 프로퍼티 규약의 수정자 메서드 방식을 사용하는 방법이다.

```java
@Component
public class OrderServiceImpl implements OrderService{

    private MemberRepository memberRepository;
    private DiscountPolicy discountPolicy;

		@Autowired
    public void setMemberRepository(MemberRepository memberRepository) {
	    this.memberRepository = memberRepository;
	  }
	  
	  @Autowired
	  public void setDiscountPolicy(DiscountPolicy discountPolicy) {
	    this.discountPolicy = discountPolicy;
	  }
}
```

- 참고로 @Autowired는 주입할 대상이 컨테이너에 없다면 오류가 발생한다.
- 주입할 대상이 없어도 작동하도록 하려면 @Autowired(required = false)로 지정해주면 된다.

### 필드 주입

- 이름 그대로 필드에 직접 주입하는 방식이다.
- 특징
    - 코드가 간결해지지만 외부에서 변경이 불가능해지기 때문에 테스트가 힘들다는 단점이 존재한다.
    - DI 프레임워크가 없는 그냥 자바 환경에선 아무것도 할 수 없어진다.
    - 위와 같은 이유들로 사용하지 않는다 → 롬복을 쓰는게 더욱 간편하다

```java
@Component
public class OrderServiceImpl implements OrderService{
		@Autowired
    private MemberRepository memberRepository;
    @Autowired
    private DiscountPolicy discountPolicy;
}
```

### 일반 메서드 주입

- 일반 메서드를 통해 주입받을 수 있다
- 특징
    - 한 번에 여러 필드를 주입 가능하다.
    - 필드 주입과 마찬가지로 일반적으로 사용하지 않는다.

```java
@Component
public class OrderServiceImpl implements OrderService{

    private MemberRepository memberRepository;
    private DiscountPolicy discountPolicy;

		@Autowired
    public void init(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
	    this.memberRepository = memberRepository;
	    this.discountPolicy = discountPolicy;
	  }
}
```

---

## 옵션 처리

만약 스프링 빈이 없어도 동작해야 할 때가 있다면 어떡해야 할까?

@Autowired만 사용한다면 required의 기본값은 true이기 때문에 주입 대상이 없다면 오류가 발생한다

자동 주입 대상을 옵션으로 처리해보자

- @Autowired(required = true): 위에서 봤듯이 자동 주입할 대상이 없다면 메서드 자체가 호출이 되지 않게 한다.
- org.springframework.lang.@Nullable: 자동 주입할 대상이 없다면 null이 입력된다.
- Optional<>: 자동 주입할 대상이 없으면 Optional.empty가 입력된다.

**옵션 처리 예제**

```java
public class AutoWiredTest {

    @Test
    void autoWiredOption() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(TestBean.class);
    }

    static class TestBean {

        @Autowired(required = false)
        public void setNoBean1(Member noBean1) {
            System.out.println("noBean1 = " + noBean1);
        }

        @Autowired
        public void setNoBean2(@Nullable Member noBean2) {
            System.out.println("noBean2 = " + noBean2);
        }

        @Autowired
        public void setNoBean3(Optional<Member> noBean3) {
            System.out.println("noBean3 = " + noBean3);
        }
    }
}
```

**출력 결과**

```java
//setNoBean1()은 호출 자체가 되지 않는다.
setNoBean2 = null
setNoBean3 = Optional.empty
```

- @Nullable과 Optional은 스프링이 지원해주는 것이기 때문에 생성자 자동 주입에서 특정 필드에만 사용해도 된다.

---

## 그래서 어떤 거 쓰라고?

과거에는 수정자 주입과 필드 주입을 많이 사용했지만 결론적으론 생성자 주입을 사용하라고 한다! 

그 이유를 알아보자

### 불변

- 대부분의 의존관계 주입은 한 번 일어나면 애플리케이션 종료 시점까지 변경할 일이 없다.
- 수정자 주입을 사용하면 setXXX 메서드를 public으로 열어두어야 한다. → 누군가 실수할 수 있다.
- 생성자 주입은 객체를 생성할 때 딱 1번만 실행되기 때문에 불변하게 설계가 가능하다.

### 누락

- 프레임워크 없이 순수한 자바 코드를 단위 테스트 하는 경우에 다음과 같이 수정자 의존관계인 경우
- 생성자 주입을 사용하면 주입 데이터를 누락했을 때 컴파일 오류가 발생한다.
→ **의존관계 누락을 사전에 방지할 수 있다.**

### final 키워드

- 생성자 주입을 사용하면 필드에 final 키워드를 사용할 수 있다.
- 혹시 생성자에서 값이 설정되지 않는다면 컴파일 오류를 발생시킨다.
    - java: variable ~~~ might not have been initialized

### 정리하자면

- 생성자 주입 방식을 선택하는 것이 프레임워크에 의존하지 않고 순수한 자바 언어의 특징을 잘 살릴 수 있다.
- 기본적으로 생성자 주입을 사용하고, 필수 값이 아닌 경우에는 수정자 주입 방식을 사용하면 된다.
- 필드 주입은 사용하지 않는 것이 좋다.

---

## 찬양하라 롬복

개발자들은 귀찮은 걸 참지 못한다. 위에 방식대로 해야 한다면 생성자도 만들고, 주입 받은 값을 대입도 시켜야하고….

그래서 준비했다, 롬복!

**롬복을 적용시킨 생성자 주입 코드**

```java
@Component
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService{

    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;
}
```

- 롬복 라이브러리가 제공하는 @RequiredArgsConstructor를 사용하면 final이 붙은 필드를 모아 생성자를 자동으로 만들어준다.
- 현재 코드에서 보이진 않지만 실제 class 파일을 보면 생성자가 추가되어 있다.

---

## 의존 관계 자동 주입 주의점

의존 관계 자동 주입 시점에서 여러가지 문제가 발생할 수 있다. 하나씩 알아보자

### 조회 빈이 2개 이상

- @Autowired는 타입으로 조회하기 때문에 마치 다음 코드와 유사하게 작동한다.
→ ac.getBean(타입)
- 타입으로 조회 시 빈이 2개 이상일 때는 문제가 발생한다.
- 해결 방법을 하나씩 알아보자

### @Autowired 필드 명 매칭

- 타입 매칭을 시도하고 여러 빈이 있으면 필드 이름, 파라미터 이름으로 빈 이름을 추가 매칭한다.

**필드 명을 빈 이름으로 변경**

```java
@Autowired
private DiscountPolicy rateDiscountPolicy
```

- 필드 명이 구현체 빈 이름이기 때문에 정상 주입 된다.

**@Autowired 매칭 순서**

1. 타입 매칭
2. 타입 매칭의 결과가 2개 이상일 때 필드 명, 파라미터 명으로 빈 이름 매칭

### @Qualifier 사용

- 추가 구분자를 붙여주는 방법이다.

**빈 등록 시 @Qualifier를 붙여준다.**

```java
@Component
@Qualifier("mainDiscountPolicy")
public class RateDiscountPolicy implements DiscountPolicy {}

@Component
@Qualifier("fixDiscountPolicy")
public class FixDiscountPolicy implements DiscountPolicy {}
```

**주입 시 @Qualifier를 붙여주고 등록한 이름을 적어준다.**

```java
//생성자 자동 주입 예시
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, @Qualifier("mainDiscountPolicy") DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}

//수정자 자동 주입 예시
@Autowired
public void setDiscountPolicy(@Qualifier("mainDiscountPolicy") DiscountPolicy discountPolicy) {
	this.discountPolicy = discountPolicy;
}
```

- 만약 @Qualifier가 @Qualifier("mainDiscountPolicy")를 찾지 못한다면 mainDiscountPolicy라는 이름의 스프링 빈을 추가로 찾는다.
- 하지만 @Qualifier는 @Qualifier를 찾는 용도로만 사용하자.

**@Qualifier 매칭 순서**

1. @Qualifier끼리 매칭
2. 빈 이름 매칭
3. NoSuchBeanDefinitionException 예외 발생

### @Primary 사용

- 우선 순위를 정하는 방법으로 여러 빈이 매칭되면 @Primary가 우선권을 가진다.

**rateDiscountPrimary 우선**

```java
@Component
@Primary
public class RateDiscountPolicy implements DiscountPolicy {}

@Component
public class FixDiscountPolicy implements DiscountPolicy {}
```

**그렇다면 @Qualifier와 @Primary 중 어떤 것을 사용해야 할까?**

- @Qualifier는 주입 받을 때 모든 코드에 @Qualifier를 붙여줘야 한다는 단점이 존재한다.
- 만약 자주 사용하는 메인 데이터베이스의 커넥션을 획득하는 스프링 빈이 있고 특정 기능에만 사용하는 서브 데이터베이스의 커넥션을 획득하는 스프링 빈이 있다고 해보자
- 메인 데이터베이스의 커넥션을 획득하는 스프링 빈에는 @Primary를 붙여 편리하게 조회하고 특정 기능을 사용할 때는 @Qualifier를 지정해 명시적으로 획득하면 코드를 깔끔하게 사용할 수 있다.

---

## 애노테이션 직접 만들기

@Qualifier(”mainDiscountPolicy”)를 사용하면 컴파일시 타입 체크가 되지 않는다. 

애노테이션을 직접 만든다면 이러한 문제를 해결할 수 있다.

**MainDiscountPolicy 애노테이션 사용**

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Qualifier("mainDiscountPolicy")
public @interface MainDiscountPolicy {
}
```

```java
//생성자 자동 주입 예시
@Autowired
public OrderServiceImpl(MemberRepository memberRepository, @MainDiscountPolicy DiscountPolicy discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}

//수정자 자동 주입 예시
@Autowired
public void setDiscountPolicy(@MainDiscountPolicy DiscountPolicy discountPolicy) {
	this.discountPolicy = discountPolicy;
}
```

- 애노테이션에는 상속이란 개념이 없다.
- 위 코드와 같이 여러 애노테이션을 모아 사용하는 기능은 스프링이 지원해주는 기능이다.
- 무분별하게 사용하지 않도록 주의하자

---

## **해당 타입의 빈이 모두 필요할 때**

- 예를 들어 클라이언트가 할인의 종류를 선택할 수 있다고 가정했을 때를 생각하고 코드를 구현해보자

```java
public class AllBeanTest {

    @Test
    void findAllBean() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AutoAppConfig.class, DiscountService.class);

        DiscountService discountService = ac.getBean(DiscountService.class);
        Member member = new Member(1L, "userA", Grade.VIP);
        int discountPrice = discountService.discount(member, 10000, "fixDiscountPolicy");

        assertThat(discountService).isInstanceOf(DiscountService.class);
        assertThat(discountPrice).isEqualTo(1000);
    }

    static class DiscountService {
        private final Map<String, DiscountPolicy> policyMap;
        private final List<DiscountPolicy> policies;

        public DiscountService(Map<String, DiscountPolicy> policyMap, List<DiscountPolicy> policies) {
            this.policyMap = policyMap;
            this.policies = policies;
            System.out.println("policyMap = " + policyMap);
            System.out.println("policies = " + policies);
        }

        public int discount(Member member, int price, String discountCode) {
            DiscountPolicy discountPolicy = policyMap.get(discountCode);
            return discountPolicy.discount(member, price);
        }
    }
}
```

- DiscountService는 모든 DiscountPolicy를 Map으로 주입받는다.
- discount 메서드에서는 discountCode의 값에 따라 어떤 스프링 빈을 찾아 실행할지 결정한다.
- Map<String, DiscountPolicy>: map의 키에 스프링 빈의 이름을 값으로는 DiscountPolicy 타입으로 조회한 모든 스프링 빈을 담는다.
- List<DiscountPolicy>: DiscountPolicy 타입으로 조회한 모든 스프링 빈을 담는다. 만약 해당하는 타입의 스프링 빈이 없다면 빈 컬렉션이나 Map을 주입한다.

---

## 자동, 수동 어떤 걸 선택해야 할까

- 결론부터 얘기하자면 점점 자동을 선호하는 추세이다.
- 스프링은 @Component 뿐만 아니라 @Controller, @Service, @Repository처럼 계층에 맞추어 일반적인 애플리케이션 로직을 자동으로 스캔할 수 있도록 지원한다.
- 수동 등록은 과정이 귀찮기도 하고 자동 등록 만으로도 OCP, DIP를 지킬 수 있기 때문이다.

### 그렇다면 수동 빈은 언제 사용할까

- 업무 로직 빈
    - 웹을 지원하는 컨트롤러, 서비스, 리포지토리 등이 업무 로직이다. 비즈니스 요구사항을 개발할 때 추가되거나 변경된다.
    - 업무 로직은 숫자도 매우 많고 한 번 개발해야 하면 어느정도 유사한 패턴이 있다.
    - 문제가 발생해도 어떤 곳에서 발생했는지 명확하게 파악이 쉽기 때문에 자동 기능을 적극 사용하는 것이 좋다.
- 기술 지원 빈
    - 기술적인 문제나 공통 관심사(AOP)를 처리할 때 주로 사용된다. 데이터베이스 연결이나 공통 로그 처리처럼 업무 로직을 지원하기 위한 하부 기술이나 공통 기술들이다.
    - 업무 로직과 비교하여 수가 매우 적고 애플리케이션 전반에 걸쳐 광범위하게 영향을 미친다.
    - 업무 로직과는 다르게 적용이 잘 되고 있는지 아닌지 조차 파악하기 어렵기 때문에 가급적 수동 빈 등록을 사용하는 것이 좋다.
