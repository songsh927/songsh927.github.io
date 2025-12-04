---
title: "[티켓 예매 서비스] 회원 서비스 개발 리팩토링 (1/2)"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/backend/large-traffic-service/4

toc: true
toc_sticky: true

date: 2025-12-01
last_modified_at: 2025-12-04
---

## JWT와 Spring Security를 사용한 로그인 로직 리팩토링

기본적인 회원 CRUD는 완성이 되었으니 토큰을 효율적으로 사용하기 위함과 Spring Security를 사용하여 리팩토링을 해보자

기존의 로그인 코드는 직접 회원을 조회하여 비밀번호를 대조하고 토큰을 발급하는 형태였다

```java

public DefaultRes login(String memberId, String memberPassword){

    MemberEntity member = memberRepository.findOneById(memberId)
        .orElseThrow(() -> new RuntimeException("ID " + memberId + "에 해당하는 사용자가 존재하지 않습니다."));

    if(member.getMember_id().equals(memberId) && passwordEncoder.matches(memberPassword, member.getMember_pw())){
            TokenInfo token = jwtProvider.create(member.getMember_idx(), member.getMember_id());
            return DefaultRes.res(true,"로그인 성공", token);
    }

    return false;
}
```
  
이전 포스트에서 언급한 Spring Security에 내장된 Authentication 인터페이스와 관련된 기능을 사용하면 더욱 간편하게 관련기능을 구현할 수 있는 것을 알았다.  
이제 로그인 API를 예시로 Spring Security의 Authentication 인터페이스를 사용하여 리팩토링해보자.

```java
@Getter
public class MemberSecurityEntity implements UserDetails {

    private final String memberId;
    private final String password;
    private final int memberIdx;
    private final Collection<? extends GrantedAuthority> authorities;

    public MemberSecurityEntity(MemberEntity member) {
        this.memberId = member.getMember_id();
        this.password = member.getMember_pw();
        this.memberIdx = member.getMember_idx();

        this.authorities = Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER"));
    }


    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return memberId;
    }

    public Integer getUserIdx(){
        return memberIdx;
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }
}
```

```java
// Controller
@PostMapping("/login")
public ResponseEntity login(@RequestBody LoginDTO loginDTO){
    DefaultRes result = memberService.login(loginDTO.getId(), loginDTO.getPassword());
    MemberSecurityEntity memberInfo = (MemberSecurityEntity) result.getData();
      
    if(result.isSuccess()){
        TokenInfo token = jwtProvider.create(memberInfo.getMemberIdx(), memberInfo.getMemberId());
        return new ResponseEntity(DefaultRes.res(true,"로그인 성공", token), HttpStatus.OK);
    }

    return new ResponseEntity(DefaultRes.res(false,"로그인 실패"), HttpStatus.BAD_REQUEST);
}
```
  
```java

// MemberService
@Service
@RequiredArgsConstructor
public class MemberService {

    // 코드 생략

    public DefaultRes login(String memberId, String memberPassword){
      Authentication authenticationToken = new UsernamePasswordAuthenticationToken(
          memberId,
          memberPassword
      );

        try{
            Authentication authenticated = authenticationManager.authenticate(authenticationToken);
            MemberSecurityEntity memberInfo = (MemberSecurityEntity) authenticated.getPrincipal();
            return DefaultRes.res(true, "로그인 성공", memberInfo);
        }catch (Exception e){
            return DefaultRes.res(false, "로그인 실패");
        }
    }
}
```
  
```java

// MemberSecurityService
@Service
@RequiredArgsConstructor
public class MemberSecurityService implements UserDetailsService {

    private final MemberRepository memberRepository;


    @Override
    public UserDetails loadUserByUsername(String memberId) throws UsernameNotFoundException {

        return memberRepository.findOneById(memberId).map(
                member -> new MemberSecurityEntity(member)
        ).orElseThrow(() -> new UsernameNotFoundException(memberId + "을 찾을 수 없습니다."));
    }
}
```
  
MemberSecurityService라는 클래스를 하나 더 생성하여 Authentication 인터페이스를 사용해야한다.  
(대부분의 예시에서는 약속한듯이 CustomUserDetailService 라고 클래스를 만들어서 사용하길래 처음에는 해당 이름의 클래스가 미리 선언되어서 Authentication 인터페이스에서 빈에 자동 등록되어 사용되는 줄 알았다...)  
  
