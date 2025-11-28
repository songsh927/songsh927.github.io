---
title: "[티켓 예매 서비스] 회원 서비스 개발(2/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/4

toc: true
toc_sticky: true

date: 2025-11-28
last_modified_at: 2025-11-29
---


Node.js 기반으로 개발을 할 때에는 앞쪽(컨트롤러 단)부터 개발하는 게 편했는데 의외로 스프링은 뒷단(레포지토리 단)부터 개발하는 게 편한듯하다.  
기분탓이다 다시 기세로 가보자

지금까지 흔히 말하는 CRUD만 만들었고 살을 붙여나가자  

오늘은 토큰을 생성과 비밀번호 encrypt 기능을 추가할 것이다.  
토큰 완료 후에는 토큰을 통해 로그아웃과 내 정보 수정 기능을 개발해보려고 한다.

## ENCRYPT

<img src="/assets/images/posts/largeTrafficService/noencrypt.png" width="50%" height="50%">  
~~군대식 비밀번호~~
  
  
항상 Node.js기반에 자유로운 형태의 프레임워크와 입맛대로 바꾸기 편한 코드 스타일을 접하다가 스프링을 접하니 적응하기 힘든 부분이 많았다.  
그중 대표적인 게 SpringSecurity같은 부분일 것 같다.  
  
유독 해당 라이브러리를 사용하면서 코드를 짜는 것보단 어노테이션을 적절히 잘 활용하면 훌륭한 서비스를 개발할 수 있겠다고 느낄 정도로 스프링은 체계적이고 이전 포스트에서 언급했던 **베스트 프렉티스**가 잘 구현되어있는 프레임워크라고 느꼈다.  
  
깃허브와 구글링등 많은 레퍼런스를 찾아보면서 지금 내 상황에서 맞게 활용할 수 있는 코드를 찾은 코드는 다음과 같다.

```java
// security/SecurityConfig.java

public class SecurityConfig {

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        // AuthenticationConfiguration 객체로부터 AuthenticationManager를 가져옵니다.
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception{
        http
                // 1. CSRF (Cross-Site Request Forgery) 방어 비활성화 (REST API 권장)
                .csrf(csrf -> csrf.disable())

                // 2. 요청별 접근 권한 설정
                .authorizeHttpRequests(auth -> auth
                        // 회원가입 요청 (POST)은 인증 없이 모두 허용 (permitAll())
                        .requestMatchers(HttpMethod.POST, "/api/members").permitAll()

                        // 기타 모든 요청은 인증된 사용자만 허용 (authenticated())
                        // .requestMatchers("/**").authenticated()

                        // 개발 단계에서는 모든 요청을 일시적으로 허용하여 401 문제를 우회할 수 있습니다.
                        .requestMatchers("/**").permitAll() // 💡 임시 해결책
                );

        // 추가적인 인증 로직 (폼 로그인, 기본 인증 등)은 상황에 따라 설정

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

}
```
처음에 해당 클래스를 등록하고 포스트맨으로 401이 떨어지면서 상당히 당황했다.  
나는 설정한 부분도 없는데 자동으로 화이트리스트를 설정한 느낌이었다.  
  
찾아보니 build.gradle에 등록한 [implementation ＇org.springframework.boot:spring-boot-starter-security＇] 라이브러리가 기본동작이 **허용되지 않은 요청에 대해 방어**를 해준다는 것이었다.  
(Nest.js, Express.js를 사용했을 땐 Apache, AWS에 ELB등 다양하게 직접 막았어야 했는데...)
  
거의 이세카이 기술이다.  
<img src="/assets/images/posts/largeTrafficService/jonnajokun.png" width="50%" height="50%">  
  
PasswordEncoder의 인터페이스 두개로 손쉽게 비밀번호를 암호화하고 비교하는 작업을 마쳤다.  
<img src="/assets/images/posts/largeTrafficService/yesencrypt.png" width="100%" height="30%">    
~~드디어 탈출한 군대식 비밀번호~~
  
  
## JWT
  


