---
title: "[티켓 예매 서비스] 실전 부하테스트 2 (대기열 도입)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/13

toc: true
toc_sticky: true

date: 2025-12-23
last_modified_at: 2025-12-31
---  
  
## 이러쿵저러쿵...  
이제 연말이다. 25년을 돌아보니 생각보다 많은 일이 있었던 한 해였다.  
아무래도 제일 큰 이슈는 퇴사였지 않았나 싶다.  
그리고 두번째 이슈는 어깨 인대 부분파열...  
아직도 안낫는것이 이제 진짜 회복이 더뎌진게 체감이 되는 요즘이다...  
  
어깨를 다친만큼 코딩에 쓸 시간이 많아졌으니 오히려 좋아  
  
---
  
## 대기열 시스템 도입  
  
저번 포스트에서 TPS 37 RPS 113 정도의 지표를 얻었다.  
발생하는 병목중에서 WS에서 발생하는 병목은 튜닝을 통해 최적화를 시켰으며 어플리케이션단에서 DB와 커넥션에서 병목이 생겨 서버가 죽는 현상을 해결해야한다.  
  
대기열시스템을 도입해서 DB커넥션에 생기는 부하를 줄여볼 예정이다.  
  
처음에는 MQ를 사용계획이였으나 EC2 인스턴스가 t3.micro인 관계로...
메모리 관리의 차원에서 이미 사용중인 Redis를 사용하여 대기열을 구현하는 것으로 결정했다.  
  
기존에 만들었던 RedisService 클래스는 순수하게 토큰관리를 위해 사용했다.  
이제는 Redis로 대기열 시스템을 구현 할 것이니 AuthRedisService로 리팩토링을 하고 QueueRedisService를 새로 만들자  
  
<img src="/assets/images/posts/largeTrafficService/newClass.png" width="50%" height="50%">  

Redis에서는 16개의 테이블을 지원하는데 기존의 토큰 관련된 Redis 테이블은 index 0에서  
대기열을 위한 테이블은 index 1에서 사용할 것이다.

```java
@Bean
public RedisConnectionFactory authRedisConnectionFactory() {
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(host, port);
    config.setDatabase(authDatabase);
    return new LettuceConnectionFactory(config);
}

@Bean(name = {"authRedisTemplate", "redisTemplate"})
@Primary
public RedisTemplate<String, Object> authRedisTemplate() {
    return createTemplate(authRedisConnectionFactory());
}

@Bean
public RedisConnectionFactory queueRedisConnectionFactory() {
    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(host, port);;
    config.setDatabase(queueDatabase);
    return new LettuceConnectionFactory(config);
}

@Bean(name = "queueRedisTemplate")
public RedisTemplate<String, Object> queueRedisTemplate() {
    return createTemplate(queueRedisConnectionFactory());
}



private RedisTemplate<String, Object> createTemplate(RedisConnectionFactory factory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(factory);

    StringRedisSerializer serializer = new StringRedisSerializer();
    template.setKeySerializer(serializer);
    template.setValueSerializer(serializer);
    template.setHashKeySerializer(serializer);
    template.setHashValueSerializer(serializer);

    template.afterPropertiesSet();
    return template;
}
```  
  
두개의 테이블을 사용하기 위해 기존의 토큰 테이블은 @Primary 어노테이션을 사용해 Spring에서 해당 Bean을 우선 호출이 가능하도록 설정해야한다.  
아니면 Spring에서 등록된 빈이 두개여서 에러가 나는 참사가 난다.
  
Redis에서는 다양한 데이터 구조를 지원하는데 그중 Sorted Set을 사용하여 대기열을 구현한다.  
  
Sorted Set은 Key Score Member의 형태로 사용되는 구조인데  
여기서 Score을 통해 우선순위 지정이 가능한 형태이다.  
  
```java
@Component
public class QueueRedisService {
    private final RedisTemplate<String, Object> redisTemplate;
    private final ZSetOperations<String, Object> zSetOps;
    private final ValueOperations<String, Object> valueOps;

    public QueueRedisService(@Qualifier("queueRedisTemplate") RedisTemplate<String, Object> queueRedisTemplate) {
        this.redisTemplate = queueRedisTemplate;
        this.zSetOps = queueRedisTemplate.opsForZSet();
        this.valueOps = queueRedisTemplate.opsForValue();
    }

    public void addQueue(String key, String userId, long score) {
        zSetOps.add(key, userId, score);
    }

    public Long getRank(String key, String userId) {
        return zSetOps.rank(key, userId);
    }

    public Set<Object> getTopOrder(String key, long count) {
        return zSetOps.range(key, 0, count - 1);
    }

    public void removeUser(String key, String userId) {
        zSetOps.remove(key, userId);
    }

    public void setValues(String key, String data, Duration duration) {
        valueOps.set(key, data, duration);
    }

    public Object getValues(String key) {
        return valueOps.get(key);
    }

    public Set<String> getAllKeys(String pattern){
        return redisTemplate.keys(pattern);
    }

    public void deleteValues(String key) {
        redisTemplate.delete(key);
    }
}
```
  
