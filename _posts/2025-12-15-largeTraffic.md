---
title: "[티켓 예매 서비스] 서비스의 1차 부하테스트"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/10

toc: true
toc_sticky: true

date: 2025-12-15
last_modified_at: 2025-12-18
---  
  
## 이러쿵저러쿵...
  
이전 직장에서 룰렛을 만들 일이 있었다.  
가중치 기반 랜덤선택 알고리즘을 활용해서 특정 그룹의 사용자에게 가중치를 더하는 등  
그런 기능이 있는 룰렛 게임이였는데  
  
실제 서비스가 돌아가는 환경에서 2만명에 가까운 사람들이 한번에 서버에 붙을 수 있는 상황도 아니였고...  
룰렛이라는 서비스 특성상 돌아가고나서 결과가 보여야 좀 있어보였기 때문에  
네트워크상태는 둘째치고 나름 대규모 트래픽을 받아낼 수 있는 아키텍처로 개발을 했었다.  
  
문제는 개발을 해놓고 테스트를 해볼 방법이 없었다.  
(왜 그랬는지는 모르겠지만 부하테스트 툴을 쓸 생각을 못했었다)  
  
그래서 당시 쉘 스크립트로 순간적으로 트래픽을 만들어내는 스크립트를 만들어 돌렸다가  
RabbitMQ로 병렬처리 한다는걸 순간적으로 큐가 폭주해서 뻗은적이 있었다.  
  
아직도 그때 생각하면 아찔하다...  
  
---
  
## 부하 테스트 툴 선택  
  
최근에 이미지 처리할게 있어서 GPT를 결제했다.  
결제한 김에 평소에 궁금했던 것들이나 개발할 때 좀 적극적으로 사용하는 편이다.  
  
부하테스트 툴을 검색하니 너무 방대하게 나와서  
GPT한테 부하테스트 툴을 정리해서 좀 보여달라고 했더니 이렇게 나왔다.

| 도구           | 스크립트 방식    | 장점        | 단점          |
| ------------ | ---------- | --------- | ----------- |
| **JMeter**   | GUI/XML    | 풍부한 기능    | 무겁고 복잡      |
| **k6**       | JavaScript | 빠르고 CI 친화 | GUI 없음      |
| **Gatling**  | Scala DSL  | 고성능       | Scala 학습 필요 |
| **nGrinder** | Groovy     | 대규모 테스트   | 세팅 복잡       |
| **Locust**   | Python     | 쉬운 스크립팅   | Python 필요   |
| **Vegeta**   | CLI        | 초경량 테스트   | 단순 기능       |
  
JMeter랑 nGrinder를 사용해야지 하다가 K6 사용방법을 찾아보니  
기존에 자바스크립트를 사용했고, GUI가 없다뿐 결국 결과값만 읽을 수 있으면 편리한 툴이겠다 싶어서 선택을 했다.  
  
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1000,
  duration: '40s'
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

let token;

function loginIfNeeded() {
  if (token) return token;

  const id = `testuser${__VU}`;
  const password = 'qwer1234';

  const res = http.post(
    `${BASE_URL}/member/login`,
    JSON.stringify({ id, password }),
    { headers: { 'Content-Type': 'application/json' } }
  );

  // 1) 상태코드 확인
  if (res.status !== 200) {
    console.error(`[LOGIN FAIL] status=${res.status} body=${res.body}`);
    throw new Error(`Login failed for ${id} (status ${res.status})`);
  }

  // 2) 안전한 JSON 파싱
  let parsed;
  try {
    parsed = res.json(); // k6 내장 JSON 파서 (권장)
  } catch (e) {
    console.error(`[LOGIN NOT JSON] body=${res.body}`);
    throw new Error(`Login response is not JSON for ${id}`);
  }

  // 3) 구조 확인
  const accessToken = parsed?.data?.accessToken;
  if (!accessToken) {
    console.error(`[LOGIN TOKEN MISSING] parsed=${JSON.stringify(parsed)}`);
    throw new Error(`No accessToken in response for ${id}`);
  }

  token = accessToken;
  return token;
}


