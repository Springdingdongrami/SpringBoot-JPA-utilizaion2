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

# 섹션 3. API 개발 고급 - 지연 로딩과 조회 성능 최적화

## Could not write JSON: Infinite recursion 에러

```java
2024-01-03T03:06:48.214+09:00 ERROR 30169 
--- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : 
Servlet.service() for servlet [dispatcherServlet] in context 
with path [] threw exception 
[Request processing failed: 
org.springframework.http.converter.HttpMessageNotWritableException: 
Could not write JSON: Infinite recursion (StackOverflowError)] 
with root cause
```

```java
@Entity
@Table(name = " orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {

    @Id @GeneratedValue
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member

		. . .
}
```

```java
@Entity
@Getter
@Setter
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @NotEmpty
    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();

		. . .
}
```

- 연관 관계에 있는 엔티티중 하나를 RestController에서JSON 형태로 반환하는 과정에서 recursion 에러가 발생한 상황
- 엔티티를 JSON으로 변경하면서 직렬화 과정을 거치는데 이때 서로가 참조하면서 무한 재귀가 발생한다.
- 해결 방법
    1. 재귀를 발생시키는 필드에 @JsonIgnore를 붙여준다.
        1. Order를 조회하고 있는 상황이라면 Memberd의 orders에 붙힌다.
    2. **엔티티를 반환하지 말고 DTO을 사용한다. (권장)**

## Type definition error: 
[simple type, class org.hibernate.proxy.pojo.bytebuddy.ByteBuddyInterceptor]] 에러

```java
2024-01-03T03:02:24.675+09:00 ERROR 30132 
--- [nio-8080-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : 
Servlet.service() for servlet [dispatcherServlet] in context 
with path [] threw exception [Request processing failed: 
org.springframework.http.converter.HttpMessageConversionException: 
Type definition error: 
[simple type, 
class org.hibernate.proxy.pojo.bytebuddy.ByteBuddyInterceptor]] 
with root cause
```

- 현재 FeatchType.LAZY(지연 로딩)로 설정 되어 있어서 참조 객체들이 프록시 객체(bytebuddy 라이브러리)로 초기화 되어 있다. 엔티티를 JSON으로 변경하기 위해 직렬화 과정을 거치는데 이때 프록시 객체(비어있음)의 직렬화 부분에서 오류가 발생했다.
- 참조 객체들이 필요할 때 DB에 쿼리를 날려 실제 값을 가져오는 작업이 필요하다. ( == 프록시 객체 초기화 작업 필요)
- 해결 방법
    1. 스프링 부트 3.0 미만 - hibernate5Module을 스프링 빈으로 등록
    2. 스프링 부트 3.0 이상 - Hibernate5JakartaModule을 스프링 빈으로 등록 
        
        ```java
        //Hibernate5JakarataModule 추가
        	implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5-jakarta';
        ```
        
        ```java
        //Application.java에 코드 추가
        @Bean
        Hibernate5JakartaModule hibernate5JakartaModule() {
        	return new Hibernate5JakartaModule();
        }
        ```
        
    3. **엔티티를 반환하지 말고 DTO을 사용한다. (권장)**

## N + 1 문제

- 연관관계가 설정된 엔티티 사이에서 한 엔티티(1)를 조회했을 때, 조회된 엔티티의 개수만큼 연관된 엔티티(N)를 조회하기 위해 추가적인 쿼리가 발생하는 문제를 의미.
- 예시 코드
    
    ```java
    @GetMapping("/api/v2/simple-orders")
    public List<SimpleOrderDto> ordersV2() {
    
        // N + 1 -> 1 + 회원 N + 배송 N
        List<Order> orders = orderRepository.findAllByString(new OrderSearch());
        List<SimpleOrderDto> result = orders.stream()
                .map(o -> new SimpleOrderDto(o))
                .collect(toList());
        return result;
    }
    
    @Data
    static class SimpleOrderDto {
    
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;
    
        public SimpleOrderDto(Order order) {
    
            orderId = order.getId();
            name = order.getMember().getName(); //LAZY 초기화
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress(); //LAZY 초기화
        }
    }
    ```
    
