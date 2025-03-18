---
title: "API Gateway를 REST API로 다시..변경..합니다.. (aka. 삽질)"
categories: [Spring Boot, MSA]
tags:
  [
    MSA,
    API Gateway,
    Aggregation,
    GraphQL,
    REST API,
    Spring Boot,
    Java,
    SKALA,
    SKALA1기,
    SK
  ]
---

> Skala 과정에서 마이크로 서비스 아키텍처 구조에 대해 새롭게 알게 되었습니다.<br>
> 배운 것을 실제로 구현해보기 위해 이 여정을 시작하기로 했습니다.<br>[처음 글부터 보러가기](<https://sermadl.github.io/posts/MSA(1)/>)

블로그의 제목에 맞게..공사장..삽질..열심히 했습니다..^^...<br>
~~레퍼런스가 많이 없는 기술은 그만한 이유가 있다는 교훈을 얻었습니다.~~

## 왜 바꿨나요?

~~GraphQL을 Apollo가 아닌 Spring Cloud Gateway에서 라우팅 할 수 있는 방법을 챗지피티도 모르고..저도 모르고..구글도 몰라서요...~~<br>

GraphQL까지 배우려는 것은 욕심이었던 것 같습니다. ㅎ<br>

보통 GraphQL을 사용하면 모든 서버의 API를 GraphQL로 설정하고 Apollo 라는 프로그램을 사용해서 라우팅 및 Aggregation(GraphQL은 Resolver..?라고 부르는 것 같습니다)을 하는데,<br>

배움의 범위가 갑자기 확 넓게 다가왔습니다,,<br>

로그인 및 인증 기능을 추가하던 중에 모든 서버에 JWT 인증 기능을 추가하는 대신<br>
인증이 필요한 url은 Gateway에서 필터를 적용하여 인증 서버에 헤더로 전달된 토큰이 유효한지 검증하고,<br>
유효한 경우 전달되는 유저 정보로 다른 마이크로 서비스에 다시 요청을 보내려고 했습니다.<br>

GraphQL로 동작하는 서버에서 필터에서 전달받은 유저 정보를 컨트롤러에서 받아오기 위해서는 Reactive Context와 관련된 기능을 써야하는 것 같던데<br>

토이프로젝트이기도 하고..얼른 Aggregation 기능을 구현해보고 싶어서 일단 후퇴(ㅎㅎ)하기로 했습니다..<br>

다음에 보자 GraphQL~

### 챗지피티에게 이 구조가 과연 맞는지 확인받아보았습니다.

> Q: 근데 Api gateway에서 인증 서버로 요청해서 인증 토큰이 유효한지 검증받는 방식이 틀린건 아니지?

> 1️⃣ 현재 방식 (API Gateway → 인증 서버 검증)은 틀리지 않음!<br>
> • 중앙 집중식 검증이 가능하고, 관리가 용이함<br><br>
> 2️⃣ 하지만 성능 최적화가 필요하면 API Gateway에서 JWT를 직접 검증하는 방식이 더 효율적!<br>
> • API Gateway에서 HMAC 또는 RSA 공개키 검증을 수행<br>
> • 인증 서버는 로그인 및 블랙리스트 관리만 담당<br>
> • API Gateway가 Redis 등을 활용하여 블랙리스트 토큰 검증 가능<br><br>
> 🚀 즉, 현재 구조도 올바르지만, 트래픽이 증가하면 API Gateway에서 직접 JWT 검증을 수행하는 방식으로 개선할 수 있습니다! 🚀

간단하게 MSA를 구현해 볼 예정이었으니까,,,다음에는,,너가 추천하는 방식으로 해볼게,,,고마워,,

<hr>

## 마치며

다음은 인증 기능 구현에 대해 작성하겠습니다..!<br>
(\*꒦ິ⌓꒦ີ)<br>

### 최최최최종 시스템 아키텍처..!

![System-Architecture-REST-API](/assets/img/system-architecture-rest-api-gateway.png)

<hr>
<br>

> 참고 자료

[하나](https://graphql-kr.github.io/learn/pagination/), [둘](https://deview.kr/data/deview/session/attach/1100_T1_%E1%84%87%E1%85%A1%E1%86%A8%E1%84%89%E1%85%A5%E1%86%BC%E1%84%92%E1%85%A7%E1%86%AB_GraphQL%20API%20%E1%84%81%E1%85%A1%E1%84%8C%E1%85%B5%E1%86%BA%E1%84%80%E1%85%A5%20%E1%84%8B%E1%85%AE%E1%86%AB%E1%84%8B%E1%85%A7%E1%86%BC%E1%84%92%E1%85%A2%E1%84%87%E1%85%A9%E1%84%8C%E1%85%B5%20%E1%84%86%E1%85%AF.pdf), [셋](https://devblog.kakaostyle.com/ko/2022-11-09-1-understanding-graphql-3-understanding-resolver/), [넷](https://velog.io/@sylvan_cottage/Graphql-Federation-%EA%B5%AC%ED%98%8413), [다섯](https://toss.tech/article/slash23-server), [여섯](https://engineering-skcc.github.io/microservice%20outer%20achitecture/outer-arch-Auth/)
