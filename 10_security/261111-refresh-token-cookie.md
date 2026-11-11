# Spring Security 3: Refresh Token · DB엔 해시만 저장 · HttpOnly Cookie 전달 · 재발급·로그아웃 API (TIL)

> 실습 환경: Spring Boot 4 + Spring Security·OAuth2 Resource Server(JWT), Spring Data JPA, MySQL `securitydb`
>
> `chap03-refresh-token`에서 Access Token에 더해 **Refresh Token**을 발급·교체·폐기하는 흐름을 만듦. 수업에서 비워 둔 `RefreshTokenService`의 `issue`·`rotate`·`findValidToken`과 `AuthService.refresh`까지 채우고, 제공된 `RefreshTokenFlowTests`(7개)·`CorsFlowTests`(4개)로 동작을 확인함

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| 왜 Refresh Token? | Access Token은 짧게, Refresh Token은 길게 해서 재로그인 없이 새 Access Token 받기 |
| 저장 | `RefreshToken` 엔티티: 회원당 1행(`@OneToOne`), 원문 대신 **SHA-256 해시** 저장 |
| 전달 | Access Token은 응답 **본문**, Refresh Token은 **HttpOnly Cookie** |
| API | `/api/auth/login`(둘 다 발급), `/refresh`(교체·재발급), `/logout`(폐기·쿠키 만료) |
| 기타 | `JwtService`에 `jti`(UUID) 추가, `InvalidRefreshTokenException` → 401, `CorsConfig` |
| 테스트 | 로그인·재발급(교체)·만료·비활성 계정·위조·쿠키 없음·로그아웃, CORS 사전 요청 |

---

## 1. Access Token만 쓸 때의 문제

| | Access Token | Refresh Token |
| :--- | :--- | :--- |
| 용도 | API 호출할 때 `Authorization: Bearer` | Access Token **재발급**에만 사용 |
| 수명 (yml) | `3600`초 (1시간) | `1209600`초 (14일) |
| 형태 | JWT (서명된 Claim) | **랜덤 문자열** (JWT 아님) |
| 서버 저장 | 안 함 (서명만 검증, stateless) | **DB에 해시 저장** (stateful) |
| 전달 위치 | 응답 본문 → 클라이언트가 헤더에 넣음 | HttpOnly Cookie → 브라우저가 자동 전송 |

* Access Token은 서버가 저장하지 않으니 **한번 발급하면 만료 전까지 취소할 수 없음** → 그래서 수명을 짧게 둠
* 그런데 짧게 두면 사용자가 1시간마다 다시 로그인해야 함 → 오래 사는 Refresh Token으로 새 Access Token을 받음
* Refresh Token은 DB에 있으니 **행을 지우면 바로 무효화**(로그아웃) 가능. "취소 가능한 긴 토큰 + 취소 불가능한 짧은 토큰" 조합

```yaml
security:
  jwt:
    secret: ${JWT_SECRET:...}
    expiration-seconds: 3600
  refresh-token:
    expiration-seconds: 1209600
    # 로컬 HTTP 실습에서는 false, 운영에서는 HTTPS + true
    cookie-secure: false
```

---

## 2. `RefreshToken` 엔티티

```java
@Entity
@Table(name = "tbl_refresh_token")
public class RefreshToken {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_account_id", nullable = false, unique = true)
    private UserAccount user;

    @Column(nullable = false, unique = true, length = 64)
    private String tokenHash;

    @Column(nullable = false)
    private Instant expiresAt;

    public void rotate(String tokenHash, Instant expiresAt) {   // 교체: 같은 행의 값만 바꿈
        this.tokenHash = tokenHash;
        this.expiresAt = expiresAt;
    }

    public boolean isExpired(Instant now) {
        return !expiresAt.isAfter(now);   // 만료 시각과 같아도 만료로 처리
    }
}
```

* `user_account_id`에 `unique = true` → **회원당 Refresh Token 1개**. 다시 로그인하면 새 행을 만드는 게 아니라 기존 행을 `rotate()`로 갱신하므로, 다른 기기에서 받은 이전 토큰은 자동으로 무효가 됨
* `tokenHash` 길이 `64` = SHA-256(32바이트)를 16진수로 쓴 길이. `unique`라서 해시로 바로 조회됨
* 변경은 setter 대신 `rotate()`라는 의미 있는 메소드로만 하고, 기본 생성자는 `protected` (JPA용)

