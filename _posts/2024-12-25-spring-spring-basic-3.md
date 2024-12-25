---
title: "[Spring] 혼자야? 어, 나 싱글톤이야 - 싱글톤 패턴"
excerpt: "하나의 빈을 여러 사용자가 동시에 요청하면 어떻게 될까?"

categories:
  - Spring
tags:
  - [spring, singleton]

permalink: /spring/spring-singleton/

toc: true
toc_sticky: true

date: 2024-12-25
last_modified_at: 2024-12-25
---

**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 싱글톤, 왜 필요할까

- 애초에 스프링은 온라인 서비스 기술을 지원하기 위해 만들어졌다
- 웹 애플리케이션은 보통 여러 고객이 동시에 요청을 한다

**스프링 없는 순수한 DI 컨테이너**

```java
@Test
    @DisplayName("스프링 없는 순수한 DI 컨테이너")
    void pureContainer() {
        AppConfig appConfig = new AppConfig();

        //1. 조회ㅣ 호출할 떄마다 객체를 생성
        MemberService memberService1 = appConfig.memberService();

        //2. 조회ㅣ 호출할 떄마다 객체를 생성
        MemberService memberService2 = appConfig.memberService();

        //참조값이 다른 것을 확인
        System.out.println("memberService1 = " + memberService1);
        System.out.println("memberService2 = " + memberService2);

        assertThat(memberService1).isNotSameAs(memberService2);
    }
```

- 예전에 만들었던 AppConfig는 클라이언트가 요청을 할 때마다 객체를 새로 생성한다
→ 트래픽이 초당 100이 나오면 초당 100개 객체가 생성되고 소멸된다
→ 메모리 낭비가 심하다
- 그럼 해당 객체가 컨테이너에 딱 1개만 생성되고 그것을 공유하도록 하면 안될까?

### 싱글톤 패턴

- 클래스의 인스턴스가 딱 1개만 생성되는 것을 보장하는 디자인 패턴이다
- 따라서 private 생성자를 사용해서 외부에서 객체 생성을 막아 2개 이상 생성하지 못하도록 한다

**싱글톤 패턴을 적용한 코드**

```java
public class SingletonService {

    private static final SingletonService instance = new SingletonService();

    public static SingletonService getInstance() {
        return instance;
    }

    private SingletonService() {
    }

    public void logic() {
        System.out.println("싱글톤 객체 로직 호출");
    }
}
```

- static 영역에 객체 instance를 미리 하나 생성해서 올려둔다
- 이 객체 인스턴스는 오직 getInstance() 메소드를 통해서만 조회할 수 있다
- 생성자를 private로 막아 외부 객체 생성을 막는다

**싱글톤 패턴을 사용한 테스트 코드**

```java
	@Test
	@DisplayName("싱글톤 패턴을 적용한 객체 사용")
	void singletonServiceTest() {
				//1. 조회: 호출할 때 마다 같은 객체를 반환
				SingletonService instance1 = SingletonService.getInstance();
        //2. 조회: 호출할 때 마다 같은 객체를 반환
        SingletonService instance2 = SingletonService.getInstance();

        //같은 참조값 확인
        System.out.println("instance1 = " + instance1);
        System.out.println("instance2 = " + instance2);

        //singletonService1 == singletonService2
        assertThat(instance1).isSameAs(instance2);
    }
```

### 싱글톤 패턴의 단점

- 코드의 양이 많아진다
- 클라이언트가 구체 클래스에 의존한다 → DIP 위반
    
    → OCP 위반 확률 높음
    
- 테스트가 어려워진다
- 내부 속성을 변경하거나 클래스를 만들기 어려워진다
- 안티패턴으로도 불린다

---

## 싱글톤 컨테이너

- 근데 생각해보면 우리가 지금까지 스프링으로 테스트를 할 때는 객체가 1개만 생성되었었다
    
    → **스프링 빈이 바로 싱글톤으로 관리되기 때문이다**
    

### 스프링 컨테이너의 똑똑함

- 스프링 컨테이너는 싱글턴 패턴을 적용하지 않아도 **객체 인스턴스를 싱글톤으로 관리**한다.
- 스프링 컨테이너는 싱글톤 컨테이너 역할을 하는데 이렇게 싱글톤 객체를 생성하고 관리하느 기능을 **싱글톤 레지스트리**라고 한다
- 이런 기능 덕분에 싱글턴 패턴의 모든 단점을 해결하면서 객체를 싱글톤으로 유지할 수 있다

**스프링 컨테이너를 사용하는 테스트 코드**

```java
@Test
    @DisplayName("싱글톤 패턴을 적용한 객체 사용")
    void singletonServiceTest() {
        //1. 조회: 호출할 때 마다 같은 객체를 반환
        SingletonService instance1 = SingletonService.getInstance();
        //2. 조회: 호출할 때 마다 같은 객체를 반환
        SingletonService instance2 = SingletonService.getInstance();

        //같은 참조값 확인
        System.out.println("instance1 = " + instance1);
        System.out.println("instance2 = " + instance2);

        //singletonService1 == singletonService2
        assertThat(instance1).isSameAs(instance2);
    }
```

## 싱글톤 방식의 주의점

- 싱글톤 방식은 여러 클라이언트가 하나의 인스턴스를 공유하기 때문에 객체는 상태를 stateful하게 설계하면 안된다
- Stateless(무상태) 설계
    - 특정 클라이언트에 의존적인 필드가 있으면 안된다
    - 특정 클라이언트가 값을 변경할 수 있는 필드가 있으면 안된다
    - 가급적 읽기만 가능해야 한다
    - 필드 대신에 자바에서 공유되지 않는, 지역변수, 파라미터, ThreadLocal 등을 사용해야 한다

