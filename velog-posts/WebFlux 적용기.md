<p>이번 프로젝트에서 MSA 기반으로 진행하여 REST통신을 어떻게 할 건지 문제였다. 이번 프로젝트에선느 비동기 기능을 제공하는 WebFlux를 활용하였다. 그래서 이번 프로젝트를 진행하면서 배웠던 WebFlux를 정리하고자 한다.</p>
<p>우선 WebFlux가 무엇이고 어떨때 사용할까?</p>
<h2 id="webflux-개념">WebFlux 개념</h2>
<p>WebFlux는 Spring Framework의 일환으로, 비동기 및 논블로킹(Non-blocking) 방식의 웹 애플리케이션을 개발하기 위한 프레임워크이다.</p>
<p>우선 WebFlux는 아래와 같은 상황에서 보통 사용한다.</p>
<ol>
<li>비동기</li>
<li>효율적으로 동작하는 고성능 웹 어플리케이션 개발</li>
<li>MSA 적합</li>
</ol>
<p>MSA에서 각 API끼리 통신이 많이 이뤄지기 때문에 WebFlux를 사용하는 것이 생산성에 좋을 것이다.</p>
<p>WebFlux는 서버 측에서 동시성을 처리하기 위해 리액티브 프로그래밍 모델을 채택하고있으며, 이는 대규모 트래픽을 효율적으로 처리하는 데 유리하다. 여기서 리액티브 스트림와 프로젝트 리액터를 이해하는 것이 좋다.</p>
<h3 id="리액티브-스트림">리액티브 스트림</h3>
<p>비동기처리를 위한 표준이라고 생각하면 편하다. 리액티브 스트림은 publisher, Subscriber, Subscription, Processor라는 4가지 인터페이스로 구성된다.</p>
<ol>
<li>Publisher : 데이터 발행 역할, subscribe 메서드를 통해 Subscriber를 등록</li>
<li>Subscriber : 데이터 소비 역할, Publisher로부터 데이터 수신</li>
<li>Subscription : Publisher와 Subscriber 간의 구독 관계</li>
<li>Processor : Publisher와 Subscriber의 역할을 모두 수행하는 중간 처리자</li>
</ol>
<h3 id="프로젝트-리액터">프로젝트 리액터</h3>
<p>WebFlux에서는 리액티브 스트림을 구현하기 위해 프로젝트 리액터를 사용한다. 쉽게 말해, Publiser의 타입으로 생각하면 편하다.</p>
<ol>
<li>Mono : 0또는 1개의 아이템을 발행하는 Publisher</li>
<li>Flux : 0또는 N개의 아이템을 발행하는 Publisher</li>
</ol>
<p>이렇게 발행하는 아이템 개수에 따라 다르게 한다.</p>
<h3 id="비동기-동작-과정">비동기 동작 과정</h3>
<ol>
<li>클라이언트 요청 수신</li>
</ol>
<p>클라이언트가 서버에 요청을 보내면 이벤트 루프가 이를 수신합니다.</p>
<ol start="2">
<li>Publisher 생성</li>
</ol>
<p>리액티브 스트림 핸들러는 요청을 처리하고, 이에 대한 응답 데이터를 발행하기 위해 Publisher(예: Flux 또는 Mono)를 생성한다.</p>
<ol start="3">
<li>Subscriber 등록</li>
</ol>
<p>이벤트 루프가 Publisher의 subscribe 메서드를 호출하여 Subscriber를 등록한다. 이 과정에서 Subscription이 생성된다.</p>
<ol start="4">
<li>데이터 요청 및 전송</li>
</ol>
<p>Subscriber가 Publisher에게 데이터를 요청하면, Publisher는 요청된 데이터 아이템을 비동기적으로 발행한다. 이벤트 루프는 이 데이터를 논블로킹 방식으로 처리하여 클라이언트에게 전송합니다.</p>
<p>그럼으로 기존의 블로킹 방식처럼 어떤 작업을 스레드 풀에 있는 스레드에 배정하고 스레드가 작업을 끝낼때마다 새로운 작업을 주는 것이 아니라, Subscriber가 작업을 Publisher에게 던지면 Publisher가 작업을 수행하고 Publisher는 계속 작업들을 수신하며 작업을 수행하기 때문에 Non-Blocking 기법으로 불린다.</p>
<p>Subscriber가 작업을 바로 수행하는 것이 아님에 헷갈리지 않도록 하자.</p>