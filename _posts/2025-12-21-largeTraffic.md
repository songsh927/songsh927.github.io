---
title: "[티켓 예매 서비스] 실전 부하테스트 1 (Nginx 튜닝)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/12

toc: true
toc_sticky: true

date: 2025-12-21
last_modified_at: 2025-12-22
---  
  
앞서 간단하게 다뤄본 부하테스트 툴인 K6를 사용해서 배포된 서버를 테스트해보자.  
  
현재 서버의 상태는  
로드밸런스 용 Nginx를 통해 Spring 서버로 들어오며 RDS를 따로 두어 EC2의 메모리 공간을 확보해둔 상태이다.  
Redis를 통해 토큰관리를 하며 토큰은 2시간 뒤에 Redis에서 자동으로 삭제된다.  
  
K6 툴을 처음 만져보다 보니 실제상황과 가깝게 만들어보기에는 아직 무리가 있어 GPT에 상황을 주어 레퍼런스를 받아 커스텀을 하는 방식으로 진행했다.  
예매 API에 비관적 락을 걸어 동시성 문제는 사실상 해결을 한 상태였지만  
비관적 락의 문제는 속도가 느리다는 것이 큰 단점이다.  
  
배포 후 간단하게 테스트를 진행했을 땐 확실히 로컬에서의 퍼포먼스와는 많이 달랐다.  
(물론 컴퓨터의 사양과 Nginx없이 테스트를 진행했다는 점이 제일 크다.)  
  
처음 테스트 시나리오 구상은 다음과 같았다.  
  
1. 티켓 오픈 전) 로그인 / 비로그인 유저의 티켓 리스트 및 상세 페이지 조회
2. 티켓 오픈 직후) 예매 API 트래픽 폭주 / 로그인 재시도 및 상세 페이지 조회 
3. 티켓 오픈 후 안정화 단계) 리스트 조회 및 재로그인과 예매 시도
  
위의 3단계에서의 시나리오를 구상했고 그에 따른 세팅은 다음과 같았다.  
  
1. 로그인 포함 혼합 시나리오
2. 예매 기능위주 시나리오(이미 로그인된 계정 토큰을 정적파일로 사용)

위의 3가지 단계의 시나리오와 각각의 세팅을 통해 부하테스트를 진행해보았다.
   
---  
  
## 로그인 포함 혼합 시나리오

먼저 실제와 비슷한 UX를 기반으로 로그인 상황을 같이 추가하여 테스트를 진행했다.  
  
500개의 가상 계정이 자신의 트랜잭션이 끝나면 지속해서 요청을 하는 형태이고,  
티켓 오픈 대기 2분 오픈 대기 이후 20초간 티켓 예매 트래픽 폭주상황과 2분 동안 간헐적인 조회와 로그인 그리고 예매 요청이 있는 상황이다.  


<img src="/assets/images/posts/largeTrafficService/loginticketingtest3.png" width="100%" height="100%">  
    
티켓 오픈 대기 중의 상태이다. 아직 리소스들이 여유롭고 딱히 부하가 없는 상태이다.  
현재의 시나리오 진행 동안은 간헐적인 로그인 요청과 함께 대부분 조회 요청이 있는 상태이다.  
  
2분이 지나고나서... 이제 헬게이트가 열린다...
  
<img src="/assets/images/posts/largeTrafficService/loginticketingtest2.png" width="100%" height="100%">  
  
cpu 사용률이 98퍼를 육박하면서 각종 에러가 터지기 시작한다.  

```log
ec 23 07:50:30 ip-172-31-42-70 java[4893]: 2025-12-23T07:50:30.331Z ERROR 4893 --- [ticketpark] [-8080-exec-1719] o.h.engine.jdbc.spi.SqlExceptionHelper   : HikariPool-1 - Connection is not available, request timed out after 30000ms (total=10, active=10, idle=0, waiting=189)

2025/12/23 06:21:38 [alert] 6705#6705: *16856 socket() failed (24: Too many open files) while connecting to upstream, client: 00.00.00.00, server: _, request: "GET /ticket/detail/1 HTTP/1.1", upstream: "http://127.0.0.1:8080/ticket/detail/1", host: "00.00.00.00"
```

위의 에러는 데몬으로 등록된 서비스의 (흔히 말하는 콘솔)의 로그이고 아래는 nginx의 에러로그이다.  
해당 에러 로그는 밑에서 다룰 것이다.
  
<img src="/assets/images/posts/largeTrafficService/loginticketingtest1.png" width="100%" height="100%">  

K6의 결과 창이다.  
  
분석을 해보면 로그인 성공률이 19%인 것을 볼 수 있다.  
  
