<h1 id="user">User</h1>
<p>가장 기본으로 우리가 벡엔드 서버에서 사용자 entity를 제어하기 위해 정의해야 한다.</p>
<p>아래와 같이 코드를 작성하였다. </p>
<pre><code class="language-java">@Getter
@Setter
@NoArgsConstructor
@Entity
@Table(name = &quot;TB_USER&quot;)
public class SiteUser {
    @JsonIgnore
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = &quot;ID&quot;)
    private Long id;

    @Column(name = &quot;USER_ID&quot;, length = 64, unique = true)
    @NotNull
    @Size(max = 64)
    private String userId;

    @Column(name = &quot;USERNAME&quot;, length = 100)
    @NotNull
    @Size(max = 100)
    private String username;

    @JsonIgnore
    @Column(name = &quot;PASSWORD&quot;, length = 128)
    @NotNull
    @Size(max = 128)
    private String password;

    @Column(name = &quot;IS_VERIFIED_EMAIL&quot;)
    @NotNull
    private boolean isVerifiedEmail;

    @Column(name = &quot;PROVIDER_TYPE&quot;, length = 20)
    @Enumerated(EnumType.STRING)
    @NotNull
    private ProviderType providerType;

    @Column(name = &quot;ROLE_TYPE&quot;, length = 20)
    @Enumerated(EnumType.STRING)
    @NotNull
    private RoleType roleType;

    @Column(name = &quot;CREATED_AT&quot;)
    @NotNull
    private LocalDateTime createdAt;

    @Column(name = &quot;MODIFIED_AT&quot;)
    @NotNull
    private LocalDateTime modifiedAt;

    public SiteUser(String userId, String username, boolean isVerifiedEmail, ProviderType providerType, RoleType roleType, LocalDateTime createdAt, LocalDateTime modifiedAt) {
        this.userId = userId;
        this.username = username;
        this.isVerifiedEmail = isVerifiedEmail;
        this.providerType = providerType;
        this.roleType = roleType;
        this.createdAt = createdAt;
        this.modifiedAt = modifiedAt;
    }
}</code></pre>
<pre><code class="language-java">@Getter
@AllArgsConstructor
public enum RoleType {
    ADMIN(&quot;ROLE_ADMIN&quot;,&quot;관리자 권한&quot;),
    USER(&quot;ROLE_USER&quot;, &quot;일반 사용자 권한&quot;),
    GUEST(&quot;ROLE_GUEST&quot;, &quot;게스트 권한&quot;);

    private final String code;
    private final String description;

    private static final Map&lt;String, RoleType&gt; roleTypeMap = Collections.unmodifiableMap(
            Stream.of(values()).collect(Collectors.toMap(RoleType::getCode, Function.identity()))
    );

    public static RoleType find(String code) {
        return Optional.ofNullable(roleTypeMap.get(code)).orElse(GUEST);
    }
}</code></pre>
<pre><code class="language-java">@Getter
public enum ProviderType {
    LOCAL,
    GOOGLE;
}</code></pre>
<h1 id="oauth2userinfo">OAuth2UserInfo</h1>
<p>OAuth2 로그인을 성공적으로 진행했을 때, AccessToken을 통해 Resource Server에 유저에 대한 정보를 해당 클래스를 통해 정보를 담고 사용할 것이다. 수정이 필요없기 때문에 get함수만 선언해서 값만 가져올 수 있도록 변경 가능성을 최소화한다.</p>
<pre><code class="language-java">public class OAuth2UserInfoFactory {
    public static OAuth2UserInfo getOAuth2UserInfo(ProviderType providerType, Map&lt;String, Object&gt; attributes) {
        switch (providerType) {
            case GOOGLE: return new GoogleOAuth2UserInfo(attributes);
            default: throw new IllegalArgumentException(&quot;Invalid Provider Type.&quot;);
        }
    }
}</code></pre>
<pre><code class="language-java">public abstract class OAuth2UserInfo {
    protected Map&lt;String, Object&gt; attributes;

    public OAuth2UserInfo(Map&lt;String, Object&gt; attributes) {
        this.attributes = attributes;
    }

    public Map&lt;String, Object&gt; getAttributes() {
        return attributes;
    }

    public abstract String getId();
    public abstract String getName();
    public abstract String getEmail();
}
</code></pre>
<pre><code class="language-java">public class GoogleOAuth2UserInfo extends OAuth2UserInfo {
    public GoogleOAuth2UserInfo(Map&lt;String, Object&gt; attributes) {
        super(attributes);
    }

    @Override
    public String getId() {
        return (String) attributes.get(&quot;sub&quot;);
    }

    @Override
    public String getName() {
        return (String) attributes.get(&quot;name&quot;);
    }

    @Override
    public String getEmail() {
        return (String) attributes.get(&quot;email&quot;);
    }
}</code></pre>
<p>임시로 Google에 대한 UserInfo만 생성하였다. 필요하다면 다른 플랫폼에 대한 UserInfo를 생성하길 바란다.</p>
<p>추가하는 방법은 별거 없다. 코딩을 하면서 Resource Server에 유저에 관한 정보를 받을 때 어떠한 key값으로 받는지 확인하고 위의 구글과 같이 추가하면 된다.</p>
<h1 id="token">Token</h1>
<p>우선 우리가 알고있는 JWT는 JWS(Json Web Signature)를 의미하는 경우이다. 즉, 어떠한 JWT를 만들 때는 signature가 필수라는 뜻이다.</p>
<p>또한, 인증시 Refresh Token을 사용했다. Refresh Token을 사용한 이유는 JWT가 만료 기간이 지났을 경우 Refresh Token을 메인으로 사용하고 보조 토큰을 해당 Refresh Token을 만들게 되면 비용 절감이 되기 때문이다.</p>
<p>이러한 이유로 처음 JWT를 만들 때 Refresh Token을 같이 만든다.</p>
<p>아래와 같이 토큰을 생성하는 코드를 작성하였다.</p>
<pre><code class="language-java">@Slf4j
@RequiredArgsConstructor
public class AuthToken {

