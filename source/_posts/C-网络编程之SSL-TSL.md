---
title: C++网络编程之SSL/TSL
date: 2016-09-23 05:11:26
tags: [网络编程, 安全]
categories: notes
---

### SSL/TSL是啥
SSL是Secure Socket Layer的缩写，从名字便可以看出来这玩意儿是干嘛的了，就是给Socket加一层安全套，提供一种安全通信的机制。SSL是著名的网景（Netscape）公司发明的，后面标准化组织接手之后，便将SSL升级成了TSL（Transport Layer Security），所以发展到现在可以认为两者是一样的。网景初衷是为使用明文传输的HTTP协议提供安全传输服务，因为使用明文传输的话，会有三个比较明显的隐患：
- 窃听（或者叫嗅探，Sniffer）：第三方可以获取通信双方的内容。
- 篡改：第三方可以修改通信双方的内容，比如ISP在返回的页面中插入广告等。
- 冒充（域名欺骗，域名劫持等）：第三方伪装成通信的某一方，从而获取机密信息。

因此，假如通信双方需要传输重要信息时，一种安全的通信机制是很有必要的。HTTP与SSL/TSL两者的结合便是HTTPS（Hyper Text Transfer Protocol over Secure Socket Layer），就是在应用层（HTTP）与传输层之间增加了一个安全层，用于建立安全的通信机制。举一个通俗的例子来说明：
```
当年上本科的时候，经常一起上下课的有两个潮汕的同学，飞宾和逸爷。说到潮汕同学，估计都能猜到我要
说什么了，没错他们两个只要碰到一起必用潮汕话交流，经常抛下一句我们要去火星啦，就开始BLABLA起
他们才能听得懂的潮汕话了，作为旁观者，我们只能众脸懵逼。他们如果用大家都听得懂的普通话（明文）
的话，那么他们聊什么，我们都是知道的，但是这样就不能当着我们面骂我们傻逼了，所以他们用了一种
他们才可以听得懂的潮汕话来交流，就相当于才用了一种加密机制，只有他们才可以破解说话的内容，我们
虽然听得到，但是并不能听懂，所以他们之间的交流内容是很安全的。
```

### SSL/TSL工作流程
SSL/TSL加密的基本思路是采用公钥加密法，大致的流程是这样的：
- 客户端向服务器端索要并验证公钥。
- 双方协商生成"对话密钥"。
- 双方采用"对话密钥"进行加密通信。

<!-- more -->

前面两步就是SSL/TSL的握手过程了，主要是验证身份，选择加密方式和密钥交换，具体的步骤如下：
```
1. ClientHello: 客户端向服务端发送TLS的版本，随机数（这个是用来生成最后加密密钥的影响因子之一，包含两部分：时间戳和随机数），session-id，加密算法套装列表（客户端支持的加密-签名算法的列表，让服务器根据自己支持的加密算法去选择一个），压缩算法和扩展字段（比如密码交换算法的参数、请求主机的名字等）
2. ServerHello: 结构跟客户端差不多，只不过跟客户端不同的是，服务端是在客户端提供的加密算法列表中选择一个进行加密，
3. ServerCertificate: 服务端向客户端展示自己的证书（证书相当于某个权威机构给服务端的一个证明，比如去派出所开的证明我是我的材料）。此时服务端也可以要求客户端提供相应的证书来证明自己(ClientSertificate)，这是个可选项。
4. ServerKey Exchange: 服务端发送加密算法的相关参数，根据选择算法的不同，参数也会不一样
5. ServerHello Done： 服务端信息传输完了
5. ClientKey Exchange: 客户端响应服务端发送的加密算法相关参数
6. ChangeCipher Spec: 客户端切换到密文模式
7. Finished：传输结束
```
从上面的流程就可以看出SSL/TSL有多麻烦，在TCP握手的基础上还要实现多次握手以及数据的加密解密，所以数据传输的开销是比较大的。一般是比较重要的数据采用加密传输，那些泄露了也没所谓的数据采用普通的HTTP进行传输就好了，这样安全性和用户体验都可以得到较好的保证。

### 使用OpenSSL实现安全传输
本站的博文向来都是以代码为导向的，一切的文字说明都是为了为代码做铺垫，解释代码-_^。
#### 安装
OpenSSL在Linux下安装很方便，两条命令解决了
- `sudo apt-get install openssl`安装OpenSSL的一些命令行工具
- `sudo apt-get install libssl-dev`开发用的

