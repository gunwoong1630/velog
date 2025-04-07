<h1 id="문제-코드">문제 코드</h1>
<pre><code class="language-java">@Getter
@Setter
@RequiredArgsConstructor
public class UserPrincipal implements OAuth2User, UserDetails, OidcUser {
    private final String userId;
    private final String password;
    private final String username;
    private final ProviderType providerType;
    private final RoleType roleType;
    private final Collection&lt;GrantedAuthority&gt; authorities;
    // 리팩토링 필요 부분
    private Map&lt;String, Object&gt; attributes;

    // OAuth2User
    @Override
    public Map&lt;String, Object&gt; getAttribute(String name) {
        return attributes;
    }

    @Override
    public String getName() {
        return (String) attributes.get(&quot;name&quot;);
    }

    // UserDetails
    @Override
    public Collection&lt;? extends GrantedAuthority&gt; getAuthorities() {
        return authorities;
    }

    @Override
    public String getUsername() {
        return username;
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
                user.getUsername(),
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
<p>해당 코드에서 UserDetails인터페이스를 구현하는데 UserDetails는 Serializable를 상속받기 때문에 해당 클래스의 필드는 모두 Serializable, 직렬화할수 있어야 한다.</p>
<p>하지만, Object객체는 Serializable를 상속받지 않기 때문에 아래와 같은 문제가 발생한다.</p>
<h1 id="원인">원인</h1>
<h2 id="fields-in-a-serializable-class-should-either-be-transient-or-serializable">Fields in a &quot;Serializable&quot; class should either be transient or serializable</h2>
<blockquote>
</blockquote>
<p>By contract, fields in a Serializable class must themselves be either Serializable or transient. Even if the class is never explicitly serialized or deserialized, it is not safe to assume that this cannot happen. For instance, under load, most J2EE application frameworks flush objects to disk.
An object that implements Serializable but contains non-transient, non-serializable data members (and thus violates the contract) could cause application crashes and open the door to attackers. In general, a Serializable class is expected to fulfil its contract and not exhibit unexpected behaviour when an instance is serialized.
This rule raises an issue on:
non-Serializable fields,
collection fields when they are not private (because they could be assigned non-Serializable values externally),
when a field is assigned a non-Serializable type within the class.</p>
<p>정리하면, Serializable 객체안의 필드들은 꼭 transient 혹은 serialzable를 상속받는 타입이어야한다.</p>
<p>하지만 해당 코드에서 attributes는 직렬화, 역직렬화 후에도 사용되어야 하기 때문에 transient를 사용하기엔 힘들다.</p>
<h1 id="해결">해결</h1>
<p>그럼으로, 아래와 같이 적절한 typeCast를 통해 수정하여 리팩토링을 진행하였다.</p>
<pre><code class="language-java">@Getter
@Setter
@RequiredArgsConstructor
public class UserPrincipal implements OAuth2User, UserDetails, OidcUser {
    private final String userId;
    private final String password;
    private final String username;
    private final ProviderType providerType;
    private final RoleType roleType;
    private final Collection&lt;GrantedAuthority&gt; authorities;

    private Map&lt;String, String&gt; attributes;

    // OAuth2User
    @Override
    public Map&lt;String, Object&gt; getAttributes() {
        return new HashMap&lt;&gt;(attributes);
    }

    @Override
    public String getName() {
        return attributes.get(&quot;name&quot;);
    }

    // UserDetails
    @Override
    public Collection&lt;? extends GrantedAuthority&gt; getAuthorities() {
        return authorities;
    }

    @Override
    public String getUsername() {
        return username;
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
        return Collections.emptyMap();
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
                user.getUsername(),
                user.getProviderType(),
                RoleType.USER,
                Collections.singletonList(new SimpleGrantedAuthority(RoleType.USER.getCode()))
        );
    }

    public static UserPrincipal create(SiteUser user, Map&lt;String, Object&gt; attributes) {
        UserPrincipal userPrincipal = create(user);
        userPrincipal.setAttributes(attributes.entrySet().stream()
                .collect(Collectors.toMap(
                        Map.Entry::getKey,
                        entry -&gt; entry.getValue().toString()
                )));

        return userPrincipal;
    }

}</code></pre>