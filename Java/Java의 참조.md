# Java의 객체 참조와 메서드 호출 방식 이해하기

## 1. 개요

Spring MVC 패턴의 컨트롤러-서비스-DAO 구조에서 특이한 코드 패턴을 발견했다. 서비스 메서드의 반환 값이 `void`인데도 불구하고, 컨트롤러에서 전달한 `Map` 객체에 서비스/DAO 계층에서 데이터가
채워져 반환되는 현상을 발견했다.

```java
// 컨트롤러
@PutMapping("/fax/getChkList")
@ResponseBody
public HashMap<String, Object> getChkList(Map<String, Object> modelMap, @RequestAttribute(value = "params") String datas) throws Exception {
    // modelMap에 값 설정
    // ...

    // 서비스 호출 - 반환값이 없는 void 메서드
    faxService.getChkList(modelMap);

    // 그런데 modelMap에 "chkList" 키가 추가되어 있음!
    HashMap<String, Object> resultMap = new HashMap<>();
    resultMap.put("chkList", modelMap.get("chkList"));
    return resultMap;
}

// 서비스
public void getChkList(Map<String, Object> modelMap) throws SQLException {
    if (modelMap.get("mapClsCd").equals("02")) {
        // DAO 결과를 modelMap에 저장
        modelMap.put("chkList", faxDAO.getChkList(modelMap));
    }
    // ...
}
```

이런 코드가 어떻게 작동할 수 있는지 이해하기 위해서는 Java의 메서드 호출 방식과 객체 참조에 대한 이해가 필요하다.

## 2. 궁금증 해결을 위한 배경지식: Java의 Pass by Value

### 2.1 Java의 기본 원칙: 항상 Pass by Value

Java에서는 **항상 Pass by Value(값에 의한 전달)** 방식만 사용한다. 이는 메서드 호출 시 인자의 "값"이 복사되어 전달된다는 의미다. 하지만 여기서 많은 개발자들이 혼동하는 지점이 있다(나도
여기가 혼란스러웠다).

1. **기본 타입(primitive types)**: 실제 값 자체가 복사됨 (int, boolean 등)
2. **객체 타입(reference types)**: 객체의 참조값(메모리 주소)이 복사됨

특히 객체 타입의 경우, 참조값을 복사해서 전달하기 때문에 마치 Pass by Reference처럼 보이는 효과가 있다. 하지만 정확히는 "참조의 값"을 전달하는 것이다.

### 2.2 참조값 변경 vs 객체 내용 변경

Java에서 객체를 메서드에 전달할 때 발생할 수 있는 동작을 이해하기 위해 두 가지 다른 상황을 살펴보자:

#### 2.2.1 객체 내용 변경 (원본 객체에 영향 O)

```java
public static void modifyContent(Person p) {
    p.setName("수정된이름");  // 참조를 통해 객체 내용 변경
}

public static void main(String[] args) {
    Person person = new Person("원래이름");
    modifyContent(person);
    System.out.println(person.getName());  // "수정된이름" 출력
}
```

이 경우 `modifyContent` 메서드는 참조를 통해 원본 객체의 내용을 변경한다. `p`라는 지역변수는 원본 객체를 가리키는 참조값을 복사받았기 때문에, 이 참조를 통해 객체에 접근하면 원본 객체가 변경된다.

#### 2.2.2 참조값 자체 변경 (원본 변수에 영향 X)

```java
public static void swapReference(Person p1, Person p2) {
    Person temp = p1;
    p1 = p2;    // 지역변수 p1의 참조값만 변경 (원본 변수에 영향 없음)
    p2 = temp;  // 지역변수 p2의 참조값만 변경 (원본 변수에 영향 없음)
}

public static void main(String[] args) {
    Person person1 = new Person("홍길동");
    Person person2 = new Person("김철수");

    swapReference(person1, person2);

    // 원본 변수는 변경되지 않음
    System.out.println(person1.getName());  // 여전히 "홍길동"
    System.out.println(person2.getName());  // 여전히 "김철수"
}
```

이 경우 `swapReference` 메서드 내에서 지역변수 `p1`, `p2`의 참조값을 서로 바꾸지만, 메서드 호출 시 전달된 원본 변수 `person1`, `person2`에는 아무런 영향을 주지 않는다.

### 2.3 비유로 이해하기

객체와 참조값의 관계를 이해하는 데 도움이 되는 비유:

- **객체**: 실제 건물
- **참조값**: 건물의 주소가 적힌 종이
- **메서드에 객체 전달**: 주소가 적힌 종이의 복사본을 전달

