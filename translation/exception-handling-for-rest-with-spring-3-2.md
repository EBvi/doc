
# 스프링 3.2에서 REST하게 예외 처리하기

- by Eugen Paraschiv (2013/2/15)
- 원문은 [Java code geeks](http://www.javacodegeeks.com/2013/02/exception-handling-for-rest-with-spring-3-2.html
)에서 확인할 수 있습니다

## 1. Overview

이 문서는 **스프링의 REST API를 사용하여 예외를 다루는 방법**에 대해 살펴볼 것입니다. 먼저 스프링 3.2 이전의 구식 해법을 보고 나서 새롭게 지원하는 스프링 3.2의 방법을 살펴볼 것입니다.

이 문서의 목표는 HTTP 상태 코드로 응용 프로그램에서 예외를 매핑하는 걸 보여주는 것입니다. 각 시나리오에서 어떤 상태 코드들이 적합할지, 또는 REST 오류 표현의 문법은 이 문서의 범위에 포함되어 있지 않습니다.

스프링 3.2 이전에 스프링 MVC 응용 프로그램에서 예외를 처리하려면 *HandlerExceptionResolver*와 *@ExceptionHandler* 어노테이션을 썼었습니다. 스프링 3.2에서는 이전의 한계를 해결하기 위해 새롭게 **@ControllerAdvice** 어노테이션이 추가되었습니다.

이러한 것들은 모두 한가지 공통점이 있는데, 표준 응용 프로그램 코드가 실패했다는 걸 보여주는 예외를 잘 던질 수 있도록, **관심사의 분리(Separation of Concerns:SoC)**가 잘 되어 있다는 점입니다.

## 2. Via Controller level @ExceptionHandler

컨트롤러 단에서 *@ExceptionHandler*로 메서드를 정의하기란 매우 쉽습니다.

```
public class FooController{
    ...
    @ExceptionHandler({ CustomException1.class, CustomException2.class })
    public void handleException() {
        //
    }
}
```

나쁘지 않지만, 이러한 방법은 한 가지 큰 문제(drawback)를 가지고 있습니다. *@ExceptionHandler* 어노테이션을 사용하는 방법은 전역적인 전체 응용 프로그램을 대상으로 하지 않고 **오직 특정(particular) 컨트롤러만 사용**할 수 있습니다. 물론 이것은 예외를 다루는 일반적인 매커니즘으로는 적합하지 않습니다.

이런 문제는 **Base Controller 클래스를 확장한 모든 컨트롤러**가 동일하게 갖고 있습니다. 그러나 이유가 뭐든 컨트롤러는 클래스로부터 확장할 수 없으므로 응용 프로그램에서 문제가 될 수 있습니다. 컨트롤러는 이미 다른 기본 클래스로부터 상속을 받았으며, jar와 같은 형태로 묶여 있으므로 직접 수정할 수 없을 것입니다.

그럼 다음으로, 우리는 전역적이거나 기존 artifact에 어떠한 변경도 가하지 않게 예외 처리 문제를 해결하는 또 다른 방법을 살펴볼 것입니다.

## 3. Via HandlerExceptionResolver

REST API에서 **일정한(uniform) 예외 처리 매커니즘**을 구현하기 위해서, 우리는 *HandlerExceptionResolver*를 작동할 필요가 있습니다. 이것은 응용 프로그램이 실행되고 있는 중(runtime)에 던져지는 예외를 해결할 것입니다. custom resolver를 보기 전에, 기존 구현을 살펴봅시다.

#### 3.1 ExceptionHandlerExceptionResolver

이 resolver는 스프링 3.1에 추가되었고, DispatcherServlet에서 기본적으로 활성화되어 있습니다. 실제적으로 @ExceptionHandler 매커니즘은 지금까지의 작업을 대표하는 핵심 컴퍼넌트입니다.

#### 3.2. DefaultHandlerExceptionResolver

