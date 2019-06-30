# select、poll和epoll
## 1. I/O复用概念
    举个例子：TCP客户端在同时处理两个输入：标准输入(fgets)和TCP套接字。当客户阻塞于fgets时，服务器进程会被杀死。由于客户端的阻塞，服务器在发送FIN结束连接时，客户端无法看到这个EOF，直到读取下一个套接字。所以进程需要一种预先告知内核的能力，使得内核一旦发现进程指定的一个或多个I/O条件就绪，就通知进程。这个能力就成为I/O复用，是由select和poll两个函数支持的。

I/O复用有以下使用场合：

- 客户端处理多个描述符(通常是交互式输入+网络套接字)
- 客户端同时处理多个套接字
- 服务器同时监听+处理已连接套接字
- 服务器同时处理TCP，又处理UDP
- 服务器同时处理多个服务或者多个协议

## 2. I/O模型
Unix下可用的5中I/O模型：

- 阻塞式I/O : 应用进程调用recvfrom，等待内核准备数据，数据报到来时将数据从内核空间复制到用户空间，复制完成后，处理数据报。在等待内核数据过程中一直是阻塞状态。
- 非阻塞式I/O : 进程调用recvfrom，内核无数据报的话直接返回EWOULDBLOCK错误，有数据报的话进行复制和处理操作，在此期间进行的是轮询操作。
- I/O复用 ：在recvfrom前面增加select或poll操作，recvfrom是等待指定套接字，而select或poll是可以等待多个套接字中的一个变为可读。一旦有某个套接字可读，就调用recvfrom操作进行操作。
- 信号驱动式I/O : 使用信号处理函数，当描述符就绪时，使用SIGIO通知，在信号处理函数中调用recvfrom进行I/O数据报处理。
- 异步I/O : 告知内核某个动作，并让内核完成操作后用指定好的信号中通知使用者，之后完成I/O操作。使用aio_read函数，告知内核描述符、缓冲区指针、缓冲区大小和文件偏移量，并指定信号让内核完成通知过程。

## 3. select函数

    该函数允许进程指示内核等待多个事件中的任何一个发生,并且只有一个或多个事件发生或经历一段指定时间后才唤醒

举例：

- 集合{1,4,5}中的任何描述符准备好读
- 集合{2,7}中的任何描述符准备好写
- 集合{1,4}中的任何描述符有异常条件待处理
- 已经经历了10.2秒
也就是说，调用select告知内核关注哪些描述符或等待多长时间，这里面的描述符除了指套接字，还可以指任何描述符。

```c++
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
//返回：若有就绪描述符则为其数目，若超时时则为0，若出错则为-1

```
timeout参数：告知内核等待所指定描述符中的任何一个就绪可花多长时间
这个参数有以下三种可能：

- 永远等待：仅有一个描述符准备好I/O时才返回。---把该参数设置为空指针
- 等待一段固定时间：在指定时间内有描述符准备好I/O时返回。---把该参数设置为指定时间长度
- 不等待：检查描述符后立即返回，即轮训。---把该参数设置为0

readset参数：内核要测试的读描述符集合
writeset参数：内核要测试的写描述符集合
exceptset参数：内核要测试的异常条件描述符

目前支持的异常条件只有两个：

- 某个套接字的带外数据的到达
- 某个已置为分组模式的伪终端存在可以从其铸锻读取的控制状态信息

使用以下四个宏进行描述符集设置：
```c++
void FD_ZERO(fd_set *fdset);          // clear all bits in fdset
void FD_SET(int fd, fd_set *fdset);   // turn on the bit for fd in fdset
void FD_CLR(int fd, fd_set *fdset);   // turn off the bit for fd in fdset
int FD_ISSET(int fd, fd_set *fdset);  // is the bit for fd on in fdset?
```

maxfdp1参数：指定待测试的描述符个数，它的值是最大描述符加1，并且该最大描述符个数不能超过1024

### 3.1 描述符就绪条件

套接字准备好***读条件***（满足条件其一即可）：

- 套接字接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的当前大小（有数据上涨超过缓冲区水位，表示可以读走了，TCP和UDP默认为1）。
- 该连接的读半部关闭（也就是接受到了FIN的TCP连接）。
- 该套接字是一个监听套接字且已完成的连接数不为0（LISTEN状态的套接字）。
- 其上有一个套接字错误待处理（返回-1，并会将error设置为确切条件）。

套接字准备好***写条件***（满足条件其一即可）：

- 套接字发送缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的当前大小，并且该套接字可发送数据（TCP需要已连接，UDP不需要连接，有数据上涨超过缓冲区水位，表示有数据需要发送，TCP和UDP默认为2048）。
- 该连接的写半部关闭（也就是发送了FIN的TCP连接）。
- 使用非阻塞式connect的套接字已经建立连接，或者connect以失败告终（也就是连接测试）。
- 其上有一个套接字错误待处理（返回-1，并会将error设置为确切条件）。

## 4.poll函数
    poll提供的功能与select类似，不过在处理流设备时，它能够提供额外的信息