Authentication 인터페이스를 사용하기 위해서는 위의 코드처럼 UserDetail 인터페이스를 상속받은 엔티티와 UserDetailService라는 인터페이스를 상속받은 서비스 클래스를 만들어 사용하면 된다.  
코드가 더 늘어나는데 어느 부분에서 좋은 것이냐... 라고 하면  
직접 아이디와 비밀번호에 대한 검증과정을 구현해야하는 과정을 생략할 수 있다.  
또한 리팩토링한 코드를 보면 Authentication 객체를 생성하여 Spring Security 내부에서 토큰을 생성하여 SecurityContext에 저장되고, 프로젝트 내부 전반에서 호출하여 사용자 정보를 사용이 가능하다.  
  
Spring Security를 효율적으로 사용하기 위해 프로젝트를 진행하면서 공부하면서 추가적으로 리팩토링을 해야겠다.  
  
  
  
컨트롤러단의 로그인 코드를 보면 서비스단에서 넘어온 성공여부 확인 후 Response Body에 토큰을 담아서 전달한다.  
Node.js로 서비스 개발을 할 당시에는 토큰에 다양한 정보를 담아 인증과 함께 미들웨어에서 인증관련 기능을 구현했다.  
  
Spring에서는 해당 기능을 Spring Security에서 filter기능으로 한다.  

```java
@Component
@RequiredArgsConstructor
public class JwtCommonFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;
    private final MemberSecurityService memberSecurityService;
    private final RedisService redisService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {

        String path = request.getRequestURI();

        String token = request.getHeader("x-access-token");
        String checkToken = (String) redisService.getValues(token);

        if(token != null || checkToken != null){
            // TODO return 401
            System.out.println(":::: token : " + token);
            System.out.println(":::: check : " + checkToken);
            System.out.println(":::: return 401");

            response.sendError(401,"로그인 정보가 유효하지 않습니다.");
        }

        if (jwtProvider.validateToken(token)) {

            Claims extractedTokenInfo = jwtProvider.extractAllClaims(token);

            UserDetails userDetails = memberSecurityService.loadUserByUsername((String) extractedTokenInfo.get("memberId"));
            Authentication authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authentication);

            Map<String, String> member = new HashMap<>();

            member.put("memberIdx", extractedTokenInfo.get("memberIdx").toString());
            member.put("memberId", (String) extractedTokenInfo.get("memberId"));
            member.put("token", token);

            request.setAttribute("user", member);
        }


        chain.doFilter(request, response);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        String path = request.getRequestURI();
        // JWT 토큰이 필요 없는 경로들에 대해 필터를 건너뜁니다
        return path.startsWith("/member/login") ||
                path.startsWith("/member/create") ||
                path.startsWith("/oauth2") ||
                path.startsWith("/login");
    }
}
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private JwtProvider jwtProvider;
    private JwtCommonFilter jwtCommonFilter;

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        // AuthenticationConfiguration 객체로부터 AuthenticationManager를 가져옵니다.
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtCommonFilter jwtCommonFilter) throws Exception{
        http
                .addFilterBefore(jwtCommonFilter, UsernamePasswordAuthenticationFilter.class)
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/**").permitAll() //임시 해결책
                );



        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

}
```

처음에는 **OncePerRequestFilter**가 아닌 **Filter**을 상속하여 개발했다.  
불편한 부분이 너무 많아 찾아보니 OncePerRequestFilter를 사용하는 것이 맞았고, 해당 클래스를 들어가보니...  
GenericFilterBean를 상속하는 것을 발견했고...  
GenericFilterBean는 놀랍게도(~~어이없게도...~~) Filter를 상속받아 구현된 구현체였다...!

<img src="/assets/images/posts/largeTrafficService/filter.png" width="100%" height="50%">  
  
  
컨트롤러단에서 도메인을 나누어 토큰검증을 받아야하는 도메인과 아닌 도메인을 나눌수 있겠지만 결국 예외상황은 벌어지게 마련이다.  
나의 경우는 Member 도메인에서 로그인과 회원가입은 토큰검증이 없어도 되는 API였고, 나머지는 토큰 검증이 필요한 API였다.  
이 역시나 Filter를 상속해서 개발했을땐 하나하나 구현을 해야했지만 OncePerRequestFilter를 사용하니 필터 예외처리도 자동으로 **shouldNotFilter** 에서 처리가 되었다.  
(역시 뭔가가 어렵고 불편하면 선배님들의 지혜를 찾아보자)  
  
토큰을 발행했으면 이를 순전히 Validation하여 사용도 가능하지만  
리프레쉬 토큰이나 로그아웃같은 예외상황에서는 토큰을 어딘가에 저장해야된다.  
이것은 Redis를 사용하여 다음 포스트에서 정리해보겠다.  
(원래는 여기에 다 쓸려고 했지만 너무 길다. 안그래도 말을 조리있게 못하는데 길게 적어놓으면 더 못해보인다.)