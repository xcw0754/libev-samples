# [ev_signal](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_signal_code_signal_me_when_a)

ev_signal就是信号的管理器，作用同`signal()`或`sigaction()`函数差不多，就是在接收到信号的时候执行注册的回调。


可使用的函数:
```
ev_signal_init (ev_signal *, callback, int signum)
ev_signal_set (ev_signal *, int signum)
```


符号定义:
```
#define EV_WATCHER(type)			\
  int active; /* private */			\
  int pending; /* private */			\
  EV_DECL_PRIORITY /* private */		\
  EV_COMMON /* rw */				\
  EV_CB_DECLARE (type) /* private */

#define EV_WATCHER_LIST(type)			\
  EV_WATCHER (type)				\
  struct ev_watcher_list *next;

typedef struct ev_signal
{
  EV_WATCHER_LIST (ev_signal)

  int signum; /* ro */
} ev_signal;
```


# sample1

先了解一下基本的使用方法。下面的例子，每次收到信号`SIGUSR1`就会调用`sigint_cb`函数(可以用kill -SIGUSR1 pid来发信号)，进程在收到信号之后不会终止，处理完回调仍然在运行。

```
void sigint_cb(struct ev_loop *loop, ev_signal *w, int revents)
{
    printf("got a SIGUSR1 signal\n");
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;
    ev_signal signal_watcher;

    ev_signal_init(&signal_watcher, sigint_cb, SIGUSR1);
    ev_signal_start(loop, &signal_watcher);

    ev_run (loop, 0);
    return 0;
}
```



# 关于fork/execve/pthread_create

`fork`用于创建子进程。子进程几乎拥有和父进程同样的镜像，子进程会继承父进程很多东西，如注册的信号回调，当收到信号时仍然会同父进程一样执行同样的回调函数。

