---
title: Spring Security custom library 만들기
author: keencho
date: 2022-09-08 08:12:00 +0900
categories: [Spring Security]
tags: [Java, Spring, Spring Security]
---

# **Spring Security custom library 만들기**  
운영중인 시스템에 적용된 Spring Security custom library를 제 방식대로 바꿔보고 겸사겸사 Spring Security의 동작 방식, 원리에 대해 더 깊게 공부하고자 작성한 글입니다.  

> 이 포스팅은 Spring Security가 어떤 방식으로 인증 / 인가 처리하는지 어느정도 이해하고 있다는 가정 하에 작성된 포스팅입니다.  

## **사용된 라이브러리 의존성**
```gradle
dependencies {
    // spring
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    // jwt
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-impl:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-jackson:0.11.5'

    // lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
```  
Spring Data JPA를 사용하여 entity 객체를 중심으로 인증이 진행되도록 하였습니다.  

## **핵심 코드 작성**  
### **1. 베이스 모델**  
첫째는 베이스 모델입니다. 추후 작성될 대부분의 코드의 제네릭에서 이 모델이 사용되기 때문에 가장 중요한 코드라고 생각됩니다.  

```java
@EqualsAndHashCode(callSuper = false)
@Data
@MappedSuperclass
public abstract class KcAccountBaseModel {

    @Column(nullable = false)
    protected String loginId;

    @Column(nullable = false)
    protected String password;

    @Column(nullable = false)
    protected boolean accountNonExpired = true;

    @Column(nullable = false)
    protected boolean accountNonLocked = true;

    @Column(nullable = false)
    protected boolean credentialsNonExpired = true;

    @Column(nullable = false)
    protected boolean enabled = true;

    @Column(nullable = false)
    protected LocalDateTime dtCreatedAt = LocalDateTime.now();

    @Column(nullable = false)
    protected LocalDateTime dtUpdatedAt = LocalDateTime.now();

    protected LocalDateTime dtPasswordChangedAt = LocalDateTime.now();

    protected LocalDateTime dtLastLoggedInAt;

    protected LocalDateTime dtLastAccessedAt;

    @Column(nullable = false)
    protected int loginAttemptCount = 0;
}
```  

각각의 테이블은 용도 (사용자, 관리자, 시스템 등..) 으로 나뉘게 되고 각각의 테이블 엔티티들은 위의 베이스 모델을 상속받아 작성될 것입니다.  

또다른 핵심 베이스 모델은 Security Account 모델입니다.  
```java
@Data
public class KcSecurityAccount implements UserDetails, CredentialsContainer {

    private final Class<?> accountEntityClass;
    private final String loginId;
    private String password;

    private final Set<GrantedAuthority> authorities;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;
    private final boolean credentialsNonExpired;
    private final boolean enabled;
    private final Object data;

    public KcSecurityAccount(Class<?> accountEntityClass, String loginId, String password, Set<GrantedAuthority> authorities,
                             boolean accountNonExpired, boolean accountNonLocked, boolean credentialsNonExpired, boolean enabled,
                             Object data) {
        this.accountEntityClass = accountEntityClass;
        this.loginId = loginId;
        this.password = password;
        this.authorities = authorities;
        this.accountNonExpired = accountNonExpired;
        this.accountNonLocked = accountNonLocked;
        this.credentialsNonExpired = credentialsNonExpired;
        this.enabled = enabled;
        this.data = data;
    }

    @Override
    public void eraseCredentials() {
        this.password = null;
    }

    @Override
    public String getUsername() {
        return null;
    }
}
```  
위 모델은 로그인시 인증 객체로 쓰일 모델이며(UserDetails, CredentialsContainer 구현) 각각의 엔티티별로 또다른 객체 데이터를 가지고 있어야 할수도 있기 때문에 `Object data` 를 통해 이를 충족시키도록 하였습니다.

### **2. 베이스 모델 리포지토리**  
```java
public interface KcAccountRepository<T extends KcAccountBaseModel, ID> extends JpaRepository<T, ID> {
    T findByLoginId(String loginId);
}
```  
`JPARepository` 를 상속하는 베이스모델 리포지토리이며 테이블 엔티티들은 이 리포지토리를 상속하는 또하나의 리포지토리를 작성해야 합니다.  

