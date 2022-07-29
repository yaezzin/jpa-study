# 스프링 데이터 JPA

## Web 확장(스프링 MVC에서 사용하는 기능들)

1. 도메인 클래스 컨버터
    - HTTP 파라미터로 넘어온 엔티티의 id로 엔티티 객체를 찾아서 바인딩
    
    ```java
    @Controller
    public class MemberController {
    	
    	@Autowired MemberRepository memberRepository;
    	
    	@RequestMapping("member/memberUpdateForm")
    	public String memberUpdateForm (@RequestParam("id") Long id, Model model) {
    	
    		Member member = memberRepository.findOne(id);
    		model.addAttribute("member", member);
    		return "member/memberSaveForm";
    	}
    }
    ```
    
    ```java
    @Controller
    public class MemberController {
    	
    	@Autowired MemberRepository memberRepository;
    	
    	@RequestMapping("member/memberUpdateForm")
    	public String memberUpdateForm (@RequestParam("id") Member member, Model model) {
    	
    		model.addAttribute("member", member);
    		return "member/memberSaveForm";
    	}
    }
    ```
    
    - 요청url은 모두 `member/memberUpdateForm?id=1`
    - HTTP 요청으로 받은 회원 아이디를 도메인 클래스 컨버터가 회원 엔티티로 변환해서 넘겨줌
    - 주의할 점: **도메인 클래스 컨버터로 넘겨 받은 엔티티는 변경해도 DB에 반영되지 않음. 단순 조회용 (컨트롤러에서는 수정하지 않는 것이 바람직)**
2. 페이징과 정렬
    - `PageableHandlerMethodArgumentResolver`와 `SortHandlerMethodArgumentResolver`
    → 스프링 데이터의 페이징과 정렬 기능을 스프링 MVC에서 편리하게 사용하도록 제공
    
    ```java
    @RequestMapping(value = "/members", method = RequestMethod.GET)
    public String list(Pageable pageable, Model model) {
    	
    	Page<Member> page = memberService.findMembers(pageable);
    	model.addAttribute("members", page.getContent());
    	return "members/memberList";
    }
    ```
    
    - 파라미터로 Pageable인터페이스 (실제로는 PageRequest 객체)를 받음
    - Pageable은 page, sort, size의 요청 파라미터 정보로 만들어짐
    - 예시
        - “/members?page=2”: 인덱스41부터 20개 (default size: 20, 인덱스는 0부터 시작)
        - “/members?page=0&size=3&sort=name,desc&sort=address.city”
    - 접두사
        - 사용할 페이징정보가 둘 이상일 때 `@Qualifier` 사용
        - /members?member_page=0&order_page=1
        
        ```java
        @RequestMapping(value = "/members", method = RequestMethod.GET)
        public String list(@Qualifier("member") Pageable memberPageable, 
        									@Qualifier("order") Pageable orderPageable, Model model) {
        	
        	Page<Member> page = memberService.findMembers(pageable);
        	model.addAttribute("members", page.getContent());
        	return "members/memberList";
        }
        ```
        
    - Pageable의 기본값 page=0, size=20 → 기본값 바꾸고 싶을 때 @PageableDefault 사용하여 개별적으로 설정 가능
    
    ```java
    @RequestMapping(value = "/members_page", method = RequestMethod.GET)
    public String list(@PageableDefault(size = 12, sort = "name", direction=Sort.Direction.DESC) Pageable pageable) {
    	...
    }
    ```
    

## 프로젝트 적용 확인

## 스프링이 QueryDSL을 지원하는 방법

### QueryDslPredicateExecutor

1. 리포지토리에서 QueryDslPredicateExecutor<>를 상속받아서 사용, 페이징과 정렬도 사용 가능
    
    ```java
    Qitem item = QItem.item;
    Iterable<Item> result = itemRepository.findAll(
    	item.name.contains("장난감").and(item.price.between(10000, 20000))
    );
    ```
    
2. join, fetch를 사용하지 못하는 기능적 한계

### QueryDslRepositorySupport

1. QueryDslRepositorySupport을 상속받은 후 JPAQuery객체 직접 생성해서 사용
