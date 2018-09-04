---
title: 校招笔试题解
key: 20180903
tags:
- Dynamic Programming
- BFS
---

## 动态规划面试题解
<!--more-->

### 网易笔试--合唱团
链接：https://www.nowcoder.com/questionTerminal/661c49118ca241909add3a11c96408c8
来源：牛客网

有 n 个学生站成一排，每个学生有一个能力值，牛牛想从这 n 个学生中按照顺序选取 k 名学生，要求相邻两个学生的位置编号的差不超过 d，使得这 k 个学生的能力值的乘积最大，你能返回最大的乘积吗？
输入描述:

每个输入包含 1 个测试用例。每个测试数据的第一行包含一个整数 n (1 <= n <= 50)，表示学生的个数，接下来的一行，包含 n 个整数，按顺序表示每个学生的能力值 ai（-50 <= ai <= 50）。接下来的一行包含两个整数，k 和 d (1 <= k <= 10, 1 <= d <= 50)。



输出描述:

输出一行表示最大的乘积。

    #include <climits>
	#include <vector>
	#include <algorithm>
	#include <iostream>

	using namespace std;

	int main(int argc, char** argv) {
	    size_t n;
	    cin >> n;
	    vector<long long> students(n, 0);
	    for (size_t i=0; i<n; ++i) {
	        int power;
	        cin >> power;
	        students[i] = power;
	    }
	    int k, d;
	    cin >> k >> d;
	
	    vector<vector<long long>> dp_max(n, vector<long long>(k, 0));
	    vector<vector<long long>> dp_min(n, vector<long long>(k, 0));
	    long long max_power = INT8_MIN;
	    for (size_t row=0; row<n; ++row) {
	        max_power = max(max_power, students[row]);
	        dp_max[row][0] = students[row];
	        dp_min[row][0] = students[row];
	    }
	
	    for (int col=1; col<k; ++col) {
	        for (int row=col; row<n; ++row) {
	            for (int idx=row-1; row-idx<=d && idx>=0; --idx) {
	                dp_max[row][col] = max(dp_max[row][col],
	                        max(dp_max[idx][col-1] * students[row], dp_min[idx][col-1]*students[row]));
	                dp_min[row][col] = min(dp_min[row][col],
	                        min(dp_min[idx][col-1] * students[row], dp_max[idx][col-1]*students[row]));
	            }
	            max_power = max(max_power, dp_max[row][col]);
	        }
	    }
	    cout << max_power <<endl;
	    return 0;
	}



### 腾讯笔试--小Q的歌单
链接：https://www.nowcoder.com/questionTerminal/f3ab6fe72af34b71a2fd1d83304cbbb3?page=1&onlyReference=false
来源：牛客网

小Q有X首长度为A的不同的歌和Y首长度为B的不同的歌，现在小Q想用这些歌组成一个总长度正好为K的歌单，每首歌最多只能在歌单中出现一次，在不考虑歌单内歌曲的先后顺序的情况下，请问有多少种组成歌单的方法。
输入描述:

每个输入包含一个测试用例。
每个测试用例的第一行包含一个整数，表示歌单的总长度K(1<=K<=1000)。
接下来的一行包含四个正整数，分别表示歌的第一种长度A(A<=10)和数量X(X<=100)以及歌的第二种长度B(B<=10)和数量Y(Y<=100)。保证A不等于B。



输出描述:

输出一个整数,表示组成歌单的方法取模。因为答案可能会很大,输出对1000000007取模的结果。

示例1
输入

5
2 3 3 3

输出

9

	#include <vector>
	#include <algorithm>
	#include <iostream>


	int main(int argc, char** argv) {
	    size_t mod = 1000000007;
	    size_t K, A, X, B, Y;
	    while (std::cin >> K >> A >> X >> B >> Y) {
	        std::vector<size_t > song_len(X+Y, 0);
	        for (size_t i=0; i<X; ++i)
	            song_len[i] = A;
	        for (size_t i=X; i<X+Y; ++i)
	            song_len[i] = B;
	        std::vector<std::vector<size_t>> dp(X+Y, std::vector<size_t>(K + 1));
	        for (size_t col=0; col<K+1; ++col) {
	            if (col != song_len[0])
	                dp[0][col] = 0;
	            else
	                dp[0][col] = 1;
	        }
	        for (size_t row=0; row<X+Y; ++row)
	            dp[row][0] = 1;
	
	        for (size_t row=1; row<X+Y; ++row) {
	            for (size_t col = 1; col<K+1; ++col) {
	                if (col >= song_len[row])
	                    dp[row][col] = (dp[row-1][col] + dp[row-1][col-song_len[row]]) % mod;
	                else
	                    dp[row][col] = dp[row-1][col] % mod;
	            }
	        }
	        std::cout << dp[X+Y-1][K] << std::endl;
	    }
	    return 0;
	}