```java
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByTokenHash(String tokenHash);   // 쿠키로 받은 토큰 찾기
    Optional<RefreshToken> findByUser(UserAccount user);        // 로그인 시 기존 행 찾기
}
```

---

## 3. `RefreshTokenService`: 원문은 쿠키로, DB엔 해시만

### 원문 만들기: `SecureRandom` + Base64 URL

```java
private static final int TOKEN_BYTE_LENGTH = 32;
private final SecureRandom secureRandom = new SecureRandom();

private String createRawToken() {
    byte[] bytes = new byte[TOKEN_BYTE_LENGTH];
    secureRandom.nextBytes(bytes);
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}
```

* `Random`이 아니라 `SecureRandom`: 예측할 수 없는 난수여야 토큰을 추측하지 못함
* URL-safe Base64 + 패딩(`=`) 제거 → 쿠키 값에 넣어도 문제 없는 문자만 남음
* Refresh Token은 안에 담을 정보가 없으니(누구 토큰인지는 DB가 앎) JWT로 만들 필요가 없음

### 해시: SHA-256

```java
private String hash(String rawToken) {
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hashed = digest.digest(rawToken.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hashed);   // 64자 16진수
    } catch (NoSuchAlgorithmException exception) {
        throw new IllegalStateException("SHA-256 알고리즘을 사용할 수 없습니다.", exception);
    }
}
```

* DB가 유출돼도 해시만으로는 원문을 복원할 수 없으니 **훔친 값으로 재발급을 받을 수 없음** (비밀번호를 해시로 저장하는 이유와 같음)
* 그런데 비밀번호처럼 **BCrypt가 아니라 SHA-256**을 쓰는 이유:

| | 비밀번호 → BCrypt | Refresh Token → SHA-256 |
| :--- | :--- | :--- |
| 원문 | 사람이 정해서 짧고 추측 가능 | 32바이트 난수라서 추측 불가 |
| 솔트 | 매번 랜덤 → 같은 원문이어도 해시가 다름 | 없음 → **같은 원문이면 같은 해시** |
| 조회 | 아이디로 행을 찾고 `matches()`로 비교 | **해시값 자체로 `findByTokenHash` 조회** |

  * 쿠키에는 토큰 원문만 오고 아이디는 없음 → 해시가 항상 같아야 DB에서 바로 찾을 수 있음
  * 원문이 충분히 무작위라서, 느린 해시로 무차별 대입을 막을 필요가 없음

### 클래스 구성

```java
@Service
@Transactional(readOnly = true)   // 기본은 읽기 전용, 쓰는 메소드만 @Transactional
public class RefreshTokenService {
    public String issue(String userId)                   // 로그인: 행 생성 또는 교체 → 원문 반환
    public Rotation rotate(String rawToken)              // 재발급: 검증 → 새 토큰으로 교체
    public void revoke(String rawToken)                  // 로그아웃: 행 삭제
    private RefreshToken findValidToken(String rawToken) // 쿠키 값 → 유효한 토큰 행

    public record Rotation(String userId, UserRole role, String refreshToken) { }
}
```

* `Rotation`: 재발급 결과로 **새 Access Token을 만들 회원 정보**(`userId`, `role`)와 **새 쿠키에 담을 원문**을 묶어서 돌려줌

### `issue`: 로그인할 때 발급

```java
@Transactional
public String issue(String userId) {
    UserAccount user = userAccountRepository.findByUserId(userId)
            .orElseThrow(() -> new InvalidRefreshTokenException("사용할 수 없는 계정입니다."));

    String rawToken = createRawToken();
    String tokenHash = hash(rawToken);
    Instant expiresAt = Instant.now().plusSeconds(expirationSeconds);

    // 회원당 한 행만 유지. 다시 로그인하면 기존 행의 해시를 바꿔 이전 토큰을 무효로 만듦
    refreshTokenRepository.findByUser(user)
            .ifPresentOrElse(
                    token -> token.rotate(tokenHash, expiresAt),
                    () -> refreshTokenRepository.save(
                            new RefreshToken(user, tokenHash, expiresAt))
            );

    return rawToken;   // DB엔 해시, 원문은 쿠키로만
}
```

