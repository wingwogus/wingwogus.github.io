---
title: "[Spring] 구현체, 내가 어디까지 해줘야 해? (2)"
excerpt: "스프링 컨테이너, 더욱 강력한 관리자의 등장"

categories:
  - Spring
tags:
  - [spring, OCP, DI]

permalink: /spring/spring-basic-2/

toc: true
toc_sticky: true

date: 2024-12-22
last_modified_at: 2024-12-22
---

**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 더욱 강력한 관리자, 스프링 컨테이너의 등장

현재까지는 단순 자바 코드만으로 구성하였지만 이제 스프링을 적용시켜보자

**스프링 코드로 변경한 AppConfig**

```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService() {
        System.out.println("call AppConfig.memberService");
        return new MemberServiceImpl(memberRepository());
    }
    
    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
}
```

- 설정을 구성한다는 뜻의 애노테이션인 **@Configuration**을 붙여준다
- 또한 각 메서드에 **@Bean**을 붙여 스프링 컨테이너에 스프링 빈으로 등록될 수 있게 한다.

**스프링 컨테이너를 적용한 MemberApp**

```java
public class MemberApp {

    public static void main(String[] args) {
        //AppConfig appConfig = new AppConfig();
        //MemberService memberService = appConfig.memberService();

        ApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
        MemberService memberService = ac.getBean("memberService", MemberService.class);

        Member memberA = new Member(1L, "memberA", Grade.VIP);
        memberService.join(memberA);

        Member findMember = memberService.findMember(1L);
        System.out.println("new member = " + memberA.getName());
        System.out.println("findMember = " + findMember.getName());
    }
}
```

### 스프링 컨테이너

- **ApplicationContext를 스프링 컨테이너**라고 한다
- **@Configuration**이 붙은 AppConfig를 설정 정보로 사용한다
- 여기서 **@Bean**이 붙은 **메서드를 모두 호출하여 반한된 객체를 스프링 컨테이너에 등록**한다
→ 이렇게 등록된 객체를 **스프링 빈**이라고 한다
- 스프링 빈은 **메서드 명을 이름으로 사용**한다(memberService, memberRepository)
- 이제부턴 개발자가 AppConfig를 사용하여 필요한 객체를 직접 조회하지 않고 스프링 컨테이너의 **getBean()** 메소드를 사용하여 스프링 빈을 조회할 수 있다.

### 스프링 컨테이너 생성과정

1. **스프링 컨테이너 생성**
    - **new AnnotationConfigApplicationContext(AppConfig.class)**
    - 스프링 컨테이너를 생성할 땐 구성 정보를 지정해줘야 하는데 AppConfig로 지정해주었다.
2. **스프링 빈 등록**
    - 스프링 컨테이너는 파라미터로 넘어온 설정 클래스 정보를 사용해서 스프링 빈을 등록한다
        
        
        | 빈 이름 | 빈 객체 |
        | --- | --- |
        | memberService | MemberServiceImpl@xxx01 |
        | memberRepository | MemoryMemberRepository@xxx02 |
    - 빈 이름은 메서드 이름을 사용한다
    - 빈 이름을 직접 부여할 수 있다.
        - @Bean(name = “memberService2”)
    - **빈 이름은 항상 다른 이름을 부여해야 하며 같은 이름을 부여할 시 설정에 따라 오류가 발생한다**
3. 스프링 빈 의존관계 설정
    - 설정 정보를 참고해서 의존관계를 주입
    - 생성자를 호출하면서 의존관계 주입도 한 번에 처리된다
    

---

## 스프링 빈, 너 누구야

**스프링 컨테이너에 실제 스프링 빈들이 잘 등록 되었는지 확인해보자**

```java
class ApplicationContextInfoTest {

    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

    @Test
    @DisplayName("모든 빈 출력하기")
    void findAllBean() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            Object bean = ac.getBean(beanDefinitionName);
            System.out.println("name = " + beanDefinitionName + " object = " + bean);
        }

    }

    @Test
    @DisplayName("애플리케이션 빈 출력하기")
    void findApplicationBean() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            BeanDefinition beanDefinition = ac.getBeanDefinition(beanDefinitionName);

            //BeanDefinition.ROLE_APPLICATION 직접 등록한 애플리케이션 빈
            //BeanDefinition.ROLE_INFRASTRUCTURE 스프링이 내부에서 사용하는 빈
            if (beanDefinition.getRole() == BeanDefinition.ROLE_APPLICATION) {
                Object bean = ac.getBean(beanDefinitionName);
                System.out.println("name = " + beanDefinitionName + " object = " + bean);
            }
        }

    }
}
```

