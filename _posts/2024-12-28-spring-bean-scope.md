---
title: "[Spring] 빈이 언제 죽는다고 생각하나? - 빈 스코프"
excerpt: "그렇다면 빈은 언제 생성되고 언제 소멸되는 것일까? 내 마음대로 설정할 수 있을까?"

categories:
  - Spring
tags:
  - [spring, bean, scope, webscope]

permalink: /spring/bean-scope/

toc: true
toc_sticky: true

date: 2024-12-28
last_modified_at: 2024-12-28
---
**본 게시글은 김영한님의 인프런 [스프링 핵심 원리 - 기본편]을 바탕으로 한 공부 내용을 작성한 글입니다.**

## 빈 스코프가 뭔데?

지금까지 우리는 스프링 빈을 스프링 컨테이너의 시작과 함께 생성되고 종료될 때까지 유지시켰다. 스프링 빈이 기본적으로 싱글톤 스코프로 생성되기 때문이다.

스코프란 말 그대로 **빈이 존재할 수 있는 범위**를 뜻한다.

### 현재까지 배운 스코프 종류

- 싱글톤: 기본 스코프, 스프링 컨테이너의 시작과 종료까지 유지되는 가장 넓은 범위의 스코프이다.
- 프로토타입: 생성과 의존관계 주입까지만 스프링 컨테이너가 관여하고 그 이상은 클라이언트에게 맡기는 매우 짧은 범위의 스코프이다.

### 컴포넌트 스캔 자동 등록

빈 스코프는 다음과 같이 @Scope를 사용해 지정할 수 있다.

```java
@Scope("prototype")
    static class PrototypeBean {
        @PostConstruct
        public void init() {
            System.out.println("PrototypeBean.init");
        }

        @PreDestroy
        public void destroy() {
            System.out.println("PrototypeBean.destroy");
        }
    }
```

---

## 넌 싱글톤이고, 난 프로토타입이야!

싱글톤 빈의 요청과 프로토타입 빈 요청의 차이점을 알아보자

### 싱글톤 빈 요청

1. 싱글톤 스코프의 빈을 스프링 컨테이너에 요청한다.
2. 스프링 컨테이너에서 관리하는 스프링 빈을 반환한다.
3. 이후에 스프링 컨테이너에 같은 요청이 와도 같은 객체 인스턴스의 스프링 빈을 반환한다.

### 프로토타입 빈 요청

1. 프로토타입 스코프의 빈을 스프링 컨테이너에 요청한다.
2. 스프링 컨테이너는 요청 시점에 프로토타입 빈을 생성하고, 필요한 의존관계를 주입한다.
3. 스프링 컨테이너는 생성한 프로토타입 빈을 클라이언트에 반환한다.
4. 이후에 스프링 컨테이너에 같은 요청이 오면 항상 새로운 프로토타입 빈을 생성해서 반환한다.

### 핵심 요약

- 스프링 컨테이너는 프로토타입 빈을 생성 → 의존관계 주입 → 초기화까지만 처리하고 클라이언트에 빈을 반환한 후 더 이상 관리하지 않는다.
- 이후에 프로토타입 빈을 관리할 책임은 클라이언트에 있기 때문에 @PreDestroy 같은 종료 메서드가 호출되지 않는다.

---

## 프로토타입, 싱글톤 빈 함께 사용 시 문제점

프로토타입 빈을 직접 요청했을 때는 클라이언트 별로 새로운 빈을 반환하기 때문에 문제가 없다. 하지만 싱글톤 빈에서 프로토타입 빈을 사용하면 어떻게 될까?

### 우리 함께할 수 없는 걸까?

clientBean이라는 싱글톤 빈이 의존관계 주입을 통해 프로토타입 빈을 사용하는 예를 들어보자

