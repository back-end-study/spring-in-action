## 1. 스프링 시큐리티 활성화하기

- 스프링 부트 스타터의 시큐리티 의존성을 빌드 명세에 추가하자 !
    - xml 추가 방식 
    ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    ```

    - gradle 추가방식
    ```groovy
    implementation 'org.springframework.boot:spring-boot-starter-security'
    ```

위와 같이 추가를 해주면 스프링 애플리케이션이 시작되면서 스프링이 우리프로젝트에 스프링 시큐리티 라이브러리를 찾아 기본적인 보안 구성을 설정해 준다.

### 스프링 시큐리티가 자동으로 제공 보안 구성

1. 모든 HTTP 요청 경로는 인증 되어야 한다.
2. 어떤 특정 역할이나 권한이 없다
3. 로그인 페이지가 따로없다.
4. 스프링 시큐리티의 HTTP 기본인증을 사용해서 인증된다
5. 사용자는 하나만 있으며, 이름은 user 이다. 비밀번호는 암호화 해준다.

## 2. 스프링 시큐리티 구성하기

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    DataSource dataSource;

    @Autowired
    private UserDetailsService userDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // HTTP 보안을 구성 하는 메서드
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/designs", "/orders")
                .access("hasRole('ROLE_USER')")
                .antMatchers("/", "/**")
                .access("permitAll") 
                .and() 
                    .formLogin() 
                    .loginPage("/login")
                    .defaultSuccessUrl("/designs", true) 
                    .failureUrl("/login?error=true")
                .and()
                    .logout()
                    .logoutSuccessUrl("/")
                .and()
                    .csrf();
    }

    // 사용자 인증 정보를 구성 하는 메서드
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
                .passwordEncoder(passwordEncoder());
    }
}
```

1. `SecurityConfig` 클래스의 경우 사용자의 HTTP 요청 경로에 대해 접근 제한과 같은 보안관련 처리를 우리가 원하는 대로 할 수 있게 해준다.
2. `SecurityConfig` 클래스는 보안 구성 클래스인 `WebSecurityConfigurerAdapter`의 서브 클래스이다.
    1. `WebSecurityConfigurerAdapter` 를 상속받게 되면 configure 2개가 생성된다
        1. `configure(HttpSecurity http)` 의 경우 HTTP 보안을 구성하는 메서드
        2. `configure(AuthenticationManagerBuilder auth)` 의 경우 사용자 인증 정보를 구성하는 메서드
3. Http보안을 구성하는 메서드에 어떤 설정이 있을까?
    1. antMatchers → 특정 리소스에 대한 권한을 설정할때 사용
    2. permitAll → antMatchers로 설정한 리소스의 접근을 인증절차 없이 사용한다
    3. hasRole → antMatchers로 설정한 리소스의 접근을 특정 ROLE 을 갖고 있는 유저만 허용한다.
    4. formLogin → 로그인 페이지나 기타 로그인 처리 및 성공 실패 처리를 사용하겠따는 의미
4. AuthenticationManagerBuilder
    1. 인증 설정을 만들기 위해 사용되는 객체

### 2-1 사용자 정보를 유지/관리 하는 사용자 스토어 구성 방법

1. **In-memory 사용자 스토어**

    ```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    
        // 인 메모리를 이용한 사용자 스토어
        auth.inMemoryAuthentication()
                .withUser("user1") // 사용자 구성 시작 username 
                .password("{noop}password1") // 비밀번호
                .authorities("ROLE_USER") // 권한
                .and()
                .withUser("user2") 
                .password("{noop}password2")
                .authorities("ROLE_USER");
    }
    ```

    1. 스프링 5부터는 반드시 비밀번호를 암호화해야한다. 만일 암호화 하지 않았을 경우 403 or 500 오류가 발생하게 된다.
    2. 인메모리 사용자 스토어의 경우 주로 테스트 목적이나 간단한 어플리케이션에 적합하다.