### **3. 로그인 매니저**  
```java
public interface KcLoginManager<T extends KcAccountBaseModel, R extends KcAccountRepository<T, ?>> extends UserDetailsService, KcAccountResolver<T> {

    Collection<? extends GrantedAuthority> getAuthorities();

    int getMaxLoginAttemptCount();

    int getMaxLongTermNonUseAllowDay();

    @Transactional(readOnly = true)
    T findByLoginId(String loginId);

    @Transactional
    void updateOnLoginSuccess(String loginId);

    @Transactional
    int updateLoginAttemptAccount(String loginId);

    @Transactional
    void lockAccount(String loginId);

}
```  
Spring Security를 접해보셨다면 한번쯤은 봤을 `loadUserByUsername(String username)` 메소드를 가지고있는 인터네이스는 `UserDetailsService`를 상속하는 로그인 매니저 인터페이스 입니다.  

뒤에 보면 `KcAccountResolver`라는 인터페이스를 상속받았는데요, 이것은 핵심 코드 작성 이후 설명할 예정입니다.  

```java
public interface KcAccountResolver<T extends KcAccountBaseModel> {
    T getAccountBySecurityAccount(KcSecurityAccount securityAccount);
}
```  

이런 형태라는것만 알아두시면 좋을것 같습니다.  

다음은 로그인 매니저를 구현하는 기본 로그인 매니저 클래스 입니다.
 
```java
public abstract class KcDefaultLoginManager<T extends KcAccountBaseModel, R extends KcAccountRepository<T, ?>> implements KcLoginManager<T, R> {

    private final R repo;

    public KcDefaultLoginManager(R r) {
        this.repo = r;
    }

    public abstract Collection<? extends GrantedAuthority> getAuthorities();

    public abstract int getMaxLoginAttemptCount();

    public abstract int getMaxLongTermNonUseAllowDay();

    public abstract UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;

    @Override
    public T findByLoginId(String loginId) {
        return repo.findByLoginId(loginId);
    }

    @Override
    public void updateOnLoginSuccess(String loginId) {
        var account = this.findByLoginId(loginId);

        Assert.notNull(account, "account must not be null!");

        var now = LocalDateTime.now();

        account.setDtLastAccessedAt(now);
        account.setDtLastLoggedInAt(now);
        account.setLoginAttemptCount(0);

        repo.save(account);
    }

    @Override
    public int updateLoginAttemptAccount(String loginId) {
        var account = this.findByLoginId(loginId);

        if (account == null) {
            return -1;
        }

        var currentCount = account.getLoginAttemptCount();
        var targetCount = currentCount + 1;

        if (targetCount >= this.getMaxLoginAttemptCount()) {
            // 잠궈야 한다면 로그인시도는 초기화한다.
            targetCount = 0;
            account.setAccountNonLocked(false);
        }

        account.setLoginAttemptCount(targetCount);

        repo.save(account);

        return targetCount;
    }

    @Override
    public void lockAccount(String loginId) {
        var account = this.findByLoginId(loginId);

        Assert.notNull(account, "account must not be null!");

        account.setAccountNonLocked(false);

        repo.save(account);
    }

    @Override
    public T getAccountBySecurityAccount(KcSecurityAccount securityAccount) {
        return this.findByLoginId(securityAccount.getLoginId());
    }
}
```  

추상메소드로 선언된 메소드들은 엔티티별, 비즈니스 로직별로 다를수 있는 부분이기 때문에 따로 작성하였습니다. (예 - 관리자는 로그인시도 3회 허용, 사용자는 로그인시도 5회 허용 등)  

그 외 계정 잠금이나 로그인 횟수 관리등 제가 생각했을때 로그인에 있어 가장 기본이라 생각되는 부분을 구현하였습니다.  

### **4. 인증 프로바이더**  
다음은 인증 프로바이더입니다. `AbstractUserDetailsAuthenticationProvider` 를 상속받은 클래스이며 `PasswordEncoder`와 `UserDetailsService`를 생성자로 주입받습니다.  

아까 기본 로그인 매니저를 작성할때 `loadUserByUsername()` 메소드를 구현해야 했지만 대부분 엔티티별로 다른 로직을 적용해야 하기 때문에 추상 메소드로 두었지요? 바로 그 메소드를 사용하기 위해 `UserDetailsService`를 주입받습니다.  

