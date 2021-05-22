课程[原地址](http://www.mit.edu/people/proven/IAP_2000/index.html)

这门课程将介绍线程是什么，为什么线程是有用的，以及如何在POSIX 1003.1c线程标准下中使用线程进行编程。



# 课程大纲

- [课程介绍](#课程介绍)
- [基础的线程函数](#基础的线程函数)
- [线程同步](#线程同步)
- [用户数据](#用户数据)



# 课程介绍

## 什么是一个线程

> a single flow of control with a process ...

线程是进程中的一个控制流 ...



## 为什么使用线程

- 利用被阻塞的时间（一个线程因为IO被阻塞了则另外一个线程进行工作）
- 利用并发
  - 并行处理
  - 分布式处理
  - 例子：分形计算器，光线追踪器
- 事件驱动的软件
  - 等待事件的软件（HTTPD，消息服务器，邮件服务器）
  - 互动型软件（实时游戏：Netrek）
  - 所有使用`select()`的软件



## 线程的类型

- 用户线程
- 内核线程
  - 不要和**多线程内核**搞混了
  - 在Linux使用`clone`函数创建
  - 在Plan9上用`rfork`函数创建
  - 在IRIX上用`sproc`函数创建
- 混合线程
  - Solaris LWP and threads.
  - WinNT threads and fibers.



# 基础的线程函数

## 创建

```c
int pthread_create(pthread_t * thread, 
                   const pthread_attr_t * attr,
                   void * (*start_routine)(void *), 
                   void *arg);
```

- 创建一个新的线程然后给这个线程分配任务
- 在传入的`thread`参数中返回一个线程的识别符
- 如果`attr`是`NULL`，则使用默认的线程属性



## 终止

```c
void pthread_exit(void * return_value);
```

- 从当前线程退出
- 如果当前线程是当前进程的最后一个线程，`exit`函数返回之后当前进程终止
- 从子线程返回相当于调用`pthread_exit`
- 从主线程返回相当于调用`exit`



## 分离和加入

```c
int pthread_detach(pthread_t thread);
```

- 标记`thread`为被分离的
- 当一个被分离的线程结束运行时，该线程所占有的资源会被自动返还给系统

```c
int pthread_join(pthread_t thread, void ** status);
```

- 等待指定的线程
- 必须指定线程
- 当前的线程被阻塞直到`thread`结束运行
- `thread`的返回值在`status`参数中存储
- 所有的线程必须被分离或者被加入



## 自身和相等

```c
pthread_t pthread_self(void);
int pthread_equal(pthread_1 t1, pthread_t t2);
```



## 一个简单的例子

```c
#include <pthread.h>
#include <stdio.h>

char * buf = "abcdefghijklmnopqrstuvwxyz";
int num_pthreads = 2;
int count = 60;
int fd = 1;

void * new_thread(void * arg)
{
    int i;

    for (i = 0; i < count; i++) {
			write(fd, arg, 1);
			sleep(1);
    }
    return(NULL);
}

main()
{
   pthread_t thread;
   int i;

   for (i = 0; i < num_pthreads; i++) {
       if (pthread_create(&thread, NULL, new_thread, (void *)(buf + i))) {
           fprintf(stderr, "error creating a new thread \n");
           exit(1);
       }
       pthread_detach(thread);
   }
   pthread_exit(NULL);
}
```



# 线程同步

## 问题一

```c
int foo = 0;
void foo_initializer()
{
    if (foo == 0)
				foo = 1;	/* This must only be done once. */
}
```

有一种可能的执行顺序...

Thread1									Thread2

`if(foo == 0)`							

​													`if(foo == 0)`

​													`foo==1`

`foo==1`

线程1和线程2都设置了`foo`这个变量.



## Mutexes

- 表示互斥锁
- 串行访问一些重要的代码区域或者数据区域
- 任何时候，一个全局的资源被多于一个线程访问时，这个资源需要一个配置互斥锁



## 创建和销毁互斥锁

```c
int pthread_mutex_init(pthread_mutex_t * mutex, pthread_mutexattr_t *attr);
// 初始化一个静态锁，并将属性设置成默认
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
// 销毁一个互斥锁
int pthread_mutex_destroy(pthread_mutex_t * mutex);
```



## 使用互斥锁

```c
int pthread_mutex_lock(pthread_mutex_t * mutex)
int pthread_mutex_trylock(pthread_mutex_t * mutex)
int pthread_mutex_unlock(pthread_mutex_t * mutex)
```



## 解决问题一

```c
int foo = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void foo_initializer()
{
    pthread_mutex_lock(&mutex);
    if (foo == 0)
				foo = 1;	/* This must only be done once. */
    pthread_mutex_unlock(&mutex);
}
```

*译者注：这个例子没有太大现实意义，主要是想说明用锁来保护某些重要的区域(critical section)，使得只可能有一个线程可以访问这个重要区域*



## 问题二

```c
#define P_M_L(x)	pthread_mutex_lock(x)
#define P_M_U(x)	pthread_mutex_unlock(x)

pthread_mutex_t foo_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t bar_mutex = PTHREAD_MUTEX_INITIALIZER;
int foo = 0, bar = 0;

void foo_bar_inc()
{
    P_M_L(&foo_mutex);
    P_M_L(&bar_mutex);
    bar = foo++;
    P_M_U(&bar_mutex);
    P_M_U(&foo_mutex);
}

void bar_foo_dec()
{
    P_M_L(&bar_mutex);
    P_M_L(&foo_mutex);
    foo = bar--;
    P_M_U(&foo_mutex);
    P_M_U(&bar_mutex);
}
```

有一种可能的执行顺序

Thread1									Thread2

​													`foo_bar_inc()`

​													`P_M_L(&foo_mutex)`

`foo_bar_dec()`		

`P_M_L(&bar_mutex)`

`P_M_L(&foo_mutex)`					

​													`P_M_L(&bar_mutex);`

在这种情况下，Thread1和Thread2都在等待互斥锁被释放。Thread1在等待`foo_mutex`，但是`foo_mutex`被Thread2持有了；Thread2在等待`bar_mutex`，但是`bar_mutex`被Thread1持有了。这种情况被称为**死锁**。



## 死锁

> Deadlocks are your friend! -- Nawaf Bitar
>
> 死锁是你的朋友！-- 纳瓦夫·比塔尔

- 死锁与竞争危害(race condition)恰好相反。
- 死锁比竞争危害更容易被找到并且解决。
- 通过合理的分级同步，可以很轻易地被避免死锁。



## 解决问题二

```c
/* Order of locking foo, bar ... */
void bar_foo_dec()
{
    P_M_L(&bar_mutex);
    // 如果foo_mutex已被持有，则直接返回
    while (pthread_mutex_trylock(&foo_mutex) {
        P_M_U(&bar_mutex);
        sleep(1);
        P_M_L(&bar_mutex);
    }
    foo = bar--;
    P_M_U(&bar_mutex);
    P_M_U(&foo_mutex);
}
```



## 问题三

```c
int i_init = 0;
pthread_mutex_t * i_mutex;

void i_init_routine()
{
    if (i_init == 0) {
        pthread_mutex_init(i_mutex, NULL);
        i_init = 1;
    }
}
```

在检查`i_init`之前，我们需要持有一个互斥锁...

```c
pthread_mutex_t * ii_mutex;
void ii_init_routine()
{
    P_M_L(ii_mutex);
    i_init_routine();
    P_M_U(ii_mutex);
}
```

我们需要初始化`ii_mutex`...



## Once Only函数

```c
pthread_once_t once_control = PTHREAD_ONCE_INIT;

int pthread_once(pthread_once_t *, void (*init_routine)(void));
```

- 不管被调用多少次，仅执行`init_routine`一次
- 所有其他的线程被阻塞，直到`init_routine`函数返回
- 被静态互斥锁淘汰



## 解决问题三

```c
pthread_once_t i_init = PTHREAD_ONCE_INIT;
pthread_mutex_t * i_mutex;

void i_init_routine()
{
    pthread_mutex_init(i_mutex, NULL);
}

void ii_init_routine()
{
    pthread_once(&i_init, i_init_routine);
}
```

*译者注：这是在解决保证锁只初始化一次的问题，如果再用一个互斥锁就套娃了，由此引入了`pthread_once`函数*



## 问题四

```c
int foo = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void foo_ready()
{
    pthread_mutex_lock(&mutex);
    while (foo != 1) {
        pthread_mutex_unlock(&mutex);
        sleep(1);
        pthread_mutex_lock(&mutex);
   }
   pthread_mutex_unlock(&mutex);
}

void foo_initializer()
{
    pthread_mutex_lock(&mutex);
    if (foo == 0)
        foo = 1;	/* This must only be done once. */
    pthread_mutex_unlock(&mutex);
}
```

执行`foo_ready`的线程需要一直等待，直到共享变量的状态改变，但是该线程花了太多时间查询共享变量的状态，而且延时花费也比较高。



## 条件变量

- 条件变量是一个线程等待的变量，直到共享变量的状态被改变
- 在当前线程进入等待的过程中，会原子性地让当前线程进入阻塞状态同时释放互斥锁
- 多个线程可能等待同一个条件变量

*译者注：这里的条件变量是condition variable的意思，共享变量是多个线程可以访问的一个变量。说明可能不是很清楚，请先看一下条件变量的相关api再回顾一下这个解释*



## 创建和销毁条件变量

```c
int pthread_cond_init(pthread_cond_t * cond, pthread_condattr_t *attr);
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int pthread_cond_destroy(pthread_cond_t * cond);
```



## 等待一个条件变量

```c
// 等待的api里传入一个互斥锁，这就是让线程进入阻塞状态时被释放的那个互斥锁
int pthread_cond_wait(pthread_cond_t * cond, pthread_mutex_t * mutex)
int pthread_cond_timedwait(pthread_cond_t * cond, pthread_mutex_t * mutex,
     const struct timespec *abstime)
```



## 唤醒等待一个条件变量的线程

```c
int pthread_cond_signal(pthread_mutex_t * cond)
int pthread_cond_broadcast(pthread_mutex_t * cond)
```



## 解决问题四

```c
int foo = 0;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void foo_ready()
{
    pthread_mutex_lock(&mutex);
    while (foo != 1) {
        pthread_cond_wait(&cond, &mutex);
    }
    pthread_mutex_unlock(&mutex);
}

void foo_initializer()
{
    pthread_mutex_lock(&mutex);
    if (foo == 0)
        foo = 1;	/* This must only be done once. */
    pthread_cond_broadcast(&cond);
    pthread_mutex_unlock(&mutex);
}
```



## 读写锁：多个线程可读，一个线程可写

```c
/* state > 0, multiple readers; state = 0, free */
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int state = 0;

int rwlock(int rw) /* (read = 0, write = 1) */
{
    if (ret = pthread_mutex_lock(&mutex) {
        if (rw) {
            while (state) {
                if (ret = pthread_cond_wait(&cond, &mutex)) {
                    (void)pthread_mutex_unlock(&mutex);
                    return(ret);
                }
            }
        } else {
            state++;
            (void)pthread_mutex_unlock(&mutex);
        }
    }
    return(ret);
}

int rwunlock(int rw) /* (read = 0, write = 1) */
{
    if (!rw) {
        if (ret = pthread_mutex_lock(&mutex) {
            return(ret);
        }
        if (!--state) {
            ret = pthread_cond_broadcast(&cond);
        }
    }
    ret = pthread_mutex_unlock(&mutex);
    return(ret);
}
```

*译者注：这是一个读写锁的实现，用了一个互斥锁和一个条件变量。在读之前调用`rwlock`，如果是读则传入参数0，如果是写则传入参数1，释放读写锁用`rwunlock`，同样，如果读则传入参数0，如果是写则传入参数1*



## 标准缓存

```c
/* It is the data that is shared so you only need one mutex. */
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond_w = PTHREAD_COND_INITIALIZER;
pthread_cond_t cond_r = PTHREAD_COND_INITIALIZER;
int buf_count = 0, buf_r_loc = 0, buf_w_loc = 0;
char buf[1024];

#define INC(x) (++x == 1024) ? (x) = 0 : (x)

char read_buf(void)
{
    char ret;

    pthread_mutex_lock(&mutex);
    while (buf_count == 0) {
        pthread_cond_broadcast(&cond_w);
        pthread_cond_wait(&cond_r, &mutex);
    }
    buf_count--;
    ret = buf[buf_r_loc];
    INC(buf_r_loc);
    pthread_cond_broadcast(&cond_w);
    pthread_mutex_unlock(&mutex);
    return(ret);
}

void write_buf(char ch)
{
    pthread_mutex_lock(&mutex);
    while (buf_count == 1024) {
        pthread_cond_broadcast(&cond_r);
        pthread_cond_wait(&cond_w, &mutex);
    }
    buf_count++;
    buf[buf_w_loc] = ch;
    INC(buf_w_loc);
    pthread_cond_broadcast(&cond_r);
    pthread_mutex_unlock(&mutex);
}
```

*译者注：这是一个standard buffer的实现，写buffer的时候往数组里写数据，如果一次性写入超过1024个字符则进入等待状态，这个buffer是一个循环数组，如果buffer写满了则从头开始写。*



# 用户数据



## 问题一

你在一个古老的软件中看到一个计数函数。每次这个函数被调用的时候，它都会返回下一个数字。现在你想要编写一个多线程软件，你仍然希望这个函数为当前线程返回下一个数字。