응답시간을 분석해보자  
로그인은 이미 medium에서 4.9초대를 기록하고 있는 반면 다른 API들은 0.01초도 안 걸리고 있는 상황이다.  
p95와 p90을 비교했을 때 전반적으로 응답시간이 길어지고 있는 현상이 보인다.  

전반적인 테스트 결과를 종합하면 로그인(인증)과 관련된 시스템에서 병목이 발생하여  
다른 도메인에도 영향을 주면서 서버의 퍼포먼스 하락이 보인다.  

**추가내용**
추가적인 테스트를 통해 과연 로그인이 얼마나 느린지를 확인해본 결과  
로직이 느린것이 아닌 **JWT 토큰 생성과 비밀번호 encrypt** 부분에서 시간소요가 생기는 것으로 확인했다.  
  
- 예매 로직의 테스트 결과
예매 시도: 1247
예매 성공률: 47.31%
TPS(예매): 4.32 tx/s
  
  
로그인을 포함한 테스트코드를 잘못 작성했을 수도 있지만 인증 같은 경우 같은 서버에 두지 않고 다른 서버에 두면 완화되는 문제이다.  
또한 예매에 대한 개선방안을 찾는 것이 목표이기 때문에 로그인을 포함하지 않은 테스트를 진행해서 퍼포먼스를 비교해보았다.  

---
  
## 예매 기능위주 시나리오

유저들의 토큰을 먼저 받기 위해 스크립트를 작성하여 토큰을 정적파일로 받은 후 테스트를 진행했다.  
K6 자체에 setup() 메소드로 테스트에 필요한 리소스들을 세팅할 수 있지만  
속도도 느리고 setup을 진행하면서 갑자기 테스트가 터지는 경우가 몇 번 있어 따로 스크립트를 만들어서 진행했다.  
  
각각의 시나리오 진행 시간과 VU는 동일하게 진행했을 때 다음과 같은 결과를 얻었다.  
(캡처를 못했다...)  

```log
█ TOTAL RESULTS 

checks_total.......: 21283 80.969871/s 
checks_succeeded...: 100.00% 21283 out of 21283 
checks_failed......: 0.00% 0 out of 21283 

✓ list <500 
✓ detail <500 
✓ reserve not 5xx (except 503) 

CUSTOM 
biz_reserve_rejected...........: 0.00% 0 out of 4227 
biz_reserve_success............: 100.00% 4227 out of 4227 
biz_reserve_throttled..........: 0.00% 0 out of 4227 
detail_latency.................: min=5.589 med=9.287 p(90)=14.354 p(95)=19.4534 max=3367.786 
list_latency...................: min=6.336 med=9.437 p(90)=14.5756 p(95)=22.2766 max=351.756 
reserve_attempts...............: 4227 16.081363/s 
reserve_latency................: min=13.222 med=1342.874 p(90)=2077.4802 p(95)=2431.6737 max=4137.956 

HTTP 
http_req_duration..............: min=5.58ms med=9.88ms p(90)=1.34s p(95)=1.49s max=4.13s 
{ expected_response:true }...: min=5.58ms med=9.88ms p(90)=1.34s p(95)=1.49s max=4.13s 
http_req_failed................: 0.00% 0 out of 21283 
http_reqs......................: 21283 80.969871/s 

EXECUTION 
iteration_duration.............: min=1.24s med=3.82s p(90)=8.2s p(95)=10.71s max=2m2s 
iterations.....................: 4208 16.009079/s 
vus............................: 33 min=33 max=544 
vus_max........................: 725 min=725 max=725 

NETWORK 
data_received..................: 13 MB 48 kB/s 
data_sent......................: 3.3 MB 13 kB/s 


running (4m22.9s), 000/725 VUs, 4208 complete and 0 interrupted iterations 
warmup_browsing ✓ [======================================] 50 VUs 2m0s 
open_spike ✓ [======================================] 500 VUs 20s 
post_open_tail ✓ [======================================] 175 VUs 2m0s  
```
  
이전보다 훨씬 좋은 지표를 보여준다.  
  
전체 요청의 RPS는 대략 81에 수렴하고  
예매 트랜잭션의 경우  
- 예매 시도 : 4,227
- 예매 성공률 : 100% (4227/4227)
- TPS(예매) : 16.08 tx/s
와 같이 나온다. 
  
이제 순수 예매 시나리오에 안정적인 수치들을 얻었으니 부하를 늘려가보자.  
  
<img src="/assets/images/posts/largeTrafficService/beforetuning.png" width="100%" height="100%">  
  
VU를 800으로 늘린 테스트 결과이다.  
아직 성공을 하긴 하지만 간당간당한 상태이다.  
  
예매 성공률과 별개로 medium, p90, p95을 비교하면 거의 3초대 이상으로 나온다.  
이는 이미 사용자로 하여금 비정상적으로 느껴져 서비스 이용에 불편을 겪는다는 결론이다.  
  