export default function () {
  const t = loginIfNeeded();
  
  const headers = {
    'Content-Type': 'application/json',
    'x-access-token': t
  };

  const res = http.post(`${BASE_URL}/ticket/reserve/1`, null, { headers });
  check(res, { 'orders 200': (r) => r.status === 200 });

  sleep(1);
}
```
  
우선 가볍게 1000개의 계정을 만들어 40초간 부하테스트를 해보았다.  
  
<img src="/assets/images/posts/largeTrafficService/k6traffictest.png" width="100%" height="50%">  
  
1000개의 계정중에 로그인은 모두 성공했고  
성공한 응답은 200개, 실패한 응답은 800개이다.  
~~실패한 응답이 800개나 되냐? 너는 실패한 개발자다!~~
가 아니라 정상적인 결과다.  
판매 가능 티켓 qty를 200으로 세팅을 하고 테스트를 진행했기 때문이다.  
(그래도 뭔가 실패했다고 나오니 기분이 나쁘다.)  
에러코드를 변경하고 예매가 끝나면 자동으로 테스트가 종료되는 코드를 넣어봐야겠다.
  
전혀 대규모 트래픽이 아닌거같아 가상 유저를 9000명으로 늘리고, 예매가능 티켓 qty도 1000개로 늘려서 테스트를 진행했다.  
  
<img src="/assets/images/posts/largeTrafficService/k6traffictestFail.png" width="100%" height="100%">  
  
이제 내가 원하는(~~원하지 않는~~) 상황이 나온다.  
밀려오는 트래픽에 피어가 부족해서 로그인시도조차 토해내는 서버의 상황이다.  
  
하지만 여기서 문제가 있다.  
인증서버에 대한 부하도 걸려야하겠지만 이 프로젝트의 목적은  
순수한 예매API에서 대규모 트래픽을 구현해보고 어떻게 해결을 하는가이다.  
위의 에러메세지를 확인해보면 대부분 로그인API에서 연결이 끊겨 다음 단계로 못넘어간 경우이다.  
  
이것을 확인해보기위해 예매된 내역을 살펴보았다.
  
<img src="/assets/images/posts/largeTrafficService/afterTestRemainTicket.png" width="50%" height="50%">  
<img src="/assets/images/posts/largeTrafficService/afterTestSuccessTicket.png" width="50%" height="50%">  
  
최초 예매가능 티켓 갯수는 1000개였으나 378개만 성공한 것이 보인다.  
애초에 로그인 API에서 병목이 걸려 넘어가지 못한 상태이다.  
  
그렇다면 k6 스크립트의 setup() 함수에서 미리 토큰을 생성해놓고 테스트를 진행해보자  
  
<img src="/assets/images/posts/largeTrafficService/k6gentoken.png" width="50%" height="50%">  
  
분석을 해보면 정상적으로 동작한 요청은 518.37ms로 나쁘지않은 결과이다.  
하지만 http_req_failed 값이 rate=91.44% 라는 것은 대부분의 요청이 응답을 받지도 못하고 터지거나 세션이 끊겼다는 것을 의미한다.  
이것은 로직상의 문제가 아닌 서버(Tomcat, OS)의 용량 및 세팅의 문제라는 것이다.  
  
---  
## 결론  
  
간단한 테스트를 해보았지만 우선 여기까지의 결론은 이정도로 정리할 수 있을것 같다.

대규모 트래픽을 받는것은 애플리케이션단에서 좋은 툴을 사용하고 쿼리 튜닝등이 필요하겠지만  
우선 애플리케이션단까지 넘어오기 이전에 WS와 네트워크 토폴로지등의 탄탄한 세팅이 되어있어야 한다.  

다음 포스트에선 좀 더 실전과 가깝게 배포를 하고 로깅을 추가하여 어느 포인트에서 개선이 필요한지를 정리해보겠다.