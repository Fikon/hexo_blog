---
title: 今天不聊风花雪月，只谈谈TCP/IP
date: 2017-04-25 01:09:21
tags: TCP/IP
categories: notes
---
*这段时间面了几家砖厂，每家必问计算机网络，提到计算机网络必问TCP/IP。一些比较大而广的东西还是能够跟面试官谈笑风声的，但是一说到比较细节的东西就有点捉鸡了，毕竟是在吃上课时的记忆和平时积累的老本。所以啊，面试准备和没准备的差别是太TMD明显了。我是吃了个大亏的，四月中旬才回校，很无奈，面试接连碰壁。扯远了，写这篇关于TCP/IP的文章，一个时为了复习下其中的一些细节，再一个就是锻炼下表达能力，看看能不能把这玩意儿说的我以后忘了之后回来看一眼就可以大(yi)彻(zhi)大(ban)悟(jie)。*

一般介绍计算机网络都习惯于分层次来进行讲解，关于计算机网络的层次结构，目前有两种比较普遍的分层方法：
- 五层因特网协议栈：应用层，传输层，网络层，数据链路层和物理层
- 七层ISO OSI参考模型：应用层，表示层，会话层，传输层，网络层，数据链路层和物理层。

### TCP/IP——传输层/网络层
不管采用哪种分层模型，TCP都是处在传输层的而IP则处于网络层。网络层提供了主机之间的逻辑通信而传输层则提供了进程之间的逻辑通信。传输层（TCP）是只运行在端系统上的，底层的数据传输则交由网络层（IP）负责，TCP负责提供多路分解和多路复用功能，将数据交付到正确的进程那里或者将进程数据打包再交由网络层进行传输。课本上举了个非常形象的例子来分析他们之间的差别：
```
考虑有两个家庭，一个位于驻马店，一个位于葫芦岛，每家都有4个小孩，并且他们是比较亲密的亲戚，所以
需要经常进行通信。每个人每个星期都会相互写一封信，并且通过单独的信封进行邮递。其中每个家庭都有一
个负责管理家里通信的小孩，驻马店的是毛蛋哥，葫芦岛的大眼妹，他们负责从邮差手里接过当天的信件并分
发给各个兄弟姐妹或者是收集他们的信件交给邮差邮递到目的地。
```
<!-- more -->
邮政服务为两个家庭提供了逻辑通信，而毛蛋哥和大眼妹通过将信件分发给各个兄弟姐妹（多路分解）或者将他们的信件交付给邮差（多路复用）为两家通信的各个亲戚之间提供了逻辑通信。从这个例子可以看出来，传输层和网络层一样是非常重要的，各自分担不同的工作。

IP的服务模型提供的是尽力而为的交付服务，其尽最大努力在主机之间交付报文段，但是并不保证交付，不保证按序交付，也不保证完整交付，所以IP被成为不可靠服务。传输层是立足于网络层之上的一个服务模型，其最基本的功能就是将两个主机之间交付扩展为运行在主机上的两个进程之间的交付，即实现多路复用（将源主机上不同socket中的数据块进行封装，生成报文并且传递给网络层）以及多路分解（将网络层中不同的报文段中的数据交付给正确的socket）。
传输层的两个典型的代表是TCP和UDP，其中UDP之完成了基本的传输层协议要求的最少工作，因而依然是不可靠的；而TCP则完成了一些额外的工作来保证可靠传输。