위와 같이 Redis에 직접적으로 붙어 사용되는 메소드들을 정의 해주고  
아래의 QueueService를 만들어 Controller단에서 호출하여 사용할 메소드들을 만들어준다.

  
```java
@Service
@RequiredArgsConstructor
public class QueueService {

    private final QueueRedisService queueRedisService;
    private static final String WAITING_KEY = "ticket:waiting:";
    private static final String WORKING_KEY = "ticket:working:";

    public Long enterQueue(String userId, String ticketIdx) {
        long score = System.currentTimeMillis();
        queueRedisService.addQueue(WAITING_KEY + ticketIdx, userId, score);
        return queueRedisService.getRank(WAITING_KEY + ticketIdx, userId);
    }

    public Long getWaitCount(String userId, String ticketIdx) {
        Long rank = queueRedisService.getRank(WAITING_KEY + ticketIdx, userId);
        return (rank != null) ? rank + 1 : -1L;
    }

    public boolean isAllowed(String userId, String ticketIdx) {
        Object data = queueRedisService.getValues(WORKING_KEY + ticketIdx + ":" + userId);
        return data != null;
    }

    public void allowUsers(String ticketIdx, long count) {
        Set<Object> users = queueRedisService.getTopOrder(WAITING_KEY + ticketIdx, count);

        for (Object user : users) {
            String userId = (String) user;

            queueRedisService.setValues(WORKING_KEY + ticketIdx + ":" + userId, "allowed", Duration.ofMinutes(5));
            queueRedisService.removeUser(WAITING_KEY + ticketIdx, userId);
        }
    }

    public void finishTicketing(String ticketIdx, String userId) {
        queueRedisService.deleteValues(WORKING_KEY + ticketIdx + ":" + userId);
    }
}
```
  
지금 보면 키 설정을 잘 못 한거같다. 순간적으로 cli환경에서 상태확인하기에 너무 비슷하게 생겼다...  
Score의 경우 unix 타임스탬프를 사용했다.  
  
Redis 세팅은 다 했으니 Controller 단 코드를 추가해야한다.  
  
```java
@PostMapping("/waiting/{idx}")
public ResponseEntity joinQueue(@RequestAttribute("user") Map<String, Object> userInfo, @PathVariable(value = "idx") String idx) {
    String userId = (String) userInfo.get("memberId");
    Long rank = queueService.enterQueue(userId, idx);
    return new ResponseEntity(DefaultRes.res(true, "대기열 진입", rank+1), HttpStatus.OK);
}

@GetMapping("/waiting/{idx}")
public ResponseEntity getStatus(@RequestAttribute("user") Map<String, Object> userInfo, @PathVariable(value = "idx") String idx) {
    String userId = (String) userInfo.get("memberId");
    if (queueService.isAllowed(userId, idx)) {
        return new ResponseEntity(DefaultRes.res(true, ""), HttpStatus.FOUND);
    }
    Long rank = queueService.getWaitCount(userId, idx);
    if(rank < 0){
        return new ResponseEntity(DefaultRes.res(false, "시간초과"), HttpStatus.REQUEST_TIMEOUT);
    }
    return new ResponseEntity(DefaultRes.res(true, "현재 대기 순번", rank), HttpStatus.OK);
}

@PostMapping("/reserve/{idx}")
public ResponseEntity reserveTicket(@RequestAttribute("user") Map<String, Object> userInfo, @PathVariable(value = "idx") String idx){

    ticketService.reserveTicket(Integer.parseInt(idx), Integer.parseInt((String) userInfo.get("memberIdx")));
    queueService.finishTicketing(idx, (String) userInfo.get("memberId"));

    return new ResponseEntity(DefaultRes.res(true, ""), HttpStatus.OK);
}
```  
  
기존에는 바로 DB에 붙어 예매를 했기에 예약 API만 필요했지만  
이제는 대기열에 들어가는(Enqueue) API, 대기열에서 몇번째 순위인지 확인하는 API가 추가적으로 필요하고  
예매시 대기열에서 빠지는 (Dequeue) 로직이 추가되어야한다.  
  
잠깐의 여담이지만 Node.js로 개발할떄는 조건문 내부에 메소드를 넣지않고 변수로 받아서 그 변수를 조건에 넣는 방식을 주로 사용했는데 (가독성의 이유로)   
자바에서는 바로 넣어서 사용하는 편인것같다. 메모리 관리 차원인가...  
  
--- 
  
## 부하테스트
  
이제 배포를 하여 테스트를 해보자.  
  
<img src="/assets/images/posts/largeTrafficService/queueFirstTest.png" width="100%" height="100%">  
  
이전 포스트의 최종 테스트와 같은 조건으로 테스트를 진행했다.  
  
