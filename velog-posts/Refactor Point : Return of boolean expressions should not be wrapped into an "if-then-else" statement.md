<h1 id="문제-코드">문제 코드</h1>
<pre><code class="language-java">    private boolean isAuthorizedRedirectUri(String uri) {
        URI clientRedirectUri = URI.create(uri);

        return appProperty.getOauth2().getAuthorizedRedirectUris()
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
    }</code></pre>
<h1 id="원인">원인</h1>
<h2 id="return-of-boolean-expressions-should-not-be-wrapped-into-an-if-then-else-statement">Return of boolean expressions should not be wrapped into an &quot;if-then-else&quot; statement</h2>
<blockquote>
</blockquote>
<p>Return of boolean literal statements wrapped into if-then-else ones should be simplified.
Similarly, method invocations wrapped into if-then-else differing only from boolean literals should be simplified into a single invocation.</p>
<p>즉, 조건문에 따라 boolean을 리턴할 때 하나로 리턴해야된다. 그럼으로 아래의 코드와 같이 수정하여 해결하였다.</p>
<h1 id="해결">해결</h1>
<pre><code>    private boolean isAuthorizedRedirectUri(String uri) {
        URI clientRedirectUri = URI.create(uri);

        return appProperty.getOauth2().getAuthorizedRedirectUris()
                .stream()
                .anyMatch(authorizedRedirectUri -&gt; {
                    // Only validate host and port. Let the clients use different paths if they want to
                    URI authorizedURI = URI.create(authorizedRedirectUri);
                    return authorizedURI.getHost().equalsIgnoreCase(clientRedirectUri.getHost())
                            &amp;&amp; authorizedURI.getPort() == clientRedirectUri.getPort();
                });
    }</code></pre>