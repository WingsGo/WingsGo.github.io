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

### 美团笔试--拼凑钱币
链接：https://www.nowcoder.com/questionTerminal/178b912722ac42a2865057a66d4e7de2
来源：牛客网

[编程题]拼凑钱币

给你六种面额 1、5、10、20、50、100 元的纸币，假设每种币值的数量都足够多，编写程序求组成N元（N为0~10000的非负整数）的不同组合的个数。
输入描述:

输入包括一个整数n(1 ≤ n ≤ 10000)



输出描述:

输出一个整数,表示不同的组合方案数

示例1
输入

1

输出

1

典型的完全背包问题，状态转移方程为**dp[i][j] = dp[i-1][j] + dp[i][j-coins[i]]**

如果是01背包问题，状态转移方程为**dp[i][j] = dp[i-1][j] + dp[i-1][j-coins[i]]**

区别在于01背包选了当前物品后就不能再次选择，而完全背包可以选择数量num(0<coins[i]*num<j)

    #include <string>
	#include <algorithm>
	#include <iostream>
	#include <vector>
	
	using namespace std;
	
	int main(int argc, char** argv) {
	    int n;
	    vector<int> coins = {1, 5, 10, 20, 50, 100};
	    while (cin >> n) {
	        vector<vector<long long>> dp(6, vector<long long>(n+1, 0));
	        for (int i=0; i<6; ++i)
	            dp[i][0] = 1;
	        for (int j=0; j<n+1; ++j)
	            dp[0][j] = 1;
	        for (int i=1; i<6; ++i) {
	            for (int j=1; j<n+1; ++j) {
	                if (j>=coins[i])
	                    dp[i][j] = dp[i-1][j] + dp[i][j-coins[i]];
	                else
	                    dp[i][j] = dp[i-1][j];
	            }
	        }
	        cout << dp[5][n] << endl;
	    }
	    return 0;
	}

### 美团笔试--最大矩形面积


给定一组非负整数组成的数组h，代表一组柱状图的高度，其中每个柱子的宽度都为1。 在这组柱状图中找到能组成的最大矩形的面积（如图所示）。 入参h为一个整型数组，代表每个柱子的高度，返回面积的值。


输入描述:

输入包括两行,第一行包含一个整数n(1 ≤ n ≤ 10000)
第二行包括n个整数,表示h数组中的每个值,h_i(1 ≤ h_i ≤ 1,000,000)



输出描述:

输出一个整数,表示最大的矩阵面积。


输入例子1:

6
2 1 5 6 2 3


输出例子1:

10



这道题可以用动态规划，暂时没想到，先上一个暴力解法


    #include <string>
	#include <algorithm>
	#include <iostream>
	#include <vector>
	
	using namespace std;
	
	int main(int argc, char** argv) {
	    int n;
	    cin >> n;
	    vector<int> rect(n, 0);
	    for (int i=0; i<n; ++i) {
	        cin >> rect[i];
	    }
	
	    vector<int> area(n, 0);
	    for (int i=0; i<rect.size(); ++i) {
	        int current_area = rect[i];
	        for (int j=i+1; j<rect.size(); ++j) {
	            if (rect[j] >= rect[i])
	                current_area += rect[i];
	            else
	                break;
	        }
	        for (int j=i-1; j>=0; --j) {
	            if (rect[j] >= rect[i])
	                current_area += rect[i];
	            else
	                break;
	        }
	        area[i] = current_area;
	    }
	    cout << *max_element(area.begin(), area.end()) <<endl;
	    return 0;
	}

使用单调栈的解法
    
    int largestRectangleArea(vector<int>& heights)
	{
	    stack<int> S;
	    heights.push_back(0); //
	    int res = 0;
	
	    for(int i = 0; i<heights.size(); i++)
	    {
	        if(S.empty() || heights[i] > heights[S.top()])
	            S.push(i);
	        else
	        {
	            int temp = S.top();
	            S.pop();
	            int j = S.top();
	            int dist = S.empty() ? i : i-S.top() -1;
	            res = max(res, dist * heights[temp]);
	            i--; // set i one step back.  to make sure the bar i will be processed.
	        }
	    }
	
	    return res;
	}

### 迅雷笔试--整数求和
[编程题] 整数求和

时间限制：1秒

空间限制：131072K
给定整数n，取若干个1到n的整数可求和等于整数m，编程求出所有组合的个数。比如当n=6，m=8时，有四种组合：[2,6], [3,5], [1,2,5], [1,3,4]。限定n和m小于120
输入描述:

整数n和m



输出描述:

求和等于m的所有组合的个数。


输入例子1:

6 8


输出例子1:

4

