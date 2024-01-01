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
    - 간한단 코드를 복잡하게 구현해야 할 경우가 생길 수 있다.