- 해결 방법 - **페치 조인 최적화**
    - 예시 코드
    
    ```java
    public List<Order> findAllWithMemberDelivery() {
    
          return em.createQuery(
                  "select o from Order o" +
                          " join fetch o.member m" +
                          " join fetch o.delivery d", Order.class
          ).getResultList();
      }
    ```
    
    - order 엔티티를 페치 조인을 사용하여 쿼리 1번에 member와 delivery도 같이 조회한다. 따라서 지연로딩이 따로 발생하지 않는다.
    - 엔티티를 조회했기 때문에 모든 정보가 넘어온다.
    
    ![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/07c43179-a669-43fa-b249-063ab1a08394)

    

## 페치 조인(Fetch Join)

- JPQL에서 성능 최적화를 위해 제공하는 기능
- 연관된 엔티티나 컬렉션들을 한번의 SQL 쿼리로 함께 조회 가능
- N + 1 문제 해결에 효과적
- 주의 사항
    - 일대다 또는 다대다 관계의 엔티티가 많이 포함되는 경우 불필요한 중복 데이터가 생기는 일이 발생한다 → **DISTINCT 키워드로 중복 데이터 제거 가능**
- 지연 로딩의 연관 엔티티 데이터를 자주 사용하는 경우에 쓰는 것이 좋다.

## JPA에서 DTO로 바로 조회

```java
public List<OrderSimpleQueryDto> findOrderDtos() {

    return em.createQuery("select new " +
                    "jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, " +
                    "o.orderDate, o.status, d.address)" +
                    " from Order o" +
            " join o.member m" +
            " join o.delivery d", OrderSimpleQueryDto.class)
            .getResultList();
}
```

- DTO로 조회했기 때문에 원하는 정보만 선택해서 받을 수 있다. (페치 조인을 했을 때와의 차이점)

![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/a97b1ebb-29f3-49c5-b12b-1d325fe18f10)


- SELECT 절에서 원하는 데이터를 직접 선택함으로 디비에서 애플리케이션 네트웍 용량 최적화에 도움이 된다(단, 큰 차이는 없음)
- 단점 - 리포지토리 재사용성이 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가는 단점

## 쿼리 방식 선택 권장 순서

1. 엔티티를 DTO로 변환하는 방법 선택
2. 필요할 경우 페치 조인으로 성능 최적화
3. 여전히 성능 이슈가 있다면 DTO로 직접 조회하는 방법 이용
4. 마지막 방버으로 JPA의 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용

<br><br><br><br>

# 섹션 4. API 개발 고급 - 컬렉션 조회 최적화

# 일대다 관계 조회 관련(OneToMany)

# **Hibernate5, Hibernate6 distinct 차이**

- Hibernate 6에서는 join fetch시 더 이상 sql구문에 distinct를 선언하지 않아도 자동으로 중복을 걸러준다.
- 예시
    - Hibernate5 - springboot 2.x
    
    ```java
    public List<Order> findAllWithItem() {
    
          return em.createQuery(
                  "select distinct o from Order o" +
                          " join fetch o.member m" +
                          " join fetch o.delivery d" +
                          " join fetch o.orderItems oi" +
                          " join fetch oi.item i", Order.class)
                  .getResultList();
      }
    ```
    
    - Hibernate6 - springboot 3.x
    
    ```java
    public List<Order> findAllWithItem() {
    
          return em.createQuery(
                  "select o from Order o" +
                          " join fetch o.member m" +
                          " join fetch o.delivery d" +
                          " join fetch o.orderItems oi" +
                          " join fetch oi.item i", Order.class)
                  .getResultList();
      
    ```
    
    - Order response data
    
    ```json
    [
        {
            "orderId": 1,
            "name": "userA",
            "orderDate": "2024-01-07T04:02:33.369413",
            "orderStatus": "ORDER",
            "address": {
                "city": "서울",
                "street": "1",
                "zipcode": "1111"
            },
            "orderItems": [
                {
                    "itemName": "JPA1 BOOK",
                    "orderPrice": 10000,
                    "count": 1
                },
                {
                    "itemName": "JPA2 BOOK",
                    "orderPrice": 20000,
                    "count": 2
                }
            ]
        },
        {
            "orderId": 2,
            "name": "userB",
            "orderDate": "2024-01-07T04:02:33.427993",
            "orderStatus": "ORDER",
            "address": {
                "city": "대구",
                "street": "2",
                "zipcode": "2222"
            },
            "orderItems": [
                {
                    "itemName": "SPRING1 BOOK",
                    "orderPrice": 30000,
                    "count": 3
                },
                {
                    "itemName": "SPRING2 BOOK",
                    "orderPrice": 40000,
                    "count": 4
                }
            ]
        }
    ]
    ```
    
    - distanct 유무의 차이가 있지만 둘다 중복을 메모리상에서 걸려서 출력 해 준 것을 볼 수 있다
    

