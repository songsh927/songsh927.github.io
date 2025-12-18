---
title: "[티켓 예매 서비스] 티켓 서비스 개발 (번외편)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/9

toc: true
toc_sticky: true

date: 2025-12-11
last_modified_at: 2025-12-12
---  
  
  
## 갑자기 생각이 났다.  
  
수요일 밤 하루의 일과를 마치고 간단하게 맥주 한캔과 온라인 아이쇼핑을 하던 중  
사이트의 상품 품절 표시를 보고나서 갑자기  
  
*어 나 티켓 구매 갯수 제한 구현 안했네?*  
  
라는 생각이 불현듯 떠올랐다...  
  
티켓 예매라고 그냥 간단하게 디비에 넣어놓는 방식으로 정리해놓고...  
정작 갯수 체크는 안하고있었다...  
  
빼먹은 부분을 채워넣어보자.  
  
```java
@Transactional
public DefaultRes reserveTicket(Integer ticket_idx, Integer member_idx){

    // 티켓 재고 확인 로직 추가

    ticketRepository.createReservation(new TicketReserveEntity(ticket_idx, member_idx));
    return DefaultRes.res(true, "");
}
```
  
저 부분에 티켓 재고 확인 로직을 넣어보자.  
  
## 비관적 락  
  
당장 학부생 시절만 생각해봐도 학교에 그 부실한 서버로 어떻게 그런 부하를 견뎠는지 모르겠다.  
학과 학부를 떠나서 학교의 모든 학생들이 달려들어 서버가 죽든말든 눌러대는 현장일텐데 말이다.  
  
그렇게 눌러대고 매크로도 쓰다보면 분명 어떤 강좌든 상품이든 중복으로 들어갈 확률이 존재한다.  
~~경험상 실무에서도 유닉스 타임스탬프를 걸었음에도 모든 자리수가 같은 시간에 신청이 들어온 적도 있었다...~~  
  
단편적인 생각으로는 들어오는 사람마다 번호표를 주고 큐에 넣어버리는 방법도 있겠지만  
그것보다 편한 방법은 예매를 하기 직전 상품의 결재(티켓의 예매) 가능 여부를 확인하면서  
**비관적 락**을 걸어버리는 방법이 있다.  
  
비관적 락은 트랜잭션이 시작될때 락을 걸어 해당 트랜잭션이 끝날떄까지 다른 요청을 처리하지 않는 동시성 문제 해결 테크닉이다.  
  
```java
public interface TicketJpaRepository extends JpaRepository<TicketMainEntity, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT tm.ticket_qty > 0 FROM TicketMainEntity tm WHERE tm.ticket_idx = :ticket_idx")
    boolean findByIdxWithPessimisticLock(@Param("ticket_idx") Integer ticket_idx);

    @Modifying
    @Query("update TicketMainEntity tm set tm.ticket_qty = tm.ticket_qty - 1 where tm.ticket_idx = :ticket_idx")
    void decreaseTicketQty(@Param("ticket_idx") Integer ticket_idx);

}
```
  
고맙게도 JPA에서 비관적 락 기능을 제공을 해준다.  
앞전에 사용을 고려했던 JpaRepository를 사용하여 비관적락을 간편하게 구현해보자.
  
이름만 가지고 간편하게 쿼리를 작성해준다길래 정성스럽게 적었지만 원하는 대로는 동작을 안한다.  
~~JPA가 비정형 데이터 분석용 AI는 안쓰나보다.~~  
  
@Lock(LockModeType.PESSIMISTIC_WRITE) 어노테이션과 옵션을 사용하여 비관적 락을 사용할 것을 선언해준다.  
그리고 쿼리를 사용하여 ticket_qty가 0보다 큰지(예매가 가능한 상태인지) 확인하여 boolean으로 리턴해주도록 한다.  

```java
@Transactional
public DefaultRes reserveTicket(Integer ticket_idx, Integer member_idx){
    boolean checkQty = ticketJpaRepository.findByIdxWithPessimisticLock(ticket_idx);

    if(!checkQty){
        System.out.println(":::: 티켓의 수량 부족");
        throw new ApiException(ExceptionEnum.TICKET_QTY_FAIL);
    }

    ticketJpaRepository.decreaseTicketQty(ticket_idx);
    ticketRepository.createReservation(new TicketReserveEntity(ticket_idx, member_idx));
    return DefaultRes.res(true, "");
}
```  
  
티켓의 갯수를 확인 후 예매가 불가능한 상태면 미리 만들어둔 ApiException을 호출한다.  
  
반대로 티켓 예매에 성공했으면 qty 갯수를 감소시킨후 기존의 로직과 똑같이 예매 기록을 남긴 후 리턴한다.  
  
이제 남은 부분들을 모두 구현했으니 부하테스트를 통해 서버가 어디까지 버티는지 확인하고 개선을 해보자.