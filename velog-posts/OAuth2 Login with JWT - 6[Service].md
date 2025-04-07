<h1 id="service">Service</h1>
<h2 id="1-customuserdetailservice">1. CustomUserDetailService</h2>
<p>기본적으로 로그인할 때, UserDetailsService 클래스의 loadUserByUsername을 통해 유저를 조회한다.</p>
<p>여기서 우리는 해당 클래스를 커스텀하게 생성하여 오버라이딩하면 우리가 원하는 기능들을 구현할 수 있을 것이다.</p>
<p>해당 프로젝트에서는 특별한 기능이 필요없음으로 담백하게 조회 후 조회한 결과를 리턴하는 코드를 구현하였다.</p>
<p>여기서 단순히 UserPrincipal객체를 생성하는 것이 아니라, 어댑터 패턴에 맞게 생성하기 위해 create함수를 사용한 것을 볼 수 있다.</p>
<p>왜냐하면, 다른 클래스에서 OAuth2 로그인에서 사용하는 loadUser함수에서도 해당 UserPrincipal을 사용함으로 create함수를 둬서 유지 보수 부분에서 용이하게 코딩하였다.</p>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
public class CustomUserDetailService implements UserDetailsService {
    private final SiteUserRepository userRepository;

    @Override
    public UserPrincipal loadUserByUsername(String username) throws UsernameNotFoundException {
        SiteUser user = userRepository.findSiteUserByUserId(username);
        if (user == null) {
            throw new UsernameNotFoundException(&quot;Can not find username&quot;);
        }
        return UserPrincipal.create(user);
    }
}</code></pre>
<h2 id="2-customoauth2userservice">2. CustomOAuth2UserService</h2>
<p>위의 CustomUserDetailService와 비슷하게 해당 클래스는 OAuth2 로그인 환경에 초점이 맞춰져 있다.</p>
<p>우리가 인가하는 과정에서 단순히 회원이 있는지 없는지 뿐만 아니라 없을 경우 회원을 생성하여 DB에 저장해야한다.</p>
<p>OAuth2 계정은 임시로 아이디를 encoding한 값을 비밀번호로 설정하였다.</p>
<p>해당 내용을 구현하면 아래와 같다.</p>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
@Slf4j
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    private final SiteUserRepository siteUserRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        log.info(&quot;gwj : CustomOAuth2UserService.loadUser&quot;);
        OAuth2User user = super.loadUser(userRequest);
        try {
            return this.process(userRequest, user);
        } catch (AuthenticationException ex) {
            throw ex;
        } catch (Exception ex) {
            ex.printStackTrace();
            throw new InternalAuthenticationServiceException(ex.getMessage(), ex.getCause());
        }
    }


    OAuth2User process(OAuth2UserRequest userRequest, OAuth2User user) {
        ProviderType providerType = ProviderType.valueOf(userRequest.getClientRegistration().getRegistrationId().toUpperCase());

        OAuth2UserInfo userInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(providerType, user.getAttributes());
        SiteUser savedUser = siteUserRepository.findSiteUserByUserId(userInfo.getId());

        if (savedUser != null) {
            if (providerType != savedUser.getProviderType()) {
                throw new OAuthProviderMissMatchException(
                        &quot;Looks like you're signed up with &quot; + providerType +
                                &quot; account. Please use your &quot; + savedUser.getProviderType() + &quot; account to login.&quot;
                );
            }
            updateUser(savedUser, userInfo);
        } else {
            savedUser = createUser(userInfo, providerType);
        }

        return UserPrincipal.create(savedUser, user.getAttributes());
    }

    private SiteUser createUser(OAuth2UserInfo userInfo, ProviderType providerType) {
        LocalDateTime now = LocalDateTime.now();
        SiteUser user = new SiteUser(
                userInfo.getId(),
                userInfo.getName(),
                false,
                providerType,
                RoleType.USER,
                now,
                now
        );

        user.setPassword(passwordEncoder.encode(userInfo.getId()));
        return siteUserRepository.saveAndFlush(user);
    }

    private SiteUser updateUser(SiteUser user, OAuth2UserInfo userInfo) {
        if (userInfo.getName() != null &amp;&amp; !user.getUsername().equals(userInfo.getName())) {
            user.setUsername(userInfo.getName());
        }
        return user;
    }
}</code></pre>
<h2 id="3-apiservice">3. ApiService</h2>
<p>우리가 로그인을 성공적으로 진행했다는 것을 보여주기 위해 해당 코드를 추가하였다.</p>
<p>간략히, WebClient과 전달받은 AccessToken(외부 플랫폼에서 제공한)을 사용해서 외부 플랫폼에 회원 정보를 성공적으로 조회하는지 확인할 것이다.</p>
<p>이를 위해 아래와 같이 서비스 코드를 작성하였다.</p>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
@Slf4j
public class ApiService {
    private final SiteUserRepository userRepository;
    private final WebClient webClient;
    private final AuthTokenProvider tokenProvider;
    private final static String LOGINPATH = &quot;http://localhost:8080/api/v1/auth/login&quot;;


    public SiteUser getUser(String userId) {
        return userRepository.findSiteUserByUserId(userId);
    }

    public ApiResponse oAuth2Login(HttpServletRequest request, HttpServletResponse response, String token) {
        ApiResponse&lt;String&gt; loginResultResponse = getApiResponse(request, response, token).block();

        return loginResultResponse;
    }

    private Mono&lt;ApiResponse&gt; getApiResponse(HttpServletRequest request, HttpServletResponse response,String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        AuthReqModel authReqModel = new AuthReqModel(authToken.getTokenClaims().getSubject(), authToken.getTokenClaims().getSubject());

        return webClient.post()
                .uri(LOGINPATH)
                .header(&quot;Authorization&quot;, &quot;Bearer &quot; + token)
                .body(BodyInserters.fromValue(authReqModel))
                .retrieve()
                .bodyToMono(ApiResponse.class);
    }

    public SiteUser getSiteUserByToken(String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        return userRepository.findSiteUserByUserId(authToken.getTokenClaims().getSubject());
    }

}</code></pre>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>