2. **JDBC 기반 사용자 스토어**

    ```java
    @Autowired
    DataSource dataSource;
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    
        // JDBC를 이용한 사용자 스토어
        auth.jdbcAuthentication()
                .dataSource(dataSource)
                .usersByUsernameQuery(
                        "select username, password, enabled from users " +
                                "where username=?")
                .authoritiesByUsernameQuery(
                        "select username, authority from authorities " +
                                "where username=?")
                .passwordEncoder(new BCryptPasswordEncoder());
    }
    ```

    1. 스프링에서는 사용자 정보를 저장하는 테이블과 열이 정해져 있고 쿼리가 미리 생성 되어 있다. 그러기 때문에 시큐리티에서 `usersByUsernameQuery` 쿼리를 날릴 수 있다.
    2. 스프링 5 부터는 암호화가 필수이다 `passwordEncoder()` 메소드를 호출하여 encode 한다
        - Pbkdf2PasswordEncoder : PBKDF2를 암호화한다.
        - BCryptPasswordEncoder : bcrypt를 해싱 암호화 한다.
        - NoOpPasswordEncoder : 암호화 하지 않는다.
        - SCryptPasswordEncoder : scrypt를 해싱 암호화 한다.
        - StandardPasswordEncoder : SHA-256를 해싱 암호화 한다.