## 일대다 컬렉션 페치 조인은 사용해도 될까?

- 안된다고 봐야 한다. 이유는 페이징 처리가 불가능해지기 때문!
- 예시
    - `findAllWithItem`에서 페이징 처리를 해줬지만 실제 SQL문에서는 limit과 offset이 나타나 있지 않다. WARN 경고를 firstResult와 MaxResult를 collection fetch 와 함께 써서 페이징 처리를 메모리에서 한다고 나와있다. ⇒ 데이터가 많아질 경우 메모리 초과가 발생한다.
    - 아래 예시를 바탕으로 설명하자면 페치 조인을 했기 때문에  Order값과 실제 DB에서 가져 오는 값이 달라 질 수 있다. 만약 한 주문에 여러 아이템들이 있는 경우 페치 조인에 의해 한 주문 당 주문된 아이템 개수 만큼 값들이 더 생기기 때문이다. 따라서 DB 영역에서는 페이징 처리가 제대로 안되기 때문에 메모리에서 하게 된다.

```java
public List<Order> findAllWithItem() {

      return em.createQuery(
              "select distinct o from Order o" +
                      " join fetch o.member m" +
                      " join fetch o.delivery d" +
                      " join fetch o.orderItems oi" +
                      " join fetch oi.item i", Order.class)
              .setFirstResult(1)
              .setMaxResults(100)
              .getResultList();
  }
```

```sql
select
        distinct o1_0.order_id,
        d1_0.delivery_id,
        d1_0.city,
        d1_0.street,
        d1_0.zipcode,
        d1_0.status,
        m1_0.member_id,
        m1_0.city,
        m1_0.street,
        m1_0.zipcode,
        m1_0.name,
        o1_0.order_date,
        o2_0.order_id,
        o2_0.order_item_id,
        o2_0.count,
        i1_0.item_id,
        i1_0.dtype,
        i1_0.name,
        i1_0.price,
        i1_0.stock_quantity,
        i1_0.artist,
        i1_0.etc,
        i1_0.author,
        i1_0.isbn,
        i1_0.actor,
        i1_0.director,
        o2_0.order_price,
        o1_0.status 
    from
        orders o1_0 
    join
        member m1_0 
            on m1_0.member_id=o1_0.member_id 
    join
        delivery d1_0 
            on d1_0.delivery_id=o1_0.delivery_id 
    join
        order_item o2_0 
            on o1_0.order_id=o2_0.order_id 
    join
        item i1_0 
            on i1_0.item_id=o2_0.item_id
```

![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/f1ce8bcc-c1c7-4e3b-a3c7-b46a46917977)


## 일대다의 컬렉션 패치 조인은 1개만 사용 가능

- 2개 이상 사용시 데이터가 부정합하게 조회될 가능성이 있다.
    - 데이터가 너무 많아지면서 jpa에서 데이터를 제대로 맞출 수 없게 될 수도 있기 때문이다.

## 일대다 컬렉션 페치 조인시 페이징 문제 해결 방법은?

1. ToOne 관계를 모두 fetch join 한다 → ToOne 관계는 row수를 증가시키지 않기 때문에 페이징 쿼리에 영향을 안준다.
2. 컬렉션은 지연 로딩(`FetchType.*LAZY`)으로 조회*
3. hibernate.default_batch_fetch_size, @BatchSize로 지연 로딩 성능 최적화 
    1. hibernate.default_batch_fetch_size : 글로벌 설정
    2. @BatchSize : 개별 최적화(세밀하게 설정할 경우 사용)
    3. 이 옵션 사용시 컬렉션이나 프록시 객체를 한꺼번엔 설정한 size만큼 IN 쿼리로 조회한다.
