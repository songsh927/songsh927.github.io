---
title: "[티켓 예매 서비스] 프로젝트 설계"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/2

toc: true
toc_sticky: true

date: 2025-11-25
last_modified_at: 2025-11-26
---

## 프로젝트 설계

가장 기본적인 형태의 서비스를 만든 후 부하테스트를 통해 기술 추가 및 로직 개선을 해 볼 것이다.  
  
기술선택에 있어서 프론트엔드는 Thymeleaf와 Vue.js를 사용할 것이다.  
admin 페이지의 경우 Thymeleaf로 구현을 할 것이지만 메인 서비스 (회원서비스 / 티켓서비스)는 Vue.js를 사용할 것이다.  
백엔드에 있어서는 Spring과 JPA, MySql을 사용하여 기본적인 형태의 서비스를 구현할 것이고 차후 상황에 따라 Redis나 RabbitMQ, NOSQL등의 기술을 추가 할 생각이다.
  
가장 기초적인 형태의 서비스는 다음과 같은 도메인과 관련 서비스가 있을것이다.  
  
**회원서비스**
- 회원가입
- 로그인 / 로그아웃
- 마이페이지 (예매 기록, 회원정보 수정)
  
**티켓서비스**
- 등록된 상품(티켓) 조회
- 티켓 정보 조회
- 티켓 예매 / 예매취소
  
**관리자서비스**
- 티켓 관리 시스템
  
**DevOps**
- CI/CD
- 테스트코드 및 테스트코드 자동화

위의 도메인을 토대로 백엔드 아키텍처를 잡을 것이다.  
이 과정에서 고민을 많이 하였다. 유지보수와 확장성의 측면에서 3계층 기준의 디렉토리 구조로 가져갈 것인지 아니면 도메인 기준으로 디렉토리 구조를 가져갈 것인지 에 대한 고민을 많이하였다.
  
우선 도메인 기준으로 디렉토리 구조를 잡는 것으로 결정했다.  
(나름의) 이유는 다음과 같다.
- MSA 구조로 마이그레이션이 용이하다.
- 도메인 별 코드가 분리되어있어 빠르게 파악이 가능하다.
- (경험상) 코드의 추가나 변경에 있어 사이드 이펙트가 적다.
  
물론 경험이 많고 기술적으로 뛰어난 개발자는 어떤 아키텍처에서도 좋은 코드를 개발하겠지만...
  
```
src
└── main
    ├── java
    │   └── com.ticketpark.ticketpark
    │       ├── admin
    │       ├── ticket
    │       ├── common
    │       └── member
    │           ├── controller
    │           ├── dto
    │           ├── entity
    │           ├── repository
    │           └── service
    └── resources
        ├── static
        │   ├── css
        │   ├── img
        │   └── js
        ├── templates //admin 페이지를 위한 템플릿 디렉토리
        └── application.yaml
```
  
이러한 구조를 가지게 될 것이다. (~~지금보니 Nest.js랑 비슷한거같기도...~~)
  
다음 포스트부터 자바와 스프링 재활시간과 무한 삽질을 시작...🤦‍♂️🤦‍♂️