```java
common/auth/JwtProvider.java

@Component
public class JwtProvider {

//    private final Key key;
    private final String key;
    private final long accessExpireTime;
    private final long refreshExpireTime;

    public JwtProvider(
            @Value("${jwt.secret}") String secretKey,
            @Value("${jwt.access-expire-time}") long accessExpireTime,
            @Value("${jwt.refresh-expire-time}") long refreshExpireTime
    ) {
//        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
//        byte[] keyBytes = Decoders.BASE64.decode("YXdlc29tZV9sb25nX2FuZF9zZWN1cmVfc2VjcmV0X2tleV9mb3Jfand0X3Rlc3Q=");
//        this.key = Keys.hmacShaKeyFor(keyBytes);
        this.key = secretKey;
        this.accessExpireTime = 3600000; //accessExpireTime;
        this.refreshExpireTime = 3600000; //refreshExpireTime;
    }

    public TokenInfo create(String memberId){
//        String memberId = authentication.getName();
        String accessToken = this.generateAccessToken(memberId);
        String refreshToken = this.generateRefreshToken(memberId);

        return TokenInfo.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .build();
    }

    private String generateAccessToken(String memberId){
        Date now = new Date();
        Date expire = new Date(now.getTime() + accessExpireTime);

        return Jwts.builder()
                .setSubject(memberId) // Payload에 사용자 ID 저장
                .setIssuedAt(now)      // 토큰 발행 시간
                .setExpiration(expire) // 토큰 만료 시간
                .signWith(SignatureAlgorithm.HS256, key) // 비밀 키로 서명
                .compact();

    }

    private String generateRefreshToken(String memberId) {
        Date now = new Date();
        Date validity = new Date(now.getTime() + refreshExpireTime);

        return Jwts.builder()
                .setSubject(memberId)
                .setIssuedAt(now)
                .setExpiration(validity)
                .signWith(SignatureAlgorithm.HS256, key)
                .compact();
    }

}

```  
위의 코드는 AccessToken과 Token을 발급하는 코드다.(로그인 서비스단 코드와 물려있다.)  
주석처리가 되어있는 부분이 많은 것은 차후에 좀 더 활용할 가능성이 있는 부분들에 대해서 삭제하지 않고 주석처리를 한 것이다.  
  
필자의 코드 스타일을 보면 당장에 필요 없다고 생각이 되어도 차후에 충분히 활용 가능성이 있고, 당장 모르더라도 학습 이후에 원활히 활용할 수 있는 코드들에 대해서는 주석처리를 하고 메모를 남겨놓는 편이다.  
  
찾아본 레퍼런스 코드들은 AccessToken에 대해서만 생성하고 리턴하는 경우가 대부분이었다.  
하지만 AccessToken만 발급하는 경우가 많진 않을 것이고...(~~무지한 필자의 기분 탓일 수도...~~)  
  
개인적인 판단으로는 당연히 AccessToken을 발급하면 Token을 발급할 것이고 그렇다면 유사한 기능을 하는데(연관된 기능일뿐더러 예매 관련 서비스라는 관점에서 봤을 때 Token이 절대적으로 필요해 보였다.) 떨어져 있을 이유가 없다고 생각을 했다.  
그렇다면 AccessToken을 발급하면서 Token을 같이 발급하면서 사용자에게 전달하는 것은 당연한 수순일것이다. 그렇기 때문에 토큰을 일괄적으로 생성하여 전달하는 DTO를 생성하고 로직들을 묶었다.

```json
{
    "success": true,
    "msg": "로그인 성공",
    "data": {
        "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0MiIsImlhdCI6MTc2NDMzNDQyOSwiZXhwIjoxNzY0MzM4MDI5fQ.g_RVBKtfuU8_R8FaAofz11eIMd9bfjvuM_UfvxN7R0o",
        "refreshToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0MiIsImlhdCI6MTc2NDMzNDQyOSwiZXhwIjoxNzY0MzM4MDI5fQ.g_RVBKtfuU8_R8FaAofz11eIMd9bfjvuM_UfvxN7R0o"
    }
}
```
  

대부분의 레퍼런스 코드와 GPT, GEMINI는 나에게 [import org.springframework.security.core.Authentication] 라는 것을 대부분 사용했다.  
하지만 대부분의 레퍼런스에서는 정확히 설명하지 못했고 솔직히 지금 포스트를 쓰는 순간에도 정확히 이해하지 못했다. (해당 라이브러리에 관해서는 공부를 하고 개별적으로 포스트를 남길 예정이다.)  
  
현재 이해한 바로는 repository단 까지 연동을 해서 로그인과 같은 로직들을 처리해주는 라이브러리 같은데 이게 과연 변동성과 특수성이 많은 프로덕션 코드에서도 활용될지 의문이 남는다.  
  
필자는 해당 라이브러리에서 제공해주는 인증과정을 구현을 했기 때문에 해당 라이브러리를 사용하지 않고 직접 회원 아이디를 전달하는 식으로 구현했다.  

많은 개발 관련 기술 블로그에서 제공하는 코드들은 언제까지나 스프링을 적극 활용해서 베스트 프렉티스에 가까운 **예쁜** 코드를 보여주는 느낌이 든다.  
하지만 모든 서비스가 그럴 수는 없다고 생각한다. 베스트 프렉티스는 말 그대로 베스트 프렉티스 일뿐...  
오히려 제한된 상황에서 최선의 로직으로 코드를 정리하고 팀원들과의 소통을 통해서 교통정리를 하면서 효율적으로 개선하는 역량이 중요하다고 생각한다.  
  
이렇게 글로 정리하는 첫 번째 프로젝트이기에 오히려 내가 겪어왔던 상황에 대입하여서 코드를 어떻게 개선해나가는지에 대한 관점으로 이 프로젝트 회고록 시리즈를 봤으면 좋겠다.