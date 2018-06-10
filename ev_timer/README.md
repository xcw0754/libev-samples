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


