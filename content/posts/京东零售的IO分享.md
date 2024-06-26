---
title: "剖析京东零售分享的伪代码———挖掘 IO 模型"
date: 2024-03-23T15:23:30+08:00
draft: false
weight: 

summary: "精读优秀博文，品味深度内涵"

# 文章和主页的图片
featuredImage: "/images/run.jpg"
# 主页的图片，优先级高于上面的
featuredImagePreview: ""

tags: [他山之石]
categories: []
---
> 之前学 IO 模型总是觉得太过抽象，网上的众多博客也是泛泛而谈，偶然搜索微信公众号的 IO 系列，竟然发现了一篇宝藏博文，特来和大家一起分享。同时有疑问也可以直接评论区留言探讨~
原文链接：[京东零售技术：IO模型介绍](https://mp.weixin.qq.com/s/pH767QK6wQOZBEbSyuw6xQ)


京东零售的博文中总共出现过四个伪代码：**BIO**，**NIO**，IO多路复用的 **select** 和 **epoll**

### BIO
首先来看看 BIO 的伪代码
```java
listenfd = socket();   // 打开一个网络通信套接字
bind(listenfd);        // 绑定
listen(listenfd);      // 监听
while(true) {
  buf = new buf[1024]; // 读取数据容器
  connfd = accept(listenfd);  // 阻塞 等待建立连接
  int n = read(connfd, buf);  // 阻塞 读数据
  doSomeThing(buf);  // 处理数据
  close(connfd);     // 关闭连接
}
```

 每当用户态来 socket 建立连接后(accept)，就直接调用 read 来阻塞读取数据了，此时其他用户态的 socket 来建立连接就卡住了，因为是单线程的。所以内核态真的就是完全串行接收请求了，通过阻塞达到串行的效果了，全由 **“主线程”** 来包揽建立连接和 read 读取数据

### NIO
接下来看看 NIO 的伪代码
```java
arr = new Arr[];
listenfd = socket();   // 打开一个网络通信套接字
bind(listenfd);        // 绑定
listen(listenfd);      // 监听
while(true) {
  connfd = accept(listenfd);  // 阻塞 等待建立连接
  arr.add(connfd);
}

// 异步线程检测 连接是否可读
new Tread(){
  for(connfd : arr){
    buf = new buf[1024]; // 读取数据容器
    // 非阻塞 read 最重要的是提供了我们在一个线程内管理多个文件描述符的能力
    int n = read(connfd, buf);  // 检测 connfd 是否可读
    if(n != -1){
       newThreadDeal(buf);   // 创建新线程处理
       close(connfd);        // 关闭连接 
       arr.remove(connfd);   // 移除已处理的连接
    }
  }
}

newTheadDeal(buf){
  doSomeThing(buf);  // 处理数据
}
```

说下改变的点：
- read 这个动作没有放在 **“主线程”** 中，主线程就负责建立连接，然后把建立连接后产生的 FD 我先放入`Arr[]`中，（这里暂时看作一个普通的数组就行，后续 select 的具体实现就是 BitsMap 实现的数组）把 read 这个操作解耦，不放在“主线程”中，而是通过后面的`new Thread()`来执行 read
- 后面的线程也就是遍历 `Arr[]`，拿到一个不管这个 FD 需要的资源有没有，read 就完事（read 有两阶段：等待读就绪：这里非阻塞，读数据：这里阻塞），由于第一阶段非阻塞，发现没有就绪就继续遍历`Arr[]`，同样也是对新一个 FD 进行 read，如果这个 FD 需要的资源就绪，就能阻塞执行第二阶段了，读取结束后还要移除这个 FD，然后继续循环拿出 FD 执行 read...

**总结下**：
- 主线程 —— >  主线程 + 异步处理线程
- read 两阶段全阻塞 ——> read 等待读就绪非阻塞，读数据阻塞
- 来个用户态连接就一条龙处理 ——> 来用户态连接先放入容器中，主线程继续处理其余连接，异步线程遍历容器处理 read 操作

> 注意区分异步和非阻塞的区别：异步线程是实现非阻塞的必要条件，必须拿出一个线程来轮询容器才能实现非阻塞 read，你一个线程既要阻塞建立连接又要轮询遍历容器，不可能的
同时异步线程只是起到解耦的作用，核心是 read 的第一阶段非阻塞。试想要是 read 两阶段全阻塞，你另外开个线程也没多大效果，运气好第一个遍历的就就绪然后完成，运气不好还得阻塞等他就绪，和一个线程没有两样

### IO 多路复用中的 select
接下来看看 IO 多路复用的 select 伪代码
```java
arr = new Arr[];
listenfd = socket();   // 打开一个网络通信套接字
bind(listenfd);        // 绑定
listen(listenfd);      // 监听
while(true) {
  connfd = accept(listenfd);  // 阻塞 等待建立连接
  arr.add(connfd);
}
```

话说怎么和 NIO 长得一样？就是少了异步线程轮询处理逻辑。
这是因为 select 是操作系统提供的系统函数，通过它我们可以将文件描述符发送给系统，让系统内核帮我们遍历检测是否可读，并告诉我们进行读取数据。而**具体实现的伪代码原作者没有给出**，但是也没有关系，能确定的就是IO 多路复用的异步线程肯定就不是轮询容器了，而是**阻塞监听**。

下面的两幅图都是 异步线程 做的事，可以对比看看不同：select 阻塞监听所有 FD 需要的资源，当资源就绪就能通过 事件通知机制 唤醒异步线程，然后就能处理普通（第一阶段阻塞还是非阻塞问题不大，因为资源此时肯定是就绪的） read 操作，完成后移除该 FD 后就能继续循环监听

![NIO](/images/image2-1.png "NIO")

![select](/images/image2-2.png "select")


### IO 多路复用中的 epoll
最后是 epoll 的伪代码
```java
listenfd = socket();   // 打开一个网络通信套接字
bind(listenfd);        // 绑定
listen(listenfd);      // 监听
int epfd = epoll_create(...); // 创建 epoll 对象
while(1) {
  connfd = accept(listenfd);  // 阻塞 等待建立连接
  epoll_ctl(connfd, ...);  // 将新连接加入到 epoll 对象
}

// 异步线程检测 通过 epoll_wait 阻塞获取可读的套接字
new Tread(){
  while(arr = epoll_wait()){
    for(connfd : arr){
        // 仅返回可读套接字
        newTheadDeal(connfd);
    }
  }
}

newTheadDeal(connfd){
    buf = new buf[1024]; // 读取数据容器
    int n = read(connfd, buf);  // 阻塞读取数据
    doSomeThing(buf);  // 处理数据
    close(connfd);        // 关闭连接 
}
```

- 和 select 不同的是主线程建立连接后，把新连接的直接加入到 epoll 对象中（里面的红黑树），而非`Arr[]`
- 这里又点明了异步线程，**复习一下**：异步线程中 NIO 是遍历容器，通过 read 无阻塞轮询是否有就绪资源；select 异步线程没有点明，但是能够知道不是遍历容器了，而是监听资源，当有资源就绪就去执行 read 操作（第一步阻塞还是非阻塞不重要）；而在 epoll 中则是通过循环调用 `epoll_wait()`，当 `epoll_wait()`返回 true 的时候才会遍历容器（这个容器不是 epoll 对象中的红黑树，而装有已经资源就绪的 FD 的容器），遍历容器只返回可读套接字（这里还是阻塞去读取）

这里就涉及**两个问题**：
1.  什么时候`epoll_wait()`返回 true，阻塞还是非阻塞的？是什么回调通知可以让他返回 true 吗？是不是根本不会返回false而是阻塞和返回 true 两个状态？
2. 遍历容器的时候什么叫做“仅返回可读套接字”？意思是可写的不行？
为了弄清上述两个疑问，我们看看原博主对`epoll_wait()`函数的系统底层逻辑图示：

![epoll_wait()](/images/image2-3.png)

“用于监听套接字事件，可以通过设置超时时间timeout来控制监听的行为为阻塞模式还是超时模式”，搭配上内核空间中的 while(1) 循环的目的：“检测就绪队列是否有事件，有就读取就绪队列并返回，没有则阻塞等待”可以得知**第一个问题的全部答案**：当就绪队列中有数据的时候才返回 true，不是什么回调通知让他返回，回调通知起到一个间接的作用（epoll 对象检测到 “数据可读或可写准备就绪的回调通知” 的时候就会往就绪队列放入这个就绪的资源，这时候就绪队列就有数据了，`epoll_wait`来读取就绪队列就能返回成功了），并且是阻塞的，因为成功就读取然后返回，失败阻塞等待。

至于第二个问题就迎刃而解了，因为要是`epoll_wait`返回 true 的话，就说明返回的arr里全是刚刚扫荡 epoll 中就绪队列里就绪的资源，先回忆一下前面几个 IO 模型中arr里都是什么妖魔鬼怪：NIO 和 select 里的`arr`都是连接后的产物`connfd`，他们的资源就没就绪都是未知的，而这里全是就绪的资源。下面的遍历都很有价值，都是精准服务（也就是轮询都是有效的，而以前轮询效率极低）。

还有“仅返回可读套接字”并不是上面疑问中说的可写的不行，为了解答这个问题，我们**先看看伪代码**中`newTheadDeal(connfd)`中做的事：注意到`buf[1024]`没有？也即是说每次epoll_wait()成功后执行一次read都只能最多读满 buf 就完了，这就导致要是传来就绪队列中的对应 FD 需要的资源大于 1024 字节的时候，你在这次 read 是搞不完的，只能读取到最多 1024 字节。这就叫“仅返回可读套接字”，而不是读和写的问题。

### 事件通知机制
好的这里又牵扯出 **最多读 1024 字节那剩余的怎么处理?** 就这么错过了？毕竟 for 循环中遍历的不止你一个 FD，还有其他资源也就绪的 FD 要执行 read。
不卖关子了。这就是 epoll 中的**事件通知机制**：`epoll_wait`返回 true 后下次碰到残余的因为 buf 限制没有 read 完的怎么处理。
1. LT：水平触发。意思就是碰到就绪的资源是之前同一个 FD 请求的也给你返回（类似消息重发机制）（怎么判断是不是残余或者怎么判断 read 操作有没有读完？这个我估计应该是读取完毕会有一个标记位给 epoll，或者没有读取到资源结尾标记符就不会发送类似 ack 这样的标记）。好处是保证 FD 需要的资源能够全部读取完毕。坏处就是要是每次都要带上重复的资源也是一种浪费
2. ET：边缘触发。意思刚好相反，碰到就绪的资源是之前同一个 FD 请求的我也不放进就绪队列中，反正我之前都已经放过了，read 自己没处理完与我无关。好处是确实减少了很多类似重发就绪资源的动作，缺点是既然只能一次返回，就需要循环读取返回回来的就绪的资源了，而不是单单靠着 `buf[1024]`一板斧打天下
这里给出其他博主更好理解的栗子

![事件通知机制](/images/image2-4.png)