또한 `AbstractUserDetailsAuthenticationProvider` 클래스의 2개의 추상 메소드를 재정의 합니다. 사용하는 용도에 따라 다른 메소드들도 재정의 할수 있겠네요.

```java
public class KcAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    private final PasswordEncoder passwordEncoder;

    @Getter
    private final UserDetailsService userDetailsService;

    public KcAuthenticationProvider(PasswordEncoder passwordEncoder, UserDetailsService userDetailsService) {
        this.passwordEncoder = passwordEncoder;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        if (authentication.getCredentials() == null) {
            throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
        }

        var passedPassword = authentication.getCredentials().toString();

        if (!passwordEncoder.matches(passedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException(messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
        }
    }

    @Override
    protected UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        UserDetails loadedUser;

        try {
            loadedUser = this.userDetailsService.loadUserByUsername(username);
        } catch (UsernameNotFoundException notFoundException) {
            throw notFoundException;
        } catch (Exception ex) {
            throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
        }

        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException("DefaultUserDetailsService returned null, it must be not null!");
        }

        return loadedUser;
    }
}
```  

### **5. 인증 프로바이더 매니저**  
인증 프로바이더들을 관리할 매니저 클래스를 작성하겠습니다.  

```java
public class KcAuthenticationProviderManagerImpl implements KcAuthenticationProviderManager {

    private final Map<Class<?>, KcAuthenticationProvider> authenticationProviderMap = new HashMap<>();

    @Override
    public Collection<KcAuthenticationProvider> getProviders() {
        return this.authenticationProviderMap.values();
    }

    @Override
    public KcAuthenticationProvider getAuthenticationProvider(Class<?> accountEntityClass) {
        return this.authenticationProviderMap.get(accountEntityClass);
    }

    public void addAuthenticationProvider(Class<?> accountEntityClass, KcAuthenticationProvider authenticationProvider) {
        Assert.isTrue(!this.authenticationProviderMap.containsKey(accountEntityClass), "authenticationProviderMap already has key");

        this.authenticationProviderMap.put(accountEntityClass, authenticationProvider);
    }
}
```

위 클래스는 추후 사용할 어플리케이션에서 bean 으로 등록되어야 합니다. 그리고 각각의 엔티티와 인증 프로바이더를 `addAuthenticationProvider` 메소드를 통해 추가해야 합니다. (이는 예제 작성할때 다루게 됩니다.)  

### **6. JWT 프로바이더**  
Spring Security는 기본적으로 세션 방식입니다. 다만 세션 사용이 제약되는 환경이 있을텐데요. 예를 들어 제 경우 앱개발을 하는데 세션이 의도한대로 동작하지 않아 몇일동안 애먹은 적이 있습니다. 
그런 경우를 대비하여 jwt 토큰방식으로 인증하는 대비책을 세워보았습니다.  

> 주의! 이 포스팅에서는 refresh token을 사용하지 않고 1개의 오리지널 토큰만을 사용합니다. 오리지널 토큰의 유효기간을 길게 잡거나 refresh token 인증 방식을 추가로 적용해보세요. 

```java
public abstract class KcDefaultJwtTokenProvider implements KcJwtTokenProvider {

    private final UserDetailsService userDetailsService;
    private final String claimsKeyName = "loginId";
    private final SecretKey secretKey;

    public KcDefaultJwtTokenProvider(String secretKey, UserDetailsService userDetailsService) {
        this.secretKey = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
        this.userDetailsService = userDetailsService;
    }

    public abstract long getExpireDays();
    public abstract String getCookieName();

    @Override
    public Authentication getAuthentication(String token) {
        var claims = this.getClaims(token);

        var loginId = claims.get(this.claimsKeyName, String.class);

        if (!StringUtils.hasText(loginId)) {
            return null;
        }

        var userDetails = userDetailsService.loadUserByUsername(loginId);

        return new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
    }

    @Override
    public String resolveToken(HttpServletRequest request) {
        var cookies = request.getCookies();
        if (cookies != null) {
            for (var cookie: cookies) {
                if (this.getCookieName().equals(cookie.getName())) {
                    return cookie.getValue();
                }
            }
        }

        return null;
    }

    @Override
    public String createToken(KcSecurityAccount securityAccount) {
        var claims = Jwts.claims().setSubject(UUID.randomUUID().toString());

        claims.put(this.claimsKeyName, securityAccount.getLoginId());
        claims.put("data", securityAccount.getData());

        var limit = LocalDateTime.now().plusDays(this.getExpireDays());
        var date = new Date();

        return Jwts.builder().setClaims(claims)
                .setIssuedAt(date)
                .setExpiration(Date.from(limit.atZone(ZoneId.systemDefault()).toInstant()))
                .signWith(this.secretKey, SignatureAlgorithm.HS256)
                .compact();
    }

    @Override
    public boolean isValidate(String jwtToken) {

        if (!StringUtils.hasText(jwtToken)) {
            return false;
        }

        try {
            var claims = this.getClaims(jwtToken);
            return claims.getExpiration().after(new Date());
        } catch (Exception e) {
            // ExpiredJwtException, MalformedJwtException, SignatureException 이 넘어올수 있다.
            // 만료가 되었다면 아예 예외가 넘어올수 있으므로 예외처리 블록으로 묶고 false를 리턴함.
            return false;
        }
    }

    private Claims getClaims(String jwtToken) {
        return Jwts
                .parserBuilder()
                .setSigningKey(this.secretKey)
                .build()
                .parseClaimsJws(jwtToken)
                .getBody();
    }
}
```  