    @Getter
    private final String token;
    private final Key key;

    private static final String AUTHORITIES_KEY = &quot;role&quot;;

    AuthToken(String id, Date expiry, Key key) {
        this.key = key;
        this.token = createAuthToken(id, expiry);
    }

    AuthToken(String id, String role, Date expiry, Key key) {
        this.key = key;
        this.token = createAuthToken(id, role, expiry);
    }

    private String createAuthToken(String id, Date expiry) {
        return Jwts.builder()
                .setSubject(id)
                .signWith(key, SignatureAlgorithm.HS256)
                .setExpiration(expiry)
                .compact();
    }

    private String createAuthToken(String id, String role, Date expiry) {
        return Jwts.builder()
                .setSubject(id)
                .claim(AUTHORITIES_KEY, role)
                .signWith(key, SignatureAlgorithm.HS256)
                .setExpiration(expiry)
                .compact();
    }

    public boolean validate() {
        return this.getTokenClaims() != null;
    }

    public Claims getTokenClaims() {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(key)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (SignatureException e) {
            log.error(&quot;Invalid JWT signature.&quot;);
        } catch (MalformedJwtException e) {
            log.error(&quot;Invalid JWT token.&quot;);
        } catch (ExpiredJwtException e) {
            log.error(&quot;Expired JWT token.&quot;);
        } catch (UnsupportedJwtException e) {
            log.error(&quot;Unsupported JWT token.&quot;);
        } catch (IllegalArgumentException e) {
            log.error(&quot;JWT token compact of handler are invalid.&quot;);
        }
        return null;
    }

    public Claims getExpiredTokenClaims() {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(key)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            log.error(&quot;Expired JWT token.&quot;);
            return e.getClaims();
        }
        return null;
    }
}</code></pre>
<pre><code class="language-java">@Slf4j
public class AuthTokenProvider {
    private final Key key;
    private static final String AUTHORITIES_KEY = &quot;role&quot;;

    public AuthTokenProvider(String secret) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes());
    }

    public AuthToken createAuthToken(String id, Date expiry) {
        return new AuthToken(id, expiry, key);
    }

    public AuthToken createAuthToken(String id, String role, Date expiry) {
        return new AuthToken(id, role, expiry, key);
    }

    public AuthToken convertAuthToken(String token) {
        return new AuthToken(token, key);
    }

    public Authentication getAuthentication(AuthToken authToken) {

        if(authToken.validate()) {
            Claims claims = authToken.getTokenClaims();
            Collection&lt;? extends GrantedAuthority&gt; authorities =
                    Arrays.stream(new String[]{claims.get(AUTHORITIES_KEY).toString()})
                            .map(SimpleGrantedAuthority::new)
                            .collect(Collectors.toList());

            log.debug(&quot;claims subject := [{}]&quot;, claims.getSubject());
            User principal = new User(claims.getSubject(), &quot;&quot;, authorities);

            return new UsernamePasswordAuthenticationToken(principal, authToken, authorities);
        } else {
            throw new TokenValidFailedException();
        }
    }
}
</code></pre>
<p>위의 코드처럼 AuthToken 클래스 안에 토큰과 키 값이 있는데 키 값을 signature로 지정해서 토큰을 생성하였다.</p>
<p>아래와 같이 Refresh Token를 코딩하였다.</p>
<pre><code class="language-java">@Getter
@Setter
@NoArgsConstructor
@Entity
@Table(name = &quot;USER_REFRESH_TOKEN&quot;)
public class UserRefreshToken {
    @JsonIgnore
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = &quot;ID&quot;)
    private Long id;

    @Column(name = &quot;USER_ID&quot;, length = 64, unique = true)
    @NotNull
    @Size(max = 64)
    private String userId;

    @Column(name = &quot;REFRESH_TOKEN&quot;, length = 256)
    @NotNull
    @Size(max = 256)
    private String refreshToken;

    public UserRefreshToken(String userId, String refreshToken) {
        this.userId = userId;
        this.refreshToken = refreshToken;
    }
}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>