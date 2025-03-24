---
title: "MSA 구현하기(8) - 각 서비스에서 API 별 권한 처리하기"
categories: [Spring Boot, MSA]
tags:
  [
    MSA,
    Spring,
    API Gateway,
    Aggregation,
    REST API,
    JWT,
    Spring Boot,
    Java,
    SKALA,
    SKALA1기,
    SK
  ]
---

> Skala 과정에서 마이크로 서비스 아키텍처 구조에 대해 새롭게 알게 되었습니다.<br>
> 배운 것을 실제로 구현해보기 위해 이 여정을 시작하기로 했습니다.<br>[처음 글부터 보러가기](<https://sermadl.github.io/posts/MSA(1)/>)

사용자 별로 권한을 설정해서 접근할 수 있는 API를 설정하고 싶어서 이 기능을 추가하게 되었습니다!<br>

게이트웨이에서 로그인 된 사용자가 보낸 토큰이 유효한지, <br>이 사용자가 실제로 등록되어 있는 사용자인지를 확인한 후에 각 API로 요청을 보내는 기능은 이전에 구현했었지만<br>
권한 별로 접근할 수 있는 API를 달리 하려면 어떻게 해야 할까 고민을 해봤습니다.<br>

### **_저는 `Header` 로 사용자의 `UserRole` 정보를 보내서 역할 별로 접근할 수 있는 API를 제한하기로 했습니다!_**

<hr>

## 왜 Header인가요?

API Gateway에서 사용자의 권한을 확인하고 각 요청의 수행 여부를 결정한다고 했을 때,<br>
Aggregation이 필요한 API로 요청이 들어왔을 때에는 사용자의 정보를 불러와서 권한을 확인한 후에 다음 API를 호출할 지 정할 수 있지만<br>

바로 라우팅 되는 API로 요청이 들어 온 경우에는 권한 별로 일일히 모든 url을 `Excluded` 변수에 저장해 둔 것 처럼 관리해줘야 합니다.<br>

게다가 이미 Aggregation으로 API Gateway가 라우팅 기능 외에 다른 기능도 수행하고 있습니다.<br>

반면 각 마이크로 서버에서 권한을 확인하는 경우 위의 단점이 보완되고<br>
추후 확장성이나 수정이 필요할 때를 고려하면 API Gateway까지 고치지 않아도, 각 서버의 코드만 수정하면 되기 때문에 단일 책임 원칙을 따르고 있습니다.<br>

권한 처리를 하는 코드가 각 서버에 중복 되어 저장되는 문제가 있고<br>
Gateway에서 사용자의 정보를 담아 API를 호출해야 합니다.<br>

> 장단점을 따져 본 결과 코드의 중복이 발생하더라도<br> > **_`각 마이크로 서버` 에서 권한을 처리하자!_** 고 결론을 내렸습니다.<br>

그에 따라 Gateway에서 라우팅을 할 때와 Aggregation으로 API를 호출할 때 일관적으로 로그인 된 사용자의 정보를 넘길 수 있는 방법이 무엇인지 찾다가<br>

`Header` 에 로그인 된 사용자의 `ID`, `UserRole` 을 담는 방법이 가장 좋을 것 같다고 판단했습니다.<br>

#### 추후에 더 좋은 방법이 있다면 바꿀 것 같습니다.

<hr>

## API Gateway에서 사용자의 정보 넘기기

권한 처리가 필요한 경우는 모두 사용자가 로그인 되어 있는지를 확인해야 하는 경우입니다.<br>

라우팅 되는 API지만 권한 확인이 필요한 경우를 먼저 다뤄보겠습니다.

### AuthenticationFilter.class 내부 함수 수정

```java
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
                    // 유저 정보를 담은 새로운 요청으로 교체
                    ServerHttpRequest newRequest = exchange.getRequest().mutate()
                            .header("x-user-id", String.valueOf(response.getId()))
                            .header("x-user-role", String.valueOf(response.getRole()))
                            .build();

                    ServerWebExchange newExchange = exchange.mutate()
                            .request(newRequest)
                            .build();

                    return chain.filter(newExchange);
                } else {

                    log.info("Invalid Response");

                    return setUnauthorizedResponse(exchange);
                }
            })
            .onErrorResume(e -> setUnauthorizedResponse(exchange));
}
```

> 이 클래스의 모든 코드를 보고 싶다면 [여기](https://sermadl.github.io/posts/API-Gateway-Auth/#authenticationfilterjava)

Header에 `x-user-id` , `x-user-role` 변수를 통해 로그인 된 사용자의 ID와 User Role을 전달합니다.<br>

<hr>

## 마이크로 서비스에서 사용자의 정보 받기

### UserController.java (User 서버)

```java
.
.
.
/** [관리자]
 * 모든 사용자 정보 가져오기
 * @param userId 로그인 된 사용자 Id
 * @param role 로그인 된 사용자 Role
 * @return 모든 사용자 정보 리스트
 */
@GetMapping("/list")
public ResponseEntity<List<UserInfoResponse>> getAllUsers(
        @RequestHeader("x-user-id") Long userId,
        @RequestHeader("x-user-role") UserRole role
    ) {
    log.info("user id({}) accessed to find all users", userId);

    RoleCheck.isAdmin(role);

    return ResponseEntity.ok(
            userService.getAll()
    );
}

.
.
.

/** [사용자]
 * 로그인 된 사용자 정보 불러오기
 * @param userId 로그인 된 사용자 User Id
 * @param role 로그인 된 사용자 Role
 * @return 로그인 된 사용자 정보
 */
@GetMapping("/my")
public ResponseEntity<UserInfoResponse> getMyInfo(
        @RequestHeader("x-user-id") Long userId,
        @RequestHeader("x-user-role") UserRole role
    ) {
    log.info("user id({}) accessed to find user", userId);
    RoleCheck.isUser(role);

    return ResponseEntity.ok(userService.getUser(userId));
}

.
.
.

```

전달된 헤더를 통해 사용자의 User Id와 Role을 받고 이를 이용해서 User의 권한을 확인합니다.<br>

만약 User가 아닌 다른 서버에서 권한을 체크하더라도, UserRole 객체와 RoleCheck 객체만 복사 붙여넣기 해주면 됩니다.<br>

`RoleCheck.java`(마이크로 서비스 공통 코드) 와 `UserRole.java`(유저의 권한에 대한 정보가 담겨 있는 코드) 의 구현은 취향껏 해주시면 되는데 혹시나 제 코드가 궁금하신 분들을 위해 아래 토글에 작성해두겠습니다.<br>

<details>
  <summary>여기를 클릭하면 내용이 열립니다</summary>

<h4>RoleCheck.java</h4>
<pre><code class="language-java">
public class RoleCheck {
    public static void isUser(UserRole role) {
        if (!role.isUser()) {
            throw new HasNoAuthorityException();
        }
    }

    public static void isSeller(UserRole role) {
        if (!role.isSeller()) {
            throw new HasNoAuthorityException();
        }
    }

    public static void isAdmin(UserRole role) {
        if (!role.isAdmin()) {
            throw new AdminOnlyException();
        }
    }

}
</code></pre>

<hr>

<h4>UserRole.java</h4>
<pre><code class="language-java">
@Getter
public enum UserRole {
    GUEST("GUEST"),
    USER("USER"),
    SELLER("SELLER"),
    ADMIN(combine("ADMIN", "USER", "SELLER"));

    private final String name;

    UserRole(String name) {
        this.name = name;
    }

    public static String combine(String... names) {
        return String.join(",", names);
    }

    private static final Map<String, UserRole> BY_LABEL =
            Stream.of(values()).collect(Collectors.toMap(UserRole::getName, e -> e));

    public static UserRole of(String name) {
        return BY_LABEL.get(name);
    }

    public boolean isAdmin() {
        return this == ADMIN;
    }

    public boolean isSeller() {
        return this == SELLER;
    }

    public boolean isUser() {
        return this == USER;
    }

}
</code></pre>

</details>

<hr>

## 마치며

마이크로 서비스 아키텍처 실습을 하면서 아키텍처 구조에 대한 고민을 굉장히 많이 하게 되는 것 같습니다..<br>
어떤 것이 효율적인 구조인지, 어떤 것이 더 보안에 좋을지, 여러가지로 신경 쓸 것이 굉장히 많은 기술이라는 것이 느껴지고 있습니다.<br>

마이크로 서비스를 처음 배울 때도 알려주신 사실이지만,<br>
서비스 별로 굳이 마이크로 서비스를 적용하지 않아도 되는 서비스가 있는 반면,<br>
마이크로 서비스를 적용하지 않으면 서버의 크기가 굉장히 커지고 무중단으로 서비스를 제공하지 못할 가능성이 큰 서비스가 있기 때문에<br>
각 서비스의 환경과 여건에 맞춰서 서비스를 도입해야 한다는 것이 작은 서비스를 더 작게 나누다 보니 느껴지고 있는 것 같습니다.<br>

확실히 마이크로 서비스를 도입하지 않는 것이 더 나은 서비스가 있을 것 같습니다,,(사이드 프로젝트 정도..?)<br>

저의 글이 누군가에게 도움이 되길,,😩

<hr>
<br>

> 참고 자료

[하나](https://velog.io/@jungse97/Spring-Cloud-Users-Microservice-%EC%9D%B8%EC%A6%9D%EA%B3%BC-%EA%B6%8C%ED%95%9C)
