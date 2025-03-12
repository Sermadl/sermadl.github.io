---
title: "MSA 구현하기(5) - API Gateway(BFF) 구성하기 (feat. Spring Cloud Gateway, GraphQL, REST API, gRPC)"
categories: [Spring Boot, MSA]
tags:
  [
    MSA,
    BFF,
    API Gateway,
    GraphQL,
    REST API,
    gRPC,
    Spring Boot,
    Java,
    SKALA,
    SKALA1기,
    SK
  ]
---

> Skala 과정에서 마이크로 서비스 아키텍처 구조에 대해 새롭게 알게 되었습니다.<br>
> 배운 것을 실제로 구현해보기 위해 이 여정을 시작하기로 했습니다.<br>[처음 글부터 보러가기](<https://sermadl.github.io/posts/MSA(1)/>)

Rest API 대신 GraphQL을 사용하여 API Gateway와 통신해보겠습니다!

## Rest API vs GraphQL

사실 Rest API가 너무너무너무(✖️♾️) 익숙해서 또 새로운 기술을 도입을 해야할까..? 라고 생각했지만<br>
이왕 배우는거...GraphQL까지 정복해버리고 말겠다(!!!!) 라는 심정으로 시작하게 되었습니다.<br>

그래도 GraphQL을 선택하게 된 이유를 설명하기 위해 두 방식의 장단점을 표로 정리해보겠습니다.<br>

| 구분     | **GraphQL**                                                                                                                                                                                                                                           | **REST API**                                                                                                                                                                                                                                                                        |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **장점** | • 클라이언트가 **`필요한 데이터만 요청`** 가능 (오버페칭 방지) <br> • `단일` 엔드포인트로 `여러 리소스를 한번에` 조회 가능 (언더페칭 방지) <br> • 버전 관리 없이 스키마 수정만으로 확장 가능 <br> • Subscription을 활용한 실시간 데이터 처리 가능     | • HTTP 캐싱이 용이하여 성능 최적화 가능 <br> • RESTful 구조가 익숙하여 학습이 `쉬움` <br> • 엔드포인트 기반으로 `접근 제어` 및 보안 관리가 용이 <br> • HTTP 상태 코드를 활용한 `직관적인 에러` 처리                                                                                 |
| **단점** | • HTTP 캐싱이 어려워 별도의 캐싱 로직이 필요 <br> • 쿼리 작성법과 스키마 정의 등 **`학습 난이도가 높음`** <br> • 클라이언트가 원하는 데이터를 자유롭게 요청할 수 있어 `보안 관리 필요` <br> • 기본적으로 REST보다 서버 리소스를 `많이` 사용할 수 있음 | • `필요 이상의 데이터` 를 받을 가능성이 있음 (오버페칭) <br> • `다중 엔드포인트` 로 인해 데이터 조회 시 `여러 번 요청` 해야 할 수도 있음 (언더페칭) <br> • 새로운 버전이 필요할 경우 엔드포인트를 추가해야 함 <br> • 실시간 데이터 처리가 어렵고 별도의 WebSocket 또는 Polling 필요 |

코드블럭으로 처리해 둔 부분이 바로 제가 GraphQL을 쓸지, REST API를 쓸지 고민하던 이유였습니다.<br>

REST API로 개발하면서 프론트엔드와의 API 연결 시 새로운 버전이 추가되거나, 수정 사항이 있는 경우 API가 너무 많고 다 비슷해보여서<br>
서로 어떤 API가 추가되었고, 수정되어야 하고, 수정이 되었는지 헷갈리는 경우가 많았습니다.<br>
필요한 Request나 Response가 조금이라도 다른 경우 또 다른 API를 만들어야 하는 것도 굉장히 번거로웠습니다.<br>

그에 비해 GraphQL은 하나의 엔드포인트를 만들어놓으면 프론트에서 원하는 데이터만 골라서 요청하고 받을 수 있다는 장점이 있었고,<br>
Subscribtion 기능을 활용할 수 있을지는 모르겠지만 데이터 변경이 생기면 실시간으로 클라이언트에 변경된 데이터를 전달할 수 있다는 것도 유용해보였습니다.<br>

GraphQL을 한 번도 사용해 본 적이 없어서 안 그래도 익숙하지 않은 기술이 어렵기까지 하다고 하니까 정말 엄두가 안 났는데,<br>
니까짓게(?) 어려우면 얼마나 어렵겠어 ㅋㅋ 라는 생각으로 시도하게 되었습니다.<br>

하지만, 모든 서버를 GraphQL을 사용해서 통신하는 것이 아니라, API Gateway 서버([BFF, Backend For Frontend](https://velog.io/@seeh_h/BFF%EB%9E%80)) 서버에만 GraphQL을 적용하고자 합니다!<br>

아무래도 GraphQL의 속도가 느리고, 서버 리소스를 많이 쓰기도 하고<br>
제대로 MSA를 경험해보기 위해서는 각 서버의 API 통신 방식이 달라야(!) 한다고 생각했습니다.<br>

그래서 gRPC도 추후에 도입을,,해보고자,,합니다,,,,<br>

<hr>

### gRPC?

| 항목               | 설명                                                                                                                                                              |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **gRPC란?**        | Google에서 개발한 **고성능 원격 프로시저 호출(Remote Procedure Call, RPC) 프레임워크**                                                                            |
| **특징**           | - HTTP/2 기반의 비동기 통신 지원<br>- Protocol Buffers(바이너리 직렬화) 사용<br>- 다중 언어 지원 (Java, Go, Python 등)                                            |
| **장점**           | - REST보다 **빠른 데이터 전송** (바이너리 포맷 사용)<br>- **Persistent Connection**으로 연결 유지 가능<br>- **양방향 스트리밍** 지원 (실시간 데이터 전송 가능)    |
| **단점**           | - **학습 난이도 높음** (ProtoBuf 학습 필요)<br>- 브라우저 직접 호출 불가 (gRPC-Web 필요)<br>- REST API보다 **디버깅 및 로깅 어려움**                              |
| **주요 사용 사례** | - **마이크로서비스 간 통신** (서비스 간 빠른 요청 처리)<br>- **대량 데이터 전송** (영상, 파일, 로그 전송)<br>- **실시간 데이터 스트리밍** (채팅, IoT 데이터 처리) |
| **gRPC vs REST**   | - **gRPC:** 바이너리 전송, HTTP/2 기반, 빠름, 실시간 처리 용이<br>- **REST:** 텍스트(JSON) 전송, HTTP/1.1 기반, 브라우저 친화적                                   |

gRPC도 학습 난이도가 굉장한 것 같아서 `일단 MSA 내부를 모두 REST API로 구현`한 다음에 <br>
모든 구조가 잘 돌아가는 모습을 확인한 이후에 도입을 해볼 예정입니다!

<hr>

## 구현하게 될 MSA 구조

결론적으로 제가 구현하게 될 마이크로 서비스 아키텍처를 그림으로 표현해봤습니다.<br>

![System-Architecture](/assets/img/system-architecture.png)

각 서버는 REST API, API Gateway(BFF)는 GraphQL로 구현합니다.

_서버의 구조는 유기적으로 변동될 것 같습니다_

## API Gateway(BFF) 서버 구성

### pom.xml

```xml
.
.
.
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway-mvc</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
.
.
.
```

Spring Cloud Gateway와 GraphQL, Eureka Client 라이브러리를 추가해주었습니다.<br>
Gateway가 Eureka 서버에 등록된 서버들을 찾기 위해서는 Client로 등록되어야합니다!

<hr>
<br>
이제 Spring Cloud 에서 자동으로 마이크로 서비스들을 찾을 수 있도록 설정해주겠습니다.

### application.yml

```yaml
.
.
.
server:
  port: 19901

spring:
  main:
    web-application-type: reactive
  cloud:
    gateway:
      globalcors:
        corsConfigurations: # cors error 방지
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
      discovery:
        locator:
          enabled: true  # Eureka에서 자동으로 서비스 찾기
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Header=Service-Type, user
          filters:
            - StripPrefix=0 # localhost:19901 뒤에 오는 url 그대로 전달하기

        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Header=Service-Type, order
          filters:
            - StripPrefix=0

        - id: item-service
          uri: lb://ITEM-SERVICE
          predicates:
            - Header=Service-Type, item
          filters:
            - StripPrefix=0


eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/  # Eureka 서버의 URL을 지정
.
.
.
```

application.yml에 위 코드를 추가해주면 등록 완료입니다!<br>

`predicates` 항목은 Gateway가 각 서버에 들어오는 요청을 구분하는 방식에 관한 변수인데,<br>
현재 구현해 둔 마이크로 서비스들이 모두 GraphQL을 사용하고 있어서,,,_굉장한 고민_ 끝에 **`Header 기반`** 으로 요청을 구분하기로 하였습니다.<br>

아직 API Gateway에서 GraphQL 관련 파일을 작성하지 않아서 임시로 이렇게 동작하게끔 만들어봤습니닷..

<hr>

### API Gateway의 요청 구분 방식

| 구분 방식                                      | 설명                                             | 예시                                                                                           |
| ---------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **경로(Path) 기반 라우팅**                     | 요청 URL의 경로에 따라 서비스를 구분             | `/users/**` → `User-Service`<br>`/orders/**` → `Order-Service`                                 |
| **도메인(Subdomain) 기반 라우팅**              | 요청 도메인 또는 서브도메인에 따라 서비스를 구분 | `user.api.com` → `User-Service`<br>`order.api.com` → `Order-Service`                           |
| **헤더(Header) 기반 라우팅**                   | 특정 HTTP 헤더 값을 기반으로 요청을 구분         | `Header: Service-Type=user` → `User-Service`<br>`Header: Service-Type=order` → `Order-Service` |
| **쿼리 파라미터(Query Parameter) 기반 라우팅** | 요청 URL의 쿼리 파라미터 값을 기준으로 구분      | `/api?service=user` → `User-Service`<br>`/api?service=order` → `Order-Service`                 |
| **메서드(HTTP Method) 기반 라우팅**            | 요청의 HTTP 메서드(GET, POST 등)에 따라 구분     | `GET /api/orders` → 조회 서비스<br>`POST /api/orders` → 생성 서비스                            |
| **JWT 토큰 기반 라우팅**                       | JWT 토큰의 특정 클레임(Claim) 값에 따라 구분     | `role=admin` → `Admin-Service`<br>`role=user` → `User-Service`                                 |
| **지리적 위치 기반 라우팅**                    | 요청의 IP 주소 또는 국가 정보를 기반으로 구분    | `한국 IP` → `KR-Server`<br>`미국 IP` → `US-Server`                                             |

이 중에 제 프로젝트와 가장 잘 맞는 방식은 헤더 기반 라우팅인 것 같아서 헤더로 각 서비스로 갈 요청을 구분하기로 했습니다.<br>

## 마치며

![Api-Gateway-Test](/assets/img/api-gateway-test-1.png)
user 서버와 정상적으로 소통이 되었습니다!

Spring Cloud Gateway 라이브러리가 제공해주는 기능이 굉장히 편리해서 생각보다 설정할 것이 많지 않았습니다! 아마 Eureka 서버와 Gateway를 함께 써서 더 쉽게 느껴지는 것일 수도 있겠지만, 이렇게 해서 Gateway 관련 설정을 모두 마쳤습니다.<br>

지금은 API Gateway 로서의 역할밖에 수행하지 않지만, 추후에 BFF로서 동작을 할 예정입니다...!
다음은 GraphQL을 통해 API Gateway와 통신해보겠습니다.,.,☃︎

<hr>
<br>

> 참고 자료

[하나](https://velog.io/@lovelys0731/GraphQL-%ED%86%BA%EC%95%84%EB%B3%B4%EA%B8%B0-1-GraphQL-%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90), [둘](https://jacksonhong.tistory.com/62), [셋](https://velog.io/@dongvelop/Spring-Boot-GraphQL-%EC%86%8C%EA%B0%9C), [넷](https://yozm.wishket.com/magazine/detail/2113/), [다섯](https://velog.io/@mrcocoball2/Spring-Cloud-Spring-Cloud-Gateway-%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90)