RPS 150  
총 요청 48655  
예매 TPS 6.5
  
이전 테스트에 비하면 예매하는 속도는 느려졌으나 p90, p95의 수치가 거의 절반 이상으로 떨어졌다.  
이전 테스트에서는 티켓 오픈 시나리오에 들어서자마자 cpu 사용율이 95프로를 왔다갔다하며 반응이 느려지는 등  
서버가 간당간당한 상태가 많았으나 지금은 cpu 로드 최고 수치가 80프로정도에서 안정적으로 트래픽을 받아내는 것을 보았다.  
  
하지만 예매 TPS는 하락했고 전체 요청(48655) 대비 예매 요청 수 (2082)는 작다.  
전체 요청의 압도적인 비율이 /ticket/waiting/:idx (폴링 API)에 치중되어있는 것을 볼 수 있다.  


  
서버는 안정되었지만 Redis 대기열에서 10초당 100명을 디큐잉하면서 전반적인 TPS가 낮아진 것으로 보인다.  
이번엔 5초당 100명을 디큐잉 한 결과이다.
  
<img src="/assets/images/posts/largeTrafficService/queueSecTest.png" width="100%" height="100%">  
  
여전히 서버는 안정적이다. 그리고 예매수가 소폭 상승한 것을 볼 수 있다.  
  
RPS 155
총 요청수 50599
예매 TPS 11.7  
  
아직 서버 리소스는 충분한 것 같으니 1초당 100명을 디큐잉 했으나...  
바로 서버가 간당간당하다가 집나간 것을 볼 수 있었다...(캡쳐는 못했다)  
  
특정 임계점 이후로 서버의 성능이 크게 떨어지는 것을 확인 후  
대략적인 계산과 추가적인 테스트를 통해 초당 60명 디큐잉 하는 것이 전반적인 아키텍쳐에서 최적의 퍼포먼스를 보여줄 수 있는 값이라는 것을 확인 후 초당 60명 값으로 고정했다.  
(최적화하고나서 신나서 캡쳐를 못했나보다... 아무리 찾아봐도 없다...)  
아래 테스트는 초당 80명 디큐잉 설정 후 vu를 1200으로 올려서 테스트한 결과다.  
서버가 죽지는 않았지만 p95값이나 예매 시도 대비 성공률이 떨어지는 것을 확인할 수 있다.
해당 테스트 이후 60명으로 고정했다.
  
<img src="/assets/images/posts/largeTrafficService/queue5test.png" width="100%" height="100%">  
[대기열 디큐잉 최적화 후 부하를 올려서 테스트]  
  
현재까지의 최적화를 통한 테스트 결과를 정리하면  
  
RPS 392
예매 TPS 27
  
과도한 폴링 API 콜로 인한 병목이 생기는 듯 하여 기존 2초당 한번 호출하던 폴링API를 4초에 한번 호출 하는 것으로 수정한 후 VU를 3000까지 증가시켜 부하테스트 결과이다.
  
<img src="/assets/images/posts/largeTrafficService/queue4thtest.png" width="100%" height="100%">  
  
RPS 367
예매 TPS 22

직전 테스트 결과와 비교를 하면 
폴링 레이트를 낮추는 전략으로 안정성은 높였지만 처리 속도에 있어 유의미한 결과를 얻지 못했다.  
  
즉 처리량은 고정인 상태로 대기열에서 대기 시간이 늘어나는 상태인 것이다.  
현재의 인스턴스와 아키텍쳐에서 Redis로 대기열을 구현하여 최적화했을때의 최고 퍼포먼스는 현재가 최적의 상태라고 판단했다.  

---

## 번외  
  
20251231 추가
현재 상태에서 이제는 최적화 방법은 DB와 커넥션을 개선하는 방법밖에 없다.  
공부가 좀 더 필요한 부분이지만 시도 해본 결과대로 작성을 해본다.
  
RDS에서 max-connection 옵션을 수정했다.  
spring에 application.yml 파일에서 다음과 같이 추가한다.  
  
```yaml
spring:
    datasource:
        hikari:
            maximum-pool-size: 50
            minimum-idle: 10
            connection-timeout: 2000
            max-lifetime: 1800000
            idle-timeout: 600000
```
   
해당 세팅은 DB와의 커넥션 풀을 늘리는 옵션과 Hikari 풀 조정을 하는 세팅이다.  
생각보다 부하를 잘 견뎌서 최종적으로 VU 3000으로 부하테스트를 진행했다. 

<img src="/assets/images/posts/largeTrafficService/queueLasttest.png" width="100%" height="100%">  
  
생각이상으로 괜찮은 퍼포먼스를 보여줬다.  
  
부하를 올렸음에도 최고 RPS인 392에서 430까지 상승했고, 성공률 또한 99%정도로 수렴했다.  
하지만 여전히 TPS는 22정도로 최고 TPS인 27에는 못미치는 수치다. 
