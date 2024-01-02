# 섹션 1. API 개발 기본

## 엔티티를 RequestBody에 직접 매핑해도 될까?

- **안하는 것을 매우 권장한다.**
- → 프레젠테이션(화면) 계층을 위한 검증 로직이 엔티티에 추가 될 뿐 아니라 다양한 API의 모든 요청 요구사항을 한 엔티티에 모든 담기 어렵다. 또한 이렇게 엔티티가 변경되면 API 스펙 또한 변경 된다.
- ⇒ API 요청 스펙에 맞추어 별도의 **DTO**(**Data Transfer Object**)를 파라미터로 받자!

## 응답값으로 엔티티를 직접 외부에 노출해도 될까?

- **안하는 것을 매우 권장한다.**
- → 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
- → 엔티티의 모든 값이 노출된다.
    - API마다 엔티티에 대해 필요한 값이 다르다.
- → 응답 스펙을 맞추기 위한 로직이 추가된다. ex) `@JsonIgnore`, 별도의 뷰 로직 등등
- → 실무에서 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기 힘들다.
- → 엔티티 변경으로 인한 API 스펙이 변한다.
- → 유지보수 시 API 스펙 변경이 힘들다.
- ⇒ API 응답 스펙에 맞추어 별도의 **DTO**(**Data Transfer Object**)를 반환하자!

## DTO를 RequestBody에 매핑 했을 때 장점

- 엔티티와 프레젠테이션 계층을 위한 로직 분리 가능
- 엔티티와 API 스펙을 명확하게 분리 할 수 있다. → 요청 파라미터값에 어떤 것을 넣어야할지 명확하게 보인다. 따라서 유지 보수에 용이하다. 또한 엔티티가 외부에 노출되지 않는다.
- 엔티티가 변해도 API 스펙은 변하지 않는다.
- 참고 : 실무에서는 엔티티가 API 스펙에 노출되면 안됨!

## 멱등성(Idempotence)

- 여러 번 동일한 요청을 보냈을 때, 서버에 미치는 의도된 영향이 동일한 경우
- `PUT`, `GET`, `DELETE` 등이 있다.

## 엔티티에서는 롬복 사용을 자제한다.

- `@Getter` 정도만 사용 권장.

## DTO에서는 롬복 사용해도 무방.

- 데이터 이동에 주로 쓰이기 때문
- 주요 비즈니스 로직이 들어 있지 않기 때문

## 커멘드와 쿼리는 분리 하자!(CQS - Command Query Separation)

- 커멘드는 상태를 변경하는 메서드를 뜻하며, 쿼리는 상태를 반환하는 메서드를 말한다.
- 이 디자인 패턴에서는 객체의 모든 메서드를 커멘드와 쿼리 둘로 나누며 하나의 메서드는 반드시 커멘드와 쿼리 둘 중 하나만 속해야 한다.

```java
/* MemberService.java */

//잘못된 예시
@Transactional
public Member update(Long id, String name) {
    Member member = memberRepository.findOne(id);
    member.setName(name);

		return member;
}

//올바른 예시
@Transactional
public void update(Long id, String name) {
    Member member = memberRepository.findOne(id);
    member.setName(name);

}
```

```java
/* MemberApiController.java */

//잘못된 예시
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV1(
        @PathVariable("id") Long id,
        @RequestBody @Valid UpdateMemberRequest request) {

    Member member = memberService.update(id, request.getName());

    return new UpdateMemberResponse(member.getId(), member.getName());
}

//올바른 예시
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV1(
        @PathVariable("id") Long id,
        @RequestBody @Valid UpdateMemberRequest request) {

    memberService.update(id, request.getName());
    Member member = memberService.findOne(id);

    return new UpdateMemberResponse(member.getId(), member.getName());
}

@Data
static class UpdateMemberRequest {

    @NotEmpty
    private String name;
}

@Data
@AllArgsConstructor
static class UpdateMemberResponse {

    private Long id;
    private String name;
}
```

- `MemberService`의 `update`에서 `member` 객체를 반환하여 `MemberApiController`의  `UpdateMemberResponse`에서 `UpdateMemberResponse` 객체를 만들 경우 상태 변경과 조회가 한 메서드에서 발생하는 경우가 되어 버린다. 다시 말해 커멘드와 쿼리가 한 곳에서 발생한다. 따라서 분리하는 것이 좋다.
- 장점
    - 성능 최적화, 유지 보수, 코드 가독성
- 단점
    - 간단한 코드를 복잡하게 구현해야 할 경우가 생길 수 있다.
 
<br><br><br><br>

# 섹션 2.  API 개발 고급 - 준비

## 스프링에서 `EntityManager`의 thread-safe를 보장 할 수 있는 이유

- 스프링에서 초기에 `EntityManager`를 주입할 때 프록시 객체로 주입 하고, 사용시에 실제 객체를 생성하여 주입하는 방법을 이용하여 thread-safe를 보장 한다.

## `@PostConstruct`