유효기간 `getExpiresDays()`와 쿠키 이름 `getCookieName()`은 추상메소드로 선언하였습니다. jwt 토큰 관련해서는 `jjwt` 라이브러리를 사용하였습니다.  

### **7. jwt 인증 필터**  
jwt 인증을 사용하려면 결국 필터단에서 cookie를 파싱하여 `SecurityContextHolder` 에 객체를 세팅해줘야 하는데요, 이를위한 공용 필터 코드를 작성 하겠습니다.  

```java
public class KcJwtAuthenticationFilter extends GenericFilterBean {

    private final KcJwtTokenProvider jwtTokenProvider;

    public KcJwtAuthenticationFilter(KcJwtTokenProvider jwtTokenProvider) {
        this.jwtTokenProvider = jwtTokenProvider;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        var token = this.jwtTokenProvider.resolveToken((HttpServletRequest) request);

        if (this.jwtTokenProvider.isValidate(token)) {
            var authentication = this.jwtTokenProvider.getAuthentication(token);

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        chain.doFilter(request, response);
    }
}
```

### **8. 로그인 서비스**  
마지막 핵심 코드인 로그인 서비스 입니다.  

```java
public abstract class KcDefaultLoginService<T extends KcAccountBaseModel, R extends KcAccountRepository<T, ?>> implements KcLoginService<T, R> {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    private final KcAuthenticationProviderManager authenticationProviderManager;
    private final KcLoginManager<T, R> accountLoginManager;
    private final KcJwtTokenProvider jwtTokenProvider;
    private boolean isUseJwtToken = false;

    public KcDefaultLoginService(KcAuthenticationProviderManager authenticationProviderManager, KcLoginManager<T, R> accountLoginManager, KcJwtTokenProvider jwtTokenProvider) {
        this.authenticationProviderManager = authenticationProviderManager;
        this.accountLoginManager = accountLoginManager;
        this.jwtTokenProvider = jwtTokenProvider;

        // jwtTokenProvider가 주입되었다면 jwt token 방식을 사용하는 것이라고 간주한다.
        if (this.jwtTokenProvider != null) {
            this.isUseJwtToken = true;
        }
    }

    public abstract Class<T> getAccountEntityClass();

    @Override
    public Object login(HttpServletResponse response, String loginId, String password) {
        Authentication authentication;
        var token = new UsernamePasswordAuthenticationToken(loginId, password);

        try {
            var authenticationProvider = authenticationProviderManager.getAuthenticationProvider(getAccountEntityClass());

            // 코딩 잘못임. bean에 엔티티 클래스 / 매니저 등록 안함.
            if (authenticationProvider == null) {
                logger.error("system error: authentication provider manager doesn't have target entity class. check your bean configuration");
                throw new KcSystemException();
            }

            authentication = authenticationProvider.authenticate(token);

            // 여기까지 들어왔으면 아이디 / 비밀번호는 일치한다는 의미임
            // 계정을 잠금처리하는 batch 따로 돌아야 하겠지만 돌기 전에 이곳에 들어왔다고 가정하고 장기 미접속 계정 잠금처리
            if (this.isLongTermNotUsed(loginId)) {
                accountLoginManager.lockAccount(loginId);
                throw new LockedException("account locked");
            }

            // jwt 인증은 별도로 구현한 인증과정을 거쳐야 한다는 뜻.
            if (!this.isUseJwtToken) {
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (BadCredentialsException ex) {
            var cnt = accountLoginManager.updateLoginAttemptAccount(loginId);

            // 계정 없음
            if (cnt == -1) {
                throw new KcLoginFailureException();
            } else {
                throw new KcLoginFailureException(cnt, accountLoginManager.getMaxLoginAttemptCount());
            }

        } catch (LockedException ex) {
            // 계정이 잠긴 이유는 일단 2가지라고 생각함.
            // 1. N회 비밀번호 틀린 경우
            // 2. 장기 미접속
            // 따라서 장기 미접속 여부 판단해야함.
            if (isLongTermNotUsed(loginId)) {
                throw new KcAccountLongTermNotUsedException();
            }

            throw new KcAccountLockedException();
        } catch (DisabledException ex) {
            throw new KcAccountDisabledException();
        } catch (KcSystemException ex) {
            throw new KcSystemException();
        } catch (Exception ex) {
            throw new KcLoginFailureException("login failure");
        }

        accountLoginManager.updateOnLoginSuccess(token.getPrincipal().toString());

        var securityUser = (KcSecurityAccount) authentication.getPrincipal();

        if (this.isUseJwtToken) {
            var jwtToken = this.jwtTokenProvider.createToken(securityUser);

            // 토큰을 쿠키에 저장한다. 이때 이름은 tokenProvider에 정의해둔 이름을 사용한다.
            // 필터가 그 이름을 알수 있어야 하는데, 여기서는 KcDefaultJwtTokenProvider에도 동일한 jwtTokenProvider를 주입받게 해두었다.
            // 구현하는 사람 맘대로 하면 된다. 어쨌든 이 쿠키 이름을 필터가 알아야 한다.
            var cookie = ResponseCookie.from(this.jwtTokenProvider.getCookieName(), jwtToken)
                    .sameSite("None")
                    .httpOnly(true)
                    .secure(true)
                    .maxAge(this.jwtTokenProvider.getExpireDays() * 86400)
                    .build();

            response.addHeader("Set-Cookie", cookie.toString());

            return jwtToken;
        }

        return securityUser.getData();
    }

    /**
     * 장기 미접속여부 판단
     * 
     * @param loginId
     * @return true if not use for long time
     */
    private boolean isLongTermNotUsed(String loginId) {
        var account = accountLoginManager.findByLoginId(loginId);
        var maxLongTermNonUseAllowDay = accountLoginManager.getMaxLongTermNonUseAllowDay();
        
        if (account != null) {
            if (maxLongTermNonUseAllowDay > 0) {
                if (account.getDtLastAccessedAt() != null) {
                    if (account.getDtLastAccessedAt().plusDays(maxLongTermNonUseAllowDay).isBefore(LocalDateTime.now())) {
                        return true;
                    }
                }
            }   
        }
        
        return false;
    }
}
```  

