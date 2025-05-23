## 문제상황

오늘 비즈니스 로직을 디버깅하는 과정에서, 특정 메서드가 어떤 상황에서 호출되는지 추적할 필요가 있었다. 보통은 로그 메시지에 직접 메서드명을 하드코딩하는 방식을 사용했는데, 이는 메서드 이름이 변경될 때마다 로그 메시지도 함께 수정해야 하는 번거로움이 있었다.

또한 여러 메서드에 로깅을 추가할 때 복사-붙여넣기 과정에서 이전 메서드 이름을 그대로 사용하는 실수를 자주 범했다. 메서드명을 동적으로 가져와 로깅하는 방법이 필요했다.

## 해결방안

Java의 스택 트레이스(Stack Trace)를 활용하여 현재 실행 중인 메서드의 이름을 동적으로 가져오는 방법을 찾았다.

```java
@Slf4j
public class SomeService {
    
    public void businessMethod() {
        log.info("메서드 호출 = {}", Thread.currentThread().getStackTrace()[1].getMethodName());
        
        // 비즈니스 로직...
    }
}
```

이 코드는 현재 실행 중인 메서드의 이름(위 예시에서는 "businessMethod")을 로그에 출력한다. 메서드 이름이 변경되어도 로그는 항상 정확한 메서드명을 표시한다.

## 동작 원리

위 코드가 어떻게 동작하는지 단계별로 살펴봤다:

1. `Thread.currentThread()`: 현재 코드가 실행되고 있는 스레드 객체를 가져온다.
2. `.getStackTrace()`: 현재 스레드의 스택 트레이스를 배열 형태로 반환한다. 이 배열은 메서드 호출 체인을 나타내며, 각 요소는 `StackTraceElement` 타입이다.
3. `[1]`: 스택 트레이스 배열에서 인덱스 1에 위치한 요소를 선택한다.
4. `.getMethodName()`: 선택된 `StackTraceElement`에서 메서드 이름만 추출한다.

## 왜 `[1]`을 사용하는가?

스택 트레이스 배열의 인덱스는 중요한 의미를 갖는다:

- `[0]`: `getStackTrace()` 메서드 자체에 대한 정보를 담고 있다. 항상 내부 구현 메서드에 관한 정보이기 때문에 일반적으로 사용하지 않는다.
- `[1]`: **현재 실행 중인 메서드**의 정보를 담고 있다. 로그를 남기는 코드가 있는 바로 그 메서드이다.
- `[2]`: 현재 메서드를 호출한 메서드(호출자)의 정보이다.
- `[3]` 이상: 호출 체인을 거슬러 올라가는 상위 메서드들의 정보이다.

따라서 현재 실행 중인 메서드의 이름을 알고 싶을 때 `[1]`을 사용한다.

## 활용 예시

호출 체인을 추적하는 로그를 남길 때도 유용하다:

```java
private void complexMethod() {
    String currentMethod = Thread.currentThread().getStackTrace()[1].getMethodName();
    String callerMethod = Thread.currentThread().getStackTrace()[2].getMethodName();
    
    log.info("현재 메서드: {}, 호출한 메서드: {}", currentMethod, callerMethod);
    
    // 비즈니스 로직...
}
```

이렇게 하면 "현재 메서드: complexMethod, 호출한 메서드: someOtherMethod"와 같은 로그를 얻을 수 있다.

## 주의사항

1. **성능 영향**: `getStackTrace()`는 가벼운 연산이 아니다. 성능이 중요한 핫 경로(hot path)에서는 사용을 자제해야 한다.
2. **람다 표현식 내부**: 람다나 익명 클래스 내부에서 사용하면 예상치 못한 결과가 나올 수 있다.
3. **최적화 옵션**: 일부 JVM 최적화 옵션에 따라 스택 트레이스 정보가 달라질 수 있다.

## 결론

이 기법은 디버깅과 로깅을 위한 강력한 도구다. 특히 복잡한 호출 흐름을 추적하거나, 메서드 이름이 자주 변경되는 프로젝트에서 유용하다. 다만 성능에 민감한 부분에서는 사용을 지양하고, 꼭 필요한 디버깅 상황에서만 활용하는 것이 좋다.