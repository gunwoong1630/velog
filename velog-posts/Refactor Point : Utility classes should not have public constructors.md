<h1 id="문제-코드">문제 코드</h1>
<pre><code class="language-java">public class OAuth2UserInfoFactory {
    public static OAuth2UserInfo getOAuth2UserInfo(ProviderType providerType, Map&lt;String, Object&gt; attributes) {
        switch (providerType) {
            case GOOGLE: return new GoogleOAuth2UserInfo(attributes);
            case FACEBOOK: return new FacebookOAuth2UserInfo(attributes);
            case GITHUB: return new GithubOAUth2UserInfo(attributes);

            default: throw new IllegalArgumentException(&quot;Invalid Provider Type.&quot;);
        }
    }
}</code></pre>
<h1 id="원인">원인</h1>
<h2 id="utility-classes-should-not-have-public-constructors">Utility classes should not have public constructors</h2>
<blockquote>
</blockquote>
<p>Utility classes, which are collections of static members, are not meant to be instantiated. Even abstract utility classes, which can be extended, should not have public constructors.
Java adds an implicit public constructor to every class which does not define at least one explicitly. Hence, at least one non-public constructor should be defined.</p>
<p>핵심은 유틸리티 클래스는 public한 생성자를 가지면 안된다는 것이다.</p>
<p>우리가 유틸리티 클래스를 생성할 때, 보통 해당 클래스를 객체로 생성해서 사용하고자 하지 않는다.</p>
<p>그럼으로 목적성에 맞게 해당 유틸리티 클래스를 생성하지 못하도록 private 생성자를 통해 객체 생성을 막는다.</p>
<h1 id="해결">해결</h1>
<pre><code class="language-java">public class OAuth2UserInfoFactory {
    private OAuth2UserInfoFactory() {
        throw new IllegalStateException(&quot;Making utility class error&quot;);
    }
    public static OAuth2UserInfo getOAuth2UserInfo(ProviderType providerType, Map&lt;String, Object&gt; attributes) {
        switch (providerType) {
            case GOOGLE: return new GoogleOAuth2UserInfo(attributes);
            case FACEBOOK: return new FacebookOAuth2UserInfo(attributes);
            case GITHUB: return new GithubOAUth2UserInfo(attributes);

            default: throw new IllegalArgumentException(&quot;Invalid Provider Type.&quot;);
        }
    }
}</code></pre>