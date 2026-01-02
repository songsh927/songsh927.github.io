---
title: "[티켓 예매 서비스] 프로젝트 회고록"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/15  

toc: true
toc_sticky: true

date: 2025-12-31
last_modified_at: 2026-01-02
---  
  
## 프로젝트를 마치며  
  
한달간의 프로젝트가 끝났다. 아쉬운 부분도 있고, 어느정도 모양새가 잡히고나서 개선사항이 보이는 것들도 있지만 처음에 예상하고 생각했던 부분은 모두 끝냈다.  
  
이전에도 개발 블로그를 작성했지만 이렇게 주기적으로 올리려고 노력하고 개발하면서 어떤부분을 내가 고민하는지 녹여낼려고 노력한 블로그는 이번이 처음이다.  
  
권유에 의해서 시작했지만 은근히 글로 풀어내다보니 내 생각도 정리가 되고 아이디어도 많이 떠오르는 방법중 하나인것 같아서 앞으로는 자주 적어야겠다는 생각이다.  
  
---  
  
## 개요  
  
이번 프로젝트는 11월 24일부터 시작하여 1월 2일인 오늘까지 대략 한달 조금 넘는 시간동안 간단한 CRUD형태의 API서버 개발을 통해 자바와 스프링에 대한 학습과 평소에 궁금했던 대규모 트래픽에 대응하여 대기열 시스템을 직접 만들어보는것이 목표였다.  
  
---  

## 프로젝트 목표  
    
프로젝트에 있어 학습 목표는 다음과 같았다.
  
1. 자바와 스프링을 활용하여 서버를 구축
2. AWS의 리소스를 사용해 자동배포환경 구축
3. 각 단계별 부하 테스트를 통해 서버 개선과정 확인  
  
이에 따른 프로젝트 마일스톤은 다음과 같다.  
  
1. Member 도메인 개발
2. Member 도메인의 부가 기능 개발
3. Ticket 도메인 개발
4. 부하테스트 툴 학습
5. EC2 세팅 및 자동배포 세팅
6. 기존 시스템 부하 테스트
7. 인프라 튜닝
8. 인프라 튜닝 후 부하 테스트
9. 대기열 시스템 도입
10. 대기열 시스템 도입 후 부하 테스트
11. Vue 활용 프론트 개발
  
---
  
## 기술 스택  
  
기술 스택에 있어서는 백엔드에서는 5개의 키워드와 프론트엔드에선 한개의 키워드로 정리할 수 있을 것 같다.

![Java](https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white) ![Spring](https://img.shields.io/badge/spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)  
프로젝트 목표중 하나가 Java와 Spring에 대한 학습이였다.  
Javascript + Node.js 와 다르게 오브젝트(객체)에 대한 접근에 있어 아직은 불편한 느낌은 있지만 타입에 있어서 엄격한 언어인점, 역사가 오래된 프레임워크여서 안정적이고 지원하는 기능이 많다는 점에서 많은 부분을 새롭게 배우는 계기가 되었다.  
  
![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)    
실무에서 가장 많이 사용한 AWS에 대한 감각을 잃지 않기 위해 AWS를 선택했고 오랜시간이 지났지만 여전히 빠른 트러블슈팅이 되는 것을 보니 아직 죽지 않았다란 생각이 든다.  
금전적인 여유와 스프링에 대한 적응이 좀 더 빨랐으면 S3라든지 인증 서버를 두어 MSA를 시도했을텐데 다음에 꼭 시도해봐야겠다.  
  
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)    
실무에서는 캐싱을 하는 용도로 사용했고 어깨너머로 배운 탓에 손에 익은 툴은 아니였으나, 빠르게 적응하고 학습하여 사용했다. 간단한 사용법과 좋은 확장성을 가지고있는 툴이라고 생각된다.  
  