**stateful 설계의 문제**

```java
public class StatefulService {

    private int price;

    public void order(String name, int price) {
        System.out.println("name = " + name + " price = " + price);
        this.price = price;//여기가 문제
    }
    
    public int getPrice() {
    return price;
}
```

```java
		@Test
    void statefulServiceSingleton() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);
        StatefulService statefulService1 = ac.getBean(StatefulService.class);
        StatefulService statefulService2 = ac.getBean(StatefulService.class);

        //ThreadA: A 사용자 10000원 주문
        statefulService1.order("userA", 10000);
        //ThreadB: B 사용자 20000원 주문
        statefulService2.order("userB", 20000);

        //ThreadA: 사용자A 주문 금액 조회
        //int price = statefulService1.getPrice();
        System.out.println("price = " + price);

        assertThat(price).isEqualTo(20000);
    }

    @Configuration
    static class TestConfig {

        @Bean
        public StatefulService statefulService() {
            return new StatefulService();
        }
    }
```

- price 필드가 공유되고 있기 때문에 순서를 보장하지 않는 멀티 스레드 환경에서는 매우 위험하다
- 따라서 객체는 무상태로 설계해야 한다

**stateless 설계로 변경**

```java
public class StatefulService {

    //private int price;

    public int order(String name, int price) {
        System.out.println("name = " + name + " price = " + price);
        return price;
    }
}
```

```java
@Test
    void statefulServiceSingleton() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);
        StatefulService statefulService1 = ac.getBean(StatefulService.class);
        StatefulService statefulService2 = ac.getBean(StatefulService.class);

        //ThreadA: A 사용자 10000원 주문
        int userAPrice = statefulService1.order("userA", 10000);
        //ThreadB: B 사용자 20000원 주문
        int userBPrice = statefulService2.order("userB", 20000);

        //ThreadA: 사용자A 주문 금액 조회
        //int price = statefulService1.getPrice();
        System.out.println("price = " + userAPrice);

        assertThat(userAPrice).isEqualTo(10000);
    }

    @Configuration
    static class TestConfig {

        @Bean
        public StatefulService statefulService() {
            return new StatefulService();
        }
    }
```

---

## @Configuraton 넌 뭐야

- 근데 AppConfig를 보면 하나의 의문이 떠오른다

```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }
    
    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }
}
```

- memberService()와 orderService()를 보면 각각 memberRepository()를 호출한다
- 그렇다면 각각 다른 MemoryMemberRepository가 생성되야 하지 않을까? 한 번 테스트 코드를 만들어보자

**테스트 코드**

```java
 @Test
    void configurationTest() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

        MemberServiceImpl memberService = ac.getBean(MemberServiceImpl.class);
        OrderServiceImpl orderService = ac.getBean(OrderServiceImpl.class);
        MemberRepository memberRepository = ac.getBean(MemberRepository.class);

        MemberRepository memberRepository1 = memberService.getMemberRepository();
        MemberRepository memberRepository2 = orderService.getMemberRepository();

        System.out.println("memberService -> memberRepository1 = " + memberRepository1);
        System.out.println("orderService -> memberRepository2 = " + memberRepository2);
        System.out.println("memberRepository = " + memberRepository);

        assertThat(memberRepository1).isSameAs(memberRepository);
        assertThat(memberRepository2).isSameAs(memberRepository);
    }

```

- 해당 테스트를 돌려보면 예상과는 다르게 성공한다 → 같은 객체를 공유하고 있다는 것이다
- 도대체 스프링 컨테이너는 어떻게 싱글톤으로 관리를 하는 것일까?

### 스프링 컨테이너의 바이트 조작

- 스프링 컨테이너는 싱글톤 레지스트리이기 때문에 스프링 빈이 싱글톤이 되도록 설계해야 한다
- 그래서 스프링은 클래스의 바이트 코드를 조작하여 싱글톤 설계를 유지시킨다. 한 번 코드를 보자

```java
@Test
    void configurationDeep() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
        AppConfig bean = ac.getBean(AppConfig.class);
        System.out.println("bean = " + bean.getClass());

        MemberServiceImpl memberService = ac.getBean("memberService", MemberServiceImpl.class);
        System.out.println("memberService = " + memberService.getClass());

    }
```

- AnnotationConfigApplicationContext에 파라미터로 넘긴 값은 스프링 빈으로 등록된다
- 그럼 한 번 AppConfig 스프링 빈을 조회한 결과를 보자

```java
bean = class hello.core.AppConfig$$EnhancerBySpringCGLIB$$xxxxx
```

- 우리의 예상과는 다른 결과가 나왔다
- 순수한 클래스라면 class hello.core.AppConfig로 출력이 되어야 한다
- 이것은 내가 만든 클래스를 스프링이 바이트 조작 라이브러인 CGLIB를 사용해서 AppConfig를 상속받은 임의의 다른 클래스를 만들고 스프링 빈으로 등록한 것이다.

**AppConfig@CGLIB 예상 코드**

```java
@Bean
public MemberRepository memberRepository() {

	if (memoryMemberRepository가 이미 컨테이너에 있으면?) {
		return 스프링 빈에서 찾아서 반환;
	} else {
		기존 로직 호출하여 스프링 빈으로 등록
		return 반환
	}
}
```

- 바로 이 기술 덕분에 스프링 컨테이너가 싱글톤을 보장하는 것이다