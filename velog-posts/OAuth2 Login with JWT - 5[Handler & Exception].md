<h1 id="handler">Handler</h1>
<p>우리는 커스텀한 로그인을 위해 해당 로그인이 성공했을 때와 실패했을 때를 구분하고 만약 AccessDenied 됐을 때, 어떻게 처리해야할지 정해야 한다.</p>
<h2 id="oauth2authenticationfailurehandler">OAuth2AuthenticationFailureHandler</h2>
<p>실패했을 경우, 리다이렉션 경로에 에러 이유를 담아 전송해야 한다. </p>
<p>해당 내용은 아래의 코드와 같다.</p>
<pre><code>@Component
@RequiredArgsConstructor
public class OAuth2AuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {
    private final OAuth2AuthorizationRequestBasedOnCookieRepository authorizationRequestRepository;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        String targetUrl = CookieUtil.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
                .map(Cookie::getValue)
                .orElse((&quot;/&quot;));

        exception.printStackTrace();

        targetUrl = UriComponentsBuilder.fromUriString(targetUrl)
                .queryParam(&quot;error&quot;, &quot;target exception occur&quot;)
                .build().toUriString();

        authorizationRequestRepository.removeAuthorizationRequestCookies(request, response);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}</code></pre><h2 id="oauth2authenticationsuccesshandler">OAuth2AuthenticationSuccessHandler</h2>
<p>만약 로그인이 성공했을 경우 우리는 리다이렉션 경로를 향해 자체적으로 만든 토큰을 담아 전송해줘야 한다. 그렇게 된다면, 해당 사용자는 받은 토큰을 통해 로그인할 수 있을 것이다.</p>
<p>추가로 AccessToken과 RefreshToken을 같이 만들고 RefreshToken은 데이터베이스에 저장해놨다가 해당 AccessToken이 만료되면 RefreshToken을 통해 사용하기 때문에 AccessToken과 다르게 repository에 저장해놔야 한다.</p>
<p>해당 내용은 아래와 같다.</p>
<pre><code class="language-java">@Component
@Slf4j
@RequiredArgsConstructor
public class OAuth2AuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {
    private final AuthTokenProvider tokenProvider;
    private final AppProperties appProperties;
    private final UserRefreshTokenRepository userRefreshTokenRepository;
    private final OAuth2AuthorizationRequestBasedOnCookieRepository authorizationRequestRepository;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
//        log.info(&quot;gwj : OAuth2AuthenticationSuccessHandler.onAuthenticationSuccess&quot;);
        String targetUrl = determineTargetUrl(request, response, authentication);

        if (response.isCommitted()) {
            logger.debug(&quot;Response has already been committed. Unable to redirect to &quot; + targetUrl);
            return;
        }