`exec`常常紧跟在fork之后，用于创建子进程后去执行一个全新的程序。只要执行了某个类`exec`的函数，进程的镜像就全变了，找不到原来的信号处理函数了。此时，注册为ignore的信号会继承下来，注册了回调的函数会被置为默认处理方式。其实`exec`之后还是会继承很多东西的(指的是继承原来的镜像，不是父进程)，比如：
```
nice value (see nice()) 
semadj values (see semop()) 
process ID 
parent process ID 
process group ID 
session membership 
real user ID 
real group ID 
supplementary group IDs 
time left until an alarm clock signal (see alarm()) 
current working directory 
root directory 
file mode creation mask (see umask()) 
file size limit (see ulimit()) 
process signal mask (see sigprocmask()) 
pending signal (see sigpending()) 
tms_utime, tms_stime, tms_cutime, and tms_cstime (see times()) 
resource limits 
controlling terminal 
interval timers
```
详见[exec手册](http://pubs.opengroup.org/onlinepubs/007908799/xsh/exec.html)

`pthread_create`用于创建线程。暂时不清楚`ev_signal`是否支持。


# 信号的特点

如果为某个信号注册了回调，当正在执行回调函数的时候，收到相同的信号会怎样？你可以用如下代码测试
```
void sigint_cb(struct ev_loop *loop, ev_signal *w, int revents)
{
    printf("got a SIGUSR1 signal\n");
    sleep(5);
    sleep(3);
    printf("finish\n");
}
int main()
{
    struct ev_loop *loop = EV_DEFAULT;
    ev_signal signal_watcher;

    ev_signal_init(&signal_watcher, sigint_cb, SIGUSR1);
    ev_signal_start(loop, &signal_watcher);

    ev_run (loop, 0);
    return 0;
}
```

程序跑起来之后多kill几次看看。比如在1秒内连续kill 3次(手动就好了，别用程序)，会看到输出
```
got a SIGUSR1 signal
finish
got a SIGUSR1 signal
finish
```

来细细解析一下为什么是这样的输出。当收到第一个信号的时候，打印`got a SIGUSR1 signal`，此时正在`sleep(5)`，收到第2个信号导致从睡眠中立刻清醒，陷入`sleep(3)`继续睡，此时接收到第3个信号，进程醒来打印了`finish`，至此首次执行完整个`sigint_cb`函数。因为kill了3次，第2、3个信号只算1个信号，不会累计，所以还会再执行一次`sigint_cb`函数。

在收到第2个信号时程序就该执行`sigint_cb`函数了，但是因为`sleep(3)`的原因，迟迟不能执行。此时还来第3个信号，但是信号不会累计，如果在同一时间收到多个相同信号，且该信号未被主力，那么只会执行一次回调函数。


# 探讨signal函数和ev_signal

`signal()`函数历史悠久，遗憾的是，它在各个类Unix系统之间不太一样，甚至在不同版本的系统之间也不太一样。这样带来的麻烦就是，要特别注意当前使用的系统，否则可能会带来预想不到的效果。

首先要注意到，`SIGKILL`和`SIGSTOP`这两个信号是不能被捕捉的，也不能被忽略，即使`signal()`函数也无法做到。其次，`signal()`在多线程中的效果是无法预料的，应该使用`sigaction()`代替之。

用`signal()`函数注册不同信号的handler，当在执行handler的时候收到信号，就会有不同的情况了:
- 在当前handler处理完成之前收到相同信号，排队等待当前handler结束后再执行handler一次
- 在当前handler处理完成之前多次收到相同信号，排队等待当前handler结束后再执行handler一次(是的，仅1次)
- 在当前handler处理完成之前收到其他信号，直接执行其他handler。这种感觉就跟多线程一样，两个handler并行执行
- 在当前handler处理完成之前多次收到其他信号，推理一下即可

这些行为和`ev_signal`有些不同，libev(努力)把它实现成同步的，就是说无论是什么信号，收到了都要排队处理，信号不叠加，即pending的信号是不会重复的。

如果你想要`signal()`和`ev_signal`混用，我的建议是**不要这样**，这样会令结果变得不可预测。


# 信号处理函数能调用哪些函数？

POSIX有个**safe function**的概念，信号处理函数中如果调用非安全函数的话可能会带来未定义的结果。POSIX.1-2004标准支持如下函数:
```
_Exit() _exit() abort() accept() access() aio_error() aio_return() aio_suspend() alarm() bind() cfgetispeed() cfgetospeed() cfsetispeed() cfsetospeed() chdir() chmod() chown() clock_gettime() close() connect() creat() dup() dup2() execle() execve() fchmod() fchown() fcntl() fdatasync() fork() fpathconf() fstat() fsync() ftruncate() getegid() geteuid() getgid() getgroups() getpeername() getpgrp() getpid() getppid() getsockname() getsockopt() getuid() kill() link() listen() lseek() lstat() mkdir() mkfifo() open() pathconf() pause() pipe() poll() posix_trace_event() pselect() raise() read() readlink() recv() recvfrom() recvmsg() rename() rmdir() select() sem_post() send() sendmsg() sendto() setgid() setpgid() setsid() setsockopt() setuid() shutdown() sigaction() sigaddset() sigdelset() sigemptyset() sigfillset() sigismember() signal() sigpause() sigpending() sigprocmask() sigqueue() sigset() sigsuspend() sleep() sockatmark() socket() socketpair() stat() symlink() sysconf()

```

不过POSIX.1-2008标准已经删除掉其中这些函数:
```
tcdrain() tcflow() tcflush() tcgetattr() tcgetpgrp() tcsendbreak() tcsetattr() tcsetpgrp() time() timer_getoverrun() timer_gettime() timer_settime() times() umask() uname() unlink() utime() wait() waitpid() write()   
```

并补充了如下这些函数:
```
tcdrain() tcflow() tcflush() tcgetattr() tcgetpgrp() tcsendbreak() tcsetattr() tcsetpgrp() time() timer_getoverrun() timer_gettime() timer_settime() times() umask() uname() unlink() utime() wait() waitpid() write()   
```
