![Nginx](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)  
이전 실무에서는 Apache를 주로 사용했고, 특정 프로젝트 시작할때 configuration파일을 세팅하는 업무가 잠깐 있었지만 그 이후로 제대로 만져볼 기회가 없었다.  
이번 기회에 프로젝트 세팅과 함께 성능 튜닝을 해보며 Apache와 다른 생태계이지만 세팅에 있어 유사하다는 것을 느꼈지만 Apache의 리버스 프록시 세팅에 비해 Nginx가 훨씬 간편하다고 느꼈다..!  
  
<img src="https://img.shields.io/badge/k6-7D64FF?style=for-the-badge&logo=k6&logoColor=white">  
처음 접해본 부하테스트 툴이다. 사용법이 명확하게 정리된 문서를 찾기 힘들어 K6 부하테스트 시나리오 작성에 있어서는 GPT의 도움을 많이 빌렸다. 물론 테스트의 큰 틀은 GPT에게 시키고 직접 시나리오와 값들을 수정해가며 사용했다.  
Grafana에서 만든 툴이다보니 Grafana GUI툴과도 같이 쓰이는 것으로 봤는데 이 부분은 좀 더 공부를 해봐야겠다.
  
![Vue.js](https://img.shields.io/badge/vuejs-%2335495e.svg?style=for-the-badge&logo=vuedotjs&logoColor=%234FC08D)  
많은 페이지가 필요한 프로젝트가 아니였기에 Thymeleaf를 사용 할 수도 있었지만 기존에 Vue를 얕게나마 알고있는 탓에 기억을 되살리기 위해 Vue를 사용했다.  
  
---

## 아키텍처

<img src="/assets/images/posts/largeTrafficService/architecture.png" width="100%" height="100%">  
  
아키텍처는 위의 그림과 같다.  
Nginx에서 트래픽을 받아 Vue로 라우팅을 해주고 Vue에서 API 요청을 할때마다 reverse_proxy를 통해 Spring서버로 요청한다.  
티켓 예매 서비스의 경우 Redis로 대기열을 구성해 서버를 보호한다.  
  
---

## 프로젝트 결과 및 성과

기초적인 형태이지만 트래픽을 발생시켜 서버에 부하를 줌으로써 어떤 방법을 사용해야 서버를 안정적으로 가동할 수 있는지 테스트를 진행했으며 각각의 상태에 따른 테스트 결과는 다음과 같다. 

| | 순수 DB 통신 | Nginx 튜닝 | Nginx 튜닝 + 대기열 도입|
|:---|:---:|:---:|:---:|
|RPS|45|114|392|
|TPS|4.5|37|27|
|CPU 부하|99% (서버다운)|92%|85%|
  
RPS 45 TPS 4.5 에서 RPS 392 TPS 27으로 개선했다.  
  
Nginx 튜닝만 했을 때 보다 대기열 시스템 도입 후에 TPS는 떨어졌지만 안정적으로 트래픽을 받았으며 서버 리소스 확인 결과 CPU, Memory 부하또한 여유있는 상태로 확인했다.  
t3a.micro에서 크래딧을 사용하여 버스트 모드를 적극 사용하거나 보다 더 높은 사양의 인스턴스를 사용하면 Nginx 튜닝값을 더 밀어붙여 요청을 여유있게 받을 것으로 예상된다.  
또한 대기열 디큐잉 또한 값을 높임과 동시에 Hikari Mysql의 max-connection 튜닝을 통해 예매 TPS를 향상시킬 수 있을 것으로 기대된다.  
  
---

## 향후 계획  
  
예매 API 뿐만 아닌 조회 API에서의 트래픽을 감소시켜 예매 API 요청에 대한 여유를 늘리는 방향을 생각했다. Redis를 사용하여 캐싱을 통해 반복적인 조회에 대한 리소스 소비를 감소시키는 방안을 구현할 계획이다.  
또한 지금은 동시성이 의미가 없는 거의 선착순에 가까운 예매 형태를 가진 서비스이지만 좌석 선점형의 서비스를 구현하여 동시성에 대한 확실한 테스트를 진행해볼 예정이다.