```c++
#include <poll.h>

int poll(struct pollfd *fdarray, unsigned long nfds, int timeout);
// 返回：若有就绪描述符则为其数目，若超时则为0，若出错则为-1

// pollfd结构
struct pollfd{
    int fd;
    short events;
    short revents;
}
```
要测试的条件由events成员指定，函数在相应的revents成员中返回该描述符的状态

测试读条件标志常值：

- POLLIN：普通或优先级带数据可读
- POLLRDNORM：普通数据可读
- POLLRDBAND：优先级带数据可读
- POLLPRI： 高优先级数据可读

测试写条件标志常值：
- POLLOUT：普通或优先级带数据可写
- POLLWRNORM：普通数据可写
- POLLWRBAND：优先级带数据可写

处理错误标志常值：

- POLLERR：发生错误
- POLLHUP：发生挂起
- POLLNVAL：描述符不是打开的文件

## 5.epoll函数

基本知识
epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

### 5.1 epoll接口
　　epoll操作过程需要三个接口，分别如下：
```c++
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

- int epoll_create(int size);
　　创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

- int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
　　epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
```c++
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```

events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

- int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
　　等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

### 5.2 工作模式

　　epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

- LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
- ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

### 5.3 测试程序

#### 5.3.1 服务器代码：
```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <sys/types.h>

#define IPADDRESS   "127.0.0.1"
#define PORT        8787
#define MAXSIZE     1024
#define LISTENQ     5
#define FDSIZE      1000
#define EPOLLEVENTS 100

//函数声明
//创建套接字并进行绑定
static int socket_bind(const char* ip,int port);
//IO多路复用epoll
static void do_epoll(int listenfd);
//事件处理函数
static void
handle_events(int epollfd,struct epoll_event *events,int num,int listenfd,char *buf);
//处理接收到的连接
static void handle_accpet(int epollfd,int listenfd);
//读处理
static void do_read(int epollfd,int fd,char *buf);
//写处理
static void do_write(int epollfd,int fd,char *buf);
//添加事件
static void add_event(int epollfd,int fd,int state);
//修改事件
static void modify_event(int epollfd,int fd,int state);
//删除事件
static void delete_event(int epollfd,int fd,int state);

int main(int argc,char *argv[])
{
    int  listenfd;
    listenfd = socket_bind(IPADDRESS,PORT);
    listen(listenfd,LISTENQ);
    do_epoll(listenfd);
    return 0;
}

static int socket_bind(const char* ip,int port)
{
    int  listenfd;
    struct sockaddr_in servaddr;
    listenfd = socket(AF_INET,SOCK_STREAM,0);
    if (listenfd == -1)
    {
        perror("socket error:");
        exit(1);
    }
    bzero(&servaddr,sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    inet_pton(AF_INET,ip,&servaddr.sin_addr);
    servaddr.sin_port = htons(port);
    if (bind(listenfd,(struct sockaddr*)&servaddr,sizeof(servaddr)) == -1)
    {
        perror("bind error: ");
        exit(1);
    }
    return listenfd;
}

static void do_epoll(int listenfd)
{
    int epollfd;
    struct epoll_event events[EPOLLEVENTS];
    int ret;
    char buf[MAXSIZE];
    memset(buf,0,MAXSIZE);
    //创建一个描述符
    epollfd = epoll_create(FDSIZE);
    //添加监听描述符事件
    add_event(epollfd,listenfd,EPOLLIN);
    for ( ; ; )
    {
        //获取已经准备好的描述符事件
        ret = epoll_wait(epollfd,events,EPOLLEVENTS,-1);
        handle_events(epollfd,events,ret,listenfd,buf);
    }
    close(epollfd);
}

static void
handle_events(int epollfd,struct epoll_event *events,int num,int listenfd,char *buf)
{
    int i;
    int fd;
    //进行选好遍历
    for (i = 0;i < num;i++)
    {
        fd = events[i].data.fd;
        //根据描述符的类型和事件类型进行处理
        if ((fd == listenfd) &&(events[i].events & EPOLLIN))
            handle_accpet(epollfd,listenfd);
        else if (events[i].events & EPOLLIN)
            do_read(epollfd,fd,buf);
        else if (events[i].events & EPOLLOUT)
            do_write(epollfd,fd,buf);
    }
}
static void handle_accpet(int epollfd,int listenfd)
{
    int clifd;
    struct sockaddr_in cliaddr;
    socklen_t  cliaddrlen;
    clifd = accept(listenfd,(struct sockaddr*)&cliaddr,&cliaddrlen);
    if (clifd == -1)
        perror("accpet error:");
    else
    {
        printf("accept a new client: %s:%d\n",inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);
        //添加一个客户描述符和事件
        add_event(epollfd,clifd,EPOLLIN);
    }
}

static void do_read(int epollfd,int fd,char *buf)
{
    int nread;
    nread = read(fd,buf,MAXSIZE);
    if (nread == -1)
    {
        perror("read error:");
        close(fd);
        delete_event(epollfd,fd,EPOLLIN);
    }
    else if (nread == 0)
    {
        fprintf(stderr,"client close.\n");
        close(fd);
        delete_event(epollfd,fd,EPOLLIN);
    }
    else
    {
        printf("read message is : %s",buf);
        //修改描述符对应的事件，由读改为写
        modify_event(epollfd,fd,EPOLLOUT);
    }
}

static void do_write(int epollfd,int fd,char *buf)
{
    int nwrite;
    nwrite = write(fd,buf,strlen(buf));
    if (nwrite == -1)
    {
        perror("write error:");
        close(fd);
        delete_event(epollfd,fd,EPOLLOUT);
    }
    else
        modify_event(epollfd,fd,EPOLLIN);
    memset(buf,0,MAXSIZE);
}

static void add_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&ev);
}

