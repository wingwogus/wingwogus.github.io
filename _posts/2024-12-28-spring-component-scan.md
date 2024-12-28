---
title: "[Spring] 누가 스프링 빈 좀 찾아줘 - 컴포넌트 스캔"
excerpt: "어느 세월에 빈을 하나씩 등록하고 있을까, 그런 당신을 위해 준비했다."

categories:
  - Spring
tags:
  - [spring, componentscan]

permalink: /spring/spring-component-scan/

toc: true
toc_sticky: true

date: 2024-12-28
last_modified_at: 2024-12-28
---
**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 개발자는 반복을 싫어해

- 지금까지 스프링 빈을 등록할 때는 @Bean을 사용하거나 <bean>을 통해 직접 정보를 등록했어야만 했다
- 만약 스프링 빈이 수십 개가 된다면 등록 과정도 귀찮고, 그 과정 속에서 누락되는 문제도 발생할 것이다
- 하지만 스프링이 누구인가, 설정 정보가 없더라도 자동으로 스프링 빈을 등록하는 컴포넌트 스캔 기능을 제공해준다

### 컴포넌트 스캔

**컴포넌트 스캔을 사용하는 AutoAppConfig**

```java
@Configuration
@ComponentScan(
        excludeFilters= @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Configuration.class)
)
public class AutoAppConfig {

}
```

- 컴포넌트 스캔을 사용하려면 우선 @ComponentScan을 붙여주면 된다
- 코드를 보면 @Bean으로 등록한 클래스가 단 하나도 없다
- 컴포넌트 스캔을 사용하면 @Configuration 태그도 자동으로 등록하기 때문에 excludeFilter를 사용해서 제외시켰다

### 작동 원리

- 컴포넌트 스캔은 이름 그대로 @Component가 붙은 클래스를 스캔하여 스프링 빈으로 등록한다.
- 근데 AppConfig에 아무 내용이 없다면 어떻게 의존관계를 설정해줄까?

### Autowired

```java
@Component
public class OrderServiceImpl implements OrderService{

    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;

    @Autowired
    public OrderServiceImpl(MemberRepository memberRepository, @MainDiscountPolicy DiscountPolicy discountPolicy) {
        this.memberRepository = memberRepository;
        this.discountPolicy = discountPolicy;
    }

    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        Member member = memberRepository.findById(memberId);
        int discountPrice = discountPolicy.discount(member, itemPrice);

				return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}
```

- AppConfig에서는 @Bean으로 직접 설정 정보를 작성하고 의존 관계 또한 직접 명시했다
- 그 역할을 Autowired가 대신 해준다

### 동작 순서

1. @ComponentScan
    - @Component가 붙은 모든 클래스를 스프링 빈으로 등록한다
    - 이때 스프링 빈의 기본 이름은 클래스명에서 맨 앞글자만 소문자로 변경하여 사용한다
        - 빈 이름을 변경하고 싶다면 @Component(”memberService2”)처럼 변경하면 된다
2. @Autowired 의존관계 자동 주입
    - 생성자에 @Autowired를 붙여주면 스프링 컨테이너가 자동으로 해당 스프링 빈을 찾아 의존관계를 주입한다
    - 이때 타입이 같은 빈을 우선적으로 찾아 주입한다

---

## 탐색 위치와 기본 스캔 대상

### 탐색할 패키지의 시작 위치 지정

```java
@Component(
	basePackages = "hello.core"
)
```

- basePackages: 탐색할 패키지의 시작 위치를 지정한다, 이 패키지를 포함해서 하위 패키지를 모두 탐색한다
    - basePackages = {”hello.core”, “hello.service”)처럼 여러 시작 위치 지정 가능
- basePackageClasses: 지정한 클래스의 패키지를 탐색 시작 위치로 지정
- 만약 지정하지 않으면 @ComponentScan이 붙은 설정 정보 클래스의 패키지가 시작 위치가 된다
- 보통은 패키지 위치를 지정하지 않고 설정 정보 클래스의 위치를 프로젝트 최상단에 두는 것을 추천한다고 한다

### 필터

- includeFilters: 컴포넌트 스캔 대상을 추가로 지정한다
- excludeFilters: 컴포넌트 스캔에서 제외할 대상을 지정한다
- 직접 애노테이션을 만들어 테스트해보자

**컴포넌트 스캔 대상에 추가할 애노테이션**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyIncludeComponent {

}
```

**컴포넌트 스캔 대상에 제외할 애노테이션**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyExcludeComponent {
}
```

**컴포넌트 스캔 대상에 추가할 클래스**

```java
@MyIncludeComponent
public class BeanA {
}
```

**컴포넌트 스캔 대상에 제외할 클래스**

```java
@MyExcludeComponent
public class BeanB {
}
```

**설정 정보와 테스트 코드**

```java
public class ComponentFilterAppConfigTest {

    @Test
    void filterScan() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(ComponentFilterAppConfig.class);

        BeanA beanA = ac.getBean("beanA", BeanA.class);
        assertThat(beanA).isNotNull();

        assertThrows(NoSuchBeanDefinitionException.class, () ->  ac.getBean("beanB", BeanB.class));
    }

    @Configuration
    @ComponentScan(
            includeFilters = @Filter(type = FilterType.ANNOTATION, classes = MyIncludeComponent.class),
            excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = MyExcludeComponent.class)
    )
    static class ComponentFilterAppConfig {

    }
}
```

### 필터 타입 옵션

- ANNOTATION: 기본값, 애노테이션을 인식해서 동작한다
- ASSIGNABLE_TYPE: 지정한 타입과 자식 타입을 인식해서 동작한다
- ASPECTJ: AspectJ 패턴 사용
- REGEX: 정규 표현식
- CUSTOM: TypeFilter라는 인터페이스를 구현해서 처리

---

## **중복 등록과 충돌**

- 만약 컴포넌트 스캔에서 같은 빈 이름이 등록이 될 수 있을까?
- 두 가지 상황을 보자

### 자동 빈 등록 vs 자동 빈 등록

- 만약 컴포넌트 스캔에 의해 같은 이름의 스프링 빈이 등록되면 스프링은 ConflictingBeanDefinitionException 예외를 발생시킨다

### 자동 빈 등록 vs 수동 빈 등록

- 만약 수동 빈 등록과 자동 빈 등록에서 이름이 같을 경우에는 수동 빈 등록이 우선권을 가진다
→ **수동 빈이 자동 빈을 오버라이딩한다**

**수동 빈 등록 시 남는 로그**

```java
Overrding bean definition for bean 'memoryMemberRepository' with a different definition: replacing
```