```java
public class SingletonWithPrototypeTest1 {

    @Test
    void prototypeFind() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(PrototypeBean.class);
        PrototypeBean prototypeBean1 = ac.getBean(PrototypeBean.class);
        prototypeBean1.addCount();
        assertThat(prototypeBean1.getCount()).isEqualTo(1);

        PrototypeBean prototypeBean2 = ac.getBean(PrototypeBean.class);
        prototypeBean2.addCount();
        assertThat(prototypeBean2.getCount()).isEqualTo(1);

    }

    @Test
    void singletonClientUsePrototype() {
        AnnotationConfigApplicationContext ac =
                new AnnotationConfigApplicationContext(PrototypeBean.class, ClientBean.class);

        ClientBean clientBean1 = ac.getBean(ClientBean.class);
        int count1 = clientBean1.logic();
        assertThat(count1).isEqualTo(1);

        ClientBean clientBean2 = ac.getBean(ClientBean.class);
        int count2 = clientBean2.logic();
        assertThat(count2).isEqualTo(2);
    }

    @Scope("singleton")
    @RequiredArgsConstructor
    static class ClientBean {
        private final PrototypeBean prototypeBean;

        public int logic() {
            prototypeBean.addCount();
            return prototypeBean.getCount();
        }

    }
    @Scope("prototype")
    static class PrototypeBean {
        private int count = 0;

        public void addCount() {
            count++;
        }

        public int getCount() {
            return count;
        }

        @PostConstruct
        public void init() {
            System.out.println("PrototypeBean.init " + this);
        }

        @PreDestroy
        public void destroy() {
            System.out.println("PrototypeBean.destroy");
        }
    }
}
```

**clientBean 생성 과정**

1. clientBean은 싱글톤이므로 스프링 컨테이너 생성 시점에 의존관계 자동 주입이 외어 프로토타입 빈을 요청한다.
2. 스프링 컨테이너는 프로토타입 빈을 생성하여 clientBean에 주입시킨다. 프로토타입 빈의 count 필드 값은 0이다.

**클라이언트 A가 login() 호출**

1. 클라이언트 A가 clientBean을 스프링 컨테이너에 요청해서 받는다.
2. 클라이언트 A가 clientBean.logic()을 호출한다.
3. clientBean은 prototypeBean의 addCount()를 호출해서 count를 증가시킨다. 
→ count = 1

**클라이언트 B가 logic() 호출**

1. 클라이언트 B가 clientBean을 스프링 컨테이너에 요청해서 받는다.
→ 싱글톤 빈이므로 같은 객체를 받는다. → 프로토타입 빈은 새로 생성되지 않는다.
2. 클라이언트 B가 clientBean.logic()을 호출한다.
3. clientBean은 prototypeBean의 addCount()를 호출해서 count를 증가시킨다.
→ count = 2

**문제점**

- 우리는 분명 클라이언트 별로 count를 다르게 하기 위해 프로토타입 빈을 주입시켰다.
- 하지만 우리의 기대와는 다르게 count가 공유 변수가 돼버렸다.
- 싱글톤 빈에서 프로토타입을 관리하는 바람에 의존관계 주입 시점에만 프로토타입 빈이 새로 생성되기 때문이다.

### 한 번 해결해보자

어떻게 하면 각 클라이언트 별로 새로운 프로토타입 빈을 생성할 수 있을까? 가장 간단한 방법은 싱글톤 빈이 프로토타입 빈을 사용할 때마다 스프링 컨테이너에 새로 요청하는 것이다. 코드를 보자

**스프링 컨테이너에 요청**

```java
public class PrototypeProviderTest1 {

    @Test
    void prototypeFind() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(PrototypeBean.class);
        PrototypeBean prototypeBean1 = ac.getBean(PrototypeBean.class);
        prototypeBean1.addCount();
        assertThat(prototypeBean1.getCount()).isEqualTo(1);

        PrototypeBean prototypeBean2 = ac.getBean(PrototypeBean.class);
        prototypeBean2.addCount();
        assertThat(prototypeBean2.getCount()).isEqualTo(1);

    }

    @Scope("singleton")
    static class ClientBean {
        @Autowired
        private ApplicationContext ac;

        public int logic() {
            PrototypeBean prototypeBean = ac.getBean(PrototypeBean.class);

            prototypeBean.addCount();
            return prototypeBean.getCount();
        }

    }
    @Scope("prototype")
    static class PrototypeBean {
        private int count = 0;

        public void addCount() {
            count++;
        }

        public int getCount() {
            return count;
        }

        @PostConstruct
        public void init() {
            System.out.println("PrototypeBean.init " + this);
        }

        @PreDestroy
        public void destroy() {
            System.out.println("PrototypeBean.destroy");
        }
    }
}
```