실제로 이 테스트는 운이 아주 좋은 케이스였다.  
이후 추가적인 테스트를 진행했을 땐 실패율이 늘어났으며 위에서 언급한 에러메세지와 함께 서버가 죽기 시작했다.  
  
---  
  
## 튜닝을 해보자  
  
AI도 쓰고 구글링도 하고 강의도 많은 요즘 생각보다 최악의 응답속도를 가진 어플리케이션을 짜기란 힘들다.  
조금만 관심을 가지고 보면 충분히 수정이 가능한 코드들이다.  
(라고 하지만 철없는 신입 시절 슬로우쿼리로 인해 서비스가 터진 적이 있다...)  
  
즉, 어느 정도 성능이 좋은 서버가 갖춰져 있으면 인프라만 튜닝을 잘해도 기본은 보여준다.  
하지만 실무에선 모든 것이 돈이다. 심지어 더 비싸게 받아간다.  
아무튼 튜닝을 해보자  
    
```log
ec 23 07:50:30 ip-172-31-42-70 java[4893]: 2025-12-23T07:50:30.331Z ERROR 4893 --- [ticketpark] [-8080-exec-1719] o.h.engine.jdbc.spi.SqlExceptionHelper   : HikariPool-1 - Connection is not available, request timed out after 30000ms (total=10, active=10, idle=0, waiting=189)

2025/12/23 06:21:38 [alert] 6705#6705: *16856 socket() failed (24: Too many open files) while connecting to upstream, client: 00.00.00.00, server: _, request: "GET /ticket/detail/1 HTTP/1.1", upstream: "http://127.0.0.1:8080/ticket/detail/1", host: "00.00.00.00"
```

위에서 언급한 중요 에러 로그이다.  
위의 에러만 봤을 때 드디어 내가 원하는 상황으로 생각하던 찰나 의심이 들었다.  
  
**과연 어플리케이션단의 병목일까?**  
  
현재 프로젝트의 구성은 Nginx가 맨 앞단에서 요청을 받는다.  
그러면 생각해야할 튜닝 포인트는 WAS도 있지만 WS도 있는 것이다.  
  
위의 두번째 에러 코드의 **Too many open files**는 아파치를 사용하더라도 트래픽이 몰리면 볼 수 있는 에러메세지이다.
  
밑의 에러코드를 통해 실제로 Nginx에서 요청을 못받는 것을 확신했다.  
```log
[alert] 648#648: 768 worker_connections are not enough
```
Nginx 자체 커넥션을 감당하지 못해 나오는 에러로 worker_connection 부족이다.  

**/etc/nginx/nginx.conf** 에 다음과 같이 추가하자.
```nginx
worker_processes auto;

events {
    worker_connections 8192;   # 또는 16384
    multi_accept on;
    use epoll;
}
```
  
위의 설정을 했으면 아래 설정도 같이 해줘야한다.  

```bash
sudo systemctl edit nginx

```  
파일을 열고 아래 내용을 추가하자

```ini
[Service]
LimitNOFILE=100000

```
간단하게 설명하면 사용자와 프로세스에 리소스를 할당하는 한계를 지정해주는 것이다.  
해당 개념은 ulimit와 관련해서 찾아보면 된다.  
  
해당 튜닝 후 서비스 데몬을 리로드 및 Nginx를 재시작 후 다시 테스트를 해보자
  
<img src="/assets/images/posts/largeTrafficService/aftertuning.png" width="100%" height="100%">  
  
전반적인 수치는 비슷하거나 개선이 되었다.  
하지만 http_req_failed와 biz_reserve_success 를 확인해보면 차이가 많이 난다.  
Nginx에서 생기는 병목은 해결한 것으로 보인다.  
  
전체 요청에 따른 퍼포먼스는 다음과 같다.
- RPS : 113.64 req/s
- RPM : 약 6,818 req/min
- 전체 요청: 34,396  
  
예매 트랜잭션에 대한 지표는 다음과 같이 나왔다.  
- 예매 시도: 11,372
- 예매 TPS: 37.57 tx/s
- 예매 성공률: 99.61%
- 거절/쓰로틀: 0%  
  
하지만 아직 예매 API에서 p50, p90, p95의 수치가 3초 이상에서 최대 5초까지 걸리는 상황이다.  
들어오는 요청이 끊기지는 않지만 여전히 포화된 상태라는 것이다.  
  
결론적으로 WS단에서 생기는 병목은 해결했고 WAS에서 생기는 병목을 해결해야한다.  
앞으로 TPS를 37, RPS를 113 기준으로 개선해 나갈 계획이다...

---
  
대략 4분정도 걸리는 테스트를 하루에 한 20번 가까이 하니 눈이 괴롭다고 소리치는것이 들린다...