jwt 토큰 프로바이더가 주입되었다면 jwt 인증방식을 사용한다고 체크합니다. 또한 jwt 인증방식을 사용한다면 ` SecurityContextHolder.getContext().setAuthentication()` 을 통해 세션에 인증정보를 저장하지 않고 쿠키를 사용합니다.  

또한 장기미접속 여부 판단을 기본적으로 하게 하였습니다. 이 또한 비즈니스 로직에 따라 재정의 될수 있겠네요.  

## **예제 작성**  
간단한 웹 어플리케이션 예제를 작성해 보겠습니다.  

관리자 와 일반 사용자가 있습니다. 이중 관리자는 jwt 토큰 인증 방식을 사용합니다.  

### **1. 엔티티 모델**  
`KcAccountBaseModel`을 상속받는 엔티티 모델을 작성합니다.  

```java
@EqualsAndHashCode(callSuper = false, onlyExplicitlyIncluded = true)
@Data
@Entity
@Table(name = "ADMIN_ACCOUNT")
public class AdminAccount extends KcAccountBaseModel {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
}
```

```java
@EqualsAndHashCode(callSuper = false, onlyExplicitlyIncluded = true)
@Data
@Entity
@Table(name = "USER_ACCOUNT")
public class UserAccount extends KcAccountBaseModel {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
}
```