- 빈 조회하기
    - **getBeanDefinitionNames():** 스프링에 등록된 모든 빈 정보를 조회하여 String 배열로 반환한다.
    - **getBean(”빈 이름”)**: 빈 이름으로 빈 객체를 조회한다.
- 빈 종류
    - 빈의 종류는 **getRole()**로 구분할 수 있다
    - **ROLE_APPLICATION**: 프로그래머가 정의한 빈
    - **ROLE_INFRASTRUCTURE**: 스프링이 내부에서 사용하는 빈

### 스프링 특정 빈 조회

```java
public class ApplicationContextExtendFindTest {

    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(TestConfig.class);

    @Test
    @DisplayName("부모 타입으로 조회 시 자식이 둘 이상 있으면 중복 오류가 발생")
    void findBeanByParentTypeDuplicate() {
        Assertions.assertThrows(NoUniqueBeanDefinitionException.class,
                () -> ac.getBean(DiscountPolicy.class));
    }

    @Test
    @DisplayName("부모 타입으로 조회 시 자식이 둘 이상 있으면 빈 이름을 지정")
    void findBeanByParentTypeBeanName() {
        DiscountPolicy bean = ac.getBean("rateDiscountPolicy", DiscountPolicy.class);
        assertThat(bean).isInstanceOf(RateDiscountPolicy.class);
    }

    @Test
    @DisplayName("특정 하위 타입으로 조회")
    void findBeanBySubType() {
        RateDiscountPolicy bean = ac.getBean(RateDiscountPolicy.class);
        assertThat(bean).isInstanceOf(RateDiscountPolicy.class);
    }

    @Test
    @DisplayName("부모 타입으로 모두 조회")
    void findAllBeanByParentType() {
        Map<String, DiscountPolicy> beansOfType = ac.getBeansOfType(DiscountPolicy.class);
        assertThat(beansOfType.size()).isEqualTo(2);
        for (String key : beansOfType.keySet()) {
            System.out.println("key = " + key + " value = " + beansOfType.get(key));
        }
    }

    @Test
    @DisplayName("부모 타입으로 모두 조회 - Object")
    void findAllBeanByObjectType() {
        Map<String, Object> beansOfType = ac.getBeansOfType(Object.class);
        for (String key : beansOfType.keySet()) {
            System.out.println("key = " + key + " value = " + beansOfType.get(key));
        }
    }
    @Configuration
    static class TestConfig {

        @Bean
        public DiscountPolicy rateDiscountPolicy() {
            return new RateDiscountPolicy();
        }

        @Bean
        public DiscountPolicy fixDiscountPolicy() {
            return new FixDiscountPolicy();
        }
    }
}
```

- getBean(”타입”)
    - 같은 타입이 둘 이상이면 오류가 발생하므로 이름까지 지정해주자
    - getBeansOfType(”타입”): 해당 타입의 모든 빈 조회하여 Map<”빈 이름”, “빈 객체”> 형식으로 반환
- getBean(”빈 이름”, “타입”)
- 조회 대상 스프링 빈이 없으면 NoSuchBeanDefinitionException 발생

### BeanFactory와 ApplicatioinContext

- BeanFactory
    - 스프링 컨테이너의 최상위 인터페이스
    - 스프링 빈을 관리하고 조회하는 역할을 담당한다
    - getBean() 메소드를 제공한다
    - 지금까지 우리가 사용했던 대부분의 기능은 BeanFactory가 제공한다
- ApplicationContext
    - BeanFactory 기능을 모두 상속 받아서 제공한다
    - 다양한 인터페이스를 상속 받아 부가 기능을 제공한다

**Application의 부가기능**

- MessageSource를 활용한 국제화 기능
    - 한국에서 들어오면 한국어로, 영어권에서 들어오면 영어로 출력
