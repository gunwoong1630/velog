<h1 id="문제점">문제점</h1>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
@Slf4j
public class OAuth2Service {
    private final AuthTokenProvider tokenProvider;
    private final AppProperty appProperty;
    private final AuthenticationManager authenticationManager;
    private final UserRefreshTokenRepository userRefreshTokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final SiteUserRepository userRepository;

    public SiteUser oAuth2Login(HttpServletRequest request, HttpServletResponse response, String token) {
        AuthReqModel authReqModel = getAuthReqModelByToken(token);
        processLogin(request, response, authReqModel);
        return getSiteUserByToken(token);
    }

    private AuthReqModel getAuthReqModelByToken(String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        String id = authToken.getTokenClaims().getSubject();
        return new AuthReqModel(id, id);
    }

    private SiteUser getSiteUserByToken(String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        String id = authToken.getTokenClaims().getSubject();
        Optional&lt;SiteUser&gt; user = userRepository.findSiteUserByUserId(id);
        if (user.isEmpty()) {
            return null;
        }
        return user.get();
    }

    private void processLogin(HttpServletRequest request, HttpServletResponse response, AuthReqModel authReqModel) {
        // 참고 부분
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        authReqModel.getId(),
                        authReqModel.getPassword()
                )
        );
        String userId = authReqModel.getId();
        SecurityContextHolder.getContext().setAuthentication(authentication);

        Date now = new Date();

        long refreshTokenExpiry = appProperty.getAuth().getRefreshTokenExpiry();
        AuthToken refreshToken = tokenProvider.createAuthToken(
                appProperty.getAuth().getTokenSecret(),
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
    }
}</code></pre>
<pre><code class="language-java">@Controller
@RequestMapping(&quot;/user&quot;)
@Slf4j
public class SiteUserController {
    private final UserManagementService userManagementService;

    public SiteUserController(UserManagementService userManagementService) {
        this.userManagementService = userManagementService;
    }

    @GetMapping(&quot;/login&quot;)
    public String login(@SessionAttribute(name = &quot;errorMsg&quot;, required = false) String errorMsg, Model model) {
        if (errorMsg != null) {
            model.addAttribute(&quot;errorMsg&quot;, errorMsg);
        }
        model.addAttribute(&quot;userLoginForm&quot;, new UserLoginForm());
        return &quot;loginForm&quot;;
    }

    @GetMapping(&quot;/signup&quot;)
    public String signup(Model model) {
        model.addAttribute(&quot;userCreateForm&quot;, new UserCreateForm());
        return &quot;signupForm&quot;;
    }

    @PostMapping(&quot;/signup&quot;)
    public String signup(@Valid UserCreateForm userCreateForm, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return &quot;signupForm&quot;;
        }
        if (!userCreateForm.getPassword1().equals(userCreateForm.getPassword2())) {
            bindingResult.rejectValue(&quot;password2&quot;, &quot;passwordInCorrect&quot;, &quot;2개의 비밀번호가 맞지 않습니다&quot;);
        }
        try {
            userManagementService.createUser(userCreateForm.getUserID(), userCreateForm.getName(), userCreateForm.getPassword1(), ProviderType.LOCAL, RoleType.USER);
            // 해당 부분에 로그인시 로그인 정보를 유지할 수 있도록 코드 추가 요망
        } catch (Exception e) {
            bindingResult.rejectValue(&quot;signupFailed&quot;, e.getMessage());
            return &quot;signupForm&quot;;
        }
        return &quot;redirect:/&quot;;
    }

    @GetMapping(&quot;/dashboard&quot;)
    public String userDashBoard(Model model) {
        model.addAttribute(&quot;users&quot;, userManagementService.findAll());
        return &quot;userDashboardForm&quot;;
    }

}</code></pre>
<p>위의 코드에서 보듯이, OAuth2Service클래스에서 oAuth2Login함수를 실행하게 되면 수동으로 로그인되어, 해당 내용을 SecurityContextHolder안의 SecurityContext에 넣음으로서 로그인시 해당 인증 정보가 유지된다.</p>
<p>그래서 로컬환경, 즉 폼 로그인 환경에서 회원가입 후 자동 로그인되도록 설정하기 위해, SiteUserController에 코드를 추가할 예정이다.</p>
<p>하지만, OAuth2Service에서 참고한 부분을 그대로 사용했을 때 제대로 인증 정보가 유지되지 않았다. 아래는 문제의 코드이다.</p>
<pre><code class="language-java">@Controller
@RequestMapping(&quot;/user&quot;)
@Slf4j
public class SiteUserController {
    private final UserManagementService userManagementService;

    public SiteUserController(UserManagementService userManagementService) {
        this.userManagementService = userManagementService;
    }