- 의존성 주입 완료 후 다른 리소스에서 호출하지 않아도 실행시켜 주는 역할
- 실행 순서
    - 생성자 호출 → 의존성 주입 완료(`@Autowired` || `@ReuqiredArgsConstructor`) → `@PostConstruct`
- `@PostConstruct`에서 `@Transactinoal` 처리 시 주의할 점
    
    ```java
    //InitDb.java 잘못된 예
    
    /**
     * 총 주문 2개
     * * userA
     *   * JPA1 BOOK
     *   * JPA2 BOOK
     * * userB
     *   * SPRING1 BOOK
     *   * SPRING2 BOOK
     */
    @Component
    @RequiredArgsConstructor
    public class InitDb {
    
        private final EntityManager em;
    
        @PostConstruct
    		@Transactional
        public void init() {
            Member member = new Member();
            member.setName("userA");
            member.setAddress(new Address("서울", "1", "1111"));
            em.persist(member);
    
            Book book1 = new Book();
            book1.setName("JPA1 BOOK");
            book1.setPrice(10000);
            book1.setStockQuantity(100);
            em.persist(book1);
    
            Book book2 = new Book();
            book2.setName("JPA2 BOOK");
            book2.setPrice(20000);
            book2.setStockQuantity(200);
            em.persist(book2);
    
            OrderItem orderItem1 = OrderItem.createOrderItem(book1, 10000, 1);
            OrderItem orderItem2 = OrderItem.createOrderItem(book2, 20000, 2);
    
            Delivery delivery = new Delivery();
            delivery.setAddress(member.getAddress());
            Order order = Order.createdOrder(member, delivery, orderItem1, orderItem2);
            em.persist(order);
        }
    }
    ```
    
    ```bash
    //오류 발생
    
    Caused by: 
    	jakarta.persistence.TransactionRequiredException: 
    		No EntityManager with actual transaction available for current thread 
    		- cannot reliably process 'persist' call
    ```
    
    - 이러한 오류가 발생하는 이유는 스프링 라이프 사이클로 인해 @PostConstruct가 먼저 실행되고 @Transactional AOP가 적용되기 때문이다. 이때 `@PostConstruct`는 의존성 주입 완료이 완료된 뒤에 실행 된다. 따라서 `EntityManager`에 객체가 주입되어 있다. 하지만 현재 `EntityManager`에는 실제 객체가 주입되어 있는것이 아니라 프록시 객체가 주입되어 있는 상태이기 때문에 위와 같은 에러가 발생한다.
- ⇒ 초기화하는 메서드와 초기화를 실행하는 메서드 둘로 분리해서 해결하자!
    
    ```java
    //InitDb.java 올바른 예
    
    /**
     * 총 주문 2개
     * * userA
     *   * JPA1 BOOK
     *   * JPA2 BOOK
     * * userB
     *   * SPRING1 BOOK
     *   * SPRING2 BOOK
     */
    @Component
    @RequiredArgsConstructor
    public class InitDb {
    
        private final InitService initService;
        private final EntityManager em;
    
        @PostConstruct
        public void init() {
            
            initService.dbInit1();
    
        }
    
        @Component
        @Transactional
        @RequiredArgsConstructor
        static class InitService {
    
            private final EntityManager em;
    
            public void dbInit1() {
    
                System.out.println("before - " + em.toString());
                Member member = new Member();
                member.setName("userA");
                member.setAddress(new Address("서울", "1", "1111"));
                em.persist(member);
                System.out.println("after - " + em);
    
                Book book1 = new Book();
                book1.setName("JPA1 BOOK");
                book1.setPrice(10000);
                book1.setStockQuantity(100);
                em.persist(book1);
    
                Book book2 = new Book();
                book2.setName("JPA2 BOOK");
                book2.setPrice(20000);
                book2.setStockQuantity(200);
                em.persist(book2);
    
                OrderItem orderItem1 = OrderItem.createOrderItem(book1, 10000, 1);
                OrderItem orderItem2 = OrderItem.createOrderItem(book2, 20000, 2);
    
                Delivery delivery = new Delivery();
                delivery.setAddress(member.getAddress());
                Order order = Order.createdOrder(member, delivery, orderItem1, orderItem2);
                em.persist(order);
            }
        }
    
    }
    ```
    
    - 이렇게 하면 `@PostConstruct` 실행 후에 `initService.dbInit1()`이 실행되고 `initService` 내부의 `@Transactional`이 실행되면서 `EntityManager`에 실제 객체 값이 주입되기 때문에 해결된다.

## 참고

[[Spring] @PostConstruct에서 @Transactional 처리 시 문제점](https://sorjfkrh5078.tistory.com/311)

[스프링과 EntityManager의 동시성 비밀](https://woodcock.tistory.com/35)

[안녕하세요, EntityManager에 대해 궁금한 점이 있어 질문 남깁니다. - 인프런](https://www.inflearn.com/questions/158967/안녕하세요-entitymanager에-대해-궁금한-점이-있어-질문-남깁니다)

<br><br><br><br>


