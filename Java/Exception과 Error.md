### 문제상황

파이썬을 공부하던 중 `try-except` 구문에 대해서 배우는 시간을 가졌다. 자바의 `try-catch` 구문의 동작과 유사하게 `try` 절에서 실행한 코드가 **문제**가 있을 경우 `except` 절에서
해당 **문제**를 `handling` 하는 구조이다. 그런데 여기서 `epcept` 절에서 잡은 **문제**가 `SyntaxError`였고 '여기서 `Error`를 왜 `except` 절에서 처리하지? 자바에서는
`Exception`과 `Error`를 구분하였고, 보통 `Exception`에 대한 핸들링을 설계했었는데...' 라는 의문을 갖게 되었다.

### Exception과 Error는 뭐가 다른데?

의문을 쫓다보니 가장 먼저 마주한 질문은 '`Exception`과 `Error`는 무엇이고 어떤 차이가 있는가?' 였다. 가장 먼저 떠오른 답은 **codable 하게 처리할 수 있다면 Exception이고
codable하게 처리할 수 없다면 Error** 였다. 그러나 이 답은 ==모호한==답이라는 것이라는 걸 이내 알게 되었다. 과연 **codable한 처리의 유무는 어떻게 나눌 수 있는가?**

다시 천천히 `Exception`과 `Error`에 대해서 정리를 해보자. Java에서 `Exception`과 `Error`는 `Throwable` 객체를 상속받은 클래스이다. 이 두 클래스(`Exception`과
`Error`)의 존재의의는 **문제 상황**을 어떻게 처리하느냐에 달려있다.

`Error`는 **프로그램 외적으로 프로그래머가 컨트롤 할 수 없는 예기치 못한 상황**을 정의한다. 대표적으로는 memory와 관련한 `Error`들(`StackOverflowError`)이나
`VirtualMachinedError`등이 있다. 우리는 이러한 상황을 **code**로 해결하기 어렵기 때문에 흔히 **codable하게 처리할 수 없는 것이 Error**라고 말한다.

`Exception`은 **프로그램 내부적으로 프로그래머가 컨트롤 할 수 있는 상황**을 정의하고, 다시 **Checked Exception**과 **Unchecked Exception**으로 구분할 수 있다. *
*Checked Exception**은 프로그램이 `complie`시점에 처리를 강제하는 `Exception`을 의미한다. 다시 말해서 JVM 외적인 상황으로 인한 `exception`들을 지칭하고 대표적으로는
SQL 관련 예외들이 있다. **Unchecked Exception**은 프로그램이 `compile`은 되었고, 코드가 동작하다가 특정 부분에서 의도치 않은 **문제 상황**을 겪는 `Exception`을 의미한다.
이러한 **Unchecked Exception**은 대표적으로 `NullPointerException` 등이 있다(프로그램이 컴파일 되어 동작하던 중 특정 부분에서 변수가 가르키는 메모리의 주소값을 갔더니 값이
null인 경우!). **Unchecked Exception**은 설계의 단계에서 프로그래머가 문제 상황을 예측하고(마치 문제가 일어날 것을 알고 있었던 것 마냥) 해당 문제 상황을 어떻게 해결할지(=핸들링할지)
정의할 수 있고 이러한 관점에서 설계의 영역으로 생각해볼 수도 있다.

앞서 살펴본 **Checked Exception**과 **Unchecked Exception**은 꼼꼼한 설계를 바탕으로 프로그래머가 **문제 상황을 예측하고 문제 상황을 핸들링 할 수 있다.** 다시 말해서 *
*codable 하게 처리할 수 있다.** 이러한 정리를 바탕으로 **Error는 codable한 해결이 힘든 영역, Exception은 codable하게 해결할 수 있는 영역**이라고 정리할 수 있는 것이다.

### 그래서?

`Exception`과 `Error`이 어떤 기준으로 구분되고 또 어떻게 대처해야 하는지에 대해서 알 수 있었다. 특히 인상적이었던 배움은 좋은 프로그래머는 `Exception` 상황을 설계의 영역에서 예측하고 이를
핸들링 할 수 있어야 한다는 점이다. 마치 '후후 네가 이렇게 행동할 줄 알았어 네 선택이 이렇다면 나는 이렇게 대응할거야!'의 경지에 올라야 한다는 것이다.

또한 우리는 **Error는 codable하게 처리할 수 없다.** 라는 말을 **Error가 발생하지 않도록 예방하는 코드를 작성할 수 없다.** 로 이해하기 보다는 **Error가 한 번 발생하면 프로그램
수준에서 그것을 적절히 복구하고 정상적인 흐름으로 돌아가는 것이 현실적으로 매우 어렵다.** 고 이해할 수 있다.