* `ifPresentOrElse`: 행이 있으면 **수정**, 없으면 **저장**. 이걸 안 하고 매번 `save`하면 `user_account_id`의 `unique` 제약 때문에 두 번째 로그인부터 에러가 남
* 수정 쪽은 `save()`를 부르지 않아도 됨: `@Transactional` 안에서 조회한 엔티티라 **변경 감지(dirty checking)** 로 커밋 때 UPDATE가 나감
* `issue`는 `authenticationManager.authenticate()`가 성공한 **뒤에만** 호출되므로 회원이 없을 일은 사실상 없지만, `orElseThrow`로 방어함

### `findValidToken`: 4단계 검사

```java
private RefreshToken findValidToken(String rawToken) {
    if (rawToken == null || rawToken.isBlank()) {
        throw new InvalidRefreshTokenException("Refresh Token이 필요합니다.");
    }

    RefreshToken token = refreshTokenRepository.findByTokenHash(hash(rawToken))
            .orElseThrow(() -> new InvalidRefreshTokenException("유효하지 않은 Refresh Token입니다."));

    if (token.isExpired(Instant.now())) {
        throw new InvalidRefreshTokenException("만료된 Refresh Token입니다.");
    }
    if (!token.getUser().isEnabled()) {
        throw new InvalidRefreshTokenException("사용할 수 없는 계정입니다.");
    }
    return token;
}
```

| 순서 | 검사 | 실패 상황 |
| :--- | :--- | :--- |
| 1 | 값이 있나 | 쿠키 없이 호출 (`@CookieValue(required = false)` → `null`). 이걸 먼저 안 하면 `hash(null)`에서 NPE → 500 |
| 2 | 해시로 행이 있나 | 위조한 값, 이미 교체된 옛 토큰, 로그아웃한 토큰 |
| 3 | 만료 안 됐나 | 14일 지남 (브라우저 쿠키는 남아 있을 수 있음) |
| 4 | 계정이 활성인가 | 토큰 발급 뒤 관리자가 계정을 정지(`enabled = false`) |

* 4번이 Refresh Token을 DB에 두는 장점이 드러나는 부분. 계정을 정지하면 **다음 재발급부터 바로 막힘** (Access Token만 쓰면 만료 때까지 못 막음)
* `token.getUser()`는 `LAZY`라 이 시점에 회원을 조회함. `open-in-view: false`여도 `rotate()`의 트랜잭션 안이라 문제없음

### `rotate`: 재발급할 때 교체

```java
@Transactional
public Rotation rotate(String rawToken) {
    RefreshToken token = findValidToken(rawToken);

    // 한 번 사용한 토큰은 새 해시로 덮어써서 다시 쓸 수 없게 함
    String newRawToken = createRawToken();
    token.rotate(hash(newRawToken), Instant.now().plusSeconds(expirationSeconds));

    UserAccount user = token.getUser();
    return new Rotation(user.getUserId(), user.getRole(), newRawToken);
}
```

* **Refresh Token Rotation**: 재발급할 때 Access Token뿐 아니라 **Refresh Token도 새로 바꿈**. 옛 원문의 해시는 DB에서 사라지니 같은 쿠키를 다시 보내면 2단계에서 401
* 효과: Refresh Token을 도난당해도 **원래 사용자나 공격자 중 한쪽이 재발급하면 다른 쪽 토큰은 바로 무효**가 됨. 그리고 만료 시각도 다시 14일로 늘어나서, 계속 사용하는 사용자는 로그아웃되지 않음

### `revoke`: 로그아웃

```java
@Transactional
public void revoke(String rawToken) {
    refreshTokenRepository.findByTokenHash(hash(rawToken))
            .ifPresent(refreshTokenRepository::delete);
}
```

