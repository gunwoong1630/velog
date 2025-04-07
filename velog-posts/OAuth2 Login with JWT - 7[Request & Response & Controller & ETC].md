<h1 id="request--response">Request &amp; Response</h1>
<p>우리가 통신을 통해 정보를 주고 받을 때 어떻게 받을 건지 중요하다.</p>
<p>아래와 같은 객체를 설정하여 api를 주고 받을 것이다.</p>
<pre><code class="language-java">@Getter
public class ApiResponse&lt;T&gt; {

    private final static int SUCCESS = 200;
    private final static int NOT_FOUND = 400;
    private final static int FAILED = 500;
    private final static String SUCCESS_MESSAGE = &quot;SUCCESS&quot;;
    private final static String NOT_FOUND_MESSAGE = &quot;NOT FOUND&quot;;
    private final static String FAILED_MESSAGE = &quot;서버에서 오류가 발생하였습니다.&quot;;
    private final static String INVALID_ACCESS_TOKEN = &quot;Invalid access token.&quot;;
    private final static String INVALID_REFRESH_TOKEN = &quot;Invalid refresh token.&quot;;
    private final static String NOT_EXPIRED_TOKEN_YET = &quot;Not expired token yet.&quot;;

    private final ApiResponseHeader header;
    private final Map&lt;String, T&gt; body;

    @JsonCreator
    public ApiResponse(@JsonProperty(&quot;header&quot;) ApiResponseHeader header, @JsonProperty(&quot;body&quot;) Map&lt;String, T&gt; body) {
        this.header = header;
        this.body = body;
    }


