<h1 id="문제-코드">문제 코드</h1>
<pre><code class="language-java">@Getter
public class ApiResponse&lt;T&gt; {
    private static final int SUCCESS = 200;
    private static final int NOT_FOUND = 400;
    private static final int FAILED = 500;
    private static final String SUCCESS_MESSAGE = &quot;SUCCESS&quot;;
    private static final String NOT_FOUND_MESSAGE = &quot;NOT FOUND&quot;;
    private static final String FAILED_MESSAGE = &quot;서버에서 오류 발생&quot;;
    private static final String INVALID_ACCESS_TOKEN = &quot;Invalid access token&quot;;
    private static final String INVALID_REFRESH_TOKEN = &quot;Invalid refresh token&quot;;
    private static final String NOT_EXPIRED_TOKEN_YET = &quot;Not expired token yet&quot;;

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
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(SUCCESS, SUCCESS_MESSAGE), map);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; fail() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, FAILED_MESSAGE), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidAccessToken() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, INVALID_ACCESS_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidRefreshToken() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, INVALID_REFRESH_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; notExpiredTokenYet() {
        # 문제의 부분 1
        return new ApiResponse(new ApiResponseHeader(FAILED, NOT_EXPIRED_TOKEN_YET), null);
    }
}</code></pre>
<pre><code class="language-java">@Getter
@Setter
@ConfigurationProperties(prefix =&quot;cors&quot;)
public class CorsProperty {
    # 문제의 부분 2
    private List allowedOrigins;
    private List allowedMethods;
    private List allowedHeaders;
    private Long maxAge;
}</code></pre>
<h1 id="원인">원인</h1>
<h2 id="raw-types-should-not-be-used">Raw types should not be used</h2>
<blockquote>
</blockquote>
<p>Generic types shouldn’t be used raw (without type parameters) in variable declarations or return values. Doing so bypasses generic type checking, and defers the catch of unsafe code to runtime.</p>
<p>즉, 제네릭을 통해 변수 선언이나 리턴할 때 제대로 제네릭 타입을 체크해야 한다.</p>
<h1 id="리팩토링-진행-결과">리팩토링 진행 결과</h1>
<pre><code class="language-java">@Getter
public class ApiResponse&lt;T&gt; {
    private static final int SUCCESS = 200;
    private static final int NOT_FOUND = 400;
    private static final int FAILED = 500;
    private static final String SUCCESS_MESSAGE = &quot;SUCCESS&quot;;
    private static final String NOT_FOUND_MESSAGE = &quot;NOT FOUND&quot;;
    private static final String FAILED_MESSAGE = &quot;서버에서 오류 발생&quot;;
    private static final String INVALID_ACCESS_TOKEN = &quot;Invalid access token&quot;;
    private static final String INVALID_REFRESH_TOKEN = &quot;Invalid refresh token&quot;;
    private static final String NOT_EXPIRED_TOKEN_YET = &quot;Not expired token yet&quot;;

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
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(SUCCESS, SUCCESS_MESSAGE), map);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; fail() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, FAILED_MESSAGE), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidAccessToken() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, INVALID_ACCESS_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; invalidRefreshToken() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, INVALID_REFRESH_TOKEN), null);
    }

    public static &lt;T&gt; ApiResponse&lt;T&gt; notExpiredTokenYet() {
        return new ApiResponse&lt;&gt;(new ApiResponseHeader(FAILED, NOT_EXPIRED_TOKEN_YET), null);
    }
}</code></pre>
<p>```java
@Getter
@Setter
@ConfigurationProperties(prefix =&quot;cors&quot;)
public class CorsProperty {
    private List allowedOrigins;
    private List allowedMethods;
    private List allowedHeaders;
    private Long maxAge;
}</p>