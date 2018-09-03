---
title: 动态规划题解
key: 20180903
tags:
- Dynamic Programming
---

## 动态规划面试题解 ##
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