* 없는 토큰이어도 예외 없이 그냥 끝남 → 로그아웃은 여러 번 호출해도 결과가 같음 (멱등)

---

## 4. `JwtService`: 재발급용 오버로드 + `jti`

재발급 때는 비밀번호 인증을 하지 않으니 `Authentication` 객체가 없음 → 회원 정보로 Access Token을 만드는 메소드가 필요해짐.

```java
// 로그인: 인증 결과에서 아이디·권한 꺼내기
public String createAccessToken(Authentication authentication) {
    List<String> authorities = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();
    return createAccessToken(authentication.getName(), authorities);
}

// 재발급: Refresh Token으로 찾은 회원 정보 사용
public String createAccessToken(String userId, UserRole role) {
    return createAccessToken(userId, List.of("ROLE_" + role.name()));
}

// 실제 발급은 한 곳에서
private String createAccessToken(String subject, List<String> authorities) {
    Instant issuedAt = Instant.now();
    JwtClaimsSet claims = JwtClaimsSet.builder()
            .issuer("chap03-refresh-token")
            .issuedAt(issuedAt)
            .expiresAt(issuedAt.plusSeconds(expirationSeconds))
            .subject(subject)
            .id(UUID.randomUUID().toString())   // jti
            .claim("authorities", authorities)
            .build();
    ...
}
```

* 두 public 메소드가 결국 같은 private 메소드를 부르므로 **Claim 구성이 한 곳에서만 관리됨**
* `"ROLE_" + role.name()`: 로그인 때 `User.withUsername().roles()`가 붙여 주던 `ROLE_` 접두사를 직접 맞춰 줌. 안 붙이면 재발급 토큰으로 `hasRole("ADMIN")`이 403이 됨
* `.id(UUID)` = `jti` Claim. `iat`이 초 단위라 같은 초에 로그인 → 즉시 재발급하면 **토큰 문자열이 완전히 똑같아질 수 있음**. 랜덤 ID를 넣어 매번 다른 토큰이 되게 함

---

## 5. `AuthService`와 `AuthTokens`

```java
/** 서비스가 만든 두 토큰을 컨트롤러까지 함께 전달하는 내부 결과 */
public record AuthTokens(TokenResponse tokenResponse, String refreshToken) { }
```

```java
public AuthTokens login(LoginRequest request) {
    Authentication authentication = authenticationManager.authenticate(
            UsernamePasswordAuthenticationToken.unauthenticated(
                    request.userId(), request.password()));

    String accessToken = jwtService.createAccessToken(authentication);
    String refreshToken = refreshTokenService.issue(authentication.getName());

    return new AuthTokens(createTokenResponse(accessToken), refreshToken);
}

public AuthTokens refresh(String refreshToken) {
    RefreshTokenService.Rotation rotation = refreshTokenService.rotate(refreshToken);
    String accessToken = jwtService.createAccessToken(rotation.userId(), rotation.role());

    return new AuthTokens(createTokenResponse(accessToken), rotation.refreshToken());
}

public void logout(String refreshToken) {
    if (refreshToken != null && !refreshToken.isBlank()) {
        refreshTokenService.revoke(refreshToken);
    }
}
```

* `refresh`는 비밀번호 인증이 없으니 `Authentication` 대신 `Rotation`의 `userId`·`role`로 Access Token을 만듦 (4장의 오버로드를 쓰는 곳)
* 반환 타입이 `TokenResponse` → `AuthTokens`로 바뀐 이유: Refresh Token은 **응답 본문에 넣으면 안 되니** `TokenResponse`에 필드를 추가하지 않고, 컨트롤러가 쿠키로 따로 꺼내 쓸 수 있게 내부용 record로 묶음
* `logout`은 쿠키가 없거나 비어 있어도 에러 없이 끝남 (이미 로그아웃한 상태여도 OK)
* 주석대로 **로그아웃해도 이미 발급된 Access Token은 만료(최대 1시간)까지 유효함**. 서버가 저장하지 않으니 막을 방법이 없음 → Access Token 수명을 짧게 두는 이유

---

## 6. `AuthController`: HttpOnly Cookie로 주고받기

