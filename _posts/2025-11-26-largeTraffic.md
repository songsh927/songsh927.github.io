---
title: "[티켓 예매 서비스] 회원 서비스 개발(1/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/3

toc: true
toc_sticky: true

date: 2025-11-26
last_modified_at: 2025-11-28
---

기본적인 회원 도메인의 서비스를 개발해보는 날이다.  
사실 어제부터 시작했지만 중간에 코드를 살짝 잘못짜기도 했고... 오랜만에 만지는 자바와 스프링이라 공부를 병행하면서 베스트 프렉티스도 찾아보느라 속도가 조금 늦은 감이 있다...

<img src="/assets/images/posts/largeTrafficService/keepgoing.png" width="50%" height="50%">  
[이러면 조지는거다...]

작업하는 디렉토리 구조는 다음과 같다

```
src
└── main
    └── java
        └── com.ticketpark.ticketpark
            ├── common  // 공통 Response DTO
            └── member
                ├── controller
                ├── dto  // 요청 DTO 및 계층간 데이터 DTO
                ├── entity  // 레포지토리에서 사용될 엔티티
                ├── repository
                └── service
```

***

## ENTITY와 DTO

사실 Express.js 로 개발할때는 계층간 응답과 컨트롤러에서 나가는 응답에 대한 DTO만 잡아놓고 (말이 DTO였지 모든 로직에 들어가는 코드였다...) 상황에 따라 응답 폼에 데이터와 메세지를 추가하여 주고받는 스타일이였다.  
Nest.js에서도 DTO와 Entity는 있었지만 효율적으로 사용하진 못했다. 그래서 이번기회에 개념을 확실히 잡고 가기위해 시간이 걸리더라도 (~~코딩은 새벽에 하더라도...~~) 개념을 확실히 잡고 공부하며 진행하기로 했다.  
  
우선 Entity와 DTO의 정의부터 알아보면  

- **Entity** : *'Entity'는 실체, 개체, 독립체라는 뜻으로, 현실 또는 가상 세계에서 식별 가능한 독립적인 존재를 의미합니다. '실체'는 사람, 사물, 장소, 사건, 개념 등 다양한 대상을 포함하며, 특히 **데이터베이스 모델링이나 프로그래밍에서 정보를 저장하고 관리하기 위한** '어떤 것(Thing)'을 가리킬 때 많이 사용됩니다.*  

- **DTO** : *DTO는 '데이터 전송 객체'(Data Transfer Object)의 약자로, **여러 계층 사이에서 데이터를 주고받기 위해 사용되는 객체**입니다. 비즈니스 로직 없이 순수하게 데이터만 담고 있으며, 주로 getter/setter 메서드로만 이루어져 있습니다.*  
~~라고 제미나이가 말해줬다.~~
  
정의를 알았다 쳐도 실제로 개발을 해보면 단순히 같은 기능을 하는 컨트롤러 메소드랑 서비스 메소드랑 구분해서 이름짓기가 힘들듯이 DB에서 정보를 그대로 가져올때는 Entity를 그대로 참조해서 가져와야되나 싶다.(나만 그런가)  
많은 사람들의 코드를 봐도 컨트롤러단까지 Entity를 사용해서 응답하는 경우를 적지않게 본것같다.(템플릿 엔진을 사용하는 경우에 특히나 그런듯하다.)  
  
사람마다 또는 조직마다의 특성이겠지만 개인적으로 특별한 경우가 있지않고서야 Entity는 레포지토리와 서비스단 사이에서 활용하고, DTO는 그 이외에 모든 계층과 라이브러리 등과 데이터를 주고받을때 사용하는 것이 맞다고 생각한다.  
즉 Entity와 DTO가 같아도 서로의 역할도 다르고 코드의 유지보수를 위해 손이 한번 더 가더라도 만들어서 쓰자.

***  

## 베스트 프렉티스를 찾아서