- 예시 코드
    - OrderApiController.java
    
    ```java
    @GetMapping("/api/v3.1/orders")
    public List<OrderDto> ordersV3Page(@RequestParam(value = "offset", defaultValue = "0") int offset,
                                       @RequestParam(value = "limit", defaultValue = "100") int limit) {
    
        List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);
    
        List<OrderDto> result = orders.stream()
                .map(o -> new OrderDto(o))
                .collect(Collectors.toList());
    
        return result;
    
    }
    ```
    
    - OrderRepository.java
    
    ```java
    public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    
          return em.createQuery(
                  "select o from Order o" +
                          " join fetch o.member m" +
                          " join fetch o.delivery d", Order.class
          )
                  .setFirstResult(offset)
                  .setMaxResults(limit)
                  .getResultList();
      }
    ```
    
    - application.yml
    
    ```java
    spring:
      jpa:
        properties:
          hibernate:
            default_batch_fetch_size: 1000
    ```
    
- 장점
    - IN 쿼리로 인해 쿼리 호출 수가 1 + N → 1 + 1로 최적화 된다.
    - 조인보다 DB 데이터 전송량이 최적화 된다.(Order와 OrderItem을 조인하면 Order가 OrderItem만큼 중복해서 조회된다. → 각각 조회하므로 전송해야할 중복 데이터가 없다.)
    - 페치 조인 방식과 비교해서 쿼리 호출수는 약간 증가. But, DB 데이터 전송량은 감소.
    - 컬렉션 패치 조인은 페이징이 불가능. But, 현재 방식은 페이징 가능.
    - 실행 되는 쿼리문 예시
        
        ```sql
        select o1_0.order_id,d1_0.delivery_id,d1_0.city,d1_0.street,d1_0.zipcode,d1_0.status,m1_0.member_id,m1_0.city,m1_0.street,m1_0.zipcode,m1_0.name,o1_0.order_date,o1_0.status 
        	from orders o1_0 
        		join member m1_0 
        			on m1_0.member_id=o1_0.member_id 
        		join delivery d1_0 
        			on d1_0.delivery_id=o1_0.delivery_id 
        	offset 0 rows fetch first 100 rows only;
        
        select o1_0.order_id,o1_0.order_item_id,o1_0.count,o1_0.item_id,o1_0.order_price 
        	from order_item o1_0 
        	where o1_0.order_id 
        		in(1,2);
        
        select i1_0.item_id,i1_0.dtype,i1_0.name,i1_0.price,i1_0.stock_quantity,i1_0.artist,i1_0.etc,i1_0.author,i1_0.isbn,i1_0.actor,i1_0.director 
        	from item i1_0 
        	where i1_0.item_id 
        		in(1,2,3,4);
        ```
        
    - 실행되는 쿼리문의 결과
        
        ![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/9693f299-7ac1-438c-b31b-bf309e9b4959)

        
- 결론
    - ToOne 관계는 페치 조인해도 페이징에 영향을 주지 않는다. 따라서 ToOne 관계는 페치조인으로 쿼리 수 를 줄이고 해결하고, 나머지는 `hibernate.default_batch_fetch_size` 로 최적화 하자.
- 참고: `default_batch_fetch_size` 의 크기는 적당한 사이즈를 골라야 하는데, 100~1000 사이를 선택하는 것을 권장한다. 이 전략을 SQL IN 절을 사용하는데, 데이터베이스에 따라 IN 절 파라미터를 1000으로 제한하기 도 한다. 1000으로 잡으면 한번에 1000개를 DB에서 애플리케이션에 불러오므로 DB에 순간 부하가 증가할 수 있다. 하지만 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 하므로 메모리 사용량이 같다. 1000으로 설정하는 것이 성능상 가장 좋지만, 결국 DB든 애플리케이션이든 순간 부하를 어디까지 견딜 수 있는 지로 결정하면 된다.