典型的01背包问题，可以和之前的拼凑钱币问题做一个比较，如果此时可以取相同的数，那么就变为完全背包问题，和拼凑钱币解法一样。

    #include <bits/stdc++.h>

	using namespace std;
	
	int main(int argc, char **argv) {
	    int n, m;
	    cin >> n >> m;
	    vector<vector<int>> dp(n, vector<int>(m + 1, 0));
	    for (int i = 0; i < n; ++i)
	        dp[i][0] = 1;
	    dp[0][1] = 1;
	
	    for (int i = 1; i < n; ++i) {
	        for (int j = 1; j < m + 1; ++j) {
	            if (j - i - 1 >= 0)
	                dp[i][j] = dp[i - 1][j] + dp[i - 1][j - i - 1];
	            else
	                dp[i][j] = dp[i - 1][j];
	        }
	    }
	    cout << dp[n-1][m] << endl;
	
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

### 剑指offer--二叉树中和为某一值得路径
输入一颗二叉树的跟节点和一个整数，打印出二叉树中结点值的和为输入整数的所有路径。路径定义为从树的根结点开始往下一直到叶结点所经过的结点形成一条路径。(注意: 在返回值的list中，数组长度大的数组靠前)

    class Solution {
	private:
	    vector<vector<int>> all_result;
	    vector<int> tmp;
	public:
	    vector<vector<int> > FindPath(TreeNode* root,int expectNumber) {
	        if (root)
	            dfs(root, expectNumber);
	        return all_result;
	    }
	
	    void dfs(TreeNode* node, int target) {
	        tmp.push_back(node->val);
			// 替换判断数值语句可转化为求树的所有路径问题
	        if (node->val == target && node->left == nullptr && node->right == nullptr)
	            all_result.push_back(tmp);
	        if (node->left)
	            dfs(node->left, target - node->val);
	        if (node->right)
	            dfs(node->right, target - node->val);
	        tmp.pop_back();
	    }
	};


## 大数系列
###大数加法
链接：https://www.nowcoder.com/questionTerminal/850fde3d987f4b678171abd88cf05710
来源：牛客网

请设计一个算法能够完成两个用字符串存储的整数进行相加操作，对非法的输入则返回error
输入描述:

输入为一行，包含两个字符串，字符串的长度在[1,100]。



输出描述:

输出为一行。合法情况输出相加结果，非法情况输出error

示例1
输入

123 123
abd 123

输出

246
Error

    #include <string>
	#include <algorithm>
	#include <iostream>
	
	using namespace std;
	
	bool vaild_input(const string &str) {
	    if (str.empty())
	        return false;
	    else if (str.size() == 1) {
	        if (!isdigit(str[0]))
	            return false;
	        else
	            return true;
	    } else {
	        int point_count = 0;
	        for (size_t i = 1; i < str.size(); ++i) {
	            if (i == 1 && !isdigit(str[i]))
	                return false;
	            else {
	                if (!isdigit(str[i])) {
	                    if (str[i] != '.') return false;
	                    else {
	                        point_count++;
	                        if (point_count >= 2)return false;
	                    }
	                }
	            }
	        }
	    }
	    return true;
	}
	
	string big_add(string &lhs, string &rhs) {
	    if (!vaild_input(lhs) || !vaild_input(rhs))
	        return "error";
	
	    reverse(lhs.begin(), lhs.end());
	    reverse(rhs.begin(), rhs.end());
	    string result;
	    size_t longer_len = lhs.size() > rhs.size() ? lhs.size() : rhs.size();
	    size_t carry = 0;
	    for (size_t i = 0; i < longer_len; ++i) {
	        size_t lhs_num = i >= lhs.size() ? 0 : lhs[i] - '0';
	        size_t rhs_num = i >= rhs.size() ? 0 : rhs[i] - '0';
	        size_t tmp = lhs_num + rhs_num + carry;
	        size_t mod = tmp % 10;
	        carry = tmp / 10;
	        result.insert(result.begin(), mod+'0');
	    }
	    if (carry)
	        result.insert(result.begin(), carry+'0');
	    return result;
	}
	
	int main(int argc, char **argv) {
	    string num_a, num_b;
	    while (cin >> num_a >> num_b) {
	        cout << big_add(num_a, num_b) << endl;
	    }
	    return 0;
	}


## 递归与回溯
### 头条笔试--还原IP
Given a string containing only digits, restore it by returning all possible valid IP address combinations.

Example:

Input: "25525511135"
Output: ["255.255.11.135", "255.255.111.35"]

    class Solution {
	public:
	    vector<string> restoreIpAddresses(string s) {
	        vector<string> res;
	        restore(s, 4, "", res);
			
			// 移除所有还原结果最后的.字符
	        for (auto& i : res)
	            i.erase(i.end() - 1);
	        return res;
	    }
	
		// s表示待分割的字符串，k表示要把s分为几部分，out表示之前分割的结果， res存储所有的还原结果
	    void restore(string s, int k, string out, vector<string> &res) {
	        if (k == 0) {
	            if (s.empty()) res.push_back(out);
	        } else {
	            for (int i = 1; i <= 3; ++i) {
	                if (s.size() >= i && isValid(s.substr(0, i))) {
	                        restore(s.substr(i), k - 1, out + s.substr(0, i) + ".", res);
	                }
	            }
	        }
	    }
	
	    bool isValid(string s) {
	        if (s.empty() || s.size() > 3 || (s.size() > 1 && s[0] == '0')) return false;
	        int res = atoi(s.c_str());
	        return res <= 255 && res >= 0;
	    }
	};