#### 步骤
主要介绍OpenSSL API的调用流程
##### 服务端
- `SSL_library_init()`初始化SSL库
- `OpenSSL_add_all_algorithms`载入SSL算法
- `SSL_load_error_strings()`载入错误消息
- `SSL_CTX_new`生成一个SSL上下文变量
- `SSL_CTX_use_PrivateKey_file`载入用户私钥
- `SSL_CTX_check_private_key`检查用户私钥
- `socket(), bind(), accept`服务端socket调用的步骤
- `SSL_new`生成一个新的SSL
- `SSL_set_fd`将Socket设置采用SSL传输
- `SSL_accept`建立SSL连接
- `SSL_read, SSL_write`通过SSL读写
- `SSL_shutdown`关闭SSL连接
- `SSL_free`释放SSL
- `close(socket)`关闭socket

#####  客户端
- `SSL_library_init()`初始化SSL库
- `OpenSSL_add_all_algorithms`载入所有的SSL算法
- `SSL_load_error_strings`载入所有的SSL错误消息
- `SSL_CTX_new`生成一个上下文变量
- `socket, connect`客户端socket的套路
- `SSL_new`产生一个新的SSL
- `SSL_set_fd`将socket设置为采用SSL传输
- `SSL_connect`建立SSL连接
- `SSL_read, SSL_write`SSL数据传输
- `SSL_shutdown`关闭SSL连接
- `SSL_free`释放SSL
- `close`关闭socket
- `SSL_CTX_free`释放上下文变量，前面介绍的服务端没有这一步，因为服务端还有处理其他的连接请求，所以需要继续使用上下文变量

看过前面的背景介绍之后，其实读这些代码还是很轻松的，并且客户端还服务端很多操作基本上是一致的，主要的差异在Socket和SSL的建立（SSL_accept or SSL_connect）。在这些了解了这些大框架之后，可以很方便按照自己的思路来进行代码编写了。

