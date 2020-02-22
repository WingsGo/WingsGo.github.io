---
title: Linux下Socket编程及IO多路复用
key: 20200222
tags: Linux Socket
---

# Linux下Socket编程及IO多路复用
从快毕业到参与工作半年了，从在校的时候编写一些单机版的PC端软件，到现在进入公司接触分布式系统，进入OLAP的领域，感觉自己也成长了不少。如今由于疫情的影响，在家宅了也快一个月，工作之余也应该把博客捡起来，也算是对自己成长的一个见证吧。

由于笔者在校期间主要做的是Windows下单机的PC端软件的开发，因此这篇博客就从Linux下的Socker编程开始，作为一个转变的分界点开篇。

和Java的万物皆对象类似，Linux下的一切都是文件，在Linux下是通过Socket进行网络编程的，Socket也是文件描述符的一种，服务端通过打开一个Socket，等待客户端连接后会返回一个新的文件描述符 `fd`，通过对该文件描述符的读写完成Server端和Client端的通信。

![Socket通信流程](https://github.com/WingsGo/WingsGo.github.io/tree/master/images/Linux-Socket.svg)

### Socket编程API

```
1、socket函数
    syntax:int socket(int domain, int type, int protocol);
功能说明：
   调用成功，返回socket文件描述符；失败，返回－1，并设置errno
参数说明：
　　domain指明所使用的协议族，通常为PF_INET，表示TCP/IP协议；
　　type参数指定socket的类型，基本上有三种：数据流套接字、数据报套接字、原始套接字
　　protocol通常赋值"0"。
　　两个网络程序之间的一个网络连接包括五种信息：通信协议、本地协议地址、本地主机端口、远端主机地址和远端协议端口。socket数据结构中包含这五种信息。

2、bind函数
    syntax:int bind(int sock_fd,struct sockaddr_in *my_addr, int addrlen);
功能说明：
   将套接字和指定的端口相连。成功返回0，否则，返回－1，并置errno.
参数说明：
    sock_fd是调用socket函数返回值,
    my_addr是一个指向包含有本机IP地址及端口号等信息的sockaddr类型的指针；
    struct sockaddr_in结构类型是用来保存socket信息的：
    struct sockaddr_in {
        short int sin_family;
        unsigned short int sin_port;
        struct in_addr sin_addr;
        unsigned char sin_zero[8];
    };
    addrlen为sockaddr的长度。

3、connect函数
    syntax: int connect(int sock_fd, struct sockaddr *serv_addr,int addrlen);
功能说明：
   客户端发送服务请求。成功返回0，否则返回－1，并置errno。
参数说明：
   sock_fd 是socket函数返回的socket描述符；serv_addr是包含远端主机IP地址和端口号的指针；addrlen是结构sockaddr_in的长度。

4、listen函数
    syntax:int listen(int sock_fd， int backlog);
功能说明：
   等待指定的端口的出现客户端连接。调用成功返回0，否则，返回－1，并置errno.
参数说明：
   sock_fd 是socket()函数返回值；
   backlog指定在请求队列中允许的最大请求数。

5、accecpt函数
    syntax:int accept(int sock_fd, struct sockadd_in* addr, int addrlen);
功能说明：
   用于接受客户端的服务请求，成功返回新的套接字描述符，失败返回－1，并置errno。
参数说明：
   sock_fd是被监听的socket描述符，
   addr通常是一个指向sockaddr_in变量的指针，
   addrlen是结构sockaddr_in的长度。

6、write函数
    syntax:ssize_t write(int fd,const void *buf,size_t nbytes)
功能说明：
    write函数将buf中的nbytes字节内容写入文件描述符fd.成功时返回写的字节数.失败时返回-1. 并设置errno变量。
    在网络程序中,当我们向套接字文件描述符写时有俩种可能：
      1)write的返回值大于0,表示写了部分或者是全部的数据.
      2)返回的值小于0,此时出现了错误.需要根据错误类型来处理.
        如果错误为EINTR表示在写的时候出现了中断错误.
        如果错误为EPIPE表示网络连接出现了问题.

7、read函数
    syntax:ssize_t read(int fd,void *buf,size_t nbyte)
函数说明：
    read函数是负责从fd中读取内容.当读成功时,read返回实际所读的字节数,如果返回的值是0 表示已经读到文件的结束了,小于0表示出现了错误.
    如果错误为EINTR说明读是由中断引起的,
    如果错误是ECONNREST表示网络连接出了问题.

8、close函数
    syntax:int close(sock_fd);
说明:
   当所有的数据操作结束以后，你可以调用close()函数来释放该socket，从而停止在该socket上的任何数据操作：
函数运行成功返回0，否则返回-1。
```

### Socket编程及注意事项

1. 网络字节顺序
    每一台机器内部对变量的字节存储顺序不同，而网络传输的数据是一定要统一顺序的。所以对内部字节表示顺序与网络字节顺序不同的机器，一定要进行转换。从程序的可移植性要求来讲，就算本机的内部字节表示顺序与网络字节顺序相同也应该在传输数据以前先调用数据转换函数，以便程序移植到其它机器上后能正确执行。真正转换还是不转换是由系统函数自己来决定的。

2. 转换函数

```
主机字节顺序转换成网络字节顺序，对无符号短型进行操作4bytes
* unsigned short int htons（unsigned short int hostshort）：
主机字节顺序转换成网络字节顺序，对无符号长型进行操作8bytes
* unsigned long int htonl（unsigned long int hostlong）：
网络字节顺序转换成主机字节顺序，对无符号短型进行操作4bytes
* unsigned short int ntohs（unsigned short int netshort）：
网络字节顺序转换成主机字节顺序，对无符号长型进行操作8bytes
* unsigned long int ntohl（unsigned long int netlong）：
注：以上函数原型定义在netinet/in.h里。
```

3. IP地址转换
有三个函数将数字点形式表示的字符串IP地址与32位网络字节顺序的二进制形式的IP地址进行转换
```
unsigned long int inet_addr(const char* cp);
说明：
    该函数把一个用数字和点表示的IP地址的字符串转换成一个无符号长整型，如：
    struct sockaddr_in ina; 
    ina.sin_addr.s_addr=inet_addr("202.206.17.101");
    该函数成功时：返回转换结果；失败时返回常量INADDR_NONE，该常量=-1，二进制的无符号整数-1相当于255.255.255.255，这是一个广播地址，所以在程序中调用iner_addr（）时，一定要人为地对调用失败进行处理。由于该函数不能处理广播地址，所以在程序中应该使用函数inet_aton（）。


int inet_aton(const char* cp, struct in_addr* inp);
说明:
    此函数将字符串形式的IP地址转换成二进制形式的IP地址；成功时返回1，否则返回0，转换后的IP地址存储在参数inp中。


（3） char* inet_ntoa(struct in-addr in);
说明:
    将32位二进制形式的IP地址转换为数字点形式的IP地址，结果在函数返回值中返回，返回的是一个指向字符串的指针。
```

4. 字节处理函数
    Socket地址是多字节数据，不是以空字符结尾的，这和C语言中的字符串是不同的。Linux提供了两组函数来处理多字节数据，一组以b（byte）开头，是和BSD系统兼容的函数，另一组以mem（内存）开头，是ANSI C提供的函数。

以mem开头的函数有：
```
void* memset(void* s，int c，size_t n);
说明:
    将参数s指定的内存区域的前n个字节设置为参数c的内容。


void* memcpy(void* dest，const void* src，size_t n);
说明:
    功能同bcopy（），区别：函数bcopy（）能处理参数src和参数dest所指定的区域有重叠的情况，memcpy（）则不能。


int memcmp(const void* s1，const void* s2，size_t n);
说明:
    比较参数s1和参数s2指定区域的前n个字节内容，如果相同则返回0，否则返回非0。
```

以b开头的函数有:
```
void bzero(void* s,int n);
说明:
    将参数s指定的内存的前n个字节设置为0，通常它用来将套接字地址清0。


void bcopy(const void *src，void * dest，int n);
说明:
    从参数src指定的内存区域拷贝指定数目的字节内容到参数dest指定的内存区域。


int bcmp(const void* s1，const void* s2，int n);
说明:
    比较参数s1指定的内存区域和参数s2指定的内存区域的前n个字节内容，如果相同则返回0，否则返回非0。

```
注：以上函数的原型定义在strings.h中。

### 示例程序
有了以上的工作流程图及API说明，我们就可以编写一个基本的Socket通信程序。

```
#include <arpa/inet.h>
#include <sys/socket.h>

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <string>
#include <thread>

#define ZERO_OR_RETURN(expr)                                       \
    do {                                                           \
        int ret = (expr);                                          \
        if (ret != 0) {                                                \
            printf("> Error: line=%d, ret=%d\n", __LINE__, ret);   \
            return ret;                                            \
        }                                                          \
    } while (0);

int Server(int port) {
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(port);
    socklen_t server_len = sizeof(server_addr);

    int socket_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (socket_fd == -1) {
        perror("> Error: open socket failed [%d].\n");
        exit(1);
    }
    printf("> Create socket [%d] succeed.\n", socket_fd);

    ZERO_OR_RETURN(bind(socket_fd, (struct sockaddr *)&server_addr, server_len));
    printf("> Bind socket port succeed.\n");

    ZERO_OR_RETURN(listen(socket_fd, 1024));
    printf("> Listen socket port succeed.\n");

    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    int fd = accept(socket_fd, reinterpret_cast<sockaddr *>(&client_addr), &client_len);
    if (fd == -1) {
        perror("> Error: failed to accept connection\n");
        exit(1);
    }
    printf("> Accept new connection [%d].\n", fd);

    std::string buffer(1024, '\0');
    int read_bytes = read(fd, &buffer[0], 1024);
    buffer.reserve(read_bytes);
    printf("> Recv %s", buffer.c_str());

    int write_bytes = write(fd, &buffer[0], 1024);
    buffer.reserve(write_bytes);
    printf("> Send %s", buffer.c_str());

    ZERO_OR_RETURN(close(fd));
    ZERO_OR_RETURN(close(socket_fd));
    return 0;
}


int main(int argc, char** argv) {
    std::thread server(Server, 19999);
    server.join();
    return 0;
}

```

使用命令 `g++ main.cpp -o main -pthread`进行编译，并使用netcat进行测试

### IO多路复用(epoll)
