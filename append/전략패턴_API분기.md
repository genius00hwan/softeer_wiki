# 전략패턴기반 API 분기 설계


## 배경

HTTP 요청을 처리하면서 URL path와 method에 따라 다른 동작을 실행해야 하는 경우는 백엔드 개발의 가장 기본적인 문제다.  
처음에는 조건문으로 시작했지만, API가 많아질수록 유지보수성과 확장성에 문제가 있을거 같았다.

- 분기 처리 코드가 더러워진다.
- 각 기능별 책임이 분리되지 않는다.
- 새로운 API를 추가할 때마다 기존 코드를 수정하게 되면 OCP를 위반하게 된다.

이에, 전략 패턴 기반으로 분기 처리하게 했다.

---

## 설계 구조

### 구성 요소

- `Dispatcher`: 요청을 받아 해당 도메인의 `AppServlet`으로 위임
- `AppServlet`: 도메인별 진입점 `AppFacade` 의 요청별 메소드 호출
- `AppFacade`: 요청 경로와 method에 따라 처리할 `BiFunction` 전략을 보유
- `RouteKey`: `(Path, Method)` 조합을 키로 사용
- `routeMap`: `Map<RouteKey, Function<HttpRequest, HttpResponse>>`
- `Function`: 각 API 요청을 처리할 함수 전략

### 흐름

요청 -> Dispatcher -> AppServlet -> AppFacade ->  routeMap -> Function 실행 -> 응답 생성

### Why?

OCP를 만족하는 확장성 있는 구조

- 새로운 API를 추가할 때 routeMap.put()만 하면 되므로 기존 코드를 수정할 필요가 없다.
- 도메인별 라우팅 책임이 AppFacade에 모여 있어 코드 흐름이 직관적이다.

비즈니스 로직과 라우팅을 분리

- 라우팅 조건과 실행 로직이 명확히 분리되어, 각 API의 책임을 이해하고 테스트하기 쉽다.

함수형 인터페이스의 장점 활용

- Function을 통해 요청과 응답 사이의 로직을 전략으로 만들어 테스트와 주입이 용이하다.

### 다른 방법은?

#### if / switch 기반 분기

- 장점: 간단하고 빠르게 구현 가능

- 단점: 

API가 많아질수록 분기문이 비대해지고 가독성이 떨어진다.
확장성과 재사용성, 테스트 용이성이 부족하다.

#### 어댑터 패턴 기반 구조 (Spring의 방식)

- 다양한 Handler 타입(JSON, HTML, 파일 응답 등)을 유연하게 처리 가능

- Dispatcher는 Handler의 supports()와 handle()만 호출하면 되므로 구조적으로 매우 깔끔함

- 새로운 핸들러가 생겨도 기존 Dispatcher 수정 없이 어댑터만 추가하면 됨.



#### 어노테이션 기반 라우팅 (Spring Boot 방식)

Spring은 `@RequestMapping`, 등의 어노테이션을 통해 요청 경로와 메서드를 선언적으로 정의할 수 있도록 지원한다.

- 라우팅 정보가 명시적으로 드러나고, 자동 등록 가능

- 외부에서 라우팅을 정의하므로 유지보수와 자동화에 유리

- Reflection, ComponentScan 을 공부해보자.



## 더 고민해볼 지점

### 애플리케이션 레벨 Request,Response 객체 

현재 구조는 `Function<HttpRequest, HttpResponse>`에 의존하고 있어 비즈니스 로직에서 다루는 데이터와 HTTP 메타정보가 섞여 있는 상태다.

예를 들어 회원가입을 처리하는 로직이 HttpRequest에서 직접 query param을 꺼내는 방식은

- 요청 유효성 검증 책임이 흐려진다.
- 도메인 계층에서 content-type, method 등 HTTP 요소에 관여하게 된다.

이런 문제가 있다.

### HandlerAdapter 추상화

현재는 `Function<HttpRequest, HttpResponse>` 으로 분기 처리를 수행하고 있다.

- JSON API 전용 핸들러
- 예외 처리 전용 핸들러

등을 만들고 싶다면 `HandlerAdapter` 추상화를 도입해야 한다.

Dispatcher는 어떤 핸들러든 supports()가 true인 Adapter만 찾으면 되고, handle()은 어댑터가 책임지므로 다형성이 생긴다.

- 다양한 핸들러를 같은 인터페이스로 다룰 수 있음
- OCP 만족: 새로운 타입이 추가될 때 기존 Dispatcher 수정 불필요
- 공통 흐름 처리 및 테스트 분리 용이

---

## Spring은 위대하다.

Spring은 많은 개발자들이 겪는 반복적이고 확장에 취약한 구조들을 해결하기 위해, 구조적 분리와 추상화, 그리고 유연한 어댑터 패턴을 도입해싿.

Function 기반 전략 패턴도 충분히 학습 가치가 있었지만, Spring의 설계 철학을 느끼게 되었다.