    @GetMapping(&quot;/login&quot;)
    public String login(@SessionAttribute(name = &quot;errorMsg&quot;, required = false) String errorMsg, Model model) {
        if (errorMsg != null) {
            model.addAttribute(&quot;errorMsg&quot;, errorMsg);
        }
        model.addAttribute(&quot;userLoginForm&quot;, new UserLoginForm());
        return &quot;loginForm&quot;;
    }

    @GetMapping(&quot;/signup&quot;)
    public String signup(Model model) {
        model.addAttribute(&quot;userCreateForm&quot;, new UserCreateForm());
        return &quot;signupForm&quot;;
    }

    @PostMapping(&quot;/signup&quot;)
    public String signup(@Valid UserCreateForm userCreateForm, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return &quot;signupForm&quot;;
        }
        if (!userCreateForm.getPassword1().equals(userCreateForm.getPassword2())) {
            bindingResult.rejectValue(&quot;password2&quot;, &quot;passwordInCorrect&quot;, &quot;2개의 비밀번호가 맞지 않습니다&quot;);
        }
        try {
            userManagementService.createUser(userCreateForm.getUserID(), userCreateForm.getName(), userCreateForm.getPassword1(), ProviderType.LOCAL, RoleType.USER);
            // 추가 부분
            Authentication authentication = authenticationManager.authenticate(
                        new UsernamePasswordAuthenticationToken(
                                userCreateForm.getUserID(),
                                userCreateForm.getPassword1()
                        )
                );
            SecurityContextHolder.getContext().setAuthentication(authentication);
        } catch (Exception e) {
            bindingResult.rejectValue(&quot;signupFailed&quot;, e.getMessage());
            return &quot;signupForm&quot;;
        }
        return &quot;redirect:/&quot;;
    }

    @GetMapping(&quot;/dashboard&quot;)
    public String userDashBoard(Model model) {
        model.addAttribute(&quot;users&quot;, userManagementService.findAll());
        return &quot;userDashboardForm&quot;;
    }

}</code></pre>
<p>어떤 부분이 문제일까?</p>
<h1 id="문제-분석">문제 분석</h1>
<p>결론적으로 Session이나 쿠키를 설정하지 않아 발생한 문제였다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/ab4b9cd3-645d-40e6-9ad4-d4e30c6f3338/image.png" /></p>
<p>위의 사진은 [관련 포스팅](<a href="https://uchupura.tistory.com/28)%EC%9D%98">https://uchupura.tistory.com/28)의</a> 내용이다.</p>
<p>해당 내용을 자세히 보면, 인증 유무에 따라서 다르게 흘러가는데 세션부분을 자세히 봐야한다.</p>
<p>위의 그림에서 request가 들어온다면 SecurityContextPersistenceFilter, HttpSecurityContextRepository가 차례로 진행된다. 주황색 다이아몬드 갈림길에서 실제로 과연 어떻게 나뉠수 있을까?</p>
<p>관련 해답을 찾기 위해 스택플로우의 <a href="https://stackoverflow.com/questions/3813028/auto-login-after-successful-registration">어떤 포스팅</a>에는 크게 2가지 방법을 소개하고 있다.</p>
<blockquote>
</blockquote>
<ol>
<li>인증 정보를 세션에 담기</li>
<li>HttpServletRequest의 login함수 사용 (Servlet 3.0이상)</li>
</ol>
<p>여기서 우리가 문제점이 된 부분이 바로 1번과 관련된 내용이었다.</p>
<p>OAuth2 로그인이 진행될 경우, 자동으로 세션에 SecurityContext가 등록되어 딱히 자동 로그인을 위해 기능을 추가할 필요가 없었다(다른 특별한 기능 추가를 하지 않는 이상).</p>
<p>반대로 로컬환경에서 회원가입 후 자동 로그인을 위해 수동 인증을 하려했지만, 인증을 완료한 결과를 SecurityContext에 저장해두었지만 해당 SecurityContext가 Session에 없기 때문에 해당 코드는 의미가 없어져 버린다.</p>
<p>아래의 결과를 보자.</p>
<ol>
<li><p>OAuth2로그인의 HttpServletRequest
<img alt="" src="https://velog.velcdn.com/images/gwj0421/post/e1d0a948-daf4-4cee-8c26-165715d1475d/image.png" /></p>
</li>
<li><p>회원가입 진행시 사용한 HttpServletRequest
<img alt="" src="https://velog.velcdn.com/images/gwj0421/post/705bc02c-c29b-4221-bec8-8fd73ef97bec/image.png" /></p>
</li>
</ol>
<p>OAuth2의 HttpServletRequest와 회원가입 진행시 사용한 HttpServletRequest의 차이점은 SecurityContext의 유무차이였다.</p>
<p>그럼으로 회원가입 진행시 HttpSErvletRequest의 세션에 SecurityContext가 없음으로 내가 아무리 Context에 authentication를 넣어도 밑 빠진 독에 물 붓기였다.</p>
<h1 id="해결-부분">해결 부분</h1>
<p>우리는 위의 스택오버플로우의 포스팅대로 서블릿 버전에 따른 해결 코드를 아래와 같이 수정하였다. 추가로 OAuth 로그인부분에서 불필요한 코드를 제거하였다.</p>
<pre><code class="language-java">@Service
@RequiredArgsConstructor
@Slf4j
public class OAuth2Service {
    private final AuthTokenProvider tokenProvider;
    private final AppProperty appProperty;
    private final AuthenticationManager authenticationManager;
    private final UserRefreshTokenRepository userRefreshTokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final SiteUserRepository userRepository;

    public SiteUser oAuth2Login(HttpServletRequest request, HttpServletResponse response, String token) {
        AuthReqModel authReqModel = getAuthReqModelByToken(token);
        processLogin(request, response, authReqModel);
        return getSiteUserByToken(token);
    }

    private AuthReqModel getAuthReqModelByToken(String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        String id = authToken.getTokenClaims().getSubject();
        return new AuthReqModel(id, id);
    }

    private SiteUser getSiteUserByToken(String token) {
        AuthToken authToken = tokenProvider.convertAuthToken(token);
        String id = authToken.getTokenClaims().getSubject();
        Optional&lt;SiteUser&gt; user = userRepository.findSiteUserByUserId(id);
        if (user.isEmpty()) {
            return null;
        }
        return user.get();
    }

    private void processLogin(HttpServletRequest request, HttpServletResponse response, AuthReqModel authReqModel) {
        // 불필요한 부분 제거
        String userId = authReqModel.getId();

        Date now = new Date();

        long refreshTokenExpiry = appProperty.getAuth().getRefreshTokenExpiry();
        AuthToken refreshToken = tokenProvider.createAuthToken(
                appProperty.getAuth().getTokenSecret(),
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
    }
}</code></pre>
<pre><code class="language-java">@Controller
@RequestMapping(&quot;/user&quot;)
@RequiredArgsConstructor
@Slf4j
public class SiteUserController {
    private final UserManagementService userManagementService;
    private final AuthenticationManager authenticationManager;

    private static final String SIGNUP_FORM_VIEW = &quot;signupForm&quot;;

    @GetMapping(&quot;/login&quot;)
    public String login(@SessionAttribute(name = &quot;errorMsg&quot;, required = false) String errorMsg, Model model) {
        if (errorMsg != null) {
            model.addAttribute(&quot;errorMsg&quot;, errorMsg);
        }
        model.addAttribute(&quot;userLoginForm&quot;, new UserLoginForm());
        return &quot;loginForm&quot;;
    }

    @GetMapping(&quot;/signup&quot;)
    public String signup(Model model) {
        model.addAttribute(&quot;userCreateForm&quot;, new UserCreateForm());
        return SIGNUP_FORM_VIEW;
    }

    @PostMapping(&quot;/signup&quot;)
    public String signup(HttpServletRequest request, HttpServletResponse response,
                         @Valid UserCreateForm userCreateForm, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return SIGNUP_FORM_VIEW;
        }
        if (!userCreateForm.getPassword1().equals(userCreateForm.getPassword2())) {
            bindingResult.rejectValue(&quot;password2&quot;, &quot;passwordInCorrect&quot;, &quot;2개의 비밀번호가 맞지 않습니다&quot;);
        }
        try {
            userManagementService.createUser(userCreateForm.getUserID(), userCreateForm.getName(), userCreateForm.getPassword1(), ProviderType.LOCAL, RoleType.USER);
            // 추가한 부분
            if (request.getServletContext().getMajorVersion() &gt;= 3) {
                request.login(userCreateForm.getUserID(), userCreateForm.getPassword1());
            } else {
                request.getSession().setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, SecurityContextHolder.getContext());
                Authentication authentication = authenticationManager.authenticate(
                        new UsernamePasswordAuthenticationToken(
                                userCreateForm.getUserID(),
                                userCreateForm.getPassword1()
                        )
                );
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            bindingResult.rejectValue(&quot;signupFailed&quot;, e.getMessage());
            return SIGNUP_FORM_VIEW;
        }


        return &quot;redirect:/&quot;;
    }

    @GetMapping(&quot;/dashboard&quot;)
    public String userDashBoard(Model model) {
        model.addAttribute(&quot;users&quot;, userManagementService.findAll());
        return &quot;userDashboardForm&quot;;
    }

}</code></pre>