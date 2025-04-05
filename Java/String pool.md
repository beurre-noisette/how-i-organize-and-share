## 개요

얕은 복사(Shallow copy)와 깊은 복사(Deep copy)에 대해서 공부하던 중 Java의 String Pool에 대해 알게 되었다. 먼저 아래의 코드를 살펴보자.

```Java
String str1 = "hello";
String str2 = "hello";
String str3 = new String("hello");

System.out.println(str1 == str2);
System.out.println(str1 == str3);
System.out.println(str1.equals(str3));
```

각 프린트의 결과와 이유는 다음과 같다.

## 1번 프린트의 결과 `true`
Java에서 `==`연산자는 원시타입일 경우 값 자체를 비교하고 참조타입일 경우에는 메모리 주소를 비교한다. 여기서 `str1`과 `str2`는 `String`(참조타입)이므로 메모리 주소를 비교하는데, 언뜻 보면
값만 동일한 서로 다른 객체를 비교하는 것 처럼 보인다. 여기서 `Spring Pool`의 개념을 알아보자. 위 상황에서 JVM은 `str1`이 선언되고 초기화 될 때 `String Pool`에 `"hello"`라는
문자열이 있는지 확인하고 없다면 새로 생성하여 `String Pool`에 저장하고 `str1`은 이 주소를 참조하게 한다. 다음 `str2`가 선언되고 초기화 될때 앞선 동작과 마찬가지로 `String Pool`에
`"hello"`라는 문자열이 있는지 확인하는데, 이때 `"hello"`가 존재하므로 JVM에 의해 `str2`는 기존 문자열(여기서는 `str1`)을 참조하게 된다.
이와 같은 이유로 1번 프린트는 서로 같은 참조값을 갖기 때문에 `true`로 판별된다.

## 2번 프린트의 결과 `false`
`str3`은 `str1`과 `str2`와 달리 리터럴 방식이 아닌 new 키워드를 통한 생성자 호출 방식으로 객체를 선언하고 초기화 하였다. 이러한 방식으로인해 `str3`는 `String Pool`을 사용하지
않고 힙 메모리에 새로운 문자열 객체를 생성하게 되고 결과적으로 서로 다른 참조값을 비교하기 때문에 `false`로 판별된다.

## 3번 프린트 결과 `true`
`.equals()`메서드는 두 객체의 '값'을 비교하는 메서드이다. 따라서 `true`로 판별된다.

`String Pool`을 이용하면 동일한 문자열을 여러 번 저장하지 않아도 된다는 점에서 **메모리 효율성**이 좋아지고 문자열 비교 시 실제 값을 비교하지 않고 참조값만 비교해도 되는 경우에는 **성능 향상**의
이점을 얻을 수 있다.