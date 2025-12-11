---
title: "[티켓 예매 서비스] 티켓 서비스 개발 (1/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/8

toc: true
toc_sticky: true

date: 2025-12-09
last_modified_at: 2025-12-11
---  
  
## JPA와 JPQL  
  
Node.js 기반의 프레임워크에서는 Sequelize나 Prism, TypeORM 같은 라이브러리들이 유명하다.  
물론 나도 Sequelize로 처음 ORM을 입문하고 실무 하는 3년동안 열심히 썼다.  
이러한 ORM들의 특징은 쿼리가 조금만 복잡해져도 라이브러리 기능으로는 표현이 힘들다.  
테이블 두세개 조인걸어서 쿼리를 날리는 것은 일상다반사였기 때문에 단일 테이블을 조회 하는 상황 빼고는  
ORM 기능을 쓴 경우가 많이 없었던 것 같다.  
(특히나 Sequelize 내장 기능으로 Join쿼리를 쓸려면 거의 극악이다...)
  
그에 비하면 JPA랑 JPQL은 그나마 괜찮은 편인듯 하다.  
  
일단 코드를 살펴보자

```java
@Repository
public class TicketRepository {

    @PersistenceContext
    private EntityManager em;

    public Page<TicketListPageDTO> findAllByPage(Pageable pageable){

        String contentQuery = "SELECT tm FROM TicketMainEntity tm WHERE tm.ticket_close > NOW()";

        List<TicketMainEntity> ticketList = em.createQuery(contentQuery, TicketMainEntity.class)
                .setFirstResult((int) pageable.getOffset())
                .setMaxResults(pageable.getPageSize())
                .getResultList();

        String countQuery = "SELECT COUNT(tm) FROM TicketMainEntity tm WHERE tm.ticket_close > NOW()";

        Long total = em.createQuery(countQuery, Long.class).getSingleResult();

        return new PageImpl<>(ticketList.stream().map(ticket -> TicketListPageDTO.from(ticket)).collect(Collectors.toList()), pageable, total);

    }

    public Page<TicketListPageDTO> findAllByOption(Pageable pageable, String searchType, String searchValue){

        String contentQuery = "SELECT tm FROM TicketMainEntity tm WHERE tm.ticket_title LIKE CONCAT('%', :searchValue,'%') AND tm.ticket_close > NOW()";

        List<TicketMainEntity> ticketList = em.createQuery(contentQuery, TicketMainEntity.class)
                .setParameter("searchValue", searchValue)
                .setFirstResult((int) pageable.getOffset())
                .setMaxResults(pageable.getPageSize())
                .getResultList();

        String countQuery = "SELECT COUNT(tm) FROM TicketMainEntity tm WHERE tm.ticket_title LIKE CONCAT('%', :searchValue,'%') AND tm.ticket_close > NOW()";

        Long total = em.createQuery(countQuery, Long.class)
                .setParameter("searchValue", searchValue)
                .getSingleResult();

        return new PageImpl<>(ticketList.stream().map(ticket -> TicketListPageDTO.from(ticket)).collect(Collectors.toList()), pageable, total);

    }

    public Optional<TicketDetailDTO> findOneByIdx(Integer ticket_idx){

        String query = "SELECT NEW TicketDetailDTO(" +
                "tm.ticket_idx, tm.ticket_title, tm.ticket_sub_title, tm.ticket_title_image, tm.ticket_open, tm.ticket_close, " +
                "td.ticket_price, td.ticket_info) " +
                "FROM TicketMainEntity tm LEFT JOIN tm.ticketDetail td " +
                "WHERE tm.ticket_idx = :ticket_idx";

        List<TicketDetailDTO> ticketInfo =  em.createQuery(query, TicketDetailDTO.class)
                .setParameter("ticket_idx", ticket_idx)
                .getResultList();

        return ticketInfo.stream().findAny();

    }

    public void createReservation(TicketReserveEntity reservationForm){
        em.persist(reservationForm);
    }

}
```
  
  
```java
@Entity
@Table(name = "`TICKET_MAIN`")
@Getter
@Setter
@NoArgsConstructor
public class TicketMainEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer ticket_idx;

    @Column(nullable = false, length = 50, unique = true, updatable = false)
    private String ticket_title;

    @Column(nullable = true, length = 200, updatable = false)
    private String ticket_title_image;

    @Column(nullable = true, length = 100, updatable = false)
    private String ticket_sub_title;

    @Column(nullable = false, updatable = false)
    private Integer ticket_qty;

    @Column(nullable = false)
    @Temporal(TemporalType.TIMESTAMP)
    private LocalDateTime ticket_open;

    @Column(nullable = false)
    @Temporal(TemporalType.TIMESTAMP)
    private LocalDateTime ticket_close;

    @OneToOne(fetch = FetchType.LAZY)

    @JoinColumn(name = "ticket_idx")
    private TicketDetailEntity ticketDetail;

}
```
  
  
```java
public class TicketDetailEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer ticket_idx;

    @Column(nullable = false)
    private Integer ticket_price;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "json")
    private Map<String, Object> ticket_info;

    @OneToOne(mappedBy = "ticketDetail", fetch = FetchType.LAZY)
    private TicketMainEntity ticketMain;
}
```

