<h1 id="문제-소개">문제 소개</h1>
<p>해당 문제는 3가지 종류의 그래프가 주어지고 하나의 정점을 생성하여 연결한 결과에서 임의로 생성한 정점의 번호와 3가지 종류의 개수를 카운트하는 문제이다.</p>
<h1 id="문제-분석">문제 분석</h1>
<p>일단 해당 문제를 푸는 과정에서 골치가 아픈 부분은 아래와 같았다.</p>
<ol>
<li>임의의 정점 찾기</li>
<li>그래프 종류에 따른 카운팅</li>
</ol>
<p>차례차례로 접근해보자</p>
<h2 id="1-임의의-정점-찾기">1. 임의의 정점 찾기</h2>
<p>아래의 그림을 보자</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/dac54ad4-ba32-4623-ad72-4a089990405e/image.png" /></p>
<p>해당 예제에서 임의의 정점은 4이고 다른 3가지의 종류의 그래프가 존재한다. 각각의 그래프는 아래와 같은 특징을 갖고 있다.</p>
<blockquote>
<p>어느 특정 노드 A에서 전송 엣지 a개와 수신 엣지 b개인 상황에서</p>
</blockquote>
<ol>
<li>도넛 모양 그래프의 하나일 경우 : a = b = 1</li>
<li>막대 모양 그래프의 하나일 경우: (a = 1 and b = 0) or (a = 1 and b = 0) or (a = 0 and b = 1)</li>
<li>8자 모양 그래프의 하나일 경우: (a = b = 2) or (a = b = 1)</li>
<li>임의로 생성한 노드 : a &gt;= 2 and b = 0</li>
</ol>
<p>왜냐하면 임의로 노드를 임의로 생성한 노드는 수신하지 않고 전체 그래프에게 하나씩 전송하기 때문이다. 그리고 최소 2개의 그래프가 존재하기에 임의 생성 노드는 최소 2개를 전송하게 된다.</p>
<p>그럼으로, 전체 노드를 서칭할 때 수신하지 않고 송신만 2개하는 노드를 찾으면 쉽게 찾을 수 있다.</p>
<pre><code class="language-java">Map&lt;Integer, Queue&lt;Integer&gt;&gt; relation = new HashMap&lt;&gt;();
        boolean[] isReceived = new boolean[1000001];
        for (int[] edge : edges) {
            if (relation.containsKey(edge[0])) {
                relation.get(edge[0]).add(edge[1]);
            } else {
                relation.put(edge[0], new LinkedList&lt;&gt;(List.of(edge[1])));
            }
            isReceived[edge[1]] = true;
            if (!relation.containsKey(edge[1])) {
                relation.put(edge[1], new LinkedList&lt;&gt;());
            }
        }
        int removeNode = -1;
        for (Map.Entry&lt;Integer, Queue&lt;Integer&gt;&gt; entry : relation.entrySet()) {
            if (entry.getValue().size() &gt;= 2 &amp;&amp; !isReceived[entry.getKey()]) {
                removeNode = entry.getKey();
            }
        }</code></pre>
<p>위와 같이 임의 생성한 노드를 찾았다면 그래프의 종류를 카운트하는 상황에서 헷갈리지 않도록 그냥 지워버리도록 하자</p>
<pre><code class="language-java">int totalCnt = relation.get(removeNode).size();
        relation.remove(removeNode);</code></pre>
<h2 id="2-그래프-종류에-따른-카운팅">2. 그래프 종류에 따른 카운팅</h2>
<p>이렇게 임의 생성한 노드를 제거하고 나면, 전체 노드를 서칭할 때 노드의 개수와 엣지의 개수를 토대로 어떤 종류의 그래프인지 카운팅하면 된다. 간단하게 BFS으로 노드 개수와 엣지 개수를 찾았다. 하지만 생각지도 못한 부분에 어려움이 봉착하였는데...</p>
<pre><code class="language-java">       int[] answer = new int[]{removeNode, 0, 0, 0};
        boolean[] isVisit = new boolean[1000001];
        for (Map.Entry&lt;Integer, Queue&lt;Integer&gt;&gt; entry : relation.entrySet()) {
            if (!isVisit[entry.getKey()]) {
                isVisit[entry.getKey()] = true;
                int nodeCnt = 1;
                int edgeCnt = 0;
                Queue&lt;Integer&gt; route = new LinkedList&lt;&gt;();
                route.add(entry.getKey());
                while (!route.isEmpty()) {
                    int node = route.poll();
                    while (!relation.get(node).isEmpty()) {
                        int nextNode = relation.get(node).poll();
                        edgeCnt++;
                        if (!isVisit[nextNode]) {
                            nodeCnt++;
                            isVisit[nextNode] = true;
                            route.add(nextNode);
                        }
                    }
                }
                int targetShape = getShapeIdx(nodeCnt, edgeCnt);
                // 왜 막대만 빼고 하는 지는 뒤에 설명한다.
                if (targetShape != 1) {
                    answer[targetShape]++;
                    totalCnt--;
                }
            }
        }</code></pre>