- ac.getBean()을 통해 logic()을 사용할 때마다 새로운 프로토타입 빈이 생성된다.
- 이처럼 의존관계를 외부에서 주입받는 게 아니라 직접 찾는 것을 Dependency Lookup(DL) 의존관계 조회라고 한다.
- 하지만 이렇게 되면 스프링 컨테이너에 종속적인 코드가 되고 단위 테스트도 어려워진다.
- 지금 필요한 기능은 컨테이너에서 프로토타입 빈을 대신 찾아주기만 하는 기능을 제공하는 무언가가 필요하다.

### 빈들의 컨테이너에 ObjectProvider의 등장이라니

믿어라 스프링, 의심하지 말지어다 스프링

지정한 빈을 컨테이너에서 대신 찾아주는 DL 서비스를 제공하는 ObjectProvider가 있다. 과거에는 ObjectFactory를 사용했지만 여기에 편의 기능을 추가한 것이다. 

```java
@Scope("singleton")
    static class ClientBean {

        @Autowired
        private ObjectProvider<PrototypeBean> prototypeBeanProvider;

        public int logic() {
            PrototypeBean prototypeBean = prototypeBeanProvider.getObject();
            prototypeBean.addCount();
            return prototypeBean.getCount();
        }

    }
```

- prototypeBeanProvider.getObject()에 의해 항상 새로운 프로토타입 빈이 생성된다.
- 스프링이 제공하는 기능을 사용하지만 기능이 단순하므로 단위테스트를 만들거나 mock 코드를 만들기는 훨씬 편해진다.
- 스프링에 의존하기 때문에 별도의 라이브러리가 필요 없고 상속,옵션,스트림 처리 등 편의 기능이 많다.

### JSR-330 Provider

다른 방법으로는 자바 표준인 jakarta.inject.Provider를 사용하는 것이다.

```java
   @Scope("singleton")
    static class ClientBean {

        @Autowired
        private Provider<PrototypeBean> prototypeBeanProvider;

        public int logic() {
            PrototypeBean prototypeBean = prototypeBeanProvider.get();
            prototypeBean.addCount();
            return prototypeBean.getCount();
        }

    }
```

- provider.get()을 통해 항상 새로운 프로토타입 빈이 생성된다.
- 자바 표준이고 기능이 단순해 단위 테스트, mock 코드 생성이 쉬워진다.
- 자바 표준이므로 스프링이 아닌 다른 컨테이너에서도 사용할 수 있지만 별도의 라이브러리가 필요하다.

### 프로토타입 빈 사용 방법

- 그렇다면 프로토타입 빈은 언제 사용해야 할까?
- 매번 사용할 때마다 새로운 객체가 필요할 때 사용하면 된다. 하지만 실무에서는 싱글톤 빈으로 대부분의 문제를 해결할 수 있다고 하니 직접 사용하는 경우는 거의 없을 것 같다.
- 두 가지의 Provider 중에서는 대부분 ObjectProvider를 사용하다고 한다. 하지만 스프링이 아닌 다른 컨테이너에서도 코드를 사용해야 한다면 JSR-330 Provider를 사용한다고 한다.(그럴 일은 거의 없다고 한다.)

---

## 웹에도 스코프가 있다고?

지금까지는 싱글톤 스코프와 프로토타입 스코프를 사용했다. 이번에는 웹 스코프를 배울 차례이다.

### 웹 스코프 너 뭔데

