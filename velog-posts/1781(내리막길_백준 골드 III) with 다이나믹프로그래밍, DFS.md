<p><a href="https://www.acmicpc.net/problem/1520">문제 링크</a></p>
<h1 id="문제-소개">문제 소개</h1>
<p>해당 문제는 어떠한 맵이 주어졌을 때 어떤 노드에서 노드로 갈 때 노드의 값이 작아지는 방향으로만 움직일 수 있을 때 왼쪽 상단에서 출발해서 오른쪽 하단에 도착할 수 있는 경우의 수를 구하는 문제이다.</p>
<h1 id="문제-분석">문제 분석</h1>
<p>우선 딱 봐도 DFS를 사용하는 것이 눈에 보여 아래와 같이 풀었다.</p>
<pre><code>import sys
sys.setrecursionlimit(10**6)


def solution():
    def dfs(y, x, visited):
        nonlocal cnt
        if y == m - 1 and x == n - 1:
            cnt += 1
            return
        for d in range(4):
            ny, nx = y + controlY[d], x + controlX[d]
            if -1 &lt; ny &lt; m and -1 &lt; nx &lt; n and not visited[ny][nx] and board[y][x] &gt; board[ny][nx]:
                visited[ny][nx] = True
                dfs(ny, nx, visited)
                visited[ny][nx] = False

    input = sys.stdin.readline
    m, n = map(int, input().split())
    board = [list(map(int, input().split())) for _ in range(m)]
    controlY, controlX = [0, 0, 1, -1], [1, -1, 0, 0]
    visited = [[False for _ in range(n)] for _ in range(m)]
    visited[0][0] = True
    cnt = 0
    dfs(0, 0, visited)
    print(cnt)


solution()</code></pre><p>위의 코드는 전형적인 DFS를 풀듯이 코드를 작성해서 제출했지만 시간 초과가 발생하였다.</p>
<p>그래서 해당 DFS에서 추가로 시간을 줄여야 한다.</p>
<h1 id="접근-1">접근 1</h1>
<p>우리가 A,B,C,D,E,F,G노드가 존재한다고 가정했을 때 시작 노드가 A이고 도착 노드가 G라 해보자.</p>
<p>A에서 G까지 갈 수 있는 경우의 수를 구하려면 어떻게 구해야 할까?</p>
<blockquote>
</blockquote>
<ol>
<li>A에서 G까지 가기 위해 경우의 수를 카운트하기 위해 하나씩 노드를 움직여 보면서, G에 도착했던 총 경우의 수를 구함</li>
<li>우선 G노드 주위에 있는 노드들부터 경우의 수를 구해봄. 이러한 방식으로 A노드까지 구해봄</li>
</ol>
<p>1번 방법과 같이 일일이 카운트하면 시간이 매우 많이 걸릴수밖에 없다.</p>
<p>하지만 2번 방법과 같은 경우에는 1번 방법과 같이 G노드에 도착하지 못하는 경우를 생략할 수 있어 시간 단축에 도움이 된다.</p>
<pre><code>    def dfs(y, x):
        if y == m - 1 and x == n - 1:
            return 1
        if dp[y][x] == -1:
            dp[y][x] = 0
            for d in range(4):
                ny, nx = y + controlY[d], x + controlX[d],
                if -1 &lt; ny &lt; m and -1 &lt; nx &lt; n and board[ny][nx] &lt; board[y][x]:
                    dp[y][x] += dfs(ny, nx)

        return dp[y][x]</code></pre><p>위의 코드를 보듯이 A노드부터 시작해서 DFS를 통해 주위 노드를 탐색한다. A노드 근처의 경우의 수를 탐색할 때 해당 노드의 근처 노드를 또 탐색해서 결국에는 2번 방법과 같이 G노드에 도착하면 1를 리턴해서 차곡차곡 더해가는 그림이 그려진다.</p>
<h2 id="전체-코드">전체 코드</h2>
<pre><code>import sys

# 재귀 호출 횟수 제한을 늘림
sys.setrecursionlimit(10 ** 6)

# DFS 탐색 함수
def dfs(y, x):
    # 도착 지점에 도달한 경우 1을 반환
    if y == m - 1 and x == n - 1:
        return 1

    # 이미 방문한 적이 있는 경우 해당 값을 반환
    if dp[y][x] == -1:
        dp[y][x] = 0
        for d in range(4):
            ny, nx = y + controlY[d], x + controlX[d]
            # 상하좌우로 이동하며 높이가 더 낮은 지점으로 이동
            if -1 &lt; ny &lt; m and -1 &lt; nx &lt; n and board[ny][nx] &lt; board[y][x]:
                dp[y][x] += dfs(ny, nx)  # 경로의 개수를 누적

    return dp[y][x]  # 경로의 개수 반환

def solution():
    input = sys.stdin.readline
    m, n = map(int, input().split())
    board = [list(map(int, input().split())) for _ in range(m)]
    controlY, controlX = [0, 0, 1, -1], [1, -1, 0, 0]
    dp = [[-1 for _ in range(n)] for _ in range(m)]  # 메모이제이션을 위한 배열

    print(dfs(0, 0))  # 시작 지점에서 DFS 탐색 시작

# solution 함수 호출
solution()</code></pre><h1 id="접근-2">접근 2</h1>
<p>해당 접근은 백준에서 가져온 코드이다. 대략적인 알고리즘은 똑같지만, 내부 구현이 최적화되어 있어 효율적임으로 속도가 더 빠른 코드여서 가져왔다.</p>
<p>코드 리뷰는 주석처리를 했으니 참고 바란다.</p>
<pre><code>import sys

direction = [(0, -1), (0, 1), (-1, 0), (1, 0)]

def rec(x, y, m, n):
    # 도착점에 도달한 경우, 경로의 수를 1로 설정하고 종료
    if m == x - 1 and n == y - 1:
        dp[m][n] = 1
        return
    # 이미 방문한 지점인 경우 더 이상의 계산을 수행하지 않음
    if dp[m][n] != -1:
        return

    dp[m][n] = 0  # 초기값 설정
    for dx, dy in direction:
        n_m, n_n = dx + m, dy + n
        # 범위 내에 있고, 내리막길인 경우
        if 0 &lt;= n_m &lt; x and 0 &lt;= n_n &lt; y and table[m][n] &gt; table[n_m][n_n]:
            # 재귀적으로 이동하며 경로의 수 누적
            rec(x, y, n_m, n_n)
            dp[m][n] += dp[n_m][n_n]

x, y = [int(v) for v in input().split()]
table = []
dp = [[-1] * y for _ in range(x)]

for _ in range(x):
    table.append(list(map(int, sys.stdin.readline().rstrip().split())))

rec(x, y, 0, 0)  # DFS 호출
print(dp[0][0])  # 결과 출력</code></pre>