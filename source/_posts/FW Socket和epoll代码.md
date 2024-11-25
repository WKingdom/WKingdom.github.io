---
title: Unix Socket和epoll代码
date: 2023-9-17 23:56:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

#### UNIX  SOCKET 

UNIX Domain SOCKET 是在Socket架构上发展起来的用于同一台主机的进程间通讯（IPC）。它不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序列号应答等。只是将应用层数据从一个进程拷贝到另一个进程。UNIX Domain SOCKET有SOKCET_DGRAM和SOCKET_STREAM两种模式，类似于UDP和TCP ，但是面向消 息的UNIX socket也是可靠的，消息既不会丢失也不会顺序错乱。

socket方法：

```c++
int socket(int protofamily, int type, int protocol);
```

protofamily：即协议域，又称为协议族（family）。Unix Socket通信情况下选择AF_UNIX，协议族决定了socket的地址类型，在通信中必须采用对应的地址，如 AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。 

type：指定socket类型。常用的socket类型有，SOCK_STREAM（TCP）、SOCK_DGRAM（UDP）。

```C++
int bind(int sockfd, const struct sockaddr addr, socklen_t addrlen);
```

sockfd：即socket描述字，它是通过socket()函数创建了，唯一标识一个socket。bind()函数就是将给这个描述字绑定一个名字。

addr：一个const struct sockaddr *指针， 指向要绑定给sockfd的协议地址。这个地址结构根据地址创建socket时的地址协议族的不同而不同。

addrlen：对应的是地址的长度。

Socket的通信过程方法调用 

服务端： socket -> bind -> listen -> accept -> read/write -> close 

客户端： socket -> connect -> read/write -> close

#### epoll主要方法

```C++
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

epoll_create：创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽

```C++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

epfd：是epoll_create()的返回值，

op：表示动作，用三个宏来表示：

- EPOLL_CTL_ADD：注册新的fd到epfd中；
- EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
- EPOLL_CTL_DEL：从epfd中删除一个fd；

event：是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

```c++
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```

events可以是以下几个宏的集合：

- EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
- EPOLLOUT：表示对应的文件描述符可以写；
- EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
- EPOLLERR：表示对应的文件描述符发生错误；
- EPOLLHUP：表示对应的文件描述符被挂断；
- EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
- EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

```C++
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

等待事件的产生，参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

**epoll工作模式**

LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下

- LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

- ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。


#### socket和epoll demo

服务端

```C++
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>
#include <ctype.h>
#include <sys/epoll.h>
#define MAXLINE 80

char *socket_path = "server-socket";

int main()
{
    struct sockaddr_un serun, cliun;
    socklen_t cliun_len;
    int listenfd, connfd, size;
    char buf[MAXLINE];
    int i, n;

    if ((listenfd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0) {
        perror("socket error");
        exit(1);
    }

    memset(&serun, 0, sizeof(serun));
    serun.sun_family = AF_UNIX;
    strncpy(serun.sun_path,socket_path ,
                   sizeof(serun.sun_path) - 1);
    unlink(socket_path);
    if (bind(listenfd, (struct sockaddr *)&serun, sizeof(struct sockaddr_un)) < 0) {
        perror("bind error");
        exit(1);
    }
    printf("UNIX domain socket bound\n");

    if (listen(listenfd, 20) < 0) {
        perror("listen error");
        exit(1);
    }
    printf("Accepting connections ...\n");
     // 4. 创建epoll树
    int epfd = epoll_create(1000);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(-1);
    }
    //5、将用于监听的lfd挂的epoll树上（红黑树）
    struct epoll_event ev;//这个结构体记录了检测什么文件描述符的什么事件
    ev.events = EPOLLIN;
    ev.data.fd = listenfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);//ev里面记录了检测lfd的什么事件
    // 循环检测 委托内核去处理
    struct epoll_event events[1024];//当内核检测到事件到来时会将事件写到这个结构体数组里


    while(1) {
        int num = epoll_wait(epfd, events, sizeof(events)/sizeof(events[0]), -1);//最后一个参数                                                                                                                                              表示阻塞
        for (int i = 0;i<num;i++) {
            if(events[i].data.fd == listenfd)//有连接请求到来
            {

                int len = sizeof(cliun);
                int connfd = accept(listenfd, (struct sockaddr *)&cliun, &len);
                if(connfd == -1)
                {
                    perror("accept");
                    exit(-1);
                }
                printf("a new client connected! \n");
                //将用于通信的文件描述符挂到epoll树上
                ev.data.fd = connfd;
                ev.events = EPOLLIN;
                epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);
            } else {
                 //通信也有可能是写事件
                if(events[i].events & EPOLLOUT)
                {
                    //这里先忽略写事件
                    continue;
                }
                char buf[1024]={0};
                int count = read(events[i].data.fd, buf, sizeof(buf));
                if(count == 0)//客户端关闭了连接
                {
                    printf("客户端关闭了连接。。。。\n");
                    //将对应的文件描述符从epoll树上取下
                    close(events[i].data.fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                }
                else
                {
                    if(count == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    else
                    {
                        //正常通信
                        printf("client say: %s\n" ,buf);
                        write(events[i].data.fd, buf, strlen(buf)+1);
                    }
                }

            }

        }
    }
    close(listenfd);
    return 0;
}

```

客户端

```c++
#include <stdlib.h>
#include <stdio.h>
#include <stddef.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#define MAXLINE 80

char *client_path = "client-socket";
char *server_path = "server-socket";

int main() {
        struct  sockaddr_un cliun, serun;
        int len;
        char buf[100];
        int sockfd, n;

        if ((sockfd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0){
                perror("client socket error");
                exit(1);
        }
//    signal(SIGPIPE, SIG_IGN);
    memset(&serun, 0, sizeof(serun));
    serun.sun_family = AF_UNIX;
    strncpy(serun.sun_path,server_path ,
                   sizeof(serun.sun_path) - 1);
    if (connect(sockfd, (struct sockaddr *)&serun, sizeof(struct sockaddr_un)) < 0){
        perror("connect error");
        exit(1);
    }
    printf("please input send char:");
    while(fgets(buf, MAXLINE, stdin) != NULL) {
         write(sockfd, buf, strlen(buf));
         n = read(sockfd, buf, MAXLINE);
         if ( n <= 0 ) {
            printf("the other side has been closed.\n");
            break;
         }else {
            printf("received from server: %s \n",buf);
         }
         printf("please input send char:");
    }
    printf("end server  date");
    close(sockfd);
    return 0;
}


```

