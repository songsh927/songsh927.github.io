---
title: "[티켓 예매 서비스] Vue로 프론트 만들어보기(번외편)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/14  

toc: true
toc_sticky: true

date: 2025-12-29
last_modified_at: 2025-12-31
---  
  
## 이러쿵저러쿵...  
  
취업하기 전 Flutter와 Vue가 점유율이 높아진다는 썰들과 배우기 쉽다는 이야기를 듣고 가볍게 공부했던적이 있다.  
물론 백엔드 개발을 희망했던 나에겐 취미(?)정도로 생각했지만 은근히 써먹을 곳이 많았다.  
  
첫 프로덕션 릴리즈였던 전북현대모터스 웹 서비스에서 갑작스럽게 일손이 필요해 작은 도메인에 두 페이지정도 구현한적도 있고  
더듬어보니 사이드 프로젝트와 내 개인 블로그를 만든다고 Vue로 삽질했던 기억도 있었다.  
물론 할때마다 HTML과 CSS때문에 고통을 받은 기억밖에 없다만... 그래도 보고 머리 몇번 긁고 구글링하다보면 만들어지는게 어디인가...  
라고 생각하며 다시한번 백엔드를 해야겠다는 생각뿐이다.  
  
---  
  
## UI/UX  
  
최근 K*증권 앱을 사용할 일이 있었다.  
개인적으로는 최악의 UI/UX와 최악의 서비스였다.  
나름 국내 최대 증권기업이면서 메뉴도 어디있는지 모르겠고 사용자에겐 너무 불친절한 UI,UX와 너무나 많은 에러메세지들...  
  
최고의 UI/UX는 카카오톡이라고 생각한다.  
직관적이고 어디있는지 알기 쉬운 메뉴들... 괜히 앱의 UI/UX는 고민하지말고 카카오톡을 레퍼런스로 삼으라는 옛말이 틀린말이 아니다.  
(최근에 욕을 어마무지하게 먹었지만...)  
  
그 외에는 넷플릭스와 쿠팡같은 레이아웃을 좋아한다.  
직관적이고 깔끔한 UI와 자연스러운 플로우의 UX 그래서 나는 메인페이지는 넷플릭스, 로그인과 회원가입은 쿠팡의 로그인 페이지를  
그리고 프로젝트 이름을 보면 알겠지만 티켓 예매 페이지는 티켓링크를 본따왔다.  
  
<img src="/assets/images/posts/largeTrafficService/mainpage.png" width="100%" height="100%">  
[메인 페이지]  
  
<img src="/assets/images/posts/largeTrafficService/loginpage.png" width="100%" height="100%">  
[로그인 페이지]  
  
<img src="/assets/images/posts/largeTrafficService/reservepage.png" width="100%" height="100%">  
[예매 페이지]  
  
아무래도 대규모 트래픽을 어떻게 처리하는가에 대해 궁금증을 가지고 시작한 프로젝트다 보니 프론트는 좀 휑하다.  
  
아래는 부하테스트를 진행하며 같이 프론트에서는 실제로 어떻게 작동하는지 보여주는 영상이다.

[![Video Label](http://img.youtube.com/vi/AUavSsak9_g/0.jpg)](https://youtu.be/AUavSsak9_g)  
  
뭔가 처음 계획은 거창했지만 날이 갈수록 현실의 벽에 부딪히면서 느려졌지만 결국 기세로 완성시켜버렸다.  
  
처음 계획한 프로젝트 기한은 끝났으니 이제 좀 더 실제 서비스같이 Admin 페이지와 캐싱을 통해 최적화를 더 시켜보는 방향으로 해당 프로젝트를 발전시켜야겠다.