#### Echo服务器代码实例
哇，很皮，又是Echo。
##### 服务端代码
```
/*
** light http server which is able to deal with https request
** kiky.jiang@gmail.com
*/


#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/wait.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <netdb.h>
#include <ctype.h>

#define PORT "4399"
#define BACKLOG 20

int socket_init(){
  int s_sock = 0;
  int res;
  int reuse = 1;
  struct addrinfo hints, *ai, *p;
  //传入一些我们期望的属性的地址链表
  bzero(&hints, sizeof hints);
  hints.ai_family = AF_UNSPEC;
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_flags = AI_PASSIVE;

  if(getaddrinfo(NULL, PORT, &hints, &ai) != 0){
    printf("error: fail to call getaddrinfo\n");
    return -1;
  }
  //遍历返回的地址链表，找一个能用的
  for(p = ai; p != NULL; p = p -> ai_next){
    if((s_sock = socket(p -> ai_family, p -> ai_socktype, p -> ai_protocol)) == -1){
      continue;
    }
    //将socket设置为可重用模式
    if(setsockopt(s_sock, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(int)) == -1){
      continue;
    }
    if(bind(s_sock, p -> ai_addr, p -> ai_addrlen) == -1){
      continue;
    }
    break;
  }
  freeaddrinfo(ai);
  if(p == NULL){
    printf("error: can not initialize address for server\n");
    return -1;
  }
  if(listen(s_sock, BACKLOG) == -1){
    printf("error: listen\n");
    return -1;
  }
  return s_sock;
}

int main(int argc, char *argv[]){

  int s_sock = socket_init();
  //SSL上下文变量
  SSL_CTX *ctx;

  //SSL库初始化
  SSL_library_init();

  //载入SSL算法
  OpenSSL_add_all_algorithms();

  //载入错误消息
  SSL_load_error_strings();

  //生成一个SSL_CTX变量，可以采用V2或者V3,或者是V2,V3兼容
  ctx = SSL_CTX_new(SSLv23_server_method());
  /* 可以用SSLv2_server_method()或者SSLv3_server_method代替 */

  if(ctx == NULL){
    ERR_print_errors_fp(stdout);
    exit(1);
  }

  /* 载入数字证书发送给客户端，证书中包含公钥, 公钥和私钥用命令行参数传入，需要提前生成 */
  if(SSL_CTX_use_certificate_file(ctx, argv[1], SSL_FILETYPE_PEM) <= 0){
    ERR_print_errors_fp(stdout);
    exit(1);
  }

  /* 私钥 */
  if(SSL_CTX_use_PrivateKey_file(ctx, argv[2], SSL_FILETYPE_PEM) <= 0){
    ERR_print_errors_fp(stdout);
    exit(1);
  }

  //检查私钥正确性
  if(!SSL_CTX_check_private_key(ctx)){
    ERR_print_errors_fp(stdout);
    exit(1);
  }
  struct sockaddr_storage client_addr;
  socklen_t client_addrlen;
  int c_sock;
  char buff[2048];
  int res = 0;
  while(1){
    SSL *ssl;
    if((c_sock = accept(s_sock, (struct sockaddr *)&client_addr, &client_addrlen)) == -1){
      perror("accept");
      exit(1);
    }
    printf("Hey, get a connection, socket: %d\n", c_sock);
    ssl = SSL_new(ctx);
    SSL_set_fd(ssl, c_sock);
    if(SSL_accept(ssl) == -1){
      perror("SSL_accept");
      close(c_sock);
      exit(1);
    }
    bzero(buff, 2048);
    res = SSL_read(ssl ,buff, 2047);
    if(res > 0){
      printf("Got a message: %s\n", buff);
      res = SSL_write(ssl, buff, strlen(buff));
      if(res <= 0){
        printf("Failed to echo\n");
        exit(1);
      }
    }
    else{
      printf("Failed to receive msg\n");
      exit(1);
    }

    //善后
    SSL_shutdown(ssl);

    SSL_free(ssl);

    close(c_sock);
  }
  close(s_sock);

  SSL_CTX_free(ctx);
}

```
##### 客户端代码
```
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <sys/socket.h>
#include <resolv.h>
#include <stdlib.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

#define MAXBUF 1024
/* 命令行传入 IP PORT MSG */
int main(int argc, char * *argv)
{
    int sockfd, len;
    struct sockaddr_in dest;
    char buffer[MAXBUF + 1];
    SSL_CTX * ctx;
    SSL * ssl;

    /* SSL 库初始化*/
    SSL_library_init();
    /* 载入所有SSL 算法*/
    OpenSSL_add_all_algorithms();
    /* 载入所有SSL 错误消息*/
    SSL_load_error_strings();
    /* 以SSL V2 和V3 标准兼容方式产生一个SSL_CTX ，即SSL Content Text */
    ctx = SSL_CTX_new(SSLv23_client_method());
    if (ctx == NULL)
    {
        ERR_print_errors_fp(stdout);
        exit(1);
    }
    /* 创建一个socket 用于tcp 通信*/
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        perror("Socket");
        exit(errno);
    }
    printf("socket created\n");
    /* 初始化服务器端（对方）的地址和端口信息*/
    bzero( &dest, sizeof(dest));
    dest.sin_family = AF_INET;
    //设置连接的端口
    dest.sin_port = htons(atoi(argv[2]));
    //设置连接的IP地址
    if (inet_aton(argv[1], (struct in_addr * ) &dest.sin_addr.s_addr) == 0)
    {
        perror(argv[1]);
        exit(errno);
    }
    printf("address created\n");
    /* 连接服务器*/
    if (connect(sockfd, (struct sockaddr * ) &dest, sizeof(dest)) != 0)
    {
        perror("Connect ");
        exit(errno);
    }
    printf("server connected\n");
    /* 基于ctx 产生一个新的SSL */
    ssl = SSL_new(ctx);
    /* 将新连接的socket 加入到SSL */
    SSL_set_fd(ssl, sockfd);
    /* 建立SSL 连接*/
    if (SSL_connect(ssl) == -1)
    {
        ERR_print_errors_fp(stderr);
    }
    bzero(buffer, MAXBUF + 1);
    strcpy(buffer, argv[3]);
    /* 发消息给服务器*/
    len = SSL_write(ssl, buffer, strlen(buffer));
    if (len < 0)
    {
        printf("消息'%s'发送失败！错误代码是%d，错误信息是'%s'\n", buffer, errno, strerror(errno));
    }
    else
    {
        printf("消息'%s'发送成功，共发送了%d 个字节！\n", buffer, len);
    }

    /* 接收对方发过来的消息，最多接收MAXBUF 个字节*/
    bzero(buffer, MAXBUF + 1);
    /* 接收服务器来的消息*/
    len = SSL_read(ssl, buffer, MAXBUF);
    if (len > 0)
    {
        printf("接收消息成功:'%s'，共%d 个字节的数据\n", buffer, len);
    }
    else
    {
        printf("消息接收失败！错误代码是%d，错误信息是'%s'\n", errno, strerror(errno));
    }
    /* 关闭连接*/
    SSL_shutdown(ssl);
    SSL_free(ssl);
    close(sockfd);
    SSL_CTX_free(ctx);
}

```
这么一个简单的Echo服务器和测试用的客户端就算是完成了，下一步再把我的EasyHTTPD给改造成支持HTTPS的吧，坑真多啊。
