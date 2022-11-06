# Spring_MVC2_Validation

기본셋팅

Annotation Processors > Enable annotation processing 체크

Gradle > Build and run using, Run tests using = IntelliJ IDEA

File Encodings > Global Encoding, Project Encoding, Default encodeing for properties files : UTF-8

cf) 해당 강의는 점진적으로 개발되어진다. v1 -> v6

**상품 등록 검증 V1**

if (!StringUtils.hasText(item.getItemName())) {
errors.put("itemName", "상품 이름은 필수입니다.");
}

검증 오류 보관
Map<String, String> errors = new HashMap<>();

만약 검증시 오류가 발생하면 어떤 검증에서 오류가 발생했는지 정보를 담아둔다.

문제점

뷰 템플릿에서 중복 처리가 많다. 뭔가 비슷하다.

타입 오류 처리가 안된다.

**프로젝트 준비 V2**

if (!StringUtils.hasText(item.getItemName())) {
bindingResult.addError(new FieldError("item", "itemName", "상품 이름은
필수입니다."));
}

주의
BindingResult bindingResult 파라미터의 위치는 @ModelAttribute Item item 다음에 와야 한다. BindingResult 가 있으면 @ModelAttribute 에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다!

필드에 오류가 있으면 FieldError 객체를 생성해서 bindingResult 에 담아두면 된다.
objectName : @ModelAttribute 이름
field : 오류가 발생한 필드 이름
defaultMessage : 오류 기본 메시지

특정 필드를 넘어서는 오류가 있으면 ObjectError 객체를 생성해서 bindingResult 에 담아두면 된다.
objectName : @ModelAttribute 의 이름
defaultMessage : 오류 기본 메시지

BindingResult , FieldError , ObjectError 를 사용해서 오류 메시지를 처리하는 방법을 알아보았다.
그런데 오류가 발생하는 경우 고객이 입력한 내용이 모두 사라진다. 이 문제를 해결해보자.

if (!StringUtils.hasText(item.getItemName())) {
bindingResult.addError(new FieldError("item", "itemName",
item.getItemName(), false, null, null, "상품 이름은 필수입니다."));
}

파라미터 목록
objectName : 오류가 발생한 객체 이름
field : 오류 필드
rejectedValue : 사용자가 입력한 값(거절된 값)
bindingFailure : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
codes : 메시지 코드
arguments : 메시지에서 사용하는 인자
defaultMessage : 기본 오류 메시지

여기서 rejectedValue 가 바로 오류 발생시 사용자 입력 값을 저장하는 필드다.

**타임리프의 사용자 입력 값 유지**

th:field="*{price}"
타임리프의 th:field 는 매우 똑똑하게 동작하는데, 정상 상황에는 모델 객체의 값을 사용하지만, 오류가
발생하면 FieldError 에서 보관한 값을 사용해서 값을 출력한다.

**오류 코드와 메시지 처리1**

오류 메시지를 체계적으로 다루어보자.

new FieldError("item", "price", item.getPrice(), false, new String[]
{"range.item.price"}, new Object[]{1000, 1000000}

codes : required.item.itemName 를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라
배열로 여러 값을 전달할 수 있는데, 순서대로 매칭해서 처음 매칭되는 메시지가 사용된다.
arguments : Object[]{1000, 1000000} 를 사용해서 코드의 {0} , {1} 로 치환할 값을 전달한다.

**오류 코드와 메시지 처리2**

FieldError , ObjectError 는 다루기 너무 번거롭다.
오류 코드도 좀 더 자동화 할 수 있지 않을까? 예) item.itemName 처럼?

rejectValue() , reject()
BindingResult 가 제공하는 rejectValue() , reject() 를 사용하면 FieldError , ObjectError 를
직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다.

if (!StringUtils.hasText(item.getItemName())) {
bindingResult.rejectValue("itemName", "required");
}

void rejectValue(@Nullable String field, String errorCode,
@Nullable Object[] errorArgs, @Nullable String defaultMessage);

field : 오류 필드명
errorCode : 오류 코드(이 오류 코드는 메시지에 등록된 코드가 아니다. 뒤에서 설명할
messageResolver를 위한 오류 코드이다.)
errorArgs : 오류 메시지에서 {0} 을 치환하기 위한 값
defaultMessage : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지

**Validator 분리2**

@PostMapping("/add")
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult
bindingResult, RedirectAttributes redirectAttributes) {
if (bindingResult.hasErrors()) {
log.info("errors={}", bindingResult);
return "validation/v2/addForm";
}
//성공 로직
Item savedItem = itemRepository.save(item);
redirectAttributes.addAttribute("itemId", savedItem.getId());
redirectAttributes.addAttribute("status", true);
return "redirect:/validation/v2/items/{itemId}";
}

동작 방식
@Validated 는 검증기를 실행하라는 애노테이션이다.
이 애노테이션이 붙으면 앞서 WebDataBinder 에 등록한 검증기를 찾아서 실행한다. 그런데 여러 검증기를
등록한다면 그 중에 어떤 검증기가 실행되어야 할지 구분이 필요하다. 이때 supports() 가 사용된다.
여기서는 supports(Item.class) 호출되고, 결과가 true 이므로 ItemValidator 의 validate() 가
호출된다.

@Component
public class ItemValidator implements Validator {
@Override
public boolean supports(Class<?> clazz) {
return Item.class.isAssignableFrom(clazz);
}
@Override
public void validate(Object target, Errors errors) {...}
}