- 웹 환경에서만 동작하고 프로토타입과 다르게 스프링 컨테이너가 해당 빈의 종료 시점까지 관리한다. 
→ 종료 메서드가 호출된다.
- 종류
    - request: HTTP 요청 하나가 들어오고 나갈 때까지 유지되는 스코프이다. 각 HTTP 요청 별 인스턴스가 생성되고 관리된다. → 한 페이지 내에서만
    - session: 웹 세션이 생성되고 종료될 때까지 유지되는 스코프이다. HTTP Session과 동일한 생명주기를 가진다. → 클라이언트 종료 직전까지
    - application: 웹의 서블릿 컨텍스트와 같은 범위로 유지되는 스코프이다. → 서버 종료 직전까지
    - websocket: 웹 소켓(클라이언트와 서버 간의 양방향 통신을 가능하게 하는 프로토콜)과 동일한 생명주기를 가진다.

### request 스코프 예제

- 우선 동시에 HTTP 요청이 올 때 각각 생성되는 request에 대해서 알아보자

**MyLogger**

```java
@Component
@Scope(value = "request")
public class MyLogger {

    private String uuid;
    @Setter
    private String requestURL;

    public void log(String message) {
        System.out.println("[" + uuid + "] " + "[" + requestURL + "] " + message);
    }

    @PostConstruct
    public void init() {
        uuid = UUID.randomUUID().toString();
        System.out.println("[" + uuid + "] request scope bean create: " + this);
    }

    @PreDestroy
    public void close() {
        System.out.println("[" + uuid + "] request scope bean close: " + this);
    }
}
```

- HTTP 요청을 구분하기 위한 로그를 출력하는 클래스이다
- @Scope(scope = “request”)를 사용하여 request 스코프로 지정했다. 해당 빈은 HTTP 요청 당 하나씩 생성되고, 요청이 끝나는 시점에 소멸된다.
- 빈이 생성되는 시점에 @PostConsturct 메서드를 통해 uuid를 생성해 저장해둔다
→ uuid는 절대 겹칠 일이 없다고 한다(로또의 로또의 로또의 ~~~ 확률이라고…)
- 요청 당 하나씩 생성되므로 uuid를 저장해두면 다른 HTTP 요청과 구분이 가능하다
- 빈이 소멸되는 시점에 @PreDestroy 메서드를 통해 종료 메시지를 남긴다.

**LogDemoController**

```java
@Controller
@RequiredArgsConstructor
public class LogDemoController {

    private final LogDemoService logDemoService;
    private final MyLogger myLogger;

    @RequestMapping("log-demo")
    @ResponseBody
    public String logDemo(HttpServletRequest request) {
        String requestURL = request.getRequestURL().toString();
        myLogger.setRequestURL(requestURL);

        myLogger.log("controller test");
        logDemoService.logic("testId");
        return "OK";
    }
}
```

- 로거 작동 확인을 위한 컨트롤러이다
- HttpServletRequest를 통해 요청 URL을 받는다.
- 받은 url을 myLogger의 setRequestURL() 메서드로 저장한다.
- 컨트롤러에서 controller test라는 로그를 남긴다.

**LogDemoService**

```java
@Service
@RequiredArgsConstructor
public class LogDemoService {

    private final MyLogger myLogger;

    public void logic(String id) {
        myLogger.log("service id = " + id);
    }

}
```

- 만약 request scope를 사용하지 않고 파라미터로 모든 정보를 서비스 계층으로 넘긴다면 파라미터가 많아져 지저분해지게 된다.
- requestURL 같은 웹과 관련된 정보가 웹과 관련없는 서비스 계층까지 넘어가게 된다.
- 웹과 관련된 부분은 컨트롤러까지만 사용하고, 서비스 계층은 웹 기술에 종속되지 않고 가급적 순수하게 자바 코드로 유지하는 것이 유지보수 관점에서 훨씬 좋은 코드이다.
- MyLogger 덕분에 파라미터로 넘기지 않고 멤버변수에 저장해 코드와 계층을 깔끔하게 유지할 수 있다.
- 하지만 스프링을 실행 시켜보면 오류가 발생한다. → request 요청이 와야 MyLogger 빈이 생성되기 때문에 MyLogger를 주입받지 못한 것이다.

