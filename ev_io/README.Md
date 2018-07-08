# [ev_io](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_io_code_is_this_file_descrip)

ev_io用于句柄可读写的检测，如果你有如下需求，你就可能需要用到它。
- 在网络对端发送的数据已到达本机时，及时读取socket中的数据
- 管道中已经写满数据，在管道可写时及时写入数据


可用函数如下:
```
ev_io_init (ev_io *, callback, int fd, int events);
ev_io_set (ev_io *, int fd, int events);

void ev_io_start(EV_P_ ev_io *w) EV_THROW;
void ev_io_stop(EV_P_ ev_io *w) EV_THROW;


```

符号定义:
```
#if EV_MINPRI == EV_MAXPRI
# define EV_DECL_PRIORITY
#elif !defined (EV_DECL_PRIORITY)
# define EV_DECL_PRIORITY int priority;
#endif

#ifndef EV_COMMON
# define EV_COMMON void *data;
#endif

#ifndef EV_CB_DECLARE
# define EV_CB_DECLARE(type) void (*cb)(EV_P_ struct type *w, int revents);
#endif

#define EV_WATCHER(type)			\
  int active; /* private */			\
  int pending; /* private */			\
  EV_DECL_PRIORITY /* private */		\
  EV_COMMON /* rw */				\
  EV_CB_DECLARE (type) /* private */

typedef struct ev_watcher_list
{
  EV_WATCHER_LIST (ev_watcher_list)
} ev_watcher_list;

#define EV_WATCHER_LIST(type)			\
  EV_WATCHER (type)				\
  struct ev_watcher_list *next; /* private */

typedef struct ev_io
{
  EV_WATCHER_LIST (ev_io)
  int fd;     /* ro */
  int events; /* ro */
} ev_io;
```

上面符号定义有点多，gcc -E xxx.c编译出来的结果:
```
typedef struct ev_io
{
  int active; 
  int pending; 
  int priority;
  void *data; 
  void (*cb)(struct ev_loop *loop, struct ev_io *w, int revents); 
  struct ev_watcher_list *next;
  int fd;
  int events;
} ev_io;
```



# sample1

这个例子可以在有标准输入时打印输入的内容。每次回调不需要重新设置监听的，默认就是一直监听着句柄的读事件。

```
void stdin_readable_cb(struct ev_loop *loop, ev_io *w, int revents)
{
    char buf[64] = {0};
    read(w->fd, buf, sizeof(buf) - 1);
    printf("%s\n", buf);
}
int main()
{
    struct ev_loop *loop = EV_DEFAULT;
    ev_io stdin_readable;

    ev_io_init(&stdin_readable, stdin_readable_cb, STDIN_FILENO, EV_READ);
    ev_io_start(loop, &stdin_readable);

    ev_run(loop, 0);
    return 0;
}
```


# sample2

sample1还存在一些问题，就是如何在socket关闭时把这个事件停掉以免浪费资源，看下面这个例子。

如果对端socket已经关闭了，那么这个句柄就是处于可读的状态，libev会一直调用`readable_cb`直到你主动把这个io事件停掉，因为下面这个例子只有一个io事件，停掉了自然就会终止进程了。检测对端socket关闭是普通的tcp常识，检测一下`recv`的返回值即可。

Tips:拿家里路由器管理页面（192.168.1.1）来测试一下。

```
// 读事件回调
void readable_cb(struct ev_loop *loop, ev_io *w, int revents)
{
    char buf[64] = {0};
    ssize_t rlen = recv(w->fd, buf, sizeof(buf) - 1, 0);
    if (rlen == 0)
    {
        // 对端关闭，stop掉fd
        printf("\n=================== stop ==================\n");
        close(w->fd);
        ev_io_stop(loop, w);
    }
    else if (rlen < 0 && (errno == EAGAIN || errno == EWOULDBLOCK))
    {
        // 被打断，重新监听即可
        printf("\n=================== break ==================\n");
    }
    else if (rlen < 0)
    {
        // 出错
        printf("%s\n", strerror(errno));
    }
    else
    {
        // 打印正常接收到的数据
        printf("%s", buf);
    }
}

#define HTTP_HEADER "GET / HTTP/1.1\r\nHost: 192.168.1.1\r\nConnection:keep-alive\r\n\r\n"

int main()
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0)
    {
        printf("create sock failed\n");
        return 0;
    }

    struct sockaddr_in addr;
    bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = inet_addr("192.168.1.1");
    addr.sin_port = htons(80);
	// 连接对端
    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0)
    {
        printf("connect failed\n");
        return 0;
    }
	// 发送GET请求
    if (send(sock, HTTP_HEADER, sizeof(HTTP_HEADER), 0) != sizeof(HTTP_HEADER)) {
        printf("send failed\n");
        return 0;
    }
	// 设置为非阻塞
    int flags = fcntl(sock, F_GETFL);
    if (fcntl(sock, F_SETFL, flags | O_NONBLOCK) != 0)
    {
        printf("%s\n", strerror(errno));
        return 0;
    }
	// 开始监听读事件
    struct ev_loop *loop = EV_DEFAULT;
    ev_io http_io;

    ev_io_init(&http_io, readable_cb, sock, EV_READ);
    ev_io_start(loop, &http_io);

    ev_run(loop, 0);
    return 0;
}
```