## ToMany JPA에서 DTO 직접 조회

- OrderApiController.java

```java
@GetMapping("/api/v4/orders")
public List<OrderQueryDto> OrderV4() {
    return orderQueryRepository.findOrderQueryDtos();
}
```

- OrderQueryRepository.java

```java
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;

    public List<OrderQueryDto> findOrderQueryDtos() {
        List<OrderQueryDto> result = findOrders(); //query 1qjs -> N번

        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId()); //query N번
            o.setOrderItems(orderItems);
        });

        return result;

    }

    private List<OrderItemQueryDto> findOrderItems(Long orderId) {

        return em.createQuery(
                "select  new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                        " from OrderItem oi" +
                        " join oi.item i" +
                        " where oi.order.id = :orderId", OrderItemQueryDto.class
        ).setParameter("orderId", orderId).getResultList();
    }

    private List<OrderQueryDto> findOrders() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name,o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderQueryDto.class
        ).getResultList();
    }
}
```

- findOrders 쿼리 하나 실행당 findOrderItems 쿼리 N번 실행
- ToOne 관계 조회 → ToMany 관계 별도 처리
    - ToOne 관계는 조인해도 데이터 row수가 증가하지 않음
    - ToMany관계는 조인하면 row수가 증가함

## ToMany JPA**에서** DTO **직접 조회** - **컬렉션 조회 최적화**

- OrderApiController.java

```java
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> OrderV5() {
    return orderQueryRepository.findAllByDtoOptimization();
}
```

- OrderQueryRepository.java

```java
public List<OrderQueryDto> findAllByDtoOptimization() {

    List<OrderQueryDto> result = findOrders();

    Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(toOrderIds(result));

    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}

private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
    List<OrderItemQueryDto> orderItems = em.createQuery(
            "select  new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                    " from OrderItem oi" +
                    " join oi.item i" +
                    " where oi.order.id in :orderIds", OrderItemQueryDto.class
    ).setParameter("orderIds", orderIds).getResultList();

    Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
            .collect(Collectors.groupingBy(orderItemQueryDto -> orderItemQueryDto.getOrderId()));
    return orderItemMap;
}

private static List<Long> toOrderIds(List<OrderQueryDto> result) {
    List<Long> orderIds = result.stream()
            .map(o -> o.getOrderId())
            .collect(Collectors.toList());
    return orderIds;
}
```

- ToOne 관계들 먼저 조회 → 얻은 식별자 orderId로 ToMany 관계인 OrderItem을 sql문 IN을 사용하여 한꺼번에 조회
- MAP을 사용해서 매칭 성능 향상 → O(1)
- fetch join select 부분의 데이터를 필요한 부분만 뽑아서 사용 가능

## ToMany JPA에서 DTO로 직접 조회, 플랫 데이터 최적화

- OrderApiController.java

```java
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> OrderV6() {

    List<OrderFlatDto> flats = orderQueryRepository.findAllByDtoFlat();

    return flats.stream()
            .collect(Collectors.groupingBy(o -> new OrderQueryDto(o.getOrderId(),
                            o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    Collectors.mapping(o -> new OrderItemQueryDto(o.getOrderId(),
                            o.getItemName(), o.getOrderPrice(), o.getCount()), Collectors.toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                    e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(),
                    e.getKey().getAddress(), e.getValue())).collect(Collectors.toList());
}
```

- OrderQueryDto.java

```java
@Data
@EqualsAndHashCode(of = "orderId")
public class OrderQueryDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemQueryDto> orderItems;

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.orderItems = orderItems;
    }

    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.orderItems = orderItems;
    }
}
```

- OrderQueryRepository.java

```java
public List<OrderFlatDto> findAllByDtoFlat() {
		
	return em.createQuery(
	        "select new" +
	                " jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
	                " from Order o" +
	                " join o.member m" +
	                " join o.delivery d" +
	                " join o.orderItems oi" +
	                " join oi.item i", OrderFlatDto.class
	).getResultList();
}
```

- OrederFlatDto.java

