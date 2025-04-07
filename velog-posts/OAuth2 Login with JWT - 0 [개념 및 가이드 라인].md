<h1 id="기본-개념">기본 개념</h1>
<h2 id="인증--인가">인증 &amp; 인가</h2>
<p>기본적으로 <strong>인증(Authentication)</strong>과 <strong>인가(Authorization)</strong>에 대해 알아야 뒤에 헷갈리지 않는다.</p>
<p>인증은 <strong>사용자가 누구인지 확인하는 단계</strong>이고 인가는 <strong>인증을 통해 검증된 사용자가 리소스에 접근할 때 해당 리소스에 접근할 권리가 있는지 확인</strong>하는 절차이다.</p>
<h2 id="spring-secucirty">Spring Secucirty</h2>
<p>기본적으로 아래와 같이 Spring security가 돌아간다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/01439b99-a9b4-49ee-8149-7ee3343d7413/image.png" /></p>
<ol>
<li>Http request시 UsernamePasswordAuthenticationFilter(위의 그림에서는 AuthenticationFilter)</li>
<li>AuthenticationFilter는 Http request에서 username과 password를 추출해서 토큰 생성</li>
<li>생성된 토큰을 AuthenticationManager에게 전달</li>
<li>ProviderManager로 impl된 AuthenticationManager가 AuthenticationProvider에게 토큰 전달</li>
<li>AuthenticationProvider가 UserDetailService에 토큰 전달</li>
<li>전달받은 토큰을 통해 데이터베이스 조회 후 확인 후 UserDetails객체를 생성</li>
<li>이렇게 생성된 UserDetails객체를 AuthenticationProvider로 전달</li>
<li>AuthenticationProvider가 AuthenticationManager에게 UserDetails객체를 전달</li>
<li>이렇게 검증된 UserDetails객체를 AuthenticationFilter에 전달</li>
<li>이러한 방식으로 검증된 토큰을 SecurityContextHolder에 있는 SecurityContext에 저장</li>
</ol>
<p>즉, 한 줄로 정리하자면, <strong>로그인을 위해 로그인 관련 정보를 request에 담아 전송한다면 AuthenticationFilter가 받아 데이터베이스를 조회하여 UserDetails객체로 SecurityContext에 저장</strong>한다.</p>
<p>그래서 우리가 로그인 정보를 SecurityContext에서 조회할 수 있을 것이다.</p>
<h2 id="oauth--jwt">OAuth &amp; JWT</h2>
<p>OAuth는 Open Authorization의 약자로 사용자 인증 및 권한 부여를 위한 개방형 표준 프로토콜이다. 자세한 내용은 아래의 게시글을 참고 바란다.</p>
<p><a href="https://velog.io/@gwj0421/IT%EC%83%81%EC%8B%9DOAuth">OAUth 개념</a></p>
<p>JWT는 정보를 Json형태로 안전한 전달을 위한 토큰이다. JWT에 필요한 것은 당연히 signature가 필요하다. 해당 signature는 인터넷 상에 자주 돌아다니는 흔한 문자열은 삼가하여 정하도록 하자.</p>
<p>JWT에 대한 자세한 정보는 아래의 게시글을 참고 바란다.</p>
<p><a href="https://api.velog.io/rss/%60%60%60%5D(https:/velog.io/@gwj0421/IT%EC%83%81%EC%8B%9DJWT)">JWT 개념</a></p>
<h1 id="초기-가이드">초기 가이드</h1>
<p>이번 프로젝트에서는 OAuth2를 기반으로 로그인기능을 구현할 것이다. 또한 해당 프로젝트는 아래의 포스팅을 참조하였다.</p>
<p><a href="https://deeplify.dev/back-end/spring/oauth2-social-login#tokenvalidfailedexception">참조한 포스팅</a></p>
<p>구현에 앞서 전체 시퀸스는 아래의 사진과 같다. 사진도 마찬가지로 해당 포스팅에서 가져왔다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/789c591e-e2da-4f19-b468-c0f0aba8fece/image.png" /></p>
<p>위의 포스팅에 가면 해당 시퀸스에 대한 설명이 자세하게 되어있다.</p>
<p>다만, 이번 프로젝트를 진행하면서 헷갈렸던 부분은 바로 AccessToken을 다루는 부분이었다.</p>
<p>처음에 AccessToken을 외부 플랫폼에서 받았다면 해당 AccessToken만 사용해야되는거 아닌가? 라고 생각했다.</p>
<p>하지만, 그렇게 될 경우 매번 로그인을 할 때마다, 외부 플랫폼에 불필요한 접근과 비용이 발생한다. 이러한 문제점을 해결하기 위해 백엔드 서버가 첫 인증과 인가를 진행하고 해당 내용을 JWT로 만들어 백엔드 서버 자체적으로 보관하도록 했다. 이러한 과정을 통해 외부 플랫폼에서 첫 인증과 인가를 진행했다면 그 다음 부터는 외부 플랫폼까지 요청할 필요도 없이 백엔드 서버가 처리할 수 있도록 된다.</p>
<p>추가적으로 아무런 설정 없이 OAuth2 로그인을 구현하게 된다면 기본적으로 첫 로그인만 외부 플랫폼에서 직접 로그인을 하고 그 뒤의 로그인은 해당 단계가 생략된다. 이러한 부분을 수정해서 매번 로그인 할 때마다 생략되지 않도록 추가할 것이다.</p>
<h1 id="초기-세팅">초기 세팅</h1>
<p>당연하게도 해당 프로젝트를 진행하려면 다양한 외부 플랫폼에 프로젝트를 만들고 OAuth2 2.0 credential을 받아야 한다(그럼으로 프로젝트 진행 전에 구글이나 페이스 북에 가서 프로젝트를 생성하고 와야함).</p>
<p>해당 내용은 다른 포스팅에서 상세히 나와있음으로 생략한다.</p>
<p>다만 설정시, 인증을 마친 후 최종 사용자가 동의 페이지에서 접근 권한을 부여한다면 이 사용자가 다시 되돌아가야 할 리다이렉트 URL을 헷갈리지 않고 정의해야한다.</p>
<p>즉, 로그인을 성공하고 백엔드에게 던져줄 경로를 헷갈리지 말라는 뜻(외부 플랫폼에서 바로 사용자에게 보여주는 것이 아니기 때문에)이다.</p>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>