이 resolver는 스프링 3.0에 추가되었고, *DispatcherServlet*에서 기본적으로 활성화되어 있습니다. *400번대*의 클라이언트 오류와 *500번대*의 서버 오류 상태 코드들과 일치하는 HTTP 상태 코드를 표준 스프링 예외로 지정하는 데 사용합니다. 스프링 예외가 다루는 모든 내역과 상태 코드를 매핑하는 방법은 [여기](http://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/mvc.html#mvc-ann-rest-spring-mvc-exceptions)에서 확인할 수 있습니다.

이 resolver는 제대로 응답의 상태 코드를 설정하는 반면에, 응답의 본문(body)에는 어떠한 것도 설정하지 않습니다. 그러나 REST API의 컨텍스트에서 상태 코드는 클라이언트의 모든 정보를 나타내는데 **충분하지 않습니다.** 응답(response)은 실패 원인에 대한 추가적인 정보를 제공하는 응용 프로그램의 본문을 잘 가지고 있어야 합니다.

이것은 View resolution과 *ModelAndView*를 통해 오류 콘텐츠를 렌더링하도록 설정하는 것으로 해결할 수는 있습니다만, 스프링 3.2에서 제공하는 옵션에 비하면 그다지 좋은 해결책이 아닙니다. 우리는 이 문서 후반에 그것을 이야기할 것입니다.

#### 3.3. ResponseStatusExceptionResolver

이 resolver 또한 스프링 3.0에 추가되었고, DispatcherServlet에서 기본적으로 활성화되어 있습니다. 이것은 custom 예외와 이러한 예외들을 HTTP 상태 코드로 매핑할 수 있도록 **@ResponseStatus** 어노테이션을 사용할 수 있게 합니다.

custom 예외는 이런 식으로 보일 겁니다.

```
@ResponseStatus(value = HttpStatus.NOT_FOUND)
public final class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException() {
        super();
    }
    public ResourceNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
    public ResourceNotFoundException(String message) {
        super(message);
    }
    public ResourceNotFoundException(Throwable cause) {
        super(cause);
    }
}
```

*DefaultHandlerExceptionResolver*와 동일하게 이 resolver가 응답의 본문을 다루는 방식은 **제한**됩니다. 응답의 상태 코드를 매핑하지만 본문은 여전히 null입니다.

#### 3.4. SimpleMappingExceptionResolver and AnnotationMethodHandlerExceptionResolver

SimpleMappingExceptionResolver는 꽤 오랫동안 사용되어 왔습니다. 꽤 오래전의 Spring MVC 모델에서 나왔으며 **REST 서비스와 전혀 관련이 없습니다.** 이 resolver는 **뷰** 이름을 예외 클래스를 매핑하는데 사용합니다.

*AnnotationMethodHandlerExceptionResolver*는 스프링 3.0에 추가되었고, *@ExceptionHandler* 어노테이션을 통해 예외를 다룰 수 있습니다. 그러나 스프링 3.2에서는 *ExceptionHandlerExceptionResolver*에 의해 **deprecated** 되었습니다.

#### 3.5. Custom HandlerExceptionResolver

*DefaultHandlerExceptionResolver*와 *ResponseStatusExceptionResolver*의 조합은 스프링 RESTful 서비스에서 제법 좋은 오류 핸들링 매커니즘이라고 볼 수 있습니다. 하지만 응답의 본문을 전혀 제어하지 않기 때문에 **새로운 예외 resolver**를 만들게 하는 문제가 있습니다.

그래서 새로운 resolver의 목표는 더 정보가 많은 응답 본문의 설정이 가능하도록 하고, 또한 클라이언트에 의해 요청된 표현 방식을 따르도록 하는 것입니다. (Accept 헤더에 기술된 대로)

```
@Component
public class RestResponseStatusExceptionResolver extends AbstractHandlerExceptionResolver {

    @Override
    protected ModelAndView doResolveException
      (HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof IllegalArgumentException) {
                return handleIllegalArgument((IllegalArgumentException) ex, response, handler);
            }
            ...
        } catch (Exception handlerException) {
            logger.warn("Handling of [" + ex.getClass().getName() + "]
              resulted in Exception", handlerException);
        }
        return null;
    }

    private ModelAndView handleIllegalArgument
      (IllegalArgumentException ex, HttpServletResponse response) throws IOException {
        response.sendError(HttpServletResponse.SC_CONFLICT);
        String accept = request.getHeader(HttpHeaders.ACCEPT);
        ...
        return new ModelAndView();
    }
}
```

여기서 주목할 점은 request를 그대로 쓸 수 있으므로, 응용 프로그램은 클라이언트에 의해 보내진 *Accept* 헤더의 값을 고려할 수 있다는 점입니다. 예를 들어, 오류 형식으로 클라이언트가 *application/json*으로 요청한다면, 응용 프로그램은 *application/json*으로 인코딩한 응답 본문을 뿌려줄 것입니다.

다른 중요한 구현 상세로 *ModelAndView*를 반환한다는 점입니다. 이것은 **응답 본문**이나 필요한 것이 무엇이든 설정하면 응용 프로그램이 그걸 허용한다는 뜻입니다.

이러한 스프링 REST 서비스의 접근법은 오류 핸들링을 일관성 있고 쉽게 구성할 수 있도록(configurable) 합니다. 그러나 저수준의 *HtttpServletResponse*으로 상호 작용하고, *ModelAndView*를 사용하는 구식 MVC 모델로 맞춰져 있다는 점에서 **한계**가 있습니다. 물론 개선의 여지는 있습니다.

## 4. Via new @ControllerAdvice (Spring 3.2 Only)

**스프링 3.2**는 *@ControllerAdvice* 어노테이션으로 전역적인 *@ExceptionHandler*에 대한 지원을 제공합니다. 이것은 *@ExceptionHandler*의 타입 안정성과 유연함을 그대로 유지하면서 *ResponseEntity*를 활용하고 구식 MVC 모델로부터 벗어날 수 있습니다.

```
@ControllerAdvice
public class RestResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(value = { IllegalArgumentException.class, IllegalStateException.class })
    protected ResponseEntity<Object> handleConflict(RuntimeException ex, WebRequest request) {
        String bodyOfResponse = "This should be application specific";
        return handleExceptionInternal(ex, bodyOfResponse,
          new HttpHeaders(), HttpStatus.CONFLICT, request);
    }
}
```

이 새로운 어노테이션은 **여기저기 흩어져 있는 하나의 전역 오류 핸들링 컴포넌트**로 통합하기 전의 *@ExceptionHandler*를 가능하게 합니다.

실제 매커니즘은 매우 간단하지만 정말 유연합니다.

- 이것은 상태 코드뿐만 아니라 응답 몸체를 완전히 제어할 수 있습니다
- 함께 처리하거나 같은 방법으로 예외들을 매핑할 수 있습니다
- RESTful *ResponseEntity* 응답을 더 잘 쓸 수 있게 해줍니다

## 5. Conclusion

스프링 REST API의 예외 처리를 구현하는 매커니즘에 대해 이야기했습니다. 실제 REST 서비스에서 예외 처리 매커니즘으로 작업한 구현체를 확인하려면, [github 저장소](https://github.com/eugenp/tutorials/tree/master/spring-security-rest-full#readme)를 확인하시기 바랍니다.