```java
@Data
public class OrderFlatDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    private String ItemName;
    private int orderPrice;
    private int count;

    public OrderFlatDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        ItemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}
```

- 장점]
    - 한번의 쿼리 실행
- 단점
    - 쿼리는 한번이지만 조인으로 인해 DB에서 애플리케이션으로 전달하는 데이터에 중복 데이터가 추가되므로 상황에 따라 “JPA**에서** DTO **직접 조회(v5)”**보다 느릴 수 도 있다.
    - 애플리케이션에서 추가 작업이 크다.(OrderFlatDto에서 OrderQueryDto로 변환 과정 등)
    - 원하는 페이징 불가능(디비에서 OrderItem을 기준으로 값을 생성하기 때문)

## 참조

### Hibernate6 distanct

[Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#hql-distinct)

<br><br><br><br>

# 섹션 5. API 개발 고급 - 실무 필수 최적화

## OSIV란

- Open Session In View
- 영속성 컨텍스트를 뷰까지 열어두는 기능을 뜻한다.
- 하이버네이트에서 부르는 명칭, JPA에서는 OEIV(Open EntityManager In View)라고 부른다. 관례상 둘다 OSIV라 부른다.
- JPA에서 설정 방법 - 기본값 True
    
    ```java
    jpa:
        open-in-view: false
    ```
    
- OSIV ON
    
    ![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/2413dfce-4d87-4dd8-a6e4-ee4dbdbefaba)

    
    - 애플리케이션 시작 시점에서 warn 로그를 남긴다.
    
    ```java
    2024-01-08T01:31:04.201+09:00  WARN 45519 --- 
    [  restartedMain] JpaBaseConfiguration$JpaWebConfiguration 
    	: spring.jpa.open-in-view is enabled by default. 
    Therefore, database queries may be performed during view rendering. 
    Explicitly configure spring.jpa.open-in-view to disable this warning
    ```
    
    - OSIV 전략은 트랜잭션 시작처럼 최초 데이터베이스 커넥션 시작부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지함 → View Template이나 API 컨트롤러에서 지연 로딩이 가능했음.
    - 실시간 트래픽이 중요한 애플리케이션에서는 장애로 이어 질 수 있음. → 커넥션이 모자라는 경우가 발생하기 때문!
- OSIV OFF
    
    ![image](https://github.com/Springdingdongrami/springboot-JPA-utilization-2/assets/74356213/0fb16f68-9864-4590-9a81-85aaa4da4d44)

    
    - OSIV를 끄면 트랜잭션을 종료할 때 영속성 컨텍스트를 닫고, 데이터베이스 커넥션도 반환한다. → 커넥션 리소스를 낭비하지 않는다.
    - OSIV를 끄면 모든 지연로딩을 트랜잭션 안에서 처리해야 한다. 따라서 지금까지 작성한 많은 지연 로딩 코드를 트랜잭 션 안으로 넣어야 하는 단점이 있다. 그리고 view template에서 지연로딩이 동작하지 않는다. 결론적으로 트랜잭션이 끝나기 전에 지연 로딩을 강제로 호출해 두어야 한다.
    - 커멘드와 쿼리 분리 방법
        - OSIV를 끈 상태로 복잡성을 관리하는 좋은 방법
        - 예시 1
            - OrderService : 핵심 비즈니스 로직
            - OrderQueryService : 화면이나 API에 맞춘 서비스(주로 읽기 전용 트랜잭션 사용)
        - 예시 2
            - OrderApiController.java
            
            ```java
            @GetMapping("/api/osiv/orders")
            public List<OrderDto> ordersOsiv() {
            
                return orderQueryService.ordersOsiv();
            }
            ```
            
            - OrderQueryService
            
            ```java
            @Service
            @Transactional(readOnly = true)
            @RequiredArgsConstructor
            public class OrderQueryService {
            
                private final OrderRepository orderRepository;
            
                public List<OrderDto> ordersOsiv() {
            
                    List<Order> orders = orderRepository.findAllWithItem();
            
                    List<OrderDto> result = orders.stream()
                            .map(o -> new OrderDto(o))
                            .collect(Collectors.toList());
            
                    return result;
                }
            }
            ```

