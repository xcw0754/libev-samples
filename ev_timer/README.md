# [ev_timer](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_timer_code_relative_and_opti)

ev_timer就是定时器，如果你有如下需求，你就可能需要用到它。
- n秒之后需要调用某个函数(如:tcp连接超时断开连接)
- 每隔y秒就调用某个函数(如:定时清理资源)


可使用的函数如下:
```
ev_timer_init (timer, callback, 60., 0.);
ev_timer_start (loop, timer);
ev_timer_stop (loop, timer);
ev_timer_set (timer, 60., 0.);
ev_timer_start (loop, timer);
ev_timer_remaining (loop, ev_timer *)
```

符号定义:
```
typedef double ev_tstamp;

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

#define EV_WATCHER(type)			    \
    int active;     /* private */		\
    int pending;    /* private */		\
    EV_DECL_PRIORITY /* private */	    \
    EV_COMMON       /* rw */			\
    EV_CB_DECLARE (type) /* private */

#define EV_WATCHER_TIME(type)			\
    EV_WATCHER (type)				    \
    ev_tstamp at;     /* private */

typedef struct ev_timer
{
    EV_WATCHER_TIME (ev_timer)
    ev_tstamp repeat; /* rw */
} ev_timer;
```



# sample1

这个例子有一个循环loop，等待了2.5秒之后调用timeout_cb，执行完这个函数就退出程序了。

```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_timer_init (&timeout_watcher, timeout_cb, 2.5, 0);
    ev_timer_start (loop, &timeout_watcher);
    
    ev_run (loop, 0);
    return 0;
}
```


# sample2

这个例子有一个循环loop，等待了2.5秒之后调用timeout_cb，之后每隔5秒就执行一次这个函数。这个程序将无限循环，不会退出。可以猜出来`ev_timer_init`的最后一个参数为0时表示不重复回调，大于0为间隔回调时间，是个浮点数。

```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    // 改一下最后一个参数试试
    ev_timer_init (&timeout_watcher, timeout_cb, 2.5, 5.0);
    ev_timer_start (loop, &timeout_watcher);
    
    ev_run (loop, 0);
    return 0;
}
```

# sample3

sample2不会退出，以下sample会来让它退出。以下程序其实和sample1一样，调用了一次timeout_cb后就退出了。
哦，libev有个宏`EV_A_`，它其实是`loop,`，用起来不太好看，不过方便一点。

```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   

    // 这里能让循环终止
    ev_break (loop, EVBREAK_ONE);
    // ev_break (EV_A_ EVBREAK_ONE); 效果同上
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_timer_init (&timeout_watcher, timeout_cb, 2.5, 5.0);
    ev_timer_start (loop, &timeout_watcher);
    
    ev_run (loop, 0);
    return 0;
}
```


# sample4

有时候并不是固定的时间间隔呀，动态地改变超时行不行？可以的，在回调里边再设一次timeout呗。
其实`ev_timer_init`调用的就是`ev_timer_set`，它们两的最后一个参数意义一样。


```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   

    ev_timer_stop (loop, w);
    ev_timer_set (w, 1., 0.);   // 第2个参数为下次timeout，可以是变化的
    ev_timer_start (loop, w);
    ev_run (loop, 0);           // 别忘了再run起来
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_timer_init (&timeout_watcher, timeout_cb, 2.5, 0);
    ev_timer_start (loop, &timeout_watcher);
    
    ev_run (loop, 0);
    return 0;
}
```

# sample5

sample4看起来有点麻烦，而且还耗时，每次都要将结构体移除再插入，不妨用以下的方式代替。


```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   

    w->repeat = 7.;     // 这里可以改变下次回调超时
    ev_timer_again(loop, w);
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_init(&timeout_watcher, timeout_cb);
    timeout_watcher.repeat = 2.;
    ev_timer_again(loop, &timeout_watcher);

    ev_run (loop, 0);
    return 0;
}
```

# sample6

其实sample5所用的`ev_timer_again`是每次都会回调的，比如下面这样写就是每间隔2秒就回调一次。

```
#include <ev.h>
#include <stdio.h>

ev_timer timeout_watcher;

void timeout_cb (struct ev_loop *loop, ev_timer *w, int revents)
{
    puts("timeout_cb has been called");   
}

int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_init(&timeout_watcher, timeout_cb);
    timeout_watcher.repeat = 2.;
    ev_timer_again(loop, &timeout_watcher);

    ev_run (loop, 0);
    return 0;
}
```

# 时间过早的问题

官方文档提到了一个例子，比如一个OS是以秒计时的，即对外提供的最小单位就是秒，然后在第500.9秒的时候我们设置了一个1秒的timeout，那么应该在什么时候触发这个回调比较好呢？

如果在第501秒触发，那么实际上只过去了0.1秒而已，如果在502秒触发，那么实际上是过去了1.1秒。libev就是后者这样的，它只能保证至少已经过去了所设的timeout秒，但是不能保证刚好就是timeout秒。比如进程接收到STOP信号，那么无法避免，它只能在恢复运行的时候触发回调，可能已经晚了很多，但这是没法保证的。


# 关于时间更新

获取系统时间是个耗时的系统调用，libev是自己维护了个时间，只有在`ev_run`收集到新事件的前后才会更新，这样会导致的问题就是时间偏差会越来越大。`ev_now()`用于获得libev的当前时间，`ev_time()`用于获取系统的当前时间。

如果想矫正libev的当前时间，可以通过`ev_timer_set (&timer, after + (ev_time () - ev_now ()), 0.);`这样来矫正它。也可以通过`ev_now_update ()`来强迫矫正`ev_now`。