## BFS题解

### 网易笔试--推箱子
大家一定玩过“推箱子”这个经典的游戏。具体规则就是在一个N*M的地图上，有1个玩家、1个箱子、1个目的地以及若干障碍，其余是空地。玩家可以往上下左右4个方向移动，但是不能移动出地图或者移动到障碍里去。如果往这个方向移动推到了箱子，箱子也会按这个方向移动一格，当然，箱子也不能被推出地图或推到障碍里。当箱子被推到目的地以后，游戏目标达成。现在告诉你游戏开始是初始的地图布局，请你求出玩家最少需要移动多少步才能够将游戏目标达成。
输入描述:

每个测试输入包含1个测试用例
第一行输入两个数字N，M表示地图的大小。其中0<N，M<=8。
接下来有N行，每行包含M个字符表示该行地图。其中 . 表示空地、X表示玩家、*表示箱子、#表示障碍、@表示目的地。
每个地图必定包含1个玩家、1个箱子、1个目的地。



输出描述:

输出一个数字表示玩家最少需要移动多少步才能将游戏目标达成。当无论如何达成不了的时候，输出-1。


输入例子1:

4 4

....

..*@

....

.X..

6 6

...#..

......

\#*##..

..##.#

..X...

.@#...


输出例子1:

3

11

    #include <queue>
	#include <iostream>
	#include <string>
	
	using namespace std;
	
	class Position {
	public:
	    Position(int x, int y, int bx, int by) : m_hx(x), m_hy(y), m_bx(bx), m_by(by) {}
	
	    int m_hx;
	    int m_hy;
	    int m_bx;
	    int m_by;
	};
	
	int main(int argc, char **argv) {
	    size_t N, M;
	    cin >> N >> M;
	    vector<string> map(N, "");
	    for (size_t row = 0; row < N; ++row) {
	        cin >> map[row];
	    }
	
	    int x_pos[4] = {-1, 1, 0, 0};
	    int y_pos[4] = {0, 0, -1, 1};
	    vector<vector<vector<vector<int>>>> visited(N,
	            vector<vector<vector<int>>>(M, vector<vector<int>>(N, vector<int>(M, 0))));
	    int h_init_x, h_init_y, b_init_x, b_init_y;
	    int d_init_x, d_init_y;
	
	    for (int row = 0; row < N; ++row) {
	        for (int col = 0; col < map[0].size(); ++col) {
	            if (map[row][col] == 'X') {
	                h_init_x = row;
	                h_init_y = col;
	            } else if (map[row][col] == '*') {
	                b_init_x = row;
	                b_init_y = col;
	            } else if (map[row][col] == '@') {
	                d_init_x = row;
	                d_init_y = col;
	            } else {}
	        }
	    }
	
	    visited[h_init_x][h_init_y][b_init_x][b_init_y] = 1;
	    Position pos(h_init_x, h_init_y, b_init_x, b_init_y);
	    queue<Position> util_queue;
	    util_queue.push(pos);
	    while (!util_queue.empty()) {
	        auto front = util_queue.front();
	        util_queue.pop();
	
	        if (front.m_bx == d_init_x && front.m_by == d_init_y) {
	            cout << visited[front.m_hx][front.m_hy][front.m_bx][front.m_by] - 1;
	            return 0;
	        }
	
	        for (int i = 0; i < 4; ++i) {
	            int hnx = front.m_hx + x_pos[i];
	            int hny = front.m_hy + y_pos[i];
	            if (hnx < 0 || hny < 0 || hnx >= N || hny >= M || map[hnx][hny] == '#') continue;
	
	            if (hnx == front.m_bx && hny == front.m_by) {
	                int bnx = front.m_bx + x_pos[i];
	                int bny = front.m_by + y_pos[i];
	                if (bnx < 0 || bny < 0 || bnx >= N || bny >= M || map[bnx][bny] == '#') continue;
	                visited[hnx][hny][bnx][bny] = visited[front.m_hx][front.m_hy][front.m_bx][front.m_by] + 1;
	                util_queue.push(Position(hnx, hny, bnx, bny));
	            } else {
	                if (visited[hnx][hny][front.m_bx][front.m_by]) continue;
	                visited[hnx][hny][front.m_bx][front.m_by] = visited[front.m_hx][front.m_hy][front.m_bx][front.m_by] + 1;
	                util_queue.push(Position(hnx, hny, front.m_bx, front.m_by));
	            }
	        }
	    }
	    cout << -1 << endl;
	}