### **2. 리포지토리**  
`KcAccountRepository`를 상속받는 리포지토리를 작성합니다.  

```java
@Repository
public interface AdminAccountRepository extends KcAccountRepository<AdminAccount, Long> {
}
```  

```java
@Repository
public interface UserAccountRepository extends KcAccountRepository<UserAccount, Long> {
}
```  

### **3. 로그인 매니저**  
`KcDefaultLoginManager` 를 상속받는 로그인 매니저를 작성합니다.  

로그인 매니저를 작성하기 전 `KcSeurityAccount` 객체의 data에 들어갈 객체를 정의하겠습니다.  
```java
@Data
public class LoginAccountData {
    private String loginId;
    private LoginAccountType loginAccountType;
    private Set<String> authorities;
}
```  

다음은 엔티티별 로그인 매니저 입니다.  

```java
@Component
public class AdminLoginManager extends KcDefaultLoginManager<AdminAccount, AdminAccountRepository> {

    public AdminLoginManager(AdminAccountRepository adminAccountRepository) {
        super(adminAccountRepository);
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        var set = new HashSet<String>();
        set.add(AccountRoleCode.ROLE_COMMON);
        set.add(AccountRoleCode.ROLE_ADMIN);

        return set.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toList());
    }

    @Override
    public int getMaxLoginAttemptCount() {
        return 5;
    }

    @Override
    public int getMaxLongTermNonUseAllowDay() {
        return 90;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        if (!StringUtils.hasText(username)) {
            throw new UsernameNotFoundException("Username must be provided!");
        }

        var account = this.findByLoginId(username);

        if (account == null) {
            throw new UsernameNotFoundException("Username not found, username=" + username);
        }

        var authorities = this.getAuthorities();

        var loginData = new LoginAccountData();
        loginData.setLoginId(account.getLoginId());
        loginData.setLoginAccountType(LoginAccountType.ADMIN);
        loginData.setAuthorities(authorities.stream().map(GrantedAuthority::getAuthority).collect(Collectors.toSet()));

        return new KcSecurityAccount(
                AdminAccount.class,
                account.getLoginId(), account.getPassword(), new HashSet<>(authorities),
                account.isAccountNonExpired(), account.isAccountNonLocked(), account.isCredentialsNonExpired(), account.isEnabled(),
                loginData
        );
    }
}
```  

```java
@Component
public class UserLoginManager extends KcDefaultLoginManager<UserAccount, UserAccountRepository> {
    public UserLoginManager(UserAccountRepository userAccountRepository) {
        super(userAccountRepository);
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        var set = new HashSet<String>();
        set.add(AccountRoleCode.ROLE_COMMON);
        set.add(AccountRoleCode.ROLE_USER);

        return set.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toList());
    }

    @Override
    public int getMaxLoginAttemptCount() {
        return 3;
    }

    @Override
    public int getMaxLongTermNonUseAllowDay() {
        return 30;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        if (!StringUtils.hasText(username)) {
            throw new UsernameNotFoundException("Username must be provided!");
        }

        var account = this.findByLoginId(username);

        if (account == null) {
            throw new UsernameNotFoundException("Username not found, username=" + username);
        }

        var authorities = this.getAuthorities();

        var loginData = new LoginAccountData();
        loginData.setLoginId(account.getLoginId());
        loginData.setLoginAccountType(LoginAccountType.USER);
        loginData.setAuthorities(authorities.stream().map(GrantedAuthority::getAuthority).collect(Collectors.toSet()));

        return new KcSecurityAccount(
                UserAccount.class,
                account.getLoginId(), account.getPassword(), new HashSet<>(authorities),
                account.isAccountNonExpired(), account.isAccountNonLocked(), account.isCredentialsNonExpired(), account.isEnabled(),
                loginData
        );
    }
}
```  

여기서 `GrantedAuthority` 에 해당되는 role들은 그냥 일반 String text로 부여해도 되지만, 제 경우 아래와 같이 상수만 가지고 있는 클래스를 하나 만들어 사용하였습니다.  

```java
public class AccountRoleCode {
    public static final String ROLE_COMMON = "ROLE_COMMON";
    public static final String ROLE_ADMIN = "ROLE_ADMIN";
    public static final String ROLE_USER = "ROLE_USER";
}
```  

이렇게 관리하는 편이 추후 `hasAuthority()`로 경로 권한 체크하는 코드에서 이점을 가질수 있을 것이라 생각했습니다.  