### TCP与UDP对比
这一节主要对TCP与UDP之间的区别进行一波分析：
- 报文头部区别：[图片来源](https://nmap.org/book/tcpip-ref.html)

UDP头部
![UDP头部](https://nmap.org/book/images/hdr/MJB-UDP-Header-800x264.png)
UDP头部包括四个字段，源端口，目的端口，长度和校验和。长度表示真个UDP报文的长度（头部加上数据），UDP提供了简单的差错检测功能，校验和用来确定报文在传输过程中比特位有没有发生变化。

TCP头部
![TCP header](https://nmap.org/book/images/hdr/MJB-TCP-Header-800x564.png)
TCP头部包括源端口，目的端口，序列号，确认号，首部长度，TCPflag位，接收窗口，校验和，紧急数据指针和被选项。序列号和确认号以及TCPflag都是用来实现可靠数据传输服务的辅助字段，接受窗口则用于流量控制，首部长度用于说明首部长度（因为有被选项存在，所以TCP首部的长度时候可变的）。

TCP为了实现可靠传输，在基本的传输层协议工作基本上实现了许多额外的功能，从头部信息就可以看出来，TCP头部至少比UDP多12字节，因为TCP在传输过程中会多许多额外的开销。因此，一些对实时性要求较高，可以忍受包丢失的应用会采用UDP来实现。

- 在使用UDP时，发送方与接收方是不需要建立连接的，而TCP开始数据传输前需要进行三次握手。
- TCP实现了拥塞控制，流量控制等机制，会使得发送方的发送速率受到抑制以避免网络过度拥塞和接收方缓冲区溢出，而UDP则只要有数据就会将其打包进UDP报文段并且立即传送给网络层。
- UDP是无连接状态的，而TCP需要在端系统中维护连接状态，包括接收和发送缓存，拥塞控制参数以及序列号和确认号的参数。
- UDP是面向报文的，应用层交给多少报文，就一次发多少报文；而TCP是面向字节流的，将应用层的报文看成是无结构的字节流。

### 可靠的传输层协议——TCP

#### 面向连接的协议
TCP被称为是面向连接的传输协议，因为在发送方与接收方在传输数据之前需要进行三次握手以建立连接。当然，这个连接只是逻辑上的连接，实际上TCP仅仅运行在端系统中，中间的网络元素是不会保存TCP连接状态的。因此，该连接只是通信的两个端系统明确一些确保数据传输的参数罢了，其连接状态保存在两个端系统上。TCP提供的是全双工服务（full-duplex service），也就是说，如果两个进程存在一个TCP连接，则他们可以相互发送数据。数据量大小受限于最大报文段（Maximum Segment Size, MSS）。MSS通常受限於底层的最大帧长度，即最大传输单元（Maximum Transmission Unit, MTU）,毕竟要保证TCP数据加上头部信息可以塞进下一层的帧中。

既然是面向连接的协议，那就来聊聊建立连接以及断开连接的过程吧。先看一个从CoolShell上面偷来的对于TCP三次握手以及四次挥手的示意图
![](http://coolshell.cn//wp-content/uploads/2014/05/tcp_open_close.jpg)
建立连接——三次握手：

- 第一步：客户端向服务端发送一个请求建立连接的报文段，该报文段不包含应用层数据，但是首部中TCP flag里面的SYN位被置为1,并且会随机一个数赋值给头部中的序列号。
- 第二步：包含TCP SYN的报文段到达服务端之后，服务端为该TCP连接分配缓存和变量，并且向其发送允许连接的报文段。该报文段也不包括应用层数据，ACK和SYN被置为1,确认号被置为收到的请求的序列号+1,然后再设置自己的序列号。
- 第三步：在接收到服务端同意连接的报文段后，客户端也给该TCP连接分配缓存和变量，并且发送另外一个报文段。该报文段允许发送应用层数据，并且ACK置为1, 序列号设置为服务端发来的确认号的值，确认号为服务端序列号+1.

三次握手的必要性：关于这个有两种解释，1)为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误，造成服务器资源浪费。例如，客户端向服务端发起了一个请求，但是这个请求由于拥塞，未能及时传到服务端，所以其又发送了一个请求，建立了连接，在数据传输完关闭了连接之后，一开始超时的请求传到了服务端，若是只进行一次或者两次握手的话，这时，服务端与客户端之间的连接就已经建立了，服务端会一直等待客户端进行传输，保留TCP连接状态，造成资源浪费；2)第二种解释是由于信道不可靠, 但是通信双方需要建立一个可靠的传输机制. 而要解决这个问题，无论你在消息中包含什么信息, 三次通信是理论上的最小值. 所以三次握手不是TCP本身的要求, 而是为了满足"在不可靠信道上可靠地传输信息"这一需求所导致的.这里的本质需求,信道不可靠, 数据传输要可靠. 三次达到了, 那后面你想接着握手也好, 发数据也好, 跟进行可靠信息传输的需求就没关系了. 因此,如果信道是可靠的, 即无论什么时候发出消息, 对方一定能收到, 或者你不关心是否要保证对方收到你的消息, 那就能像UDP那样直接发送消息就可以了.


断开连接——四次挥手：首先，客户端向服务端发送一个结束连接的请求，这个请求将首部中TCP flag里的FIN置为1。当服务端收到后，给其发送一个确认报文，将ACK置为1。此时，客户端对服务端的发送端口关闭了（TCP是全双工了），而服务端仍可以向服务端发送数据，若服务端无数据进行发送了，可以向客户端发送一个终止报文段，将FIN置为1。最后客户端向服务端发送一个确认报文段进行确认，服务端收到确认报文后，释放用于该TCP连接的资源，而客户端在发送了最后一个ACK后，进入了time_wait状态，之后并释放用于该TCP的资源。

客户端为什么要进入time_wait状态：time_wait是客户端发送完最后一个ACK之后的一段持续2MSL（Maximum Segment Lifetime）的等待期。time_wait有两个目的，一个是为了保证在最后一个ACK丢失后，在收到服务端的FIN请求时可以重发ACK，保证全双工的TCP正常终止；另一个是保证过时的报文在网络中消逝，防止超时的报文被后续的连接（相同的IP加端口）错误解析。

#### TCP重传机制
TCP有两种重传机制，一个是超时重传，另一个是快速重传机制：
- 超时重传
```
超时长度如何设定?显然超时长度应该大于RTT（报文往返时间），而且还应该是动态确定的，因为网络的拥塞状况是会随着时间改变的。TCP才用了下面这种方法来对超时长度进行估计
EstimatedRTT = (1 - a) * EstimatedRTT + a * SampleRTT, EstimatedRTT表示预估的RTT，sampleRTT表示采样的RTT。
DevRTT = (1 - b) * DevRTT + b * |SampleRTT - EstimatedRTT|, DevRTT表示对上面预估的RTT与采样RTT的差的估算值。
最终，超时长度的估算值为 TimeoutInterval = EstimatedRTT + 4 * DevRTT
```
- 快速重传
```
超时重传的一个明显问题是超时周期可能会比较长，这样会使得端对端的时延增加。因此，可以通过在超时发生之前注意
冗余ACK来较好地检测丢包情况。冗余ACK指重复确认某个报文段的ACK。在收到冗余ACK时，说明已经有报文丢失了（因为
TCK不是用否定确定）。快速重传机制就是在受到连续三个冗余的ACK时，便开始重传丢失的报文段
```
- 如何重传——GBN or SR
```
GBN（Go Gack N）: 回退N步，指重传丢失的报文段之后发送的所有报文段
SR(Selected Repeat)：选择重传，只重段丢失的报文，这种机制需要额外的缓存来存储那些已经接收，但是不能组成有序系列的报文段。
举例来说两种机制：假设发送方发送了一组报文段1,2...N，其中n < N丢失了，其他的报文段均正确接收了，GBN会重传报文段n + 1, n + 2...N，而SR只重传n。
TCP采用累积确认的方式，即确认号N表示，N之前的所有报文段已经正常接收了，这点类似于GBN；而在重传的时候却又是仅仅重传丢失的报文段，因此，TCP更像是
GBN与SR的结合
```

#### TCP流量控制与拥塞控制
##### 流量控制
TCP连接的双方都为该连接设置了缓存，当TCP接收到正确，按序的字节后，将其放入缓存中，相关的应用进程从该缓存中读取数据。若应用进程读取的速率较慢而发送方发送的速率过快就会导致缓存区溢出。所以TCP提供了流量控制机制，以消除缓存区溢出的可能。从这个角度上说，流量控制是一个速度匹配服务，让发送方的发送速率不超过接收方的接收速率。流量控制机制是通过让发送方维护一个接收窗口（首部字段）来实现的，简单的说就是接收窗口就是告诉发送方接收方的缓存区还有多大空间。当接收窗口为0时，发送方并不会停止发送报文段，而是发送只有一个字节数据的报文段来进行探测，若缓存区有空间了，接收方会给发送方发送一个非0的接收窗口。
##### 拥塞控制
TCP提供了拥塞控制，以防止大量的报文涌入网络中造成网络过载，导致丢包。大致的思路是这样的，如果发送方感知到他从它到目的地之间的路径上并没有什么拥塞，则加大发送速率；若其感知到该路径上有拥塞，则降低其发送速率。
- TCP发送方如何限制其发送速率： 运行在发送方的TCP维护一个拥塞窗口（cwnd）,他对发送方能发送到网络中的流量进行了限制。在一个发送方中未被确认的数据量不能超过拥塞窗口和接收窗口中的最小值。
- TCP如何感知从其到目的地的路径上存在拥塞：1)一个丢失的报文段意味着拥塞（超时或连续三个冗余ACK）；2)一个确认报文段指示该网络正在向接收方交付发送方的报文段，因此，当对先前未确认的报文段的确认到达时，可以增加发送速率；
- 当发送方感知到拥塞时，如何调整其发送速率：