3. **LDAP 사용자 스토어**

    ```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    
        auth.ldapAuthentication()
                .userSearchBase("ou=people")
                .userSearchFilter("(uid={0})")
                .groupSearchBase("ou=groups")
                .groupSearchFilter("(member={0})")
                .contextSource()
                .root("dc=tacocloud,dc=com")
                .ldif("classpath:users.ldif")
                .and()
                .passwordCompare() // 비밀번호 비교
                .passwordEncoder(new BCryptPasswordEncoder())
                .passwordAttribute("userPasscode");
    }
    ```

   1. LDAP 이란?
      1. 디렉터리 서비스를 조회하고 수정하는 응용프로토콜이다.
      2. 주로 인증을 위한 다른 서비스에서 자주 사용된다.
      3. 사용자 시스템 네크워크, 앱 정보를 공유하기 위한 목적으로 사용 되며 사용자 정보를 중앙 집중적으로 관리하는데 유용하다
      4. 디렉토리 형식의 트리계층 구조이며 Client- Server 구조 기반이다
      5. 어떻게 활용될까?
         1. LDAP 을 이용해 회사에서는 구성원의 조직도나 팀별 이메일 주소를 LDAP 서비스로 관리하고 인증을 위한 용도로 많이 사용함
      6. `ldapAuthentication()` 이 갖고 있는 메서드
          - `userSearchFilter()`와 `groupSearchFilter`메서드는 LDAP 기본 쿼리의 필터를 제공하기 위해 사용
          - `userSearchBase()`는 사용자를 찾기 위한 기준점 쿼리를 제공
          - `groupSearchBase()`는 그룹을 찾기 위한 기준점 쿼리를 지정
      7. LDAP의 기본인증 전략은 사용자가 직접 LDAP 서버에서 인증을 받도록 하는것
          1. 그러나 비밀번호를 비교하는 방법도 존재함
          2. 만약 비밀번호 비교 방식으로 사용하려면 `passwordCompare` 를 사용하면 된다
      8. 원격 LDAP 서버 참조하기
          1. 기본적으로 스프링 시큐리티의 LDAP 인증에서는 [localhost:33389](http://localhost:33389) 포트로 LDAP 서버가 접속된다고 간주한다.
          2. 만약 LDAP 서버가 다른 컴퓨터에서 실행중이라면 `contextSource` 를 이용하여 해당 서버의 위치를 구성할 수 있다.

          ```java
          // 이런식으로 구성이 가능하다.
          .contextSource()
          .url("ldap://tacocloud.com:389/dc=tacocloud,dc=com");
          ```

      9. 내장된 LDAP 서버 구성하기

          ```xml
              <dependency>
                  <groupId>org.springframework.boot</groupId>
                  <artifactId>spring-boot-starter-data-ldap</artifactId>
              </dependency>
              <dependency>
                  <groupId>org.springframework.ldap</groupId>
                  <artifactId>spring-ldap-core</artifactId>
              </dependency>
              <dependency>
                  <groupId>org.springframework.security</groupId>
                  <artifactId>spring-security-ldap</artifactId>
              </dependency>
          ```

          - 이런식으로 pom.xml 에 추가하여 LDAP 서버를 구성할 수 있음

4. **사용자 인증의 커스터 마이징**

    ```java
    public class User implements UserDetails {
            private Long id;
            private final String username;
            private final String password;
    
            // userDetails 상속
            @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"));
        }
    
        @Override
        public boolean isAccountNonExpired() {
            return true;
        }
    
        @Override
        public boolean isAccountNonLocked() {
            return true;
        }
    
        @Override
        public boolean isCredentialsNonExpired() {
            return true;
        }
    
        @Override
        public boolean isEnabled() {
            return true;
        }
    }
    ```

    - User 클래스는 스프링 시큐리티의 UserDetails 인터페이스를 구현
        - `getAuthorities()` : 사용자에게 부여된 권한을 저장한 컬렉션을 반환한다.
        - `isAccountNonExpired()`, `isCredentialsNonExpired()` 등은 사용자 계정 활성화 또는 비활성화 여부를 나타내는 boolean 값을 리턴
    - 사용자 명세 서비스 생성하기
        - 스프링 시큐리티의 UserDetailsService 는 다음과 같이 간단한 인터페이스이다.

        ```java
        public interface UserDetailsService {
                UserDetails loadUserByUsername(String username) 
                    throws UsernameNotFoundException;
        }
        ```

### 3. 웹 요청 보안 처리하기

보안 규칙을 구성하려면 SecurityConfig 클래스 다음에 configure 메서드를 오버라이딩 해야한다.
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
}
```
`configure(HttpSecurity http)` 메서드는 `HttpSecurity` 를 객체 인자로 받으며 이 객체는 웹수준에서 보안을 처리하는 방법을 구성하는데 사용된다.

`HttpSecurity` 로 구성할 수 있는것은 무엇이 있을까?

- HTTP 요청 처리를 허용하기 전에 충족되어야 할 특정 보안 조건을 구성한다.
- 커스텀 로그인 페이지를 구성한다
- 사용자가 애플리케이션의 로그아웃을 할 수 있도록한다
- CSRF 공격으로 부터 보호 하도록 구성한다.

```java
// 이것 저것 적용한 예시
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .antMatchers("/designs", "/orders") // ROLE_USER 권한이 있는 유저만 허용
            .access("hasRole('ROLE_USER')")
            .antMatchers("/", "/**")
            .access("permitAll") // 이 외의 요청은 모두 허용
            .and() // 인증 구성이 끝나서 http 구성을 적용할 준비가 되었다
                .formLogin() // 커스텀 로그인 폼을 구성하기 위해 호출
                .loginPage("/login")
                //.loginProcessingUrl("/authenticate") // /authenticate 경로의 로그인 처리
                //.defaultSuccessUrl("/designs") // 로그인 페이지로 이동한 후 로그인 성공시 /design 페이지로 이동
                .defaultSuccessUrl("/designs", true) // 로그인 전에 어떤 페이지에 있었던 로그인 성공시 /design 페이지로 이동
                .failureUrl("/login?error=true")
                //.usernameParameter("user")
                //.passwordParameter("pwd")
            .and()
                .logout()
                .logoutSuccessUrl("/")
            .and()
                .csrf();
}
```
### 3-1. 웹요청 보안 처리하기

`authorizeRequests()`는 `ExpressInterceptUrlRegistry` 객체를 반환한다. 이 객체를 사용하면 URL 경로와 패턴 및 해당경로의 보안 요구사항을 구성할수 있다.

이때 중요한 것은 순서가 중요하다. 순서를 잘못 걸면 예상과는 다르게 작동 할 수 있다.

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .antMatchers("/designs", "/orders")
            .access("hasRole('ROLE_USER')")
            .antMatchers("/", "/**")
            .access("permitAll")
}
```
예를들어 antMatchers() 에서 지정된 경로의 패턴일치를 검사하므로 먼저 지정된 보안규칙이 우선 적용된다. 만약 antMatehrs() 의 순서를 앞뒤로 바꾸게 되면 permitAll() 이 먼저 작동하게 되므로 /designs", "/orders" 는 무의미 해지게 된다.


