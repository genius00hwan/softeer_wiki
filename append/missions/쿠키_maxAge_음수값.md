# 쿠키 `Max-Age` 음수값에 대한 의문과 정리

## WAS 구현 중 생긴 의문

WAS를 직접 구현하면서 쿠키를 다루다 보니, 세션 쿠키를 디폴트로 만들기 위해 `max-age`를 음수값으로 설정했다.  
하지만 브라우저에서 확인해보니 쿠키가 아예 존재하지 않았다.

코드를 살펴봤지만 문제를 못찾았고, RFC 문서를 찾아보았다.

---

## RFC 6265

[RFC 6265 §4.1.2.2](https://datatracker.ietf.org/doc/html/rfc6265#section-4.1.2.2)에는 다음과 같이 정의돼 있다.

> If the attribute-value is less than or equal to zero (0), let expiry-time be the earliest representable date and time.

즉, `Max-Age`가 0 이하이면 즉시 만료되도록 규정하고 있다.  
내가 음수를 넣었기 때문에 브라우저는 쿠키를 바로 만료시킨 것이다.

### 나는 왜 음수값으로 설정했나?

Spring `ResponseCookie`를 사용해 로그인 기능을 만들 때, 세션 쿠키를 위해 `maxAge`를 -1로 정의했고, 이를 표준적인 방식이라고 생각했었다.

> A negative value results in no "Max-Age" attribute in which case the cookie is removed when the browser is closed


`org.springframework.http.ResponseCookie`

```java
/**
 * Set the cookie "Max-Age" attribute.
 *
 * <p>A positive value indicates when the cookie should expire relative
 * to the current time. A value of 0 means the cookie should expire
 * immediately. A negative value results in no "Max-Age" attribute in
 * which case the cookie is removed when the browser is closed.
 */
ResponseCookieBuilder maxAge(Duration maxAge);
```
---

## Spring은 왜 이렇게 모호하게 만들었을까?

사실 Spring의 잘못은 아니다.
Spring은 `javax.servlet.http.Cookie` 의 표준 설계를 그대로 따르고 있다.

```java
/**
 * Sets the maximum age of the cookie in seconds.
 * A positive value indicates that the cookie will expire after that many seconds have passed.
 * A negative value means that the cookie is not stored persistently and will be deleted
 * when the Web browser exits.
 * A zero value causes the cookie to be deleted.
 */
public void setMaxAge(int expiry){}
```

즉, 음수 = 세션 쿠키라는 해석은 자바에서 시작된 것이다.

---

## 톰캣의 실제 동작

톰캣도 이 자바 표준을 따른다.
하지만 톰캣은 RFC 표준을 맞추기 위해 음수값일 때 Max-Age를 아예 헤더에 포함하지 않는다.

톰캣 `org.apache.tomcat.util.http.Rfc6265CookieProcessor`

```java
@Override
public String generateHeader(cookie, request){
  //...
  if (maxAge > -1) {
      // Negative Max-Age is equivalent to no Max-Age
      header.append("; Max-Age=");
      header.append(maxAge);
  
      if (maxAge == 0) {
          header.append("; Expires=");
          header.append(ANCIENT_DATE);
      } else {
          COOKIE_DATE_FORMAT.get().format(new Date(System.currentTimeMillis() + maxAge * 1000L),
                                          header, new FieldPosition(0));
      }
  }
  
```

즉, `maxAge < 0`이면 `Max-Age`와 `Expires`를 모두 설정하지 않는다.
브라우저는 이를 세션 쿠키로 인식하며, 동작 자체는 RFC와 완전히 일치한다.

---

## 왜 이런 차이가 발생했을까?

자바 초기(1997년) 넷스케이프 표준 시절
- 당시 넷스케이프 쿠키 표준에서는 세션 쿠키 = Expires 미설정 방식이었다. 
- 자바는 명시적 API 설계를 선호했고, 세션 쿠키를 나타내기 위해 음수를 상태값으로 사용했다.

자바 초기의 언어적·설계적 제약
- `maxAge`는 원시 타입(int)으로 설계되었기 때문에 `null`을 사용할 수 없었다. 
- 당시에는 `Optional`이나 자동 박싱(autoboxing)도 없었고, 단순성을 위해 래퍼 타입(`Integer`) 대신 원시 타입을 선호했다. 
- 그래서 `>0` = 영속 쿠키, `0` = 즉시 만료, `<0` = 세션 쿠키라는 간단한 규칙이 채택되었다.

RFC 6265(2011) 이후
- RFC에서는 "세션 쿠키 = Max-Age 미설정"으로 명확히 규정. 
- 하지만 이미 자바와 톰캣, JBoss 등의 서버는 -1 관례를 사용하고 있었고, 동작상 문제도 없었기 때문에 그대로 유지되었다.


---

## 결론

처음엔 Spring의 처리 방식만 보고 의문을 제기했고, 자바와 톰캣의 모호함이 문제라고 생각했다.

다만 이걸 바꾸려면 방대한 자바 생태계 전반을 수정해야 하고, 이미 -1 관례는 자바 개발자들에게 사실상의 표준으로 자리 잡았다.
개발 표준은 결국 문서가 아니라 많은 사람들이 쓰면서 자연스럽게 굳어지는 것이니까.

**제일 중요한건 동작 자체에는 아무런 문제가 없다는 점이다.**

그래서 나는 이렇게 결론 내렸다.
의문을 제기하는 태도도 중요하지만, 기존 생태계가 만든 현실을 받아들이는 것도 필요하다.