# 阻塞与非阻塞

**句柄最好设置成非阻塞模式**，除非你知道自己在做什么。官网提到了要小心可能会收到假的触发事件，比如给某个fd设置了`EV_READ`事件，在回调的函数里可能会发现并没有数据到达，具体的表现就是阻塞型fd使用`read`函数会阻塞，非阻塞型fd使用`read`函数会收到`EAGAIN`。这就是假的触发事件，这种情况应该当作啥事都没有发生过，继续监听事件到达。

[select手册](https://linux.die.net/man/2/select) 中的Bugs章节提到了这样一个bug:

> Under Linux, select() may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks. This could for example happen when data has arrived but upon examination has wrong checksum and is discarded. There may be other circumstances in which a file descriptor is spuriously reported as ready. Thus it may be safer to use O_NONBLOCK on sockets that should not block.

大意就是，由于各种原因，select可能会因为假的事件而返回，比如select已经返回，但是数据包的checksum不对而被丢弃，所以导致`read`读不到数据。但是bug为何至今不修复，应该是有其原因的，stackoverflow上有个同样的问题[Spurious readiness notification for Select System call](https://stackoverflow.com/questions/858282/spurious-readiness-notification-for-select-system-call)。


如果你由于某种原因必须使用阻塞模式，你必须在读之前再次测试是否真的有数据可读，否则可能永远阻塞在那里了。有些人就会用`SIGALRM`和定时器来保证进程不会被永久阻塞。不过推荐最好还是用非阻塞模式。


# 复制的句柄

**复制的句柄可能只有其中一个会收到触发的事件**，比如先将标准出错句柄`2`关闭，再使用`dup`函数将标准输出`1`复制到句柄`2`，现在就有两个句柄(即`2`和`1`）表示标准输出了，如果为两个句柄都注册同样事件的话，那么只有其中一个句柄注册的事件会被触发。


# 文件的读写

**文件句柄不同于socket**，文件所需的磁盘IO时间消耗是躲不过去的。比如你打开了一个文件，设置了`EV_READ`和`EV_WRITE`事件，想以此来减少磁盘IO时间。可惜的是这样并没有效果，很可能libev会一直触发事件，但是你去`read`或者`write`依然要花时间在磁盘IO上面。所以socket、管道、字符设备等等这些句柄才能达到减少消耗的效果。

建议在普通文件上用异步的模式(如libeio)来减少IO消耗，除非你知道自己在做什么。比如STDIN、STDOUT还是可以考虑用io_watcher的。


# fork带来的问题

epoll和kqueue是不支持`fork`的，而libev是支持的，只不过要在子进程里告诉它你已经执行`fork`了。想在支持`fork`，应该在执行`fork`之后在子进程里调用`ev_loop_fork ()`，打开`EVFLAG_FORKCHECK`，或者`resort to EVBACKEND_SELECT or EVBACKEND_POLL`。


# accept带来的问题

比如承载力较大的server通常会将有限的句柄消耗完，这时`accept()`就会返回`ENFILE`，但是呢，又不会拒绝连接，导致libev每次循环都回触发可读事件，这样就会造成进程的CPU消耗过高。

最简单的方式就是忽略这个问题，当server过载的时候就会忙等待，直到负载降下来自然问题就解决了。

推荐在进程中将所有错误给LOG下来，除了`EAGAIN`和`EWOULDBLOCK`，其他都可以考虑LOG下来，这样在出现奇怪的问题的时候可能有点用处，但要保证别LOG过多的垃圾信息。可以的话，在必要的时候将ev_io给stop掉一小段时间，这样能减少一点CPU消耗。

如果是单线程的话，可以先打开一些虚句柄(如/dev/null)来占位，等到`accept()`返回`ENFILE`或`EMFILE`的时候，关闭掉一个虚句柄，再`accept()`，实句柄用完之后关闭，再打开虚句柄。据说这可以在程序过载的时候优雅地拒绝即将到来的连接。