```java
@PostMapping("/login")
public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request) {
    return tokenResponse(authService.login(request));
}

@PostMapping("/refresh")
public ResponseEntity<TokenResponse> refresh(
        @CookieValue(name = "refresh_token", required = false) String refreshToken) {
    return tokenResponse(authService.refresh(refreshToken));
}

@PostMapping("/logout")
public ResponseEntity<Void> logout(
        @CookieValue(name = "refresh_token", required = false) String refreshToken) {
    authService.logout(refreshToken);
    return ResponseEntity.noContent()                 // 204
            .header(HttpHeaders.SET_COOKIE, clearRefreshCookie().toString())
            .build();
}

private ResponseEntity<TokenResponse> tokenResponse(AuthTokens tokens) {
    return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, refreshCookie(tokens.refreshToken()).toString())
            .body(tokens.tokenResponse());            // 본문엔 Access Token만
}
```

* `@CookieValue(required = false)`: 쿠키가 없을 때 Spring이 400을 내지 않고 `null`을 넘김 → 서비스에서 401로 처리하게 함
* 설정값은 생성자에서 `@Value("${security.refresh-token.cookie-secure:false}")`처럼 받음 (`:false` = 기본값)

### `ResponseCookie` 옵션

```java
private ResponseCookie refreshCookie(String refreshToken) {
    return ResponseCookie.from("refresh_token", refreshToken)
            .httpOnly(true)
            .secure(cookieSecure)
            .sameSite("Strict")
            .path("/api/auth")
            .maxAge(Duration.ofSeconds(refreshExpirationSeconds))
            .build();
}
```

| 옵션 | 의미 | 막는 것 |
| :--- | :--- | :--- |
| `httpOnly(true)` | JavaScript(`document.cookie`)로 읽을 수 없음 | XSS로 토큰 탈취 |
| `secure(...)` | HTTPS 요청에만 전송. 로컬 HTTP 실습은 `false`, 운영은 `true` | 평문 전송 중 가로채기 |
| `sameSite("Strict")` | 다른 사이트에서 시작된 요청엔 쿠키를 안 붙임 | CSRF |
| `path("/api/auth")` | `/api/auth` 하위 요청에만 전송 | 일반 API 요청마다 쿠키가 따라가는 것 |
| `maxAge(...)` | 브라우저 보관 기간 = Refresh Token 수명(14일) | |

* 쿠키 삭제는 따로 API가 없음 → **같은 이름·같은 path**로 값 `""`, `maxAge(Duration.ZERO)`인 쿠키를 내려 주면 브라우저가 지움 (`clearRefreshCookie()`). path가 다르면 다른 쿠키로 취급돼서 안 지워짐
* 왜 Access Token은 쿠키에 안 넣나: 쿠키는 브라우저가 **자동으로** 붙이니 CSRF 위험이 생김. Access Token은 헤더에 직접 넣는 방식으로 두고, 자동 전송이 필요한 Refresh Token만 `SameSite=Strict` + 좁은 `path`로 제한함
* 그래서 Refresh Token을 쓰면서도 `csrf().disable()`이 가능함: 쿠키를 쓰는 곳이 `/api/auth`뿐이고 `SameSite=Strict`가 다른 사이트 요청을 막아 줌

---

## 7. 예외 처리와 CORS

```java
@ExceptionHandler(InvalidRefreshTokenException.class)
public ResponseEntity<Map<String, String>> handleInvalidRefreshToken(
        InvalidRefreshTokenException exception) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(Map.of("message", exception.getMessage()));
}
```