| 메서드 | 하는일 |
| --- | --- |
| access(String) | 인자로 전달된 SpEL 표현식이 true 면 접근 허용 |
| anonymous() | 익명의 사용자에게 접근 허용 |
| authenticated() | 익명이 아닌 사용자로 인정된 경우 접근 허용 |
| denyAll() | 무조건 접근을 거부 |
| fullyAuthenticated() | 익명이 아니거나 또는 remember-me 가 아닌 사용자로 인증되면 접근 허용 |
| hasAnyAuthority(String…) | 지정된 권한 중 어떤 것이라도 사용자가 갖고 있으면 접근 허용 |
| hasAnyRole(String…) | 지정된 권한 중 어느 하나라도 사용자가 갖고 있으면 접근 허용 |
| hasAuthority(String) | 지정된 권한을 사용자가 갖고 있으면 접근 허용 |
| hasIpAddress(String) | 지정된 IP 주소로 부터 요청이 오면 접근 허용 |
| hasRole(String) | 지정된 역항을 사용자가 갖고 있으면 접근 허용 |
| not() | 다른 접근 메서드들의 효력 무효화 |
| permitAll() | 무조건 접근 허용 |
| rememberMe() | remember-Me(이전 로그인 정보를 쿠키나 디비에 저정한 후 일정 기간내 다시 접근 시 저장된 정보로 자동 로그인 됨)을 통해 사용자의 접근 허용 |
- 대부분의 메서드는 요청 처리의 기본적인 보안 규칙을 제공한다. **BUT** 각 메서드에 정의된 보안 규칙만 사용된다는 제약이 있다.
- 따라서 이 대안으로 `access()` 메서드를 사용하면 더 풍부한 보안규칙을 선언하기 위해 SpEL(Spring Expression Language)를 사용할 수 있다.

### 스프링 시큐리티에서 확장된 SpEL

| 보안표현식 | 산출결과 |
| --- | --- |
| authentication | 해당 사용자의 인증 객체 |
| denyAll | 항상 false를 산출한다 |
| hasAnyRole(list of roles) | 지정된 역할 중 어느하나라도 해당 사용자가 갖고 있으면 true |
| hasRole(role) | 지정된 역할을 해당 사용자가 갖고있으면 ture |
| hasIpAddress(IP address) | 지정된 IP주소로 부터 요청이 온 것이면 ture |
| isAnonymous() | 해당 사용자가 익명이면 true |
| isAuthenticated() | 해당 사용자가 익명이 아닌 사용자 인증되었으면 true |
| isFullyAuthenticated() | 해당 사용자가 익명이 아니거나 또는 remember-be가 아닌 사용자로 인증되어있으면 true |
| isRememberMe() | 해당 사용자가 remmber-me로 인증되었으면 true |
| permitAll | 항상 true |
| principal | 해당 사용자의 principal 객체 |
- SpEL 을 사용하면 어떤 보안 규칙도 작성할 수 있다.

### 3-2. 커스텀 로그인 페이지 생성하기