static void delete_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_DEL,fd,&ev);
}

static void modify_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&ev);
}
```


#### 5.3.2 客户端代码
客户端也用epoll实现，控制STDIN_FILENO、STDOUT_FILENO、和sockfd三个描述符，程序如下所示:

```c++
#include <netinet/in.h>
#include <sys/socket.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <time.h>
#include <unistd.h>
#include <sys/types.h>
#include <arpa/inet.h>

#define MAXSIZE     1024
#define IPADDRESS   "127.0.0.1"
#define SERV_PORT   8787
#define FDSIZE        1024
#define EPOLLEVENTS 20

static void handle_connection(int sockfd);
static void
handle_events(int epollfd,struct epoll_event *events,int num,int sockfd,char *buf);
static void do_read(int epollfd,int fd,int sockfd,char *buf);
static void do_read(int epollfd,int fd,int sockfd,char *buf);
static void do_write(int epollfd,int fd,int sockfd,char *buf);
static void add_event(int epollfd,int fd,int state);
static void delete_event(int epollfd,int fd,int state);
static void modify_event(int epollfd,int fd,int state);

int main(int argc,char *argv[])
{
    int                 sockfd;
    struct sockaddr_in  servaddr;
    sockfd = socket(AF_INET,SOCK_STREAM,0);
    bzero(&servaddr,sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET,IPADDRESS,&servaddr.sin_addr);
    connect(sockfd,(struct sockaddr*)&servaddr,sizeof(servaddr));
    //处理连接
    handle_connection(sockfd);
    close(sockfd);
    return 0;
}


static void handle_connection(int sockfd)
{
    int epollfd;
    struct epoll_event events[EPOLLEVENTS];
    char buf[MAXSIZE];
    int ret;
    epollfd = epoll_create(FDSIZE);
    add_event(epollfd,STDIN_FILENO,EPOLLIN);
    for ( ; ; )
    {
        ret = epoll_wait(epollfd,events,EPOLLEVENTS,-1);
        handle_events(epollfd,events,ret,sockfd,buf);
    }
    close(epollfd);
}

static void
handle_events(int epollfd,struct epoll_event *events,int num,int sockfd,char *buf)
{
    int fd;
    int i;
    for (i = 0;i < num;i++)
    {
        fd = events[i].data.fd;
        if (events[i].events & EPOLLIN)
            do_read(epollfd,fd,sockfd,buf);
        else if (events[i].events & EPOLLOUT)
            do_write(epollfd,fd,sockfd,buf);
    }
}

static void do_read(int epollfd,int fd,int sockfd,char *buf)
{
    int nread;
    nread = read(fd,buf,MAXSIZE);
        if (nread == -1)
    {
        perror("read error:");
        close(fd);
    }
    else if (nread == 0)
    {
        fprintf(stderr,"server close.\n");
        close(fd);
    }
    else
    {
        if (fd == STDIN_FILENO)
            add_event(epollfd,sockfd,EPOLLOUT);
        else
        {
            delete_event(epollfd,sockfd,EPOLLIN);
            add_event(epollfd,STDOUT_FILENO,EPOLLOUT);
        }
    }
}

static void do_write(int epollfd,int fd,int sockfd,char *buf)
{
    int nwrite;
    nwrite = write(fd,buf,strlen(buf));
    if (nwrite == -1)
    {
        perror("write error:");
        close(fd);
    }
    else
    {
        if (fd == STDOUT_FILENO)
            delete_event(epollfd,fd,EPOLLOUT);
        else
            modify_event(epollfd,fd,EPOLLIN);
    }
    memset(buf,0,MAXSIZE);
}

static void add_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_ADD,fd,&ev);
}

static void delete_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_DEL,fd,&ev);
}

static void modify_event(int epollfd,int fd,int state)
{
    struct epoll_event ev;
    ev.events = state;
    ev.data.fd = fd;
    epoll_ctl(epollfd,EPOLL_CTL_MOD,fd,&ev);
}
```
#### 5.3.3测试结果
![1](https://images0.cnblogs.com/blog/305504/201308/17013658-d6aa0df188b24c6da6759c4c611570b3.png)
![2](https://images0.cnblogs.com/blog/305504/201308/17013713-0bd444572e064c8e80d58d6221277eb4.png)
![3](https://images0.cnblogs.com/blog/305504/201308/17013723-571312abdc2945a68016a080690fc3fb.png)