* 없는·만료된·폐기된 Refresh Token은 모두 **401**. 클라이언트는 401을 받으면 "다시 로그인" 화면으로 보내면 됨

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("http://localhost:3000"));   // React 개발 서버
    configuration.setAllowedMethods(List.of("GET", "POST", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("Content-Type", "Authorization"));
    configuration.setAllowCredentials(true);   // 쿠키 주고받기 허용

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", configuration);
    return source;
}
```

* `CorsConfigurationSource` Bean만 만들면 Security에는 적용되지 않음. `SecurityConfig`에서 `.cors(Customizer.withDefaults())`를 켜야 그 Bean을 찾아 **인증 필터보다 먼저** CORS를 처리함. 이게 없으면 브라우저의 사전 요청(`OPTIONS`, 토큰 없음)이 401로 막힘
* 프론트(3000)와 API(8080)는 포트가 달라서 다른 출처(origin) → 쿠키를 보내려면 `allowCredentials(true)` + 프론트에서 `fetch(..., { credentials: 'include' })`가 필요
* `allowCredentials(true)`일 때는 `allowedOrigins`에 `*`를 쓸 수 없어서 출처를 정확히 적음

---

## 8. 실행 흐름 (`api-test.http`)

```
회원 가입 → 로그인 ─┬─ 본문: accessToken  → client.global.set("accessToken", ...)
                    └─ Set-Cookie: refresh_token (HttpOnly)
GET /api/users/me  (Bearer accessToken)                → 200
POST /api/auth/refresh  (쿠키 자동 전송)               → 새 accessToken + 새 쿠키
GET /api/users/me  (새 accessToken)                    → 200
POST /api/auth/logout                                  → 204 + 쿠키 만료, DB 행 삭제
POST /api/auth/refresh                                 → 401 예상
```

* IntelliJ HTTP Client는 앞 응답의 쿠키를 같은 호스트의 다음 요청에 자동으로 붙여 줌 → `/refresh`·`/logout`에 쿠키를 직접 적을 필요 없음

---

## 9. `SecurityConfig` 수정

```java
http
        .cors(Customizer.withDefaults())
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(authorize -> authorize
                // 재발급·로그아웃은 Access Token이 만료된 상태에서도 호출되므로 열어 둠
                .requestMatchers(HttpMethod.POST,
                        "/api/auth/signup", "/api/auth/login",
                        "/api/auth/refresh", "/api/auth/logout").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
        .oauth2ResourceServer(resourceServer -> resourceServer
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter)))
        .authenticationProvider(authenticationProvider);
```

* `/refresh`를 열어 두지 않으면, **Access Token이 만료돼서 재발급하려는 건데** "인증이 필요하다"며 401로 막히는 모순이 생김. 재발급 요청의 인증 수단은 Access Token이 아니라 쿠키의 Refresh Token이고, 그 검사는 `RefreshTokenService`가 함
* 11/10에 "다음 시간"으로 남겨 둔 `oauth2ResourceServer().jwt()` 연결은 chap03 제공 코드에 이미 들어 있음 → `Authorization: Bearer` 토큰이 검증돼서 `/api/users/me`가 동작함

---

## 10. 제공 테스트로 확인

`provided-code/`의 `RefreshTokenFlowTests`·`CorsFlowTests`를 `src/test/java`로 옮겨 `./gradlew test` → **11개 모두 통과**

| 테스트 | 확인하는 것 |
| :--- | :--- |
| 로그인 | Access Token이 비어 있지 않음, 쿠키가 `HttpOnly`·`SameSite=Strict`, **DB의 `tokenHash` ≠ 쿠키 원문** |
| 재발급 | 200 + 새 Access Token·새 쿠키 둘 다 이전 값과 다름, **옛 쿠키로 다시 요청하면 401** (Rotation) |
| 만료 | `jdbcTemplate`로 `expires_at`을 1초 전으로 바꾼 뒤 → 401 "만료된 Refresh Token입니다." |
| 비활성 계정 | `enabled = false`로 바꾼 뒤 → 401 "사용할 수 없는 계정입니다." |
| 위조 | `forged-token` → 401 "유효하지 않은 Refresh Token입니다." |
| 쿠키 없음 | 401 "Refresh Token이 필요합니다." (NPE·500이 아님) |
| 로그아웃 | 204 + `Set-Cookie`에 `Max-Age=0`, DB 행 삭제, 그 쿠키로 재발급하면 401 |
| CORS | 허용 출처(3000)의 사전 요청은 200 + `Allow-Credentials: true`, 다른 출처(4000)는 403, **출처가 허용돼도 토큰 없는 실제 요청은 401** |

* 테스트가 **에러 메시지 문자열까지** 검사하므로, `findValidToken`의 예외 메시지를 테스트와 똑같이 맞춤
* 만료·비활성 테스트는 JPA를 거치지 않고 `jdbcTemplate`로 DB를 직접 바꾼 뒤 `entityManager.clear()`를 함. 1차 캐시에 남아 있는 옛 엔티티를 지워야 다음 조회가 DB의 바뀐 값을 읽음 (11/04 영속성 컨텍스트 내용)
* 마지막 CORS 테스트처럼 **CORS는 "어느 출처의 브라우저 요청을 받을지"일 뿐 인증을 대신하지 않음**

### 같이 고친 것

* `SecurityBasicApplication.java` 맨 끝의 닫는 `}` 뒤에 **백틱(`` ` ``) 한 글자**가 들어가 있어서 컴파일 에러 → 삭제

