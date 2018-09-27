---
title: 从零实现atoi函数
key: 20180927
tags:
- atoi
---

## atoi的实现
<!--more-->
需要考虑的几个点：
- 整形溢出的问题
- 符号位的问题
- 输入有效性校验及异常处理


    #include <climits>

	enum STATUS {
	    valid = 0,
	    inValid = 1
	}
	
	g_status = valid;
	
	int str2int_core(const char* str, bool minus) {
	    long long res = 0;
	    while (*str != '\0') {
	        if (*str < '0' || *str > '9') {
	            res = 0;
	            break;
	        }
	        else {
	            int flag = minus ? -1 : 1;
	            res = 10 * res + flag * (*str - '0');
	            if ((!minus && res > INT_MAX) || (minus && res < INT_MIN)) {
	                res = 0;
	                break;
	            }
	            str++;
	        }
	    }
	    if (*str == '\0')
	        g_status = inValid;
	    return static_cast<int>(res);
	}
	
	int str2int(const char* str) {
	    g_status = inValid;
	    int res = 0;
	    bool minus = false;
	    if (str != nullptr && *str != '\0') {
	        if (*str == '+')
	            str++;
	        else if (*str == '-') {
	            minus = true;
	            str++;
	        }
	
	        if (*str != '\0')
	            res = str2int_core(str, minus);
	    }
	    return res;
	}