        clearAuthenticationAttributes(request, response);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }

    protected String determineTargetUrl(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
        Optional&lt;String&gt; redirectUri = CookieUtil.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
                .map(Cookie::getValue);
        if (redirectUri.isPresent() &amp;&amp; !isAuthorizedRedirectUri(redirectUri.get())) {
            throw new IllegalArgumentException(&quot;Sorry! We've got an Unauthorized Redirect URI and can't proceed with the authentication&quot;);
        }

        String targetUrl = redirectUri.orElse(getDefaultTargetUrl());

        OAuth2AuthenticationToken authToken = (OAuth2AuthenticationToken) authentication;
        ProviderType providerType = ProviderType.valueOf(authToken.getAuthorizedClientRegistrationId().toUpperCase());

        OidcUser user = ((OidcUser) authentication.getPrincipal());
        OAuth2UserInfo userInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(providerType, user.getAttributes());
        Collection&lt;? extends GrantedAuthority&gt; authorities = ((OidcUser) authentication.getPrincipal()).getAuthorities();

        RoleType roleType = hasAuthority(authorities, RoleType.ADMIN.getCode()) ? RoleType.ADMIN : RoleType.USER;

        Date now = new Date();
        AuthToken accessToken = tokenProvider.createAuthToken(
                userInfo.getId(),
                roleType.getCode(),
                new Date(now.getTime() + appProperties.getAuth().getTokenExpiry())
        );

        // refresh 토큰 설정
        long refreshTokenExpiry = appProperties.getAuth().getRefreshTokenExpiry();
        AuthToken refreshToken = tokenProvider.createAuthToken(
                appProperties.getAuth().getTokenSecret(),
                new Date(now.getTime() + refreshTokenExpiry)
        );

        // DB 저장
        UserRefreshToken userRefreshToken = userRefreshTokenRepository.findUserRefreshTokenByUserId(userInfo.getId());
        if (userRefreshToken != null) {
            userRefreshToken.setRefreshToken(refreshToken.getToken());
        } else {
            userRefreshToken = new UserRefreshToken(userInfo.getId(), refreshToken.getToken());
            userRefreshTokenRepository.saveAndFlush(userRefreshToken);
        }

        int cookieMaxAge = (int) refreshTokenExpiry / 60;

        CookieUtil.deleteCookie(request, response, REFRESH_TOKEN);
        CookieUtil.addCookie(response, REFRESH_TOKEN, refreshToken.getToken(), cookieMaxAge);

        return UriComponentsBuilder.fromUriString(targetUrl)
                .queryParam(&quot;token&quot;, accessToken.getToken())
                .build().toUriString();
    }

    protected void clearAuthenticationAttributes(HttpServletRequest request, HttpServletResponse response) {
        super.clearAuthenticationAttributes(request);
        authorizationRequestRepository.removeAuthorizationRequestCookies(request, response);
    }

    private boolean hasAuthority(Collection&lt;? extends GrantedAuthority&gt; authorities, String authority) {
        if (authorities == null) {
            return false;
        }

        for (GrantedAuthority grantedAuthority : authorities) {
            if (authority.equals(grantedAuthority.getAuthority())) {
                return true;
            }
        }
        return false;
    }

    private boolean isAuthorizedRedirectUri(String uri) {
        URI clientRedirectUri = URI.create(uri);

        return appProperties.getOauth2().getAuthorizedRedirectUris()
                .stream()
                .anyMatch(authorizedRedirectUri -&gt; {
                    // Only validate host and port. Let the clients use different paths if they want to
                    URI authorizedURI = URI.create(authorizedRedirectUri);
                    if (authorizedURI.getHost().equalsIgnoreCase(clientRedirectUri.getHost())
                            &amp;&amp; authorizedURI.getPort() == clientRedirectUri.getPort()) {
                        return true;
                    }
                    return false;
                });
    }

}</code></pre>
<h2 id="tokenaccessdeniedhandler">TokenAccessDeniedHandler</h2>
<p>만약 해당 토큰의 접근이 거부될 경우 해당 내용을 쿠키에 담아서 전달해야 한다. 해당 내용은 아래와 같다.</p>
<pre><code class="language-java">@Component
@RequiredArgsConstructor
@Slf4j
public class TokenAccessDeniedHandler implements AccessDeniedHandler {
    private final HandlerExceptionResolver handlerExceptionResolver;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
//        response.sendError(HttpServletResponse.SC_FORBIDDEN, accessDeniedException.getMessage());
        response.addCookie(new Cookie(&quot;isTokenAccessDenied&quot;,&quot;Y&quot;));
        handlerExceptionResolver.resolveException(request, response, null, accessDeniedException);
    }
}</code></pre>
<h1 id="exception">Exception</h1>
<h2 id="1-oauthprovidermissmatchexception">1. OAuthProviderMissMatchException</h2>
<p>해당 예외는 Provider의 property들을 auto mapping하는 과정에서 잘못될 경우 해당 예외를 던지기 위해 생성한 코드이다.</p>
<pre><code class="language-java">public class OAuthProviderMissMatchException extends RuntimeException {
    public OAuthProviderMissMatchException(String message) {
        super(message);
    }
}</code></pre>
<h2 id="2-restauthenticationentrypoint">2. RestAuthenticationEntryPoint</h2>
<p>해당 예외는 인증시 오류가 발생할 경우 예외를 던지기 위한 코드이다.</p>
<pre><code class="language-java">@Slf4j
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        authException.printStackTrace();
        log.info(&quot;Responding with unauthorized error. Message := {}&quot;, authException.getMessage());
        response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                authException.getLocalizedMessage()
        );
    }
}</code></pre>
<h2 id="3-tokenvalidfailedexception">3. TokenValidFailedException</h2>
<p>해당 코드는 토큰 생성시 오류가 발생할 경우 예외를 던지기 위한 코드이다.</p>
<pre><code class="language-java">public class TokenValidFailedException extends RuntimeException {
    public TokenValidFailedException() {
        super(&quot;Failed to generate Token.&quot;);
    }

    private TokenValidFailedException(String message) {
        super(message);
    }
}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>