### **4. bean 주입**  
다음은 `KcAccountResolverManager` 와 관리자 엔티티 인증에서 사용될 `KcJwtTokenProvider` 를 bean으로 주입해야 합니다. `PasswordEncoder`의 경우 `BcryptPasswordEncoder`를 사용하였습니다.  

```java
@Configuration
public class SecurityBeanContainer {

    private static final String JWT_SECRET_KEY = "Qi0P4mU8ABq6M9nMZG5y67E6hmNad14n";

    @Autowired
    AdminLoginManager adminAccountLoginManager;

    @Autowired
    UserLoginManager userLoginManager;

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public KcAuthenticationProviderManager authenticationProviderManager() {
        var authenticationProviderManager = new KcAuthenticationProviderManagerImpl();

        authenticationProviderManager.addAuthenticationProvider(
                AdminAccount.class,
                new KcAuthenticationProvider(this.bCryptPasswordEncoder(), this.adminAccountLoginManager)
        );

        authenticationProviderManager.addAuthenticationProvider(
                UserAccount.class,
                new KcAuthenticationProvider(this.bCryptPasswordEncoder(), this.userLoginManager)
        );

        return authenticationProviderManager;
    }

    @Bean("adminJwtTokenProvider")
    public KcJwtTokenProvider adminJwtTokenProvider() {
        return new KcDefaultJwtTokenProvider(JWT_SECRET_KEY, this.adminAccountLoginManager) {
            @Override
            public long getExpireDays() {
                return 30;
            }

            @Override
            public String getCookieName() {
                return "KEENCHO_JWT_TOKEN";
            }
        };
    }
}
```  

`JWT_SECRET_KEY` 의 경우 이곳에서는 상수로 선언하였지만 따로 .properties 나 .yml 설정파일로 빼는 편이 보안상 좋을것 같습니다.  

각각의 로그인 서비스를 작성할때 `KcJwtTokenProvider`를 주입받아야 하는데 `@Autowired`는 사용하지 못하고 `@Qualifier`를 사용해야 하기 때문에 `KcJwtTokenProvider`의 경우 별도의 이름을 부여하였습니다.  

### **5. 로그인 서비스** 
다음은 로그인 서비스 입니다. 앞서 말한대로 관리자의 경우 `KcJwtTokenProvider`를 생성자 생성시 주입합니다.  

```java
@Service
public class AdminLoginService extends KcDefaultLoginService<AdminAccount, AdminAccountRepository> {
    public AdminLoginService(
            KcAuthenticationProviderManager authenticationProviderManager,
            AdminLoginManager accountLoginManager,
            @Qualifier("adminJwtTokenProvider")
            KcJwtTokenProvider jwtTokenProvider
    ) {
        super(authenticationProviderManager, accountLoginManager, jwtTokenProvider);
    }

    @Override
    public Class<AdminAccount> getAccountEntityClass() {
        return AdminAccount.class;
    }
}
```  

```java
@Service
public class UserLoginService extends KcDefaultLoginService<UserAccount, UserAccountRepository> {

    public UserLoginService(
            KcAuthenticationProviderManager authenticationProviderManager,
            UserLoginManager accountLoginManager
    ) {
        super(authenticationProviderManager, accountLoginManager, null);
    }

    @Override
    public Class<UserAccount> getAccountEntityClass() {
        return UserAccount.class;
    }
}
```  

### **6. 웹 예제**  
첫째로 커스텀 securityFilterChain을 bean으로 주입합니다.  

