# [ev_periodic](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_periodic_code_to_cron_or_not)

ev_periodic和ev_timer也是一种定时器，有时它们是可以通用的，有时也是有区别的。ev_timer不适用于长时间的超时，比如一周后、一个月后，它在一定程度上有延时的风险，回调的超时越大，这个时间就越不准。其实从命名上也能看出区别，ev_periodic适用于周期性的回调，比如每天早上5点需要清理冗余的数据。ev_periodic它是根据时刻来回调的，也就是说，只有本地时钟刚好走到了那个点才会触发回调。

考虑一下特殊情况，如果设置了10分钟后的回调，再把本地时钟调到去年，那么这个回调就要等1年的时间了。如果把时钟往后调呢？....


可使用的函数如下:
```
ev_periodic_init(ev_periodic *, callback, ev_tstamp offset, ev_tstamp interval, reschedule_cb)
ev_periodic_set(ev_periodic *, ev_tstamp offset, ev_tstamp interval, reschedule_cb)
ev_periodic_again(loop, ev_periodic *)
ev_tstamp ev_periodic_at(ev_periodic *)
```

符号定义:
```

```


#sample1

这个例子每5秒就调用一次clock_cb。

```
void clock_cb(struct ev_loop *loop, ev_periodic *w, int revents)
{
    printf("clock_cb\n");
}
int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_periodic tick;
    ev_periodic_init(&tick, clock_cb, 0., 5., 0);
    ev_periodic_start(loop, &tick);

    ev_run (loop, 0);
    return 0;
}
```


#sample2

这个例子和sample1一样，不过使用到了`ev_periodic_init`的最后一个参数，略微规则略微复杂。当执行`ev_run`之后立刻调用了`my_scheduler_cb`计算出下一次调用`clock_cb`的时间，此后，每次调用`clock_cb`之前都会调用`my_scheduler_cb`计算下一次回调`clock_cb`的时间。除了首次之外，它们都是依次调用的。这种写法可以满足不定长超时的回调，在`my_scheduler_cb`里边计算下次回调的时间即可。

严格来讲，这个例子的执行顺序是这样的。第0秒，调用`my_scheduler_cb`。第5秒，调用`my_scheduler_cb`后立即调用`clock_cb`。第10秒，调用`my_scheduler_cb`后立即调用`clock_cb`。依此类推。

注:`ev_periodic_init`设置了末参数的话，第3、4个参数就自动失效。

```
void clock_cb(struct ev_loop *loop, ev_periodic *w, int revents)
{
    printf("clock_cb\n");
}
ev_tstamp my_scheduler_cb(ev_periodic *w, ev_tstamp now)
{
    printf("my_scheduler_cb\n");
    return now + 5;
}
int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_periodic tick;
    ev_periodic_init(&tick, clock_cb, 0., 0., my_scheduler_cb);
    ev_periodic_start(loop, &tick);

    ev_run(loop, 0);
    return 0;
}
```

# sample3

下面这个例子在整点的时候就会回调`clock_cb`。

```
#include <math.h>
void clock_cb(struct ev_loop *loop, ev_periodic *w, int revents)
{
    printf("clock_cb\n");
}
int main()
{
    struct ev_loop *loop = EV_DEFAULT;

    ev_periodic tick;
    ev_periodic_init(&tick, clock_cb, fmod(ev_now(loop), 3600.), 3600., 0);
    ev_periodic_start(loop, &tick);

    ev_run (loop, 0);
    return 0;
}
```