- EnvironmentCapable(환경 변수)
    - 로컬, 개발, 운영 등을 구분해서 처리
- ApplicationEventPublisher
    - 이벤트를 발생시키고 구독하는 모델을 편리하게 지원
- ResourceLoader
    - 파일, 클래스패스, 외부 등에서 리소스를 편하게 조회

### 다양한 설정 형식 지원

- 스프링 컨테이너는 자바 코드, XML, Groovy 등 다양한 형식의 설정 정보를 받아들일 수 있게 되어있다.

**애노테이션 기반 자바 코드 설정 사용**

- @Configuration, @Bean
- AnnotationCponfigApplicationContext 클래스를 사용하면서 자바 코드로 된 설정 정보를 넘기면 된다.

**XML 설정 사용**

- <beans>, <bean> 등 태그 사용
- **컴파일 없이 빈 설정 정보를 변경할 수 있는 장점**이 있다.
- GenericXmlApplicationContext를 사용하여 xml 설정 파일을 넘기면 된다.
- 자세한 설명은 생략한다

---

**스프링은 이렇게나 다양한 설정 형식을 지원하는 데 어떻게 지원하는 것일까?**

## BeanDefinition 너 누군데?

- 실은 스프링 컨테이너는 AppConfig가 자바 코드인지 XML인지 모른다.
- BeanDefinition, 즉 빈 설정 메타 정보가 @Bean, <bean>당 각각 하나씩 메타 정보를 생성한다.
- 스프링 컨테이너는 이 메타 정보를 기반으로 스프링 빈을 생성한다.

**그럼 한 번 BeanDefinition이라는 친구와 더욱 친해지는 시간을 가져보자**

- AnnotationConfigApplicationContext는 AnnotatedBeanDefinitionReader를 사용해서 AppConfig.class를 읽은 후 BeanDefinition을 생성한다.
- 이렇든 새로운 형식의 설정 정보가 추가되면, xxxBeanDefinitionReader를 만들어서 BeanDefinition을 생성하면 된다.
- BeanDefinition을 직접 생성해서 등록할 수도 있지만 실무에선 그럴 일은 없다고 한다

**BeanDefinition 정보**

```java
public class BeanDefinitionTest {

    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
    //GenericXmlApplicationContext ac = new GenericXmlApplicationContext("appconfig.xml");
    @Test
    @DisplayName("빈 설정 메타정보 확인")
    void findApplicationBean() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            BeanDefinition beanDefinition = ac.getBeanDefinition(beanDefinitionName);

            if (beanDefinition.getRole() == BeanDefinition.ROLE_APPLICATION) {
                System.out.println("beanDefinitionName = " + beanDefinitionName +
                        " beanDefinition = " + beanDefinition);
            }
        }
    }
}

//결과
beanDefinitionName = appConfig beanDefinition = Generic bean: class [hello.core.AppConfig$$SpringCGLIB$$0]; scope=singleton; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodNames=null; destroyMethodNames=null
beanDefinitionName = memberService beanDefinition = Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=appConfig; factoryMethodName=memberService; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = orderService beanDefinition = Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=appConfig; factoryMethodName=orderService; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = memberRepository beanDefinition = Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=appConfig; factoryMethodName=memberRepository; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
beanDefinitionName = discountPolicy beanDefinition = Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=appConfig; factoryMethodName=discountPolicy; initMethodNames=null; destroyMethodNames=[(inferred)]; defined in hello.core.AppConfig
```

- BeanClassName: 생성할 빈의 클래스명
- factoryBeanName: 팩토리 역할의 빈을 사용할 경우 이름 ex) appConfig
- factoryMethodName: 빈을 생성할 팩토리 메서드 지정  ex) memberService
- Scope: 싱글톤(싱글톤에 대해선 후에 설명한다)
- lazyInit: 스프링 컨테이너를 생성하 때 빈을 생성하는 것이 아니라, 실제 빈을 사용할 때까지 최대한 생성을 지연 처리할것인지에 대한 여부
- InitMethodName: 빈을 생성하고, 의존관계를 적요한 뒤 호출되는 초기화 메서드 명
- DestoryMethodName: 빈의 생명 주기가 끝나 제거하기 직전에 호출되는 메서드 명
- Constructure arguments, Properties: 의존관계 주입에서 사용