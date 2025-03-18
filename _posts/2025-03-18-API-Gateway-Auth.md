---
title: "API Gateway에서 GraphQL to REST API로 요청 변환하기 - Aggregation 시작하기 (feat. Aggregation, MSA)"
categories: [Spring Boot, MSA]
tags: [MSA, API Gateway, REST API, Spring Boot, Java, SKALA, SKALA1기, SK]
---

> Skala 과정에서 마이크로 서비스 아키텍처 구조에 대해 새롭게 알게 되었습니다.<br>
> 배운 것을 실제로 구현해보기 위해 이 여정을 시작하기로 했습니다.<br>[처음 글부터 보러가기](<https://sermadl.github.io/posts/MSA(1)/>)

이번 기능은 생각(_GraphQL을 포기하지 못했던 때_)보다 잘 작동해서<br>
다시 추진력을 얻었습니다,,

## System Architecture

![System-Architecture](/assets/img/system-architecture-rest-api-gateway.png)

이제 진짜진짜진짜 최최최종 시스템 구조입니다...🥲

Auth Server를 따로 구성하기에는 간단하게 구현하려다가 일이 커질 것만 같아서<br>
기능 구현 단계에서 최종 목표인 Aggregation을 하루 빨리 진행하기 위해<br>
회원 가입과 로그인 기능을 같이 두기로 했습니다! (분리를 하는 서비스도 분명 있긴하겠죠..??)

## User Server

### UserController.java

```java
@RestController
@Slf4j
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public ResponseEntity<List<UserInfoResponse>> getAllUsers() {
        return ResponseEntity.ok(
                userService.getAll()
        );
    }

    @GetMapping("/{userId}")
    public ResponseEntity<UserInfoResponse> getUserById(@PathVariable("userId") Long userId) {
        return ResponseEntity.ok(userService.getUser(userId));
    }

    @PostMapping("/register")
    public ResponseEntity<UserInfoResponse> createUser(@RequestBody RegisterUserRequest request) {
        log.info(request.getUsername());
        log.info(request.getPassword());
        log.info(request.getEmail());
        log.info(request.getPhone());
        return ResponseEntity.ok(userService.register(request));
    }
}
```

`DTO` 는 각자 설정에 맞게 구성해주시면 됩니다<br>

<hr>

### AuthController.java

```java
@RestController
@Slf4j
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @RequestBody @Validated LoginRequest request
    ) {
        return ResponseEntity.ok(
                authService.login(request)
        );
    }

    @GetMapping("/validation")
    public ResponseEntity<ValidTokenResponse> validate(
            JwtAuthentication auth
    ) {
        return ResponseEntity.ok(
                authService.validToken(auth.getUserId())
        );
    }
}
```

`ValidTokenResponse` 는 전달된 JWT Token을 발급받은 사용자의 정보를 담고 있습니다!<br>
사용자의 권한에 따라 접근할 수 있는 url을 달리 할 것이기 때문에 사용자의 역할(권한)에 대한 정보도 담겨있습니다<br>

JWT 설정에 관련된 건 [여기](https://sermadl.github.io/posts/Spring-3.0-JWT-Spring-Security/)를 참고해주세요 ^,^

<hr>

## API Gateway Server

### WebClientConfig.java

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced  // ✅ LoadBalancer 활성화
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```

Eureka Client로 등록된 이름으로 webClient Url을 작성하기 위해 필요한 클래스입니다<br>

<hr>

### CustomPreFilter.java

```java
@Component
@Slf4j
public class CustomPreFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest response = exchange.getRequest();
        log.info("Pre Filter: Request URI is {}", response.getURI());
        // Add any custom logic here

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

Global 하게 작용할 PreFilter를 커스텀하기 위해 필요한 클래스입니다.<br>
당장 사용하진 않지만, 요청이 라우팅 되었음을 이 PreFilter를 통해 알 수 있기 때문에 보다 자세한 디버깅(?)을 위해 추가했습니다.<br>

### AuthenticationFilter.java

```java
@Component
@Slf4j
public class AuthenticationFilter extends AbstractGatewayFilterFactory<AuthenticationFilter.Config> {

    private final WebClient webClient;

    private static final List<String> EXCLUDED = List.of(
            "/login",
            "/register",
            "/getAllItems"
    );

    public AuthenticationFilter(WebClient.Builder webClient) {
        super(Config.class);
        this.webClient = webClient.baseUrl("http://USER-SERVICE").build();
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) ->
                isExcluded(exchange.getRequest().getURI().getPath()) ?
                        chain.filter(exchange) : validToken(exchange, chain);
    }

    // 특정 url은 인증 제외
    private boolean isExcluded(String requestPath) {
        return EXCLUDED.stream().anyMatch(requestPath::startsWith);
    }

    // JWT 검증을 위해 User Server와 통신
    private Mono<Void> validToken(ServerWebExchange exchange, GatewayFilterChain chain) {
        String header = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

        log.info("Header: {}", header);

        if (header == null || !header.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return webClient.get()
                .uri("/validation")
                .header(HttpHeaders.AUTHORIZATION, header)
                .retrieve()
                .bodyToMono(ValidTokenResponse.class)
                .flatMap(response -> {
                    if(response.isValid()) {
                        log.info("Valid Response");

                        return chain.filter(exchange);
                    } else {
                        log.info("Invalid Response");

                        return setUnauthorizedResponse(exchange);
                    }
                })
                .onErrorResume(e -> setUnauthorizedResponse(exchange));
    }

    // 401 Unauthorized 응답 설정
    private Mono<Void> setUnauthorizedResponse(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }

    public static class Config {
    }
}
```

`EXCLUDED` 변수에 인증이 필요하지 않은 path를 설정합니다.<br>

`@RequiredArgsConstructor` 를 사용하여 편하게 생성자를 만들 수도 있었지만,<br>
`baseURL` 을 지정해주기 위해 이런 구조를 사용했습니다.<br>

`apply()` 함수가 이 필터가 실행될 때 실질적으로 실행되는 함수라고 생각하면 됩니다.<br>

#### 필터 작동 방식

1. 먼저 EXCLUDED에 포함된 인증이 필요없는 url인지 확인한 후에<br>

2. Authorization Header로 전달된 값이 Bearer 토큰이 맞는지 확인하고<br>

3. 해당 JWT 토큰이 유효한지 확인하기 위해 User 서버로 요청을 보냅니다.<br>

4. 만약 유효한 토큰이라면 정상적으로 사용자의 정보가 담긴 `ValidTokenResponse` 를 반환할 것이고,<br>
   그렇지 않은 경우에는 401 에러코드를 반환합니다. (아직 이 부분은 커스텀 에러가 적용되지 않았습니다)<br>

`Config` 생성자에는 추가 구성을 위한 변수를 선언할 수 있다고 합니다.<br>
추후에 활용을 할 수도 있겠지만 일단 비워뒀습니다!<br>

### application.yml

```yml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
      discovery:
        locator:
          enabled: true  # Eureka에서 자동으로 서비스 찾기
          lower-case-service-id: true # 대소문자 구분 없이 서버 아이디 등록 가능
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Header=Service-Type, user
          filters:
            - name: AuthenticationFilter
            - StripPrefix=0

            .
            .
            .
```

path가 아니라 Header로 라우팅 될 서버를 구분해 줄 것이고,<br>
요청된 path 그대로 라우팅 할 서버에 전달합니다!<br>

## 실행 화면

### 인증이 필요 없는 path로 요청을 보낼 때

로그인은 사용자 인증이 필요없는 기능입니다.<br>
해당 url로 요청을 보내 라우팅이 잘 동작하는지 살펴보겠습니다.<br>

![API-Gateway-Authentication](/assets/img/api-gateway-authorization-1.png)
해당 API를 요청하여 User 서버로 라우팅이 제대로 되게 하려면 Header에 Service-Type 변수를 넣고 user 값을 넣어줍니다!<br>

![API-Gateway-Authentication](/assets/img/api-gateway-authorization-2.png)
User Server의 `/login` 으로 DTO에 맞게 요청을 보내면 위와 같이 JWT Token이 정상적으로 전달되는 모습을 확인할 수 있습니다<br>

![Log1](/assets/img/1-2-log.png)
로그에서도 정상적으로 라우팅이 되어 GlobalFilter로 지정해주었던 CustomFilter를 지난 것을 확인할 수 있습니다<br>

<hr>

### 인증이 필요한 path로 요청을 보낼 때

모든 사용자의 정보를 반환하는 url은 사용자 인증이 필요합니다.<br>
해당 url로 요청을 보내보겠습니다<br>

#### Header에 토큰을 전달하지 않았을 때

![No-Header](/assets/img/need-authorization-but-no-header.png)
인증이 필요하지만 토큰이 전달되지 않았을 때, 구현한대로 `401` 을 반환합니다.

![No-Header-Log](/assets/img/no-header-log.png)
라우팅이 제대로 되었지만, Header가 제대로 전달되지 않은 것을 로그에서도 확인할 수 있습니다.<br>

<hr>

#### 올바른 토큰을 전달하지 않았을 때

![Invalid-Header](/assets/img/need-authorization-but-invalid.png)
인증이 필요하지만 토큰이 올바르지 않을 때, 구현한대로 `401` 을 반환합니다.<br>

정확히 어떤 이유 때문에 에러 코드가 반환되었는지 구별하기 위해 추후에 커스텀 에러필터를 적용하여 구현해야 할 것 같습니다.<br>

![Invalid-Header-Log](/assets/img/invalid-header-log.png)
라우팅이 제대로 되었지만, 전달된 Header가 제대로 된 토큰의 형태를 띄지 않고 있음을 로그에서도 확인할 수 있습니다.<br>

#### 정상적인 토큰이 전달되었을 때

![Success](/assets/img/authorization-success.png)
요청이 정상적으로 동작하였습니다!<br>

![Success-log](/assets/img/success-log.png)
Token과 Response 모두 정상적으로 전달된 것을 로그에서도 확인할 수 있습니다.<br>

## 마치며

~~GraphQL만 포기하면..이렇게 간단한..~~

레퍼런스가 적은 기술은 독학하기엔 너무 어려운 것 같습니다,,<br>
GPT야 어서 더 똑똑해지렴,,,<br>

다음은 필터에서 전달받은 `ValidTokenResponse` 를 활용해 사용자의 역할에 따라 API에 대한 접근 권한을 나눠보도록 하겠습니다!<br>

이름값 하는 블로그,,🚧🦺🚜💭,,🪏<br>

<hr>
<br>

> 참고 자료

[하나](https://pixx.tistory.com/287), [둘](https://velog.io/@jungse97/), [셋](Spring-Cloud-Users-Microservice-%EC%9D%B8%EC%A6%9D%EA%B3%BC-%EA%B6%8C%ED%95%9C), [넷](https://medium.com/@premchandu.in/spring-webflux-aggregation-of-responses-from-different-microservices-acfb0e5f1fc5), [다섯](https://lincoding.tistory.com/99)
