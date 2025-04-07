<h1 id="util">Util</h1>
<p>단순한 작업(쿠키, 헤더에 대한 작업)을 함수화하여 유지 보수 용이성을 높혔다.</p>
<h2 id="1-cookieutil">1. CookieUtil</h2>
<p>기본적으로 HttpServletRequest에서 특정 쿠키를 찾거나 찾은 쿠키를 삭제하는 기능과, HttpServletResponse에 쿠키를 삽입하는 기능, 쿠키를 직렬화, 역직렬화하는 기능을 아래의 코드와 같이 구현하였다.</p>
<pre><code class="language-java">public class CookieUtil {
    public static Optional&lt;Cookie&gt; getCookie(HttpServletRequest request, String name) {
        Cookie[] cookies = request.getCookies();

        if (cookies != null &amp;&amp; cookies.length &gt; 0) {
            for (Cookie cookie : cookies) {
                if (name.equals(cookie.getName())) {
                    return Optional.of(cookie);
                }
            }
        }
        return Optional.empty();
    }

    public static void addCookie(HttpServletResponse response, String name, String value, int maxAge) {
        Cookie cookie = new Cookie(name, value);
        cookie.setPath(&quot;/&quot;);
        cookie.setHttpOnly(true);
        cookie.setMaxAge(maxAge);

        response.addCookie(cookie);
    }

    public static void deleteCookie(HttpServletRequest request, HttpServletResponse response, String name) {
        Cookie[] cookies = request.getCookies();
        if (cookies != null &amp;&amp; cookies.length &gt; 0) {
            for (Cookie cookie : cookies) {
                if (name.equals(cookie.getName())) {
                    cookie.setValue(&quot;&quot;);
                    cookie.setPath(&quot;/&quot;);
                    cookie.setMaxAge(0);
                    response.addCookie(cookie);
                }
            }
        }
    }

    public static String serialize(Object obj) {
        return Base64.getUrlEncoder()
                .encodeToString(SerializationUtils.serialize(obj));
    }

    public static &lt;T&gt; T deserialize(Cookie cookie, Class&lt;T&gt; cls) {
        return cls.cast(
                SerializationUtils.deserialize(
                        Base64.getUrlDecoder().decode(cookie.getValue())
                )
        );
    }
}</code></pre>
<h2 id="2-headerutil">2. HeaderUtil</h2>
<p>우리는 AccessToken을 HttpServletRequest에 헤더에 넣어서 받기 때문에, 이 AccessToken을 받기 위해 헤더를 조회해서 리턴하는 과정이 필요하다.</p>
<pre><code class="language-java">public class HeaderUtil {
    private final static String HEADER_AUTHORIZATION = &quot;Authorization&quot;;
    private final static String TOKEN_PREFIX = &quot;Bearer &quot;;

    public static String getAccessToken(HttpServletRequest request) {
        String headerValue = request.getHeader(HEADER_AUTHORIZATION);

        if (headerValue == null) {
            return null;
        }

        if (headerValue.startsWith(TOKEN_PREFIX)) {
            return headerValue.substring(TOKEN_PREFIX.length());
        }

        return null;
    }
}</code></pre>
<h1 id="repository">Repository</h1>
<p>우리는 SiteUser와 RefreshToken에 대한 테이블을 갖고 있기 때문에 아래와 같이 정의하였다.</p>
<pre><code class="language-java">@Repository
public interface SiteUserRepository extends JpaRepository&lt;SiteUser, Long&gt; {
    SiteUser findSiteUserByUserId(String userId);
}</code></pre>
<pre><code class="language-java">@Repository
public interface UserRefreshTokenRepository extends JpaRepository&lt;UserRefreshToken, Long&gt; {
    UserRefreshToken findUserRefreshTokenByUserId(String userId);
    UserRefreshToken findUserRefreshTokenByUserIdAndRefreshToken(String userId,String refreshToken);

}</code></pre>
<p>추가로 우리가 인가 후 결과를 처리하기 위해 아래와 같이 코딩하였다. 우리는 쿠키를 기반으로 만약, 다음 접근을 위해 response에 인증요청에 관한 쿠키를 넣어 반환한다.</p>
<pre><code class="language-java">public class OAuth2AuthorizationRequestBasedOnCookieRepository implements AuthorizationRequestRepository {
    public final static String OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME = &quot;oauth2_auth_request&quot;;
    public final static String REDIRECT_URI_PARAM_COOKIE_NAME = &quot;redirect_uri&quot;;
    public final static String REFRESH_TOKEN = &quot;refresh_token&quot;;
    private final static int cookieExpireSeconds = 180;

    @Override
    public OAuth2AuthorizationRequest loadAuthorizationRequest(HttpServletRequest request) {
        return CookieUtil.getCookie(request, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME)
                .map(cookie -&gt; CookieUtil.deserialize(cookie, OAuth2AuthorizationRequest.class))
                .orElse(null);
    }

    @Override
    public void saveAuthorizationRequest(OAuth2AuthorizationRequest authorizationRequest, HttpServletRequest request, HttpServletResponse response) {
        if (authorizationRequest == null) {
            CookieUtil.deleteCookie(request, response, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME);
            CookieUtil.deleteCookie(request, response, REDIRECT_URI_PARAM_COOKIE_NAME);
            CookieUtil.deleteCookie(request, response, REFRESH_TOKEN);
            return;
        }

        CookieUtil.addCookie(response, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME, CookieUtil.serialize(authorizationRequest), cookieExpireSeconds);
        String redirectUriAfterLogin = request.getParameter(REDIRECT_URI_PARAM_COOKIE_NAME);
        if (StringUtils.isNotBlank(redirectUriAfterLogin)) {
            CookieUtil.addCookie(response, REDIRECT_URI_PARAM_COOKIE_NAME, redirectUriAfterLogin, cookieExpireSeconds);
        }
    }

//    @Override
//    public OAuth2AuthorizationRequest removeAuthorizationRequest(HttpServletRequest request,Jttp) {
//        return this.loadAuthorizationRequest(request);
//    }

    @Override
    public OAuth2AuthorizationRequest removeAuthorizationRequest(HttpServletRequest request, HttpServletResponse response) {
        return this.loadAuthorizationRequest(request);
    }

    public void removeAuthorizationRequestCookies(HttpServletRequest request, HttpServletResponse response) {
        CookieUtil.deleteCookie(request, response, OAUTH2_AUTHORIZATION_REQUEST_COOKIE_NAME);
        CookieUtil.deleteCookie(request, response, REDIRECT_URI_PARAM_COOKIE_NAME);
        CookieUtil.deleteCookie(request, response, REFRESH_TOKEN);
    }
}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>