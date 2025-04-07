<p>이번 프로젝트의 핵심 기능은 OAuth2 로그인이다. 해당 파트에 대한 개념적인 설명은 초반(0번 포스팅부터)에 했음으로 생략하고 PostMan을 사용해서 기능 구현을 확인할 것이다.</p>
<p>임시로, 구글과 로그인만 구현했기 때문에 구글 OAuth 로그인을 해 볼 것이다.</p>
<h1 id="구글-로그인">구글 로그인</h1>
<p>우리가 SecurityConfig에서 설정했던 것처럼, &quot;<a href="http://localhost:8080/oauth2/authorization/%7BClient-id%7D&quot;%EB%A5%BC">http://localhost:8080/oauth2/authorization/{Client-id}&quot;를</a> 입력하게 되면 해당 플랫폼에 대한 provider의 로그인 화면으로 이동하게 된다.</p>
<p>우리는 구글 로그인을 진행할 것임으로 &quot;<a href="http://localhost:8080/oauth2/authorization/google?redirect_uri=http://localhost:8080/oauth/redirect&quot;%EB%A5%BC">http://localhost:8080/oauth2/authorization/google?redirect_uri=http://localhost:8080/oauth/redirect&quot;를</a> 입력하게 될 경우 아래와 같은 화면으로 이동될 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/25d58db8-1736-45bb-920c-d5dcefe9fd9b/image.png" /></p>
<p>이렇게 로그인할 경우 파라미터로 넘긴 redirect_uri로 이동하게 되면서 전에 컨트롤러에서 정의한 경로로 넘어가 아래와 같이 출력하게 될 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/e5ea47af-8730-43ec-97d9-471d23621aa1/image.png" /></p>
<p>이렇게 확인을 했고 직접 데이터베이스를 확인해보면 회원 정보와 RefreshToken이 정상적으로 저장됐음을 볼 수 있다.</p>
<p>그 후, postman을 사용해서 정상적인 로그인 기능을 확인해볼 것이다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/147c0d5f-86d0-498c-a6ad-30560b150542/image.png" /></p>
<p>위의 사진과 같이 post를 했을 때 결과값을 받을 수 있다.</p>
<h1 id="고려-사항">고려 사항</h1>
<p>이번 프로젝트를 진행함에 있어서 두가지 상황을 고려해서 코드를 작성하였다.</p>
<blockquote>
</blockquote>
<ol>
<li>백엔드만 사용할 경우</li>
<li>백엔드 + 프론트엔드 사용할 경우</li>
</ol>
<p>1번과 같은 경우에는 백엔드에서 다 구현해야 함으로 UserController의 checkOAuth2Login에 추가로 authenticate하고 Refresh토큰과 AccessToken을 생성하고 다루는 부분과 쿠키를 다루는 부분을 추가해야할 것이다(AuthController의 login함수 참고).</p>
<p>2번과 같은 경우에는 적절히 redirectUri를 설정해서 AuthController의 login함수를 메인으로 사용하면 될 것이다.</p>
<p>추가로 Refresh토큰을 다룰 떄는 JDBC보다는 in memory형식으로 사용한다. 그럼으로 Redis를 사용한다면 엑세스 시간을 단축시킬수 있을 것이다.</p>
<h1 id="전체-코드-github">전체 코드 Github</h1>
<p><a href="https://github.com/gwj0421/OAuth2Login/tree/main">https://github.com/gwj0421/OAuth2Login/tree/main</a></p>