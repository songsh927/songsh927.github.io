---
title: "[티켓 예매 서비스] 티켓 서비스 개발 (1/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/7

toc: true
toc_sticky: true

date: 2025-12-06
last_modified_at: 2025-12-07
---
  

## 시작하기 전 이러쿵저러쿵...
  
자바랑 스프링을 다룬지 2주 조금 안 되는 시간 만에 많이 익숙해졌다.  
  
대학교 2학년 기말과제가 자바로 개발하고 싶은걸 개발해서 제출하는 과제였는데 순수 자바가지고 할 게 뭐가 있으려나 하다가 지뢰찾기를 만들었다.  
swing으로 GUI 만들고 기세와 어거지로 코딩을 했던 그 시절이 머슬메모리로 기억이 나버리면서...  
  
점점 프로젝트 코드가 산으로 가는 느낌이다...  
  
<img src="/assets/images/posts/largeTrafficService/shxt.png" width="50%" height="50%">  
[~~증거자료 1~~] 나름 A+을 받았다...!  
  
지금 생각해보면 객체지향도 아니고, 가독성은 쥐뿔도 없지만  
당시의 나에겐 큰 도전이었고, 결국 해냈다는 것이 행복이었던 것 같다.  
  
그때의 깡을 좀 다시 불러와야 할 시기인 것 같으니...  
다시 기세로 해보자  
  
---  
  
## 티켓 도메인 개발
  
티켓 도메인을 개발하면서 사실상 프로젝트의 제일 핵심 서비스이고,  
부하테스트를 염두해둔 도메인이기에  
실제 서비스와 어느 정도까지 비슷하게 구현을 하느냐에 고민이 많았다.  
  
실제 티X링크나 인X파크처럼 직접 좌석을 선택하고, 피 말리는 티켓팅을 구현할 수도 있겠지만  
이번 프로젝트는 부하를 발생시켜 어느 상황에서 어떤 기술을 도입하고, 튜닝을 해야 효율적으로  
대규모 트래픽을 안정적으로 받아낼 수 있는지를 검증해보는 것이 목표이기에  
  
티켓팅과 관련된 UI나 로직은 간단하게 구현을 하는 것으로 결정했다.  
  
그럼에도 실제 서비스와 비슷한 상황을 구현하기 위해 티켓테이블의 정규화를 통해 JOIN쿼리 사용과  
회원정보 검증 로직들을 적극 이용할 생각이다.  
  
  
티켓 도메인에서는 다음과 같은 API만 나오면 될듯하다.
- 티켓 조회
- 티켓 상세 조회
- 티켓 예매  
  
부하테스트를 위해 아마도 한개의 API가 하나 더 필요할 것이지만 우선 가장 간단한 서비스에서부터 발전해나가는 방향으로 프로젝트를 진행하고 있으니  
우선은 위와 같이 API를 만들어보자.  
  
```java
@GetMapping("/list")
    public ResponseEntity getTicketList(
            @RequestParam(value = "page", defaultValue = "1", required = false) Integer page,
            @RequestParam(value = "searchType", required = false) String searchType,
            @RequestParam(value = "searchValue", required = false) String searchValue
    ){

        DefaultRes result = ticketService.getTicketListByOption(page, searchType, searchValue);

        return new ResponseEntity(DefaultRes.res(result.isSuccess(), result.getMsg(), result.getData()), HttpStatus.OK);
    }
```  
  
컨트롤러단 코드이다.  
이전에 실무에서 조회기능과 페이징이 필요한 도메인을 개발할때 파라미터를 이런식으로 받는 것을 선호했었다.  
  
```java
public DefaultRes getTicketListByOption(Integer page, String searchType, String searchValue) {

        // TODO user-agent가 PC가 아닐때 pageSize 고려
        Pageable pageable = PageRequest.of(page > 1? page -1 : 0, 10);

        Page<TicketListPageDTO> ticketList = null;

        if(searchType != null){
            ticketList = ticketRepository.findAllByOption(pageable, searchType, searchValue);
        } else {
            ticketList = ticketRepository.findAllByPage(pageable);
        }

        return DefaultRes.res(true, "", ticketList);
    }
```  
  
위의 컨트롤러 코드에서 호출하는 서비스단 코드이다.  
Node.js로 개발할 당시에는 페이징 코드가 따로 없었기때문에  

```javascript
const offsetSetter = (page) => {
    return (page - 1)*_limit;
}
```  
  
이런 식으로 함수를 만들어놓고 사용했었다.  
  
스프링은 놀랍게도 **Pageable** 객체를 제공하여 페이징을 도와준다.  
~~실로 놀라운 일이 아닐수가 없다...~~
  
이러한 조회 API를 만들 때마다 결국 파라미터만 다르고 같은 기능을 하는데 어디서 나눠야 하는 것에 대한 고민을 많이 했다.  
상황에 따라 다르겠지만, 현재 나의 기조는 *＇프론트 단에서 페이지 단위가 바뀌는 요청은 서비스단 코드에서 나누고, 같은 페이지 안에서의 요청은 레포지토리단에서 나누자＇* 이다.  
페이지가 바뀌는 (예를 들어 네이버 메인 화면에서 메일로 들어가는 등) 상황에서는 같은 디비와 테이블을 조회하는 상황이 잘 없다.  
하지만 페이지가 바뀌지 않는 (예를 들어 네이버 메인 화면에서 하단의 컴포넌트들만 조회하는 등) 상황에서는 같은 디비와 테이블을 조회하는 경우가 많았다.

결국 쿼리만 달라지는 경우가 많기 때문에 나는 서비스단보단 레포지토리단에서 나누는 것을 선호하는 편이다.  

  
또 길어지면 가독성이 없어지니 다음 포스트에서 제대로 삽질한 상황을 적어야겠다.