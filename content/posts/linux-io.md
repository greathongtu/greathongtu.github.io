+++
title = 'Linux 网络I/O复用并发模型'
date = 2023-11-17T01:04:58+08:00
draft = false
tags = ['linux', 'io']
+++

# 阻塞非阻塞
阻塞占用资源少，但是一个线程通过阻塞IO，只能处理一个请求。
非阻塞需要一直轮询，占用CPU资源大。

* 解决方法一：阻塞+多线程/多进程。 浪费资源
* 解决方法二：非阻塞忙轮询
```cpp
while(true) {
    for i in stream[] {
        if(i has data) {
            read/ other process
        }
    }
}
```
* 方法三：IO多路复用：既能阻塞等待，不浪费CPU资源，也能同一时刻监听多个IO请求的状态
select:
```cpp
while(true) {
    select(stream[]); // 阻塞 max 1024个
    for i in stream[] {
        if(i has data) {
            read/ other process
        }
    }
}
```
epoll:
```cpp
while(true) {
    可处理的流 = epoll_wait(epoll_fd);// 阻塞 max `cat /proc/sys/fs/file-max` 个
    for i in 可处理的流[] {
        read/ other process
    }
}
```

## select/ poll
select 实现多路复用的方式是，将已连接的 Socket 都放到一个文件描述符集合，然后调用 select 函数将文件描述符集合拷贝到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。

所以，对于 select 这种方式，需要进行 2 次「遍历」文件描述符集合，一次是在内核态里，一个次是在用户态里 ，而且还会发生 2 次「拷贝」文件描述符集合，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中。

## epoll
第一点，epoll 在内核里使用红黑树来跟踪进程所有待检测的文件描述字，把需要监控的 socket 通过 epoll_ctl() 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 O(logn)。而 select/poll 内核里没有类似 epoll 红黑树这种保存所有待检测的 socket 的数据结构，所以 select/poll 每次操作时都传入整个 socket 集合给内核，而 epoll 因为在内核维护了红黑树，可以保存所有待检测的 socket ，所以只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。

第二点， epoll 使用事件驱动的机制，内核里维护了一个链表来记录就绪事件，当某个 socket 有事件发生时，通过回调函数内核会将其加入到这个就绪事件列表中，当用户调用 epoll_wait() 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率。

## epoll API

1. 创建epoll，在内核创建一颗红黑树
```cpp
// @param size 告诉内核监听的数目
// @returns 返回一个epoll句柄（fd）
int epoll_create(int size);
```

2. 控制epoll，在红黑树上crud
```cpp
// @param op: EPOLL_CTL_ADD; EPOLL_CTL_MOD; EPOLL_CTL_DEL
// @param event 常见的有EPOLLIN; EPOLLOUT
// @returns 成功返回0，失败-1，errno查看错误信息
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// example：
struct epoll_event new_event;
new_event.events = EPOLLIN | EPOLLOUT;
new_event.data.fd = 5;
epoll_ctl(epfd, EPOLL_CTL_ADD, 5, &new_event);
```

3. 等待epoll，触发阻塞 
```cpp
// @param event 从内核得到的事件集合
// @param maxevents 告知内核这个events有多大
int epoll_wait(int epfd, struct epoll_event *event, int maxevents, int timeout);

// example:
struct epoll_event my_event[1000];
int event_cnt = epoll_wait(epfd, my_event, 1000, -1);
```

epoll 编程框架
```cpp
int epfd = epoll_create(1000);

epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &listen_event);

while(1) {
    int active_cnt = epoll_wait(epfd, events, 1000, -1);
    for(int i=0; i<active_cnt; i++) {
        if(events[i].data.fd == listen_fd) {
            // accept 三次握手。并且将新accept的fd 加进epoll
        } else if(events[i].events & EPOLLIN) {
            // 对此fd 进行读操作
        } else if(events[i].events & EPOLLOUT) {
            // 对此fd 进行写操作
        } 
    }
}
```

### 水平触发LT边缘触发ET
LT：只要用户不处理就一直把这些事件返回，涉及到内核态到用户态的拷贝，上下文的转换，稳定但消耗大。
ET：只把事件返回一次，性能高。
类似TCP/UDP 可靠重传

# 常见多路IO复用并发模型

## 模型一 单线程Accept

Server:
1. create ListenFd
2. Bind + Listen 
3. Accept(ListenFd) // 这里会阻塞
4. Read + Write

Client:
1. Connect
2. Read + Write

缺点：其他client会阻塞。非并发。

## 模型二 单线程Accept + 多线程读写业务（无I/O复用）

Server：
Accept 之后创建新thread进行读写，主线程只Accept

缺点：要的线程太多，也会增加CPU切换成本；对于长连接，客户端只要不关闭server就要保持这个链接的状态，占用连接资源和线程的开销。

## 模型三 单线程多路I/O复用

Server:
1. create ListenFd
2. Bind + Listen 
3. 多路I/O复用：创建 ListenFd 并监听。有Client1 Connect 请求，检测到ListenFd触发读事件，则进行Accept 建立连接，并将新生成的connFd1加入到监听I/O集合中
4. 读写的时候不能及时相应新Client的连接请求。

并发少的时候，消息延迟要求不高，可以采用。

## 模型四 单线程多路I/O复用 + 多线程业务工作池

读取数据交给worker pool工作线程池，里面的的线程只处理消息业务，不进行socket读写操作。
工作池处理完业务，触发connFd写事件，将回执客户端的消息通过main thread写给对方。
实际上读写的业务并发为1，但是业务流程的并发为worker pool 线程数量。
缺点：读写依然是main thread单独处理，最高的读写并行通道依然为1；返回给Client依旧需要排队。

## 模型五 单线程多路I/O复用 + 多线程多路I/O复用（线程池）

main thread 只用来监听ListenFd 并 Accept，线程池负责ConnFds

缺点：多个身处同一个Thread的客户端会出现读写延迟现象。

## 模型五 单线程多路I/O复用 + 多进程多路I/O复用（进程池）

多进程抢占处理ListenFd事件，进程池中的某个进程进行Accept，然后ConnFd

多进程内存资源空间占用稍微大一些

# 模型六 单线程多路I/O复用 + 多线程多路I/O复用 + 多线程

线程池中的线程还会继续 new thread。
过于理想化，需要很多核


模型五是最合适的
