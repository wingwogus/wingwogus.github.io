---
title: "[HTTP] U씨 삼형제 등장 - URI, URL, URN"
excerpt: "셋의 개념은 뭐고 어떤 차이점이 있는걸까?"

categories:
  - HTTP
tags:
  - [http, cs]

permalink: /http/uri-url-urn/

toc: true
toc_sticky: true

date: 2025-01-01
last_modified_at: 2024-01-01
---
## 각 개념에 대해 정리해보자

우리는 흔히 URL에 대해서는 익숙하지만 URI, URN에 대해서는 잘 들어보지 못 했을것이다.

### URI(Uniform Resource Identifier)

- 통합 자원 식별자로써 인터넷에 있는 자원을 어디에 있는지 자원 자체를 식별하는 방법이다.
    - Uniform: 리소스를 식별하는 통일된 방식
    - Resource: 자원 → URI로 식별할 수 있는 모든 것
    - Identifier: 다른 항목과 구분하는데 필요한 정보
- URI의 존재는 인터넷에서 요구되는 기본 조건으로 인터넷 프로토콜에 항상 붙어 다닌다.
- 하위 개념으로 URL, URN이 있다.

### URL(Uniform Resouce Locator)

- 파일 식별자로써 네트워크 상에서 자원이 어디 있는지 위치를 가르쳐주는 방법이다.
- 웹 사이트 주소로만 알고 있지만 뿐만 아니라 컴퓨터 네트워크 상의 모든 자원을 나타내는 표기법
- 해당 주소에 접속하려면 URL에 맞는 프로토콜을 알아야하고 동일 프로토콜로 접속해야 한다.

**URL 구성 요소**

```java
scehme://[userinfo@]host[:port][/path][?query][#fragment]
->
https://www.google.com/search?q=hello
```

- scheme
    - 주로 프로토콜을 사용(http, https)
- userinfo
    - URL에 사용자 정보를 포함해서 인증 하는 것인데 거의 사용하지 않는다.
- host
    - 호스트명
    - 도메인명 또는 IP 주소를 직접 사용 가능하다.
- PORT
    - 접속 포트로 일반적으로 생략 가능하다
    - http는 80, https는 443
- path
    - 리소스 경로, 계층적 구조로 이루어져 있다.
    - 말 그대로 서버 파일의 경로라고 생각하면 된다.
- query parameter or query string
    - key, value 형태로 이루어져 있다.
    - ?시작하고 &로 추가가 가능하다.
    - 웹 서버에 제공하는 파라미터이기 때문에 query parameter라고 불리고 항상 문자 형태로 넘어가기 때문에 query string이라고도 불린다.
- fragment
    - 메인 리소스 내에 존재하는 서브 리소스에 접근할 떄 이름을 식별하기 위한 정보
    - html 내부 북마크 등에 사용한다.
    - 서버에 전송하는 정보가 아니다.

### URN(Uniform Resource Name)

- URL이 리소스가 있는 위치를 지정한다면 URN은 리소스에 이름을 부여하는 것이다.
- 프로토콜을 포함하지 않고 page 이후 부분까지 포함한다.

**URN 구성 요소**

```java
scheme:NID:NSS
->
urn:isbn:123456
```

- scheme
    - 다른 URI와 구분하기 위해 urn으로 고정해서 사용한다 → RFC 2141 표준
- NID(Namespace Identifier)
    - 각 URN이 속한 namespace로 자원이 저장된 저장소를 의미한다.
    - 도서에 관한 URN인 경우 isbn을 예로 들 수 있다.
    - 해당 식별자를 통해 어떤 종류의 자원인지 파악할 수 있다.
- NSS(Namespace Specific String)
    - namespace 내에서 고유한 자원을 식별하는 문자열이다.
    - 도서에 관한 NSS인 경우 isbn의 일련 변호를 나타낸다