많은 블로그들의 설명을 봤을때  
*@OneToOne* 어노테이션을 붙이고 각각의 키에 대해 정의하면 된다고 나와있어서  
당연히 각각 엔티티의 조인 할 키에 붙였더니 에러파티였다.  
  
대략 한시간 정도 컬러풀한 욕설과 깊은 한숨과 함께 알아보면서 삽질을 한 결과  
각각 엔티티의 맨 밑에 있는 것 처럼 각 엔티티를 정의하고 JoinColumn을 붙이는 것이였다.  
  
처음에는 사용할 줄 몰랐으니 이제 시작이라고 생각을 했지만...  
놀랍게도 이게 끝이였다.  
JPQL은 자바 객체단위로 쿼리를 생성해 주기때문에 이러한 연결들만 해놓으면 좀 더 편하게 쿼리를 만들 수 있었다.  
  
## Exception 
  
Nest.js에서의 에러 핸들링과 비슷하게 스프링에서도 에러 핸들링을 공식적으로 지원한다.  
*Exception.class*를 통해서 포괄적으로 기본적인 예외처리만 구현할까 란 생각도 있었지만 로깅을 위해 단계를 나누기로 했다.  
  
```java
@RestControllerAdvice
public class ApiExceptionAdvice {

    @ExceptionHandler(ApiException.class)
    public ResponseEntity exceptionHandler(HttpServletRequest http, ApiException e){
        return new ResponseEntity(DefaultRes.res(false, e.getError().getMessage()), e.getError().getStatus());
    }

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity exceptionHandler(HttpServletRequest http,final RuntimeException e){
        System.out.println("[ERROR]::::"+ e.getMessage());
        return new ResponseEntity(DefaultRes.res(false, "error" ), HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity exceptionHandler(HttpServletRequest http,final Exception e){
        System.out.println("[ERROR]::::"+ e.getMessage());
        return new ResponseEntity(DefaultRes.res(false, "error"), HttpStatus.UNAUTHORIZED);
    }
}
```
  
앞선 포스트에서도 언급했듯이 예상된 예외 (로직상 정상적인 실패)는 에러가 아니다.  
그러한 예외들은 첫번째 메서드(ApiException.class)로 핸들링을 하고,  
나머지 에러 핸들링은 아래 두 메서드를 통해 핸들링 할 것이다.  
  
<img src="/assets/images/posts/largeTrafficService/serviceException.png" width="100%" height="50%">  
서비스단에서 예외처리 된 응답  
  

앞서 개발한 필터에도 예외처리를 추가하기 위해 추가된 ApiException을 설정하였으나 정상적으로 동작하지 않았다.  
이 또한 두시간정도 삽질을 하다보니 에러 핸들링 컴포넌트는 컨트롤러단 이하부터 에러를 감지하는 것을 알았다.  
필터에서는 SpringSecurity에서 계정의 등급조정을 위한 기능이 이미 추가가 되어있으므로  
토큰에 대한 검증만 필요한 상태니 하드코딩으로 401 코드와 함께 응답을 하는 것으로 마무리했다.  
  
```java
String token = request.getHeader("x-access-token");
String checkToken = (String) redisService.getValues(token);

if(token == null || checkToken == null){
    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    response.setContentType("application/json;charset=UTF-8");
    String errorMessage = "{\"success\": false, \"message\": \"유효하지 않거나 만료된 토큰입니다.\", \"data\": null}";
    response.getWriter().write(errorMessage);
    return;
}
```
/common/filter/JwtCommonFilter에 추가된 401상태에 대한 처리
  
<img src="/assets/images/posts/largeTrafficService/filterException.png" width="100%" height="50%">  
필터에서 예외처리 된 응답
    
이제 기본적인 작업이 끝났으니 다음 포스트부터는 부하테스트를 통해 코드를 수정해보자