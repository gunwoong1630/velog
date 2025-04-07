<h1 id="filter">Filter</h1>
<h2 id="tokenauthenticationfilter">TokenAuthenticationFilter</h2>
<p>우리가 로그인을 진행하기 전에 HttpServletRequest를 탐색해서 만약, AccessToken을 받았을 경우 SecurityContextHolder에 해당 인증 정보를 저장해야 한다.</p>
<p>해당 내용을 아래의 코드와 같이 정리하였다.</p>
<pre><code class="language-java">@Slf4j
@RequiredArgsConstructor
public class TokenAuthenticationFilter extends OncePerRequestFilter {
    private final AuthTokenProvider tokenProvider;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        log.info(&quot;gwj : TokenAuthenticationFilter.doFilterInternal&quot;);

        // HTTP 요청에서 Access Token을 가져옵니다.
        String tokenStr = HeaderUtil.getAccessToken(request);

        // Access Token을 이용하여 AuthToken 객체로 변환합니다.
        AuthToken token = tokenProvider.convertAuthToken(tokenStr);

        // AuthToken이 유효한지 확인하고, 유효하다면 인증을 가져온 후 SecurityContext에 설정합니다.
        if (token.validate()) {
            Authentication authentication = tokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        // 다음 필터로 체인을 전달합니다.
        filterChain.doFilter(request, response);
    }
}</code></pre>
<h1 id="userprincipal">UserPrincipal</h1>
<p>우리는 0번째 포스팅에서 얘기했듯이, 인증 및 인가 성공시 인증 정보가 SecurityContext에 저장된다고 하였다. 과연 어떻게 저장할 지 아래와 같이 코딩하였다.</p>
<p>여기서 중요하게 봐야할 점은 &quot;우리가 로그인할 떄 local로 로그인할 때와 oAuth2으로 로그인할 때를 어떻게 처리하냐&quot; 이다.</p>
<p>단순히, 따로따로 처리하게 되면 쓸데없이 유지 보수 비용이 증가할 것이다. 그래서 어댑터 패턴을 사용해서 두 가지 경우에도 하나의 클래스로 처리할 수 있도록 하였다.</p>
<p>나중에 OAuth2UserService의 loadUser를 오버라이딩할 때, 해당 UserPrincipal을 기반으로 생성할 것이다.</p>
<pre><code class="language-java">@Getter
@Setter
@RequiredArgsConstructor
public class UserPrincipal implements OAuth2User, UserDetails, OidcUser {
    private final String userId;
    private final String password;
    private final ProviderType providerType;
    private final RoleType roleType;
    private final Collection&lt;GrantedAuthority&gt; authorities;
    private Map&lt;String, Object&gt; attributes;

    // OAuth2User
    @Override
    public Map&lt;String, Object&gt; getAttribute(String name) {
        return attributes;
    }

    @Override
    public String getName() {
        return userId;
    }

    // UserDetails
    @Override
    public Collection&lt;? extends GrantedAuthority&gt; getAuthorities() {
        return authorities;
    }

    @Override
    public String getUsername() {
        return userId;
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

    // OidcUser

    @Override
    public Map&lt;String, Object&gt; getClaims() {
        return null;
    }

    @Override
    public OidcUserInfo getUserInfo() {
        return null;
    }

    @Override
    public OidcIdToken getIdToken() {
        return null;
    }

    public static UserPrincipal create(SiteUser user) {
        return new UserPrincipal(
                user.getUserId(),
                user.getPassword(),
                user.getProviderType(),
                RoleType.USER,
                Collections.singletonList(new SimpleGrantedAuthority(RoleType.USER.getCode()))
        );
    }

    public static UserPrincipal create(SiteUser user, Map&lt;String, Object&gt; attributes) {
        UserPrincipal userPrincipal = create(user);
        userPrincipal.setAttributes(attributes);

        return userPrincipal;
    }

}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>