기본 로그인 페이지를 교체하려면 우선 우리의 커스텀 로그인 페이지가 있는 경로를 스프링 시큐리티에게 알려줘야한다.

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .antMatchers("/designs", "/orders") 
            .access("hasRole('ROLE_USER')")
            .antMatchers("/", "/**")
            .access("permitAll")
            .and() 
            .formLogin() 
            .loginPage("/login")
}
```

- `loginPage("/login")`: 로그인 페이지로 이동한다.
- `loginProcessingUrl("/authenticate")`: "/authenticate" 경로로 요청 로그인 처리하라
- `usernameParameter("user")`, `passwordParameter("pwd")`: 사용자 이름과 비밀번호의 필드 이름은 user, pwd
- `defaultSuccessUrl("url")` :  로그인 성공 시 해당 url 로 이동한다.
- `defaultSuccessUrl("url", true)`:  로그인 전에 어떤 페이지에 있었던 로그인 성공시 해당 페이지로 이동
- `failureUrl("/login?error=true")`: 로그인 실패 시 이동할 페이지
- `logout**()`:** 로그아웃
- `logoutSuccessUrl("/")`: 로그아웃 성공 시 이동할 페이지

## 4. CSRF 공격 방어하기

CSRF(Cross-Site Request Forgery)는 많이 알려진 보안 공격이다

즉 사용자가 웹사이트에 로그인한 상태에서 악의적인 코드(사이트 간의 요청을 위조하여 공격하는) 가 삽입된 페이 지를 열면 공격 대상이 되는 웹사이트에 자동으로 폼이 제출되고 이 사이트는 위조된 공격 명령이 믿을 수 있는 사용자로부터 제출된 것으로 판단하게 되어 공격에 노출된다.

스프링 시큐리티에는 CSRF 방어 기능이 존재한다. `_csrf` 라는 이름의 요청 속성에 넣으면 된다.

실제 업무용 애플리케이션에서는 csrf 를 비활성화 하지 말자.

단 REST API 서버로 실행되는 애플리케이션의 경우 CSRF를 disable 해야한다.

→ CSRF의 경우 웹 양식 제출이 있을 경우 대부분 발생하는 공격 그러나 REST API 의 경우 JSON 형태로 데이터를 주고 받기 때문 그래서 disabled 하는것이 좋음

## 4-1. 사용자 인지하기

사용자가 누구인지 결정하는 방법은 여러가지가 존재한다.  그 중 가장 많이 사용되는 방식은 아래와 같다.

- Princial 객체를 컨트롤러에 주입한다.
- `Authentication` 객체를 컨트롤러에 주입한다.
- SecurityContextHolder를 사용해서 보안 컨텍스트를 얻는다.
- `@AuthenticationPrincipal` 어노테이션을 메서드에 지정한다.

`Authentication` 를 이용하는 방식은 타입 변환이 필요하다.

```java
@PostMapping
public String processOrder(
	@Valid Order order,
  Errors errors,
  SessionStatus sessionStatus,
  Authentication auth ) {
	
	(User) auth.getPrincipal(); // 타입 변환이 필요 함
}
```

`@AuthenticationPrincipal` 를 이용하면 명확하게 `User` 객체를 받을 수 있다.

또한 `Authentication` 객체와 동일 하게 보안 특정 코드만 갖는다

```java
@PostMapping
public String processOrder(
	@Valid Order order,
  Errors errors,
  SessionStatus sessionStatus,
	@AuthenticationPrincipal User user) {

		user.getName() // 타입변환 X
}
```

`SecurityContextHolder` 를 이용 하면 아래와 같이 여러가지 보안코드들이 많이 들어간다

하지만 가장 큰 장점은 애플리케이션 어디서든 사용 할 수 있는것이 가장 큰 장점이다

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

User user = (User) auth.getPrincipal()
```

- 스프링 시큐리티의 자동-구성은 보안을 시작하는 데 좋은 방법이다. 그러나 대부분의 애플리케이션에서는 나름의 보안 요구사항을 충족하기 위해 별도의 보안 구성이 필요하다.
- 사용자 정보는 여러 종류의 스토어에 저장되고 관리 될 수 있다. 예를 들어 관계형 데이터 베이스, LDAP 이다.
- 스프링 시큐리티는 자동으로 CSRF 공격을 방어한다
- 인증된 사용자에 관한 정보는 SecurityContext 객체(`SecurityContextHolder.getContext()`에서 반환됨)를 통해서 얻거나, `@AuthenticationPrincipal`을 사용해서 컨트롤러에 주입하면 된다.