이 비유에서:

1. **객체 내용 변경**: 복사한 주소를 가지고 실제 건물에 가서 리모델링
2. **참조값 변경**: 종이에 적힌 주소를 다른 주소로 교체 (원본 종이는 그대로)

## 3. 실제 코드 동작 원리: void 메서드가 데이터를 반환하는 이유

이제 처음 질문으로 돌아가서, 서비스 메서드가 `void` 타입인데도 어떻게 데이터를 "반환"할 수 있는지 이해해보자.

### 3.1 Map 객체의 전달과 수정

```java
// 컨트롤러에서 서비스 호출
Map<String, Object> modelMap = /* 스프링이 생성한 맵 */;
faxService.

getChkList(modelMap);  // void 메서드

// 서비스에서 Map 수정
public void getChkList(Map<String, Object> modelMap) {
    // modelMap은 컨트롤러의 modelMap 객체를 가리키는 참조값의 복사본
    modelMap.put("chkList", faxDAO.getChkList(modelMap));
    // 원본 Map 객체에 "chkList" 키와 값이 추가됨
}
```

이 과정을 단계별로 살펴보면:

1. 컨트롤러의 `modelMap` 변수가 특정 Map 객체를 가리킴
2. 서비스 메서드 호출 시, 이 참조값의 복사본이 서비스 메서드의 매개변수로 전달됨
3. 서비스 메서드 내에서 `modelMap.put()`을 호출하면, 원본 Map 객체에 새 키-값 쌍이 추가됨
4. 서비스 메서드가 종료되어도, 원본 Map 객체의 내용은 이미 변경되었기 때문에 컨트롤러에서 볼 수 있음

이것이 가능한 이유는:

- `modelMap`이 참조하는 Map 객체는 힙(heap) 메모리에 저장됨
- 컨트롤러와 서비스 메서드의 `modelMap` 변수는 모두 같은 객체를 가리킴
- `put()` 메서드는 참조를 통해 원본 객체의 내용을 수정

### 3.2 이러한 패턴의 장단점

#### 장점:

- 여러 결과값을 한번에 전달할 수 있음
- `Map`을 통해 다양한 타입의 데이터를 유연하게 전달할 수 있음
- 레거시 시스템에서 널리 사용되는 패턴으로 기존 코드와의 호환성이 좋음

#### 단점:

- 타입 안전성(Type Safety)이 떨어짐
- 메서드 시그니처만 봐서는 어떤 데이터가 추가/변경되는지 알기 어려움
- 코드 가독성과 유지보수성이 낮아짐
- 오타나 키 불일치 등으로 인한 버그 발생 가능성이 높음

### 3.3 현대적인 대안

최신 Java/Spring 개발에서는 DTO(Data Transfer Object) 패턴을 사용하여 명확한 타입과 의도를 표현하는 것이 권장된다:

```java
// DTO 정의
public class FaxChecklistRequest {
    private String mapClsCd;
    // getter, setter
}

public class FaxChecklistResponse {
    private List<Map<String, Object>> chkList;
    // getter, setter
}

// 컨트롤러
@PutMapping("/fax/getChkList")
public FaxChecklistResponse getChkList(@RequestBody FaxChecklistRequest request) {
    return faxService.getCheckList(request);
}

// 서비스
public FaxChecklistResponse getCheckList(FaxChecklistRequest request) {
    List<Map<String, Object>> results = faxDAO.getCheckListByType(request.getMapClsCd());

    FaxChecklistResponse response = new FaxChecklistResponse();
    response.setChkList(results);
    return response;
}
```

이 방식은 코드의 의도와 데이터 흐름을 더 명확하게 표현하며, 컴파일 시점에 타입 검증이 가능하다.

## 4. 결론

Java에서 메서드 인자로 객체를 전달할 때 참조값이 복사되어 전달되는 Pass by Value 방식을 사용한다. 이로 인해:

1. 메서드 내에서 객체의 내용을 변경하면 원본 객체가 변경된다.
2. 메서드 내에서 참조값 자체를 변경해도 호출자의 변수에는 영향이 없다.

이런 특성을 활용하여 `void` 메서드에서도 Map 같은 컨테이너 객체를 수정함으로써 데이터를 "반환"하는 패턴이 레거시 시스템에서 흔히 사용된다. 하지만 현대적인 개발에서는 명확한 DTO와 반환값을 사용하는 것이
권장된다.