<h1 id="위기">위기</h1>
<p>도넛 모양, 막대 모양, 8자 모양은 노드의 개수마다 다른 엣지 개수를 갖고 있기 때문에 분류하는 부분에서 어려움이 없었다. 하지만 아래의 표와 같이 방향성에 따른 차이점이 존재하는 것을 깨달았다.</p>
<blockquote>
</blockquote>
<ul>
<li>도넛, 8자 = 순환 구조, 어떤 노드를 골라도 중복 방문만 잘 처리하면 OK</li>
<li>막대 = 비순환 구조, 노드를 잘못 고르면 하나의 막대에서 2개(?)의 막대가 나온다;;</li>
</ul>
<p>아래의 그림을 보자</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/84b2b582-bde0-403f-b405-05b082cd0e61/image.png" /></p>
<p>위의 예시에서 3번과 4번 노드는 막대 그래프이다. 3번 노드를 먼저 조회하게 되면 3번 노드만 존재하는 막대로 취급하고 4번 노드를 조회하게 되면 4번 노드만 따로 조회하게 된다. 그래서 만약 4번 노드 뒤에 5번 노드까지 존재하게 된다면 원래 하나의 연결된 막대가 아닌 3개로 나눠진 막대를 따로따로 취급하게 된다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/gwj0421/post/44f758d7-5087-4621-86c5-1b9c0a2f9b84/image.png" /></p>
<p>그렇다면 이를 어떻게 해결할 수 있을까?</p>
<h1 id="해결">해결</h1>
<p>해당 문제는 방향성에 대한 미흡한 생각에 발생한 문제이다. 다른 문제와 같으면 엣지의 방향성을 양방향으로 가정하고 3번 노드를 조회했을 때 어떻게 연결되었는지 판단했을 것이다. 하지만 그렇게 바꿔버리면 막대 그래프와 도넛 그래프의 특징이 모호해져서 더 어려워 진다. 그래서 양방향으로 바꿔 푸는 방법은 진작에 포기했다.</p>
<p>해당 문제는 이전의 1번에서 임의 생성 노드 부분에서 해결책을 떠올렸다.</p>
<p>임의 생성한 노드는 분명 전체 그래프마다 하나씩 연결되어 있다 했다. 그렇다면 임의 생성 노드의 엣지 개수는 전체 그래프의 개수임이 틀림없다.</p>
<p>그렇다면 전체 그래프 개수에서 확실하게 구할 수 있는 도넛과 8자 모양 그래프의 합을 빼면 막대 그래프의 개수일 것이다(주어진 그래프는 3종류 중 하나라는 문제 가정을 기반).</p>
<p>그럼으로 이전에 구했던 임의 생성 노드의 간선 개수에서 도넛, 8자 모양 그래프의 개수를 뺀 개수를 막대 모양 그래프의 개수로 판단하고 결과에 넣었다.</p>
<p>그래서 아래의 코드와 같이 targetShape가 1이 아닐때(막대 그래프가 아닐 때), 전체 개수를 하나씩 빼고 해당 그래프는 카운트하는 방식으로 해결하였다.</p>
<pre><code class="language-java">int targetShape = getShapeIdx(nodeCnt, edgeCnt);
                // 왜 막대만 빼고 하는 지는 뒤에 설명한다.
                if (targetShape != 1) {
                    answer[targetShape]++;
                    totalCnt--;
                }</code></pre>
<h1 id="전체-코드">전체 코드</h1>
<pre><code class="language-java">import java.io.IOException;
import java.util.*;

class Solution {
    public int[] solution(int[][] edges) {
        Map&lt;Integer, Queue&lt;Integer&gt;&gt; relation = new HashMap&lt;&gt;();
        boolean[] isReceived = new boolean[1000001];
        for (int[] edge : edges) {
            if (relation.containsKey(edge[0])) {
                relation.get(edge[0]).add(edge[1]);
            } else {
                relation.put(edge[0], new LinkedList&lt;&gt;(List.of(edge[1])));
            }
            isReceived[edge[1]] = true;
            if (!relation.containsKey(edge[1])) {
                relation.put(edge[1], new LinkedList&lt;&gt;());
            }
        }
        int removeNode = -1;
        for (Map.Entry&lt;Integer, Queue&lt;Integer&gt;&gt; entry : relation.entrySet()) {
            if (entry.getValue().size() &gt;= 2 &amp;&amp; !isReceived[entry.getKey()]) {
                removeNode = entry.getKey();
            }
        }

        int totalCnt = relation.get(removeNode).size();
        relation.remove(removeNode);
        int[] answer = new int[]{removeNode, 0, 0, 0};
        boolean[] isVisit = new boolean[1000001];
        for (Map.Entry&lt;Integer, Queue&lt;Integer&gt;&gt; entry : relation.entrySet()) {
            if (!isVisit[entry.getKey()]) {
                isVisit[entry.getKey()] = true;
                int nodeCnt = 1;
                int edgeCnt = 0;
                Queue&lt;Integer&gt; route = new LinkedList&lt;&gt;();
                route.add(entry.getKey());
                while (!route.isEmpty()) {
                    int node = route.poll();
                    while (!relation.get(node).isEmpty()) {
                        int nextNode = relation.get(node).poll();
                        edgeCnt++;
                        if (!isVisit[nextNode]) {
                            nodeCnt++;
                            isVisit[nextNode] = true;
                            route.add(nextNode);
                        }
                    }
                }
                int targetShape = getShapeIdx(nodeCnt, edgeCnt);
                if (targetShape != 1) {
                    answer[targetShape]++;
                    totalCnt--;
                }
            }
        }
        answer[1] = totalCnt;
        return answer;
    }

    private int getShapeIdx(int nodeCnt, int edgeCnt) {
        if (nodeCnt == edgeCnt) {
            return 1;
        } else if (nodeCnt-1 == edgeCnt) {
            return 2;
        } else if (nodeCnt+1== edgeCnt) {
            return 3;
        }
        throw new IllegalArgumentException(&quot;잘못된 입력&quot;);
    }
}</code></pre>