---

## 📝 오늘의 정리

1. Access Token은 짧게(1시간, 취소 불가), Refresh Token은 길게(14일, DB에서 취소 가능) 두고, Refresh Token은 **재발급에만** 씀
2. Refresh Token은 JWT가 아니라 `SecureRandom` 32바이트를 URL-safe Base64로 만든 **랜덤 문자열**
3. DB에는 원문 대신 **SHA-256 해시**만 저장. 솔트가 없어서 같은 원문이면 같은 해시가 나와 `findByTokenHash`로 바로 조회됨 (BCrypt를 안 쓰는 이유)
4. `RefreshToken`은 회원당 1행(`@OneToOne` + `unique`). 재로그인·재발급 때 새 행이 아니라 `rotate()`로 해시·만료 시각만 교체 → 이전 토큰은 자동 무효
5. `JwtService`는 `Authentication`용과 `(userId, role)`용 오버로드가 같은 private 메소드를 쓰고, `jti`(UUID)로 같은 초에 발급해도 토큰을 구분함
6. 서비스는 `AuthTokens`로 두 토큰을 넘기고, 컨트롤러가 Access Token은 **본문**, Refresh Token은 **`HttpOnly`·`Secure`·`SameSite=Strict`·`path=/api/auth` 쿠키**로 나눠 보냄
7. 재발급은 `findValidToken`(값 있음 → 해시 존재 → 만료 → 계정 활성) 통과 후 **Refresh Token까지 새로 교체(Rotation)** → 옛 쿠키는 한 번 쓰면 401
8. 로그아웃 = DB 행 삭제 + 같은 이름·path의 `Max-Age=0` 쿠키. 이미 발급된 Access Token은 만료까지 남음
9. `/refresh`·`/logout`은 Access Token 없이 호출되므로 `permitAll`, `CorsConfigurationSource`는 `.cors(Customizer.withDefaults())`로 켜야 적용됨

---

## 느낀 점

* 처음엔 "토큰을 두 개나 쓰는 게 더 복잡하기만 하다"고 생각했는데, JWT는 취소할 수 없다는 약점을 **DB에 저장되는 토큰 하나로 보완**하는 구조라는 걸 알고 나니 납득됨. stateless의 편리함과 stateful의 통제력을 섞은 느낌
* 같은 "해시 저장"인데 비밀번호는 BCrypt, Refresh Token은 SHA-256이라는 게 처음엔 헷갈렸음. 원문이 추측 가능한지, 해시로 조회해야 하는지에 따라 고르는 거라는 기준이 생김
* 쿠키 옵션 하나하나가 각각 다른 공격(XSS·CSRF·가로채기)을 막는다는 게 표로 정리하니 확실히 보임. 특히 쿠키를 지울 때 path까지 똑같아야 한다는 건 직접 안 해 봤으면 몰랐을 것 같음
* 재발급 API를 "인증이 필요한 API"로 두면 재발급이 필요한 순간에 오히려 막힌다는 게, 11/10의 "로그인 API를 `permitAll`에 안 넣으면 로그인을 못 한다"와 같은 실수라서 기억에 남음
* 재발급 토큰에 `ROLE_`을 직접 붙여야 한다는 것처럼, 로그인 때 프레임워크가 해 주던 일을 다른 경로에서도 똑같이 맞춰 줘야 한다는 점을 계속 신경 써야겠음