> 참고로 Spring Security 최신버전에서는 WebSecurityConfigurerAdapter 가 deprecated되었고 이제는 그냥 bean으로 주입하면 됩니다.  

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfiguration {

    @Autowired
    KcAuthenticationProviderManager kcAuthenticationProviderManager;

    @Autowired
    KcJwtTokenProvider kcJwtTokenProvider;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    
        for (var provider : kcAuthenticationProviderManager.getProviders()) {
            http.authenticationProvider(provider);
        }

        // POST 요청 오픈
        http.csrf().disable();
        http.headers().frameOptions().disable();
        http.httpBasic().disable();

        // jwt token 인증부 필터등록
        http.addFilterBefore(new KcJwtAuthenticationFilter(kcJwtTokenProvider), UsernamePasswordAuthenticationFilter.class);

        http
                .antMatcher("/**").authorizeRequests()
                //
                .antMatchers("/api/auth/test/admin").hasAuthority(AccountRoleCode.ROLE_ADMIN)
                //
                .antMatchers("/api/auth/test/user").hasAuthority(AccountRoleCode.ROLE_USER)
                //
                .antMatchers("/**").permitAll()
                //
                .anyRequest().authenticated();

        return http.build();
    }
}
```  

`api/auth/test/**` 의 경우 각각 경로에 따라 role을 가진 사용자만 접근할 수 있도록 설정하였습니다.  

다음은 로그인 컨트롤러 부분입니다.  
```java
@RestController
@RequestMapping("/api")
public class ApiController {

    @Autowired
    List<KcLoginService<? extends KcAccountBaseModel, ? extends KcAccountRepository<?, ?>>> kcLoginService;

    Map<Class<? extends KcAccountBaseModel>, KcLoginService<? extends KcAccountBaseModel, ? extends KcAccountRepository<?, ?>>> loginServiceMap;

    @PostConstruct
    public void initMap() {
        loginServiceMap = new HashMap<>();

        for (var svc : kcLoginService) {
            loginServiceMap.put(svc.getAccountEntityClass(), svc);
        }
    }

    @PostMapping("/login")
    public Object login(
            @RequestBody Map<String, String> map,
            @RequestParam String type,
            HttpServletResponse response
    ) {
        var id = map.get("id");
        var pw = map.get("pw");

        var clazz = "admin".equals(type) ? AdminAccount.class : UserAccount.class;

        return this.loginServiceMap.get(clazz).login(response, id, pw);
    }

    @GetMapping("/auth/test/admin")
    public String authTestAdmin() {
        return "success";
    }

    @GetMapping("/auth/test/user")
    public String authTestUser() {
        return "success";
    }
}
```  

최초 시작시 각각의 로그인 서비스들을 map에 넣어 로그인시 엔티티를 통해 올바른 서비스를 찾아갈 수 있도록 하였습니다.  

마지막으로 위 api를 테스트하는 html을 만들어 테스트하면 모두 완료입니다. 다만 여기에서는 따로 다루지 않겠습니다. (깃허브 링크를 참고하세요.)  

## **마무리**  
커스텀 라이브러리를 만들어보며 Spring Security에 대해 조금더 자세히 알아볼수 있었습니다. 위 라이브러리에는 몇개의 문제가 있는데요, 가장 큰 문제는 중복 인증 문제입니다.  

A 엔티티가 jwt 토큰 인증방식을 사용하고 B 엔티티는 기존 세션 인증방식을 사용한다고 가정했을때, jwt 인증 필터가 항상 세션 인증 필터 뒤에 있기 때문에 A와 B 모두 로그인된 상태해서 인증을 진행하면 A만 인증이 되는 문제가 일어나게 됩니다. 
하지만 이는 하나의 어플리케이션 일때만 문제가 되며 만약 엔티티별로 다른 모듈을 만들어 각각의 bean을 주입받는 방식으로 멀티모듈 프로젝트를 구성하면 문제가 되지 않습니다.  

여기서는 설명하지 않았지만 `WebMvcConfigurer` 인터페이스의 `addArgumentResolvers` 메소드를 통해 `ControllerAdvice` 혹은 `Controller`의 시작점에 현재 인증된 인증 객체 정보를 가져올수 있게 할수도 있습니다. 예를들어 아래와 같이 말이지요. (깃허브 링크를 참고하세요.)  

```java
@GetMapping("/auth/test/admin")
public AdminAccount authTestAdmin(
        @KcsAccount(required = true) AdminAccount adminAccount,
        @KcsAccount(accountType = KcsAccountType.SECURITY_ACCOUNT) KcSecurityAccount securityAccount
        ) {
    return adminAccount;
}
```  

어쨌든 이 포스팅이 자신만의 Spring Security 인증을 구현하려는 분들께 도움이 되었으면 좋겠습니다.  

> 라이브러리: [https://github.com/keencho/lib-spring/tree/master/src/main/java/com/keencho/lib/spring/security](https://github.com/keencho/lib-spring/tree/master/src/main/java/com/keencho/lib/spring/security)  
> 예제: [https://github.com/keencho/spring/tree/master/security](https://github.com/keencho/spring/tree/master/security)

