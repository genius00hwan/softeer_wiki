# DIContainer 개선 회고

## 문제

### 발생한 오류
- `java.lang.IllegalStateException: Recursive update` 예외가 발생했다.
- 싱글톤 객체 생성 중, 동일 객체를 다시 요청하는 상황이 고려되지 않았다.
- `computeIfAbsent`는 초기화 중 같은 key가 다시 요청되면 이를 "중복 업데이트"로 간주하고 예외를 던진다.

### 상황 요약
- `StaticServlet` 생성 도중 `AuthSession` 싱글톤이 필요했고,
- `AuthSession`도 초기화 중 다른 의존성으로 인해 `getInstance`가 다시 호출되면서 충돌이 발생했다.

```java
return (T) singletonInstances.computeIfAbsent(clazz, c -> {
    log.info("[DI] 싱글톤 생성 시작: {}", c.getName());
    return createInstance(c); // 초기화 중 getInstance 재호출 -> Recursive update 발생
});
```
---

## 해결

### 해결 방안
- 싱글톤 생성이 완료되기 전이라도 Map에 먼저 등록하여 중복 초기화를 방지했다.
- Spring처럼 생성 중임을 표시하는 placeholder를 두고, 초기화 완료 후 이를 실제 인스턴스로 교체하는 방식으로 수정했다.

```java
private static final Object PLACEHOLDER = new Object();

if (clazz.isAnnotationPresent(Singleton.class)) {
    if (singletonInstances.containsKey(clazz)) {
        return (T) singletonInstances.get(clazz);
    }

    singletonInstances.put(clazz, PLACEHOLDER); // 초기화 중임을 먼저 등록

    T instance = createInstance(clazz);
    singletonInstances.put(clazz, instance);

    return instance;
}

```
### 결과
- 초기화 중 동일 싱글톤 요청이 들어와도 `Recursive update`가 발생하지 않게 되었다.
- DIContainer가 순환 의존이나 초기화 중 의존성을 보다 안전하게 처리할 수 있게 되었다.

---

## 배운 점

### DI 설계의 핵심
- 싱글톤 생성은 동시 요청과 초기화 순서 문제를 전제로 설계해야 한다.
- 초기화 상태 관리(placeholder나 프록시)가 필수적이다.

### `computeIfAbsent`의 한계
- 단순한 원자적 초기화에는 적합하지만, 초기화 중 동일 key가 다시 요청되는 DI 환경에는 적합하지 않다.

### Spring DI의 철학
- Spring은 Bean 생성 시 먼저 등록 후 초기화하는 전략을 사용한다.
- 이번 개선으로 DI에서 초기화 시점 관리가 설계의 중요한 부분임을 배웠다.