많은 똑똑하신 개발자들이 *"자바로 서비스 개발은 이렇게 하면 되는거야!"* 라고 프리티어같은걸 깃에 올려주셨음 좋겠다.(있는데 못찾는거일수도;)  
매번 느끼지만 오늘 내가 짠 코드가 가독성이 좋고 효율적인 패턴이라고 든 생각이 그날 자정을 못넘기는 경우가 많았다.  
이틀동안 이 간단한 코드를 짜고나서 글쓰는 이 순간에도 어떻게 해야 더 좋은 변수명과 클래스, 메서드 이름을 지어주고, 어떻게 해야 더 효율적으로 자바를 쓰고, 로직이 간단해지고, 나의 코드를 보는(보나?) 사람들이 더 이해하기 쉽게 짤수있을까 란 고민이 든다.  
한풀이는 여기까지 하고 기세로 밀어붙인 코드를 살펴보자  


```
member/controller

@GetMapping("/find")
public ResponseEntity findMember(@RequestParam(required = true) String idx){
    GetMemberInfoDTO member = memberService.findMemberByIdx(Integer.parseInt(idx));

    return new ResponseEntity(DefaultRes.res(true, "회원정보 조회 성공", member), HttpStatus.OK);
}
```  
차후에 Vue.js를 도입할 것을 염두해두고 @RestController 어노테이션을 사용하여 JSON으로 응답하는 컨트롤러단 코드이다.  
Spring에서 지원하는 ResponseEntity를 사용했다. ResponseEntity 클래스를 살펴보면 첫번째 파라미터가 Nullable로 되어있는 Response Body를 받게 되어있다.  
이 Body에는 DefaultRes 클래스를 통해 요청 성공 여부를 담는 success, 요청에 대한 메세지를 담는 msg, 요청에 대한 응답 데이터를 위한 data로 구성을 했다.
```
/member/find?idx=1

ResponseBody
{
    "success": true,
    "msg": "회원정보 조회 성공",
    "data": {
        "idx": 1,
        "id": "test1",
        "email": "test@naver.com",
        "createdAt": "2025-11-25T12:00:00"
    }
}
```  

참고를 했던 코드에서는 success위치에 http status 코드를 넣어줬는데 ResponseEntity의 파라미터를 살펴보면 HttpStatus.OK 와 같이 http status 코드를 보내주기때문에 해당 부분은 단순히 boolean타입으로 요청에 대한 성공 여부로 바꿨다.
  
```
member/service

@Transactional
    public DefaultRes createMemberInfo(JoinDTO joinDTO){

        DefaultRes checkDuplicate = validateDuplicationMemberInfo(joinDTO);

        if(checkDuplicate.isSuccess()){
            memberRepository.saveMember(joinDTO.toMember());
            return DefaultRes.res(true, "회원가입 성공", new GetMemberInfoDTO(joinDTO.toMember()));
        }
        return DefaultRes.res(false, checkDuplicate.getMsg());
    }
```  
위에는 회원가입 API의 서비스단 코드다. return에 보면 컨트롤러단과 비슷하게 DefaultRes 클래스를 사용하는 것을 볼 수 있다.  
물론 Exception 클래스를 사용하여 예외처리를 하고 실패에 대한 로그를 남기고, 응답을 보낼순 있겠지만  
예외상황으로 인한 에러와 정상적인 로직을 통한 실패는 다르다.(물론 나의 스프링 사용 역량이 부족해서일 수도 있다.)  
서비스 코드(비즈니스 로직)에서 컨트롤러 코드로 데이터를 보낼때 성공여부와 메세지, 데이터를 보내야하는 로직들이 있기때문에 DefaultRes를 공통으로 사용하기로 했다.  

레포지토리는 JPA DATA를 사용해서 인터페이스로 구현하는 방법이 제일 편하지만 우선 코드로 작성을 했고, 차후 리팩토링을 통해 개선을 할 생각이다.
