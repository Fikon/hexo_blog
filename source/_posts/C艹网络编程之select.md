---
title: C艹网络编程之select
date: 2016-09-04 10:34:11
tags: [那些年的坑]
categories: [学习心得, 网络编程]
---
*select()是一种比较古老的IO复用机制了，或许正是由于历史久远，才使得select可打开的最大文件数有限制，毕竟考虑到上个世纪当时性能限制和应用场景的简单，1024其实已经够用了。不过随着计算机性能的飞速提升和应用场景的复杂化
select已经明显疲于应付各种多路IO复用场景了，所以现在处理需要监听大量IO事件的场景都使用epoll了。*

### select()函数介绍——一个结构，一个函数和四种操作

#### 一个结构——fd_set
这个似乎没啥好说明的，就是一个保存所有监听的所有文件描述符的集合。

#### 一个函数——select()
整个select IO复用机制就一个select函数，相当的简单。`int select(int nfds, fd_set *readfds, fd_set* writefds, fd_set *exceptfds, struct timeval *timeout);`
- nfds：这个是readfds，writefds和exceptfds三个集合当中最大的文件描述符数值加1，我的理解是在有文件描述符就绪后，select需要遍历集合中所有的文件描述符来确定哪个是就绪了，所以需要一个最大数值的描述符来明确遍历的边界
- readfds，writefds，exceptfds：分别用来保存读就绪，写就绪和发生错误的文件描述符的集合，select函数调用之后，可以通过判断某个文件符是否在这三个集合中来确定该文件描述符是否就绪。
- timeout：select的等待时间，没有那么卑贱的等待，对吧，select也是有脾气的，它在timeout时间内仍未找到就绪的文件描述符的话，就会返回。

<!-- more -->

#### 四种操作
- FD_ZERO(fd_set *set)：这个是对集合的初始化操作，将集合清空
- FD_CTL(int fd, fd_set *set)：将fd移除出集合
- FD_SET(int fd, fd_set *set)：将fd添加至集合
- FD_ISSET(int fd, fd_set *set)：判断fd是否在集合中，在遍历整个fd集合的时候，就是通过这个函数还判断某个fd的就绪状态(判断其是否在readfds, writefds或者exceptfds中)

### 具体的例子——一个炒鸡简单的chat服务器
程序用select实现了一个简单的聊天服务器功能，在接受到客户端发来的消息是，会将消息发送给所有在监听集合中的客户端，废话不多说，直接上代码：
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define PORT "4399"   // 监听端口，哇，4399有点皮


int main(void)
{
    fd_set master;    // 监听时间的集合
    fd_set read_fds;  // 用于判断是否有读就绪的文件描述符，为了简单，这里只关注读事件
    int fdmax;        // 集合中最大文件描述符的数字（好别扭英文是file descriptor number，说实在的，这种东西硬翻译过来真难听）

    int s_sock;     // 监听的sock
    int c_client;        // 与服务器建立连接的sock
    struct sockaddr_storage client_addr; // client address
    socklen_t addrlen;

    char buf[256];    // 消息缓冲区
    int nbytes;


    int reuse = 1;        // 用于设置sock地址的reusage属性，不设置的话，在关闭程序后会有点时间不能使用该地址，我在想会不会是因为正处于time_wait状态，哈哈哈哈

    int i, j, res;

    struct addrinfo hints, *ai, *p;

    FD_ZERO(&master);    // 初始化
    FD_ZERO(&read_fds);

    // 给getaddrinfo一些hints，即我们希望构造的地址的一些属性，然后它返回满足这些属性的地址给我们
    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;  //不指定ipv4/ipv6都可以
    hints.ai_socktype = SOCK_STREAM;  //tcp socket
    hints.ai_flags = AI_PASSIVE;    //被动模式，一般服务器这样设置
    if ((rv = getaddrinfo(NULL, PORT, &hints, &ai)) != 0) {
      printf("hei, error happened in getaddrinfo\n");
      exit(1);
    }
    //遍历返回的地址节点，找到可用的就跳出循环
    for(p = ai; p != NULL; p = p->ai_next) {
        s_sock = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (s_sock < 0) {
            continue;
        }

        // 设置socket可重用
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(int));

        if (bind(listener, p->ai_addr, p->ai_addrlen) < 0) {
            close(s_sock);
            continue;
        }

        break;
    }

    // p = NULL则是没找到
    if (p == NULL) {
      printf("哇，很难受\n");
      exit(1);   //没找到可用的地址，没必要继续了
    }

    freeaddrinfo(ai);

    // 监听，参数10 表示等待连接的队列最大为10
    if (listen(listener, 10) == -1) {
      printf(”玩呢，兄弟，有出错了\n");
      exit(1);
    }

    // 吧s_sock装进集合，若其读就绪，表示可以接受新的连接了
    FD_SET(s_sock, &master);

    fdmax = s_sock;

    while(true) {
        read_fds = master; //把主集合中的fd全弄进去监听读事件
        if (select(fdmax+1, &read_fds, NULL, NULL, NULL) == -1) {
            printf("select\n");
            exit(1);
        }

        // 遍历整个集合，找到读就绪的fd
        for(i = 0; i <= fdmax; i++) {
            if (FD_ISSET(i, &read_fds)) {
                if (i == s_sock) {   //可以接受新的连接了

                    addrlen = sizeof client_addr;
                    c_sock = accept(s_sock,
                        (struct sockaddr *)&client_addr,
                        &addrlen);

                    if (c_sock == -1) {
                        printf("accept\n");
                    } else {
                        FD_SET(c_sock, &master); // 加入监听集合的豪华全家痛
                        if (c_sock > fdmax) {    // 更新max file descriptors number，用英文感觉就好多了-_-||
                            fdmax = c_sock;
                        }
                        printf("加入了一个新哥们:%s\n",c_sock);
                    }
                } else {
                    // 如果是客户端sock已经就绪了，则接受消息，并且将消息发送给集合中所有的客户端
                    if ((nbytes = recv(i, buf, sizeof buf, 0)) <= 0) {
                        // 出错或者连接中断了
                        if (nbytes == 0) {
                            // 远端sock已经关闭了
                            printf("走了一个\n");
                        } else {
                            printf("recv\n");
                        }
                        //善后
                        close(i);
                        FD_CLR(i, &master);
                    } else {
                        // 没什么问题就吧这消息发出去了
                        for(j = 0; j <= fdmax; j++) {
                            if (FD_ISSET(j, &master)) {
                                if (j != listener && j != i) {
                                    if (send(j, buf, nbytes, 0) == -1) {
                                        printf("send\n");
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    return 0;
}
```
上面的代码，逻辑是很简单的，代码也容易看懂，基本上了看一遍大概就知道怎么用select了，这里部队select做更加深入的研究，仅记录下其用法。
