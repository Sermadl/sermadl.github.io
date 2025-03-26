---
title: "Error Code Custom 하기!"
categories: [Spring Boot, MSA]
tags:
  [
    MSA,
    Spring,
    API Gateway,
    Aggregation,
    Custom Error,
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

이번에는 커스텀 에러 처리에 대해 작성해보겠습니다!<br>
어떤 에러가 발생했는지 한 눈에 확인할 수 있다는 점이 아주 매력적입니다 🤓<br>
그리고 한 번 작성하면 JWT 설정처럼 복사..붙여넣기...(ㅎㅎ) 할 수 있다는 아주 큰 장점이 있습니다<br>

<hr>

## 에러를 굳이 커스텀해야 하나요?

### _네!_

1. 표즌 예외를 사용할 때보다 더 자세하게 어떤 이유에서 에러가 발생했는지 알 수 있습니다.
2. 로그를 살펴보지 않아도 어떤 문제가 발생한 것인지 알 수 있습니다.
3. 반복되는 예외 처리 구문을 객체화 할 수 있습니다.
4. 한 번 해보세요. ~~아니 진짜 맛있는 예외 처리를 안 해봐서 그래~~

## 어떻게 해요?

`@ControllerAdvisor` 와 `@ExceptionHandeler` 를 이용하면 전역 또는 원하는 에러가 발생하였을 때, 적절한 예외 처리를 해줄 수 있습니다.<br>
[공식 문서 1](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html), [공식 문서 2](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)

표준 예외를 사용해서 예외처리를 하는 방식은 `throw new RuntimeException()` 을 작성하면 됩니다.<br>

커스텀 한 예외도 똑같이 저 방식으로 사용할 수 있도록 코드를 작성하겠습니다!<br>

<hr>

### ControllerAdvisor.java

```java
@RestControllerAdvice
@RequiredArgsConstructor(access = AccessLevel.PROTECTED)
@Slf4j
public class ControllerAdvisor {
    private final MessageSource messageSource; // 다중 언어로 에러처리를 하고 싶을 때 사용합니다!

    // 전반적인 에러를 처리 할 함수입니다.
    @ExceptionHandler
    protected ResponseEntity<ErrorResponse> handleException(CustomException e, Locale locale) {
        log.error("Exception: {}", e.getMessage(), e);

        // 다중 언어를 적용하지 않으시는 분들은 messageSource와 locale을 매개변수로 사용하지 않는 함수를 작성하시면 됩니다.
        ErrorResponse response = ErrorResponse.of(e, messageSource, locale);

        return ResponseEntity.status(e.getStatus()).body(response);
    }

    // WebFlux 기반으로 동작하는 서버는 "BadRequest(클라이언트가 잘못된 입력값을 전달했을 때)"에 대한 오류가 발생하면 WebExchangeBindException이 발생합니다.
    // MVC에서는 다른 exception class에서 처리됩니다.
    @ExceptionHandler(WebExchangeBindException.class)
    public ResponseEntity<ErrorResponse> webClientException(WebExchangeBindException e, Locale locale) {
        log.error("Validation Exception: {}", e.getMessage());

        // 다중 언어를 적용하지 않으시는 분들은 messageSource와 locale을 매개변수로 사용하지 않는 함수를 작성하시면 됩니다.
        ErrorResponse response = ErrorResponse.of(e, messageSource, locale);
        return ResponseEntity.badRequest().body(response);
    }
}
```

저는 에러 메시지를 다중 언어를 지원하여 생성하고자 했기 때문에 MessageSource를 이용했지만,<br>
굳이 다중 언어를 지원하고 싶지 않으신 분들은 적절히 필요 없는 함수나 매개변수를 지우시면 됩니다.<br>

<hr>

### ErrorResponse.java

```java
@Getter
@Builder
public class ErrorResponse {
    private final String trackingId;
    private final LocalDateTime timestamp;
    private final int status;
    private final String code;
    private final String message;

    public static ErrorResponse of(CustomException e, MessageSource messageSource, Locale locale) {
        return ErrorResponse.builder()
                .trackingId(UUID.randomUUID().toString())
                .timestamp(LocalDateTime.now())
                .status(e.getStatus().value())
                .code(e.getCode())
                .message(e.getLocalizedMessage(messageSource, locale))
                .build();
    }

    public static ErrorResponse of(WebExchangeBindException e,
                                   MessageSource messageSource, Locale locale) {
        return ErrorResponse.builder()
                .trackingId(UUID.randomUUID().toString())
                .timestamp(LocalDateTime.now())
                .status(e.getStatusCode().value())
                .code(e.getClass().getSimpleName())
                .message(Arrays.toString(e.getDetailMessageArguments(messageSource, locale)))
                .build();
    }
}
```

여기도 마찬가지로 `getLocalizedMessage` 함수가 다중 언어를 지원하기 위해 만들어진 함수이기 때문에,<br>
`e.getMessage()` 함수 등을 상황에 맞게 적절히 사용하시기 바랍니다!<br><br>

> ### 생성자를 사용해도 되는데 `of` 함수를 사용한 이유?
>
> 코드에서도 볼 수 있다시피, 함수를 사용하면 생성자를 사용할 때 보다 전달해야 하는 변수를 마음대로 설정할 수 있다는 이점이 있습니다.<br>
>
> 그래서 어떤 변수를 어떤 순서대로 넣어줘야하는지 체크하고, 만약 매개변수의 자료형이 겹친다면 순서가 굉장히 중요해지는데<br>
> 여기서 순서를 잘못 입력하게 되면, 디버깅을 하기 굉장히 어려워지기 때문에 함수를 사용해서 객체를 생성했습니다.

<hr>

### CustomException.java

```java
@Getter
public class CustomException extends RuntimeException {

    private final ErrorCode errorCode;
    private final Object[] args; // 다중 언어를 지원하지 않을 예정이라면 필요 없는 변수입니다!

    public CustomException(ErrorCode errorCode, Object... args) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
        this.args = args;
    }

    public CustomException(Throwable e, ErrorCode errorCode, Object... args) {
        super(e.getMessage());
        this.errorCode = errorCode;
        this.args = args;
    }

    public HttpStatus getStatus() {
        return errorCode.getStatus();
    }

    public String getCode() {
        return errorCode.name();
    }

    // 다중 언어를 지원하지 않을 예정이라면 필요 없는 함수입니다!
    public String getLocalizedMessage(MessageSource messageSource, Locale locale) {
        return messageSource.getMessage(errorCode.name(), args, locale);
    }
}
```

`super()` 를 이용해서 메시지를 주입합니다<br>
이 코드가 없으면 메시지가 정상적으로 저장되지 않아 메시지로 `null` 이 출력됩니다.<br>

<hr>

### ErrorCode.java

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_SERVER_ERROR"),
    UNKNOWN_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "UNKNOWN_ERROR"),
    INVALID_TOKEN(HttpStatus.UNAUTHORIZED, "INVALID_TOKEN");
    // [Enum 변수]([상태 코드], [에러 메시지]) 형식으로 커스텀 할 수 있습니다!
    // 다중 언어를 지원하지 않는 경우에는 'INVALID_TOKEN' 같은 형식 보다는 '토큰이 유효하지 않습니다' 와 같은 형식으로 작성해주세요

    private final HttpStatus status; // 상태 코드
    private final String message; // 에러 메시지
}
```

이 class를 통해 모든 에러 메시지를 한꺼번에 관리할 수 있습니다.<br>

> [에러 메시지가 다중 언어를 지원하도록 만들기!](https://shinsunyoung.tistory.com/79)
>
> messages.properties, messages_ko_KR.properties 등에 [에러 메시지]="[나타내고 싶은 메시지]"를 작성하면 자동으로 지역에 맞게 메시지가 변합니다.
>
> Spring 3 에서는 따로 MessageSourceConfig와 관련된 클래스를 작성하지 않아도 자동으로 파일을 찾아준다고 합니다.!

<hr>

### 이제 에러 코드를 커스텀 해봅시다!

`ErrorCode` 에 미리 작성해둔 커스텀 에러를 반환하는 클래스를 만들면 표준 예외로 예외처리를 할 때와 똑같이 예외 처리가 가능합니다!!<br>

<hr>

#### InvalidTokenException.java

```java
public class InvalidTokenException extends CustomException {
    public InvalidTokenException() {
        super(ErrorCode.INVALID_TOKEN);
    }
}
```

이렇게 적어주고, 사용할 때에는

```java
if (token == null || !header.startsWith("Bearer ")) {
    throw new InvalidTokenException();
}
```

이렇게!<br>

<hr>

## 이거 하면 어떻게 되는데요?

![Custom-Error](/assets/img/custom-error.png)

짜잔~~<br>

이제 로그를 보지 않아도 뭐가 문제인지 알 수 있게 되었습니다!<br>
디버깅하기 편하려면 커스텀 예외를 더더 많이 만들고 메시지를 더 자세하게, 코드를 더 명확하게 작성하는 것이 좋을 것 같습니다<br>

## 마치며

언젠가는 스스로 커스텀 에러를 꼭 만들고 말겠다는 야망이 있었는데,<br>
스프링과 자바에 어느정도 익숙해지고 나니 쪼금은 쉽게 느껴졌습니다 ㅎㅎ<br>

디버깅이 한층 더 편해진 평화로운 코딩 생활 하세용 ~.~<br>

😶‍🌫️

<hr>
<br>

> 참고 자료

[하나](https://blog.dukcode.org/spring/spring-custom-exception-and-exception-strategy/), [둘](https://dev.gmarket.com/66), [셋](https://mangkyu.tistory.com/205), [넷](https://mangkyu.tistory.com/204), [다섯](https://tecoble.techcourse.co.kr/post/2020-08-17-custom-exception/), [여섯](https://www.bezkoder.com/spring-boot-controlleradvice-exceptionhandler/)