    public static &lt;T&gt; ApiResponse&lt;T&gt; success(String name, T body) {
        Map&lt;String, T&gt; map = new HashMap&lt;&gt;();
        map.put(name, body);

        return new ApiResponse(new ApiResponseHeader(SUCCESS, SUCCESS_MESSAGE), map);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; fail() {
        return new ApiResponse(new ApiResponseHeader(FAILED, FAILED_MESSAGE), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidAccessToken() {
        return new ApiResponse(new ApiResponseHeader(FAILED, INVALID_ACCESS_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidRefreshToken() {
        return new ApiResponse(new ApiResponseHeader(FAILED, INVALID_REFRESH_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; notExpiredTokenYet() {
        return new ApiResponse(new ApiResponseHeader(FAILED, NOT_EXPIRED_TOKEN_YET), null);
    }
}</code></pre>
<pre><code class="language-java">@Setter
@Getter
public class ApiResponseHeader {
    private int code;
    private String message;

    @JsonCreator
    public ApiResponseHeader(@JsonProperty(&quot;code&quot;) int code,@JsonProperty(&quot;message&quot;) String message) {
        this.code = code;
        this.message = message;
    }
}</code></pre>
<h1 id="controller">Controller</h1>
<p>우리가 원하는 OAuth2 로그인 기능을 간략하게 정리하면 아래와 같다.</p>
<ol>
<li>외부 플랫폼에 로그인하면 해당 내용을 토대로 인증 및 인가 실시</li>
<li>인증 정보를 백엔드 서버에 저장해둬서 다시 로그인할 경우 해당 정보 사용</li>
<li>토큰을 사용하여 로그인하기 때문에 RefreshToken을 저장해두고 AccessToken이 만료기간이 끝날경우 대체로 사용할 것</li>
</ol>
<p>컨트롤러에서 임시로 3번을 처리할 것이다. 후에, 서비스 파트로 리팩토링을 하는 것을 권한다.</p>
<pre><code class="language-java">@RestController
@RequestMapping(&quot;/api/v1/auth&quot;)
@RequiredArgsConstructor
public class AuthController {

    private final AppProperties appProperties;
    private final AuthTokenProvider tokenProvider;
    private final AuthenticationManager authenticationManager;
    private final UserRefreshTokenRepository userRefreshTokenRepository;

    private final static long THREE_DAYS_MSEC = 259200000;
    private final static String REFRESH_TOKEN = &quot;refresh_token&quot;;

    @PostMapping(&quot;/login&quot;)
    public ApiResponse login(
            HttpServletRequest request,
            HttpServletResponse response,
            @RequestBody AuthReqModel authReqModel
    ) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        authReqModel.getId(),
                        authReqModel.getPassword()
                )
        );

        String userId = authReqModel.getId();
        SecurityContextHolder.getContext().setAuthentication(authentication);

        Date now = new Date();
        AuthToken accessToken = tokenProvider.createAuthToken(
                userId,
                ((UserPrincipal) authentication.getPrincipal()).getRoleType().getCode(),
                new Date(now.getTime() + appProperties.getAuth().getTokenExpiry())
        );

        long refreshTokenExpiry = appProperties.getAuth().getRefreshTokenExpiry();
        AuthToken refreshToken = tokenProvider.createAuthToken(
                appProperties.getAuth().getTokenSecret(),
                new Date(now.getTime() + refreshTokenExpiry)
        );

        // userId refresh token 으로 DB 확인
        UserRefreshToken userRefreshToken = userRefreshTokenRepository.findUserRefreshTokenByUserId(userId);
        if (userRefreshToken == null) {
            // 없는 경우 새로 등록
            userRefreshToken = new UserRefreshToken(userId, refreshToken.getToken());
            userRefreshTokenRepository.saveAndFlush(userRefreshToken);
        } else {
            // DB에 refresh 토큰 업데이트
            userRefreshToken.setRefreshToken(refreshToken.getToken());
        }

        int cookieMaxAge = (int) refreshTokenExpiry / 60;
        CookieUtil.deleteCookie(request, response, REFRESH_TOKEN);
        CookieUtil.addCookie(response, REFRESH_TOKEN, refreshToken.getToken(), cookieMaxAge);

        return ApiResponse.success(&quot;token&quot;, accessToken.getToken());
    }

    @GetMapping(&quot;/refresh&quot;)
    public ApiResponse refreshToken(HttpServletRequest request, HttpServletResponse response) {
        // access token 확인
        String accessToken = HeaderUtil.getAccessToken(request);
        AuthToken authToken = tokenProvider.convertAuthToken(accessToken);
        if (!authToken.validate()) {
            return ApiResponse.invalidAccessToken();
        }

        // expired access token 인지 확인
        Claims claims = authToken.getExpiredTokenClaims();
        if (claims == null) {
            return ApiResponse.notExpiredTokenYet();
        }

        String userId = claims.getSubject();
        RoleType roleType = RoleType.find(claims.get(&quot;role&quot;, String.class));

        // refresh token
        String refreshToken = CookieUtil.getCookie(request, REFRESH_TOKEN)
                .map(Cookie::getValue)
                .orElse((null));
        AuthToken authRefreshToken = tokenProvider.convertAuthToken(refreshToken);

        if (authRefreshToken.validate()) {
            return ApiResponse.invalidRefreshToken();
        }

        // userId refresh token 으로 DB 확인
        UserRefreshToken userRefreshToken = userRefreshTokenRepository.findUserRefreshTokenByUserIdAndRefreshToken(userId, refreshToken);
        if (userRefreshToken == null) {
            return ApiResponse.invalidRefreshToken();
        }

        Date now = new Date();
        AuthToken newAccessToken = tokenProvider.createAuthToken(
                userId,
                roleType.getCode(),
                new Date(now.getTime() + appProperties.getAuth().getTokenExpiry())
        );

        long validTime = authRefreshToken.getTokenClaims().getExpiration().getTime() - now.getTime();

        // refresh 토큰 기간이 3일 이하로 남은 경우, refresh 토큰 갱신
        if (validTime &lt;= THREE_DAYS_MSEC) {
            // refresh 토큰 설정
            long refreshTokenExpiry = appProperties.getAuth().getRefreshTokenExpiry();

            authRefreshToken = tokenProvider.createAuthToken(
                    appProperties.getAuth().getTokenSecret(),
                    new Date(now.getTime() + refreshTokenExpiry)
            );

            // DB에 refresh 토큰 업데이트
            userRefreshToken.setRefreshToken(authRefreshToken.getToken());

            int cookieMaxAge = (int) refreshTokenExpiry / 60;
            CookieUtil.deleteCookie(request, response, REFRESH_TOKEN);
            CookieUtil.addCookie(response, REFRESH_TOKEN, authRefreshToken.getToken(), cookieMaxAge);
        }

        return ApiResponse.success(&quot;token&quot;, newAccessToken.getToken());
    }

}</code></pre>
<p>추가로 나중에 OAuth2 로그인 시 적절하게 redirect했는지, 외부 플랫폼에서 제공한 토큰이 적절한 토큰인지 확인하기 위해 아래와 같은 컨트롤러를 세팅해서 확인할 것이다.</p>
<pre><code class="language-java">@RestController
@Slf4j
@RequiredArgsConstructor
public class UserController {

    private final ApiService apiService;

    @GetMapping(&quot;/oauth/redirect&quot;)
    public ApiResponse checkOAuth2Login(
            HttpServletRequest request,
            HttpServletResponse response,
            @RequestParam String token) {
        return apiService.oAuth2Login(request, response, token);
    }

}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>