```
TCP拥塞控制算法：
1. 慢启动
  当一条TCP连接开始时cwnd被设置为1MSS，然后每当传输的报文段被首次确认就给cwnd加1,所以每个RTT，cwnd就会翻倍，呈指数增长。1)当到达慢启动阀值时，进入拥塞避免阶段；2)当发生由于超时指示的丢包时，将cwnd设置为1,并且将慢启动阀值设置为发生拥塞时cwnd的一半；3)当检测到三个冗余ACK时，TCP执行快速重传，并且进入快速恢复状态，即将慢启动阀值设置为当前cwnd的一半，然后将cwnd设置为当前值的一半，再加上3(三个ACK)，然后进入拥塞避免阶段。
2. 拥塞避免
   一旦进入拥塞避免阶段，cwnd大概是上次发生拥塞时的一半，所以不能再指数增长了。cwnd线性增加，每过一个RTT增加1MSS。1)若此时发生超时，跟慢启动一样的处理方式；2)接收到冗余ACK，处理亦跟慢启动一样
```
#### TCP服务端与客户端的生命周期
一张状态转移图，这个是客户端与服务端结合在一起了，看起来有点花里胡哨的，还是《计算机网络——自顶向下方法》里面的图比较简单明了，不过我找不到清晰的图，只能凑合用这个了。
![](http://coolshell.cn//wp-content/uploads/2014/05/tcpfsm.png)
看到这十二个状态的有限状态机就知道TCP的状态转移略复杂，不过仔细看的话，所有的过程前面都有提到过了，主要就是三次握手建立连接，四次挥手白了个白。（有一次有个面试官小哥就问了我这个图说什么情况下会出现close_wait啊，我一脸懵逼，压根记不住有个close_wait，于是跟他说了一下整个流程，不过是用我的话来描述的，因为记不住这些close_wait，time_wait了，最后这兄弟还是不太满意的样子，诶-_-||）

TCP/IP大概就这些东西了吧，写的时候参考了[库壳](http://coolshell.cn/articles/11564.html)和《计算机网络——自顶向下的方法》。
