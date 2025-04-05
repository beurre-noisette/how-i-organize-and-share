사내 직원역량평가로 토이프로젝트를 진행하던 중 발주와 입고 그리고 재고 사이에서 숫자를 더하고 뺄 상황이 있었다. 물건을 발주할 때 정수로 주문을 하는 경우도 많이 있지만, KG으로 발주를 할 때와 같은 경우엔
소숫점을 이용한 발주를 할 상황도 있지 않을까? 하는 생각이 있었다. 그래서 DB(MySQL)에서 이를 관리하는 테이블은 decimal(10, 3), 10자리를 넘어가거나 소숫점이 3자리 이하로 상세히 발주하지는
않을 것 같았다. 타입으로 선언하고 이를 어플리케이션에서 어떻게 관리해야할지에 대해서 고민을 해보기 시작했다.
고민거리는 크게 다음과 같았다.

1. (백엔드) 사용자가 소숫점을 이용하여 값을 주고 받을 때 오차는 생기지 않을까? - Java의 float과 double은 근삿값을 담고 있기에 오차가 생길 수 있다.
2. (프론트엔드) 사용자가 보기에 정수로 떨어지거나, 소숫점 1자리 혹은 2자리 까지만 있는 숫자의 경우 깔끔하게 보여줘야 하지 않을까?

첫번째 고민인 소숫점을 이용한 연산에서의 오차에 대한 해결은 float이나 double타입을 사용하는 것 대신 BigDecimal 타입을 사용하는 것으로 시작했다. 물론 내가 이번 토이프로젝트에서 다루는 실수가 매우
큰 형태의 소숫점을 가지지는 않지만, 정확한 계산을 보장하기 위해 BigDecimal 타입을 사용해보기로 했다. 먼저 float과 double 타입이 오차를 내는 이유는 부동소수점 방식으로 수를 다루기 때문이다.
이에 대한 자세한 설명은 BigDecimal A to Z 를 참고했다.

실제 내가 백엔드 로직에서 수를 다루는데 고민을 했었던 부분은 다음과 같았다.

1. 수를 더하고 빼는데 어떻게 해야하는가? - 기본적인 BigDecimal 타입의 사용법
2. 최솟값과 최댓값은 어떻게 처리하는가? - 나의 경우에는 최솟값은 0, 최댓값은 DB설정인 decimal(10, 3)에 따라 9999999.999 였다.
3. BigDecimal 타입의 비교는 어떻게 하는가?

이에 대한 내 처리방법은 아래의 코드블록을 통해서 정리해 보겠다.

```java
// ManageOrderService.java
@Service
@Transactional
public class ManageOrderService {
// 필요한 Mapper와 Service 필드들
private static final BigDecimal ZERO = BigDecimal.ZERO; // 최솟값인 0으로 사용하기 위한 상수
private static final BigDecimal MAX_QUANTITY = new BigDecimal("9999999.999"); // 최댓값으로 사용하기 위한 상수, 생성자 선언 방식을보면 알겠지만 문자열로 저장한다.

	public Integer addToManageOrderCart(RequestGoodsCdDTO goodsCdDTO) {  
  
    Integer result = 0;  
  
    final BigDecimal ONE = BigDecimal.ONE;  
  
    for (String goodsCd : goodsCdDTO.getGoodsCds()) {  
        if (manageOrderMapper.checkExistingCart(goodsCd) != null) {  
            ManageOrderCartUpdateDTO updateDTO = ManageOrderCartUpdateDTO.builder()  
                    .manageOrdrCartCd(manageOrderMapper.checkExistingCart(goodsCd).getManageOrdrCartCd())  
                    .ordrQty(manageOrderMapper.checkExistingCart(goodsCd).getOrdrQty().add(ONE))  // 카트에 항목이 존재할 경우 1 증가시키는 로직
                    .build();  
            manageOrderMapper.addOrdrQty(updateDTO);  
        } else {  
            String generatedCode = codeGenerationService.generateCode("CART");  
  
            ManageOrderCartCreateDTO cartItem = ManageOrderCartCreateDTO.builder()  
                    .manageOrdrCartCd(generatedCode)  
                    .goodsCd(goodsCd)  
                    .ordrQty(BigDecimal.ZERO)  // 새로운 항목일 경우 0으로 등록하는 로직
                    .build();  
            manageOrderMapper.createManageOrderCart(cartItem);  
  
            result++;  
        }  
    }  
  
    return result;  
  
	}

	public Integer updateManageOrderDtl(ManageOrderHistoryUpdateDTO updateDTO) {  
    // 기존 발주 상세 정보 조회  
    ManageOrderHistoryDTO currentOrder = manageOrderMapper.getManageOrderDetails(updateDTO.getManageOrdrDtlCd());  
    if (currentOrder == null) {  
        throw new BuisnessException(ErrorCode.NON_EXISTENT);  
    }  
    // 수량 검증  
    validateQuantity(updateDTO.getWrhsQty(), "입고수량");  
  
    // 발주 상세 정보 업데이트  
    Integer result = manageOrderMapper.updateManageOrderDtl(updateDTO);  
  
    // 입고 수량이 있을 경우 재고 처리  
    if (updateDTO.getWrhsQty() != null) {  
        try {  
            InventoryInfoDTO inventory = inventoryMapper.getInventoryByGoodsCd(currentOrder.getGoodsCd());  
  
            if (inventory == null) {  
                inventoryService.createInventory(currentOrder.getGoodsCd(), updateDTO.getWrhsQty());  
            } else {  
                BigDecimal oldWrhsQty = currentOrder.getWrhsQty() == null ? ZERO : currentOrder.getWrhsQty();  
  
                BigDecimal newInventoryQty;  
                if (updateDTO.getWrhsQty().compareTo(ZERO) == 0) {  // 소숫점은 제외한 수가 0인지 비교하는 부분
                    newInventoryQty = inventory.getNowQty().subtract(oldWrhsQty);  
                } else {  
                    newInventoryQty = inventory.getNowQty()  
                            .subtract(oldWrhsQty)  
                            .add(updateDTO.getWrhsQty());  
                }  
  
                if (newInventoryQty.compareTo(ZERO) < 0) {  
                    throw new BuisnessException(ErrorCode.NEGATIVE_QUANTITY_NOT_ALLOWED);  
                }  
  
                InventoryUpdateDTO inventoryUpdateDTO = InventoryUpdateDTO.builder()  
                        .invntryCd(inventory.getInvntryCd())  
                        .goodsCd(inventory.getGoodsCd())  
                        .nowQty(newInventoryQty)  
                        .trgtQty(inventory.getTrgtQty())  
                        .build();  
  
                inventoryService.updateInventory(inventoryUpdateDTO);  
            }  
        } catch (Exception e) {  
            throw new BuisnessException(ErrorCode.INVENTORY_UPDATE_FAILED);  
        }  
    }  
  
    return result;  
	}
}
```
두번째 고민인 프론트엔드에서의 사용자가 마주하는 숫자를 보기좋게 포맷팅하는 과정은 아래의 코드와 같았다.

```javascript
function formatNumber(value) {
	if (!value && value !== 0) {
		return '-';
	}
	const num = parseFloat(value);
	return num % 1 === 0 ? num.toString() : num.toFixed(3).replace(/\.?0+$/, ''); // 소숫점이 없을 경우 toString으로 정수부를 보여주고, 그렇지 않을 경우(소숫점이 있는 경우)에는 소수점 3자리까지 표시하고 끝에있는 불필요한 0들을 제거한다.
}
```
참고 [BigDecimal A to Z](https://dev.gmarket.com/75)