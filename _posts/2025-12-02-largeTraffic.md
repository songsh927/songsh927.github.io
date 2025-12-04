---
title: "[티켓 예매 서비스] 회원 서비스 개발 리팩토링 (2/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/6

toc: true
toc_sticky: true

date: 2025-12-01
last_modified_at: 2025-12-04
---

## 이러쿵저러쿵.

실무에서 AWS를 사용할때는 DynamoDB를 사용했었다.  
관리하기도 편하고 자동으로 프로비저닝도 해주고... 무엇보다 정말 빠르고 ~~비쌌다~~.  
하지만 개인프로젝트에선 그런거 없다 다 돈이다. 배포에서는 EC2와 ELB정도만 사용할것이다.  

NoSql의 장점이라면 속도 이런것도 있겠지만 사용이 편하다는 것이 나에게는 가장 큰 장점이다.  
JSON 형태나 Map 형태의 데이터를 자주 다루다보니 데이터셋도 눈으로 파악하기 쉽고, 프론트와 통신할때도 주로 JSON을 사용하다보니  
KEY - VALUE 형태에 익숙해졌다.  
Redis는 키-밸류 형태로 데이터를 읽고, 쓰기에 적합한 툴이다. 또한 인메모리 기반의 DB이기에 속도가 매우 빠르다.(그리고 오픈소스다!!)  
  
기능단위로 회고록을 작성하면서도 Spring 사용해보기 인지 대규모 트래픽을 핸들링하는 서비스 개발인지 가끔 헷갈리지만  
다시한번 상기하자면 **대규모 트래픽 핸들링 서비스**를 개발하는 것이 나의 목표이다.  
  
오늘은 기세로 Redis를 길들여보자.
  
---
  
## Redis를 사용해보자
  
왜 Redis냐?  
앞에서도 말했듯이 오픈소스(~~공짜!~~)다.  
그리고 속도가 빠르다.  
  
일반적인 트래픽이 아닌 초당 몇십건의 트래픽을 받아야되는 서비스에서 생각보다 부하가 걸리는 쪽은 인증 시스템이다.(경험상)  
해당 사용자가 로그인은 했는지 부터 시작해서 해당 상품구매에 대한 권한은 있는지, 엑세스토큰 관리와 엑세스 토큰의 최신화를 위한 리프레쉬 토큰 관리등  
인증 시스템에 대한 부하도 만만치 않다.  
  
순간적으로 몰릴때 빠르고 안전하게 토큰을 저장하고 불러올 툴로 Redis를 선택했다.
  
Spring에서 Redis를 사용하는 방법은 두가지가 있다.  
- RedisRepository
- RedisTemplate
  
단순 토큰 저장을 위해 사용하기때문에 Entity를 사용할 필요가 없다고 생각했으며,  
차후에 Entity를 활용한 CRUD가 필요할 경우 마이그레이션이 가능하다는 장점으로 RedisTemplate를 선택했다.

```java
// RedisConfig

@Configuration
@EnableRedisRepositories
public class RedisConfig {

    @Value("${spring.data.redis.host}")
    private String host;

    @Value("${spring.data.redis.port}")
    private int port;

    // Redis 연결 팩토리 설정
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(host);
        redisStandaloneConfiguration.setPort(port);

        
        return new LettuceConnectionFactory(redisStandaloneConfiguration);
    }


    @Bean
    public RedisTemplate<String, Object> redisTemplate() {

        // Redis와 통신할 때 사용할 템플릿 설정
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());

        // key, value에 대한 직렬화 방법 설정
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());

        // hash key, hash value에 대한 직렬화 방법 설정
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());

        return redisTemplate;
    }
}
```  
    
```java
// RedisService

@Component
public class RedisService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ValueOperations<String, Object> values;


    public RedisService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.values = redisTemplate.opsForValue();
    }

    // 기본 데이터 저장
    public void setValues(String key, String data) {
        values.set(key, data);
    }

    public void setValues(String key, String data, Duration duration) {
        values.set(key, data, duration);
    }

    public Object getValues(String key) {
        return values.get(key);
    }

    public void deleteValues(String key) {
        redisTemplate.delete(key);
    }

}
```
   
토큰 입출력 할 것이기때문에 **opsForValue()**를 사용한다.  
참고로  
1. opsForValue()
  - 문자열 값을 처리할때 사용
  
1. opsForList()
  - Redis 리스트에 대한 연산 제공
  
1. opsForSet()
  - Redis 집합에 대한 연산 제공
  
1. opsForHash()
  - Redis 해시에 대한 연산 제공
  
1. opsForZSet()
  - Redis 정렬된 집합에 대한 연산 제공

위와 같이 기능 구현을 했으면 Redis를 설치해보자.  
요즘은 왠만하면 도커로 설치해서 사용한다. 로컬에 설치해서 사용하는 것 보다 훨씬 빠르고 가벼워서 로컬에서 사용하는 메시지큐, MSSQL, Redis는 모두 도커에서 사용한다.  
  
<img src="/assets/images/posts/largeTrafficService/docker.png" width="100%" height="50%">  
  
Redis가 올라갔으면 이제 로그인쪽 코드를 다시 확인해보자.  
이전 포스트에서 로그인 성공후 JwtProvider에서 create() 메소드를 통해 엑세스토큰과 리프레쉬토큰을 발급받는다.

```java

public TokenInfo create(Integer memberIdx, String memberId){
    String accessToken = generateAccessToken(memberIdx, memberId);
    String refreshToken = generateRefreshToken(memberIdx, memberId);

    redisService.setValues(accessToken, memberId + "-accessToken", Duration.ofMillis(accessExpireTime));
    redisService.setValues(refreshToken, memberId + "-refreshToken", Duration.ofMillis(refreshExpireTime));

    return TokenInfo.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .build();
}
```  
  
토큰을 생성하면서 넘어온 사용자의 PK인 idx(회원번호)와 id를 받아 토큰을 생성하고, 토큰은 생성즉시 Redis에 키값으로는 토큰, 밸류값으로는 **{사용자ID}-accessToken** 과 같은 형태로 저장된다.
  
이제 로그인 이후 로그아웃까지 테스트를 해보며 토큰이 정상적으로 저장이 되고, 삭제가 되는지 확인해보자.  
  
  
<img src="/assets/images/posts/largeTrafficService/login.png" width="100%" height="50%">  
로그인에 성공했고 토큰을 받았다.  
  
<img src="/assets/images/posts/largeTrafficService/myinfo.png" width="100%" height="50%">  
토큰을 헤더에 담아 불러오면 잘 불러와진다.

<img src="/assets/images/posts/largeTrafficService/loginRedis.png" width="100%" height="50%">  
Redis에도 정확히 들어온 것이 보인다.  

이제 로그아웃 후 토큰이 Redis에서 삭제되고, 내 정보를 조회했을때 에러메세지가 나와야한다.  
  
<img src="/assets/images/posts/largeTrafficService/logoutSuccess.png" width="100%" height="50%">  
<img src="/assets/images/posts/largeTrafficService/logoutRedis.png" width="100%" height="50%">  
원하는대로 동작을 한다.  
  