### Provider를 써볼까

우선 앞에 배운대로 Provider를 사용해 필요한 시점에만 빈을 받는 것이다.

**Provider를 사용한 LogDemoController**

```java
@Controller
@RequiredArgsConstructor
public class LogDemoController {

    private final LogDemoService logDemoService;
    private final ObjectProvider<MyLogger> myLoggerProvider;

    @RequestMapping("log-demo")
    @ResponseBody
    public String logDemo(HttpServletRequest request) {
        String requestURL = request.getRequestURL().toString();
        MyLogger myLogger = myLoggerProvider.getObject();
        myLogger.setRequestURL(requestURL);

        myLogger.log("controller test");
        logDemoService.logic("testId");
        return "OK";
    }
}
```

**Provider를 사용한 LogDemoService**

```java
@Service
@RequiredArgsConstructor
public class LogDemoService {

    private final ObjectProvider<MyLogger> myLoggerProvider;

    public void logic(String id) {
		    MyLogger myLogger = myLoggerProvider.getObject();
        myLogger.log("service id = " + id);
    }

}
```

- ObjectProvider 덕분에 request scope 빈의 생성을 지연하고 로직을 사용할 때만 받아올 수 있게 되었다.
- LogDemoController와 LogDemoService에서 각각 따로 호출해도 같은 HTTP 요청이라면 같은 스프링 빈이 반환된다.

### 프록시를 쓰면 더 간단해진다

이번에는 프록시 방식을 사용해보자

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MyLogger {

    private String uuid;
    @Setter
    private String requestURL;

    public void log(String message) {
        System.out.println("[" + uuid + "] " + "[" + requestURL + "] " + message);
    }

    @PostConstruct
    public void init() {
        uuid = UUID.randomUUID().toString();
        System.out.println("[" + uuid + "] request scope bean create: " + this);
    }

    @PreDestroy
    public void close() {
        System.out.println("[" + uuid + "] request scope bean close: " + this);
    }
}
```

- proxyMode = ScopedProxyMode.TARGET_CLASS를 추가해보자
    - 적용 대상이 클래스라면 TARGET_CLASS
    - 적용 대상이 인터페이스면 INTERFACES
- MyLogger의 가짜 프록시 클래스를 만들어두고 HTTP request와 상관 없이 다른 빈에 미리 주입해둘 수 있다.

### 프록시 동작 원리

주입된 MyLogger의 프록시 클래스를 조회해보자

```java
myLogger = class hello.core.common.MyLogger$$EnhancerBySpringCGLIB$$xxx
```

- 예전에 본 그놈이 또 나왔다 → CGLIB라는 라이브러로 내 클래스를 상속 받은 가짜 프록시 객체를 만들어서 주입한다.
- @Scope의 proxyMode를 설정하면 스프링 컨테이너는 CGLIB라는 바이트코드 조작 라이브러리를 사용해 MyLogger를 상속한 가짜 프록시 객체를 생성한다.
- 스프링 컨테이너에 myLogger라는 이름으로 가짜 프록시 객체가 등록되어있다.

**가짜 프록시 객체에 요청이 온다면?**

- 가짜 프록시 객체는 내부에 진짜 myLogger 클래스를 찾는 방법을 가지고 있다.
- 클라이언트가 myLogger.log()를 호출하면 가짜 프록시 객체의 메서드가 호출되어 request 스코프의 진짜 myLogger.log()를 호출한다.
- 가짜 프록시 객체는 원본 클래스를 상속받아 만들어졌기 때문에 클라이언트 입장에선 프록시인지, 원본인지도 모르게 동일하게 사용할 수 있다.

### 주의점

- 마치 싱글톤을 사용하는 거 같지만 실제로는 다르게 동작하기 때문에 주의해서 사용해야 한다.
- 이런 특별한 기능을 하는 scope는 유지 보수가 어려워지기 때문에 필요한 곳에서만 최소화해서 사용해야한다.
