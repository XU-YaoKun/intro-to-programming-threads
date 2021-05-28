课程[原地址](http://www.mit.edu/people/proven/IAP_2000/index.html)

这门课程将介绍线程是什么，为什么线程是有用的，以及如何在POSIX 1003.1c线程标准下中使用线程进行编程。



# 课程大纲

- [课程介绍](#课程介绍)
- [基础的线程函数](#基础的线程函数)
- [线程同步](#线程同步)
- [线程私有数据](#线程私有数据)
- [线程属性](#线程属性)
- [线程调度](#线程调度)
- [同步属性](#同步属性)
- [取消](#取消)



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



# 线程私有数据

## 问题一

你在一个古老的软件中看到一个计数函数。每次这个函数被调用的时候，它都会返回下一个数字。现在你想要编写一个多线程软件，你仍然希望这个函数为当前线程返回下一个数字。

```c
static int i = 0;
int counter()
{
    return (++i);
}
```

- 需要一个用线程id作为索引的数组
- 当一个线程退出时，相应的数据需要被清除
- 可以被多个函数调用



## 线程私有的数据

```c
int pthread_key_create(pthread_key_t * key, void (*destructor(void *)) );
int pthread_key_delete(pthread_key_t key);
int pthread_setspecific(pthread_key_t * key, const void *value)
void * pthread_getspecific(pthread_key_t * key)
```

- 每个线程都可以访问`key`这个变量，但是每个线程对应`key`的`value`值是可以不一样的
- 在这个api的实现中，允许最多有`PTHREAD_KEYS_MAX`个`key`
- 在线程结束的时候，如果`value`是非空的，`destructor`会被调用
- 在线程结束的时候，只要`value`是非空的，`destructor`会被持续调用`PTHREAD_DESTRUCTOR_ITERATIONS`次



## 解决问题一

```c
static pthread_mutex_t counter_mutex = PTHREAD_MUTEX_INITIALIZER;
static pthread_key_t counter_key;
static counter_init = 0;

int counter()
{
    int *i;
    P_M_L(&counter_mutex);
    if (counter_init == 0) {
        if (pthread_key_create(&counter_key, free)) {
            counter_init = 1;
            P_M_U(&counter_mutex);
            exit(1);
        }
    } 
    P_M_U(&counter_mutex);

    if ((i = pthread_getspecific(&counter_key)) == NULL) {
        i = malloc(sizeof(int));
        pthread_setspecific(counter_key, &i)
    }
    return(++(*i));
}
```



## 小提示

- `pthread_getspecific()`不会返回错误码，即使输入的`key`是无效的
- 调用规则跟大多数的线程函数不太一样

```c
int pthread_getspecific(pthread_key_t * key, void **)
```



## 清理句柄

- 多线程编程很可能出现资源未释放的问题
- 清理句柄是个不错的工具，保证资源在线程结束之后释放
- 清理句柄应该和异常处理搭配来使用

```c
void pthread_cleanup_push(void (*routine)(void *), void *arg);

void pthread_cleanup_pop(int execute);
```

经常被实现为

```c
struct _pthread_handler_rec {
};

#define pthread_cleanup_push(rtn, arg) {				\
    struct _pthread_handler_rec __cleanup_handler, **__head; 	\
    __cleanup_handler.rtn = rtn;					\
    __cleanup_handler.arg = arg;					\
    head = pthread_getspecific(_pthread_handler_key);		\
    __cleanup_handler.next = *__head;				\
    *head = &__cleanup_handler;

#define pthread_cleanup_pop(ex)						\
    *__head = cleanup_handler.next;					\
    if (ex) cleanup_handler.rtn(__cleanup_handler.arg);		\
  }
```

*译者注：这里说的非常不清楚...总之大概的意思是用来保存一些释放资源的函数避免资源泄漏，详情可以去看看官方文档对`pthread_cleanup_push`和`pthread_cleanup_pop`的解释*



## 线程退出

- 所有的清理句柄会被运行
- 所有的线程私有数据会被析构

```c
extern void (*dest[])(void *);

void __pthread_cleanup(void)
{
    int key, i;
    void * data;

    for (i = 0; i < PTHREAD_DESTRUCTOR_ITERATIONS; i++) {
        for (key = 0; key < PTHREAD_KEYS_MAX; key++) {
            if (data = pthread_getspecific(key) {
                if (data) {
                    pthread_setspecific(key, NULL);
                    if (dest[key]) {
                        dest[key](data);
                    }
                }
            }
        }
    }
}
```

*译者注：这里的代码只描述了析构线程私有数据的部分，还有清理句柄也会被运行但是在这个函数中没有体现出来*



# 线程属性

## 初始化

```c
int pthread_attr_init(pthread_attr_t * attr);
int pthread_attr_destroy(pthread_attr_t * attr);
```

- `pthread_attr_init`函数生成一个默认的线程属性，用这个函数返回的`attr`来创建线程相当于在`pthread_create`函数的第二个参数传入`NULL`，可以用这个函数找到默认的线程属性。
- 没有静态的线程属性初始化器（*译者注：有静态的互斥锁初始化器*）



## 分离状态

```c
int pthread_attr_getdetachstate(pthread_attr_t * attr, int * state);
int pthread_attr_setdetachstate(pthread_attr_t * attr, int state);
```

- `state`的返回状态为`PTHREAD_CREATE_DETACHED`或者`PTHREAD_CREATE_JOINABLE`.
- 默认状态为`PTHREAD_CREATE_JOINABLE`.
- 没有运行时的方法来获取线程的分离状态.



## 栈属性

```c
int pthread_attr_getstacksize(pthread_attr_t * attr, size_t * stacksize);
int pthread_attr_setstacksize(pthread_attr_t * attr, size_t stacksize);
int pthread_attr_getstackaddr(pthread_attr_t * attr, void ** stackaddr);
int pthread_attr_setstackaddr(pthread_attr_t * attr, void * stackaddr);
```



## 清理

```c
int pthread_attr_setcleanup(pthread_attr_t * attr, void * arg);
```

- 用于清理传入线程的参数所占用的内存。
- 不是POSIX标准下的函数。



# 线程调度

## 调度空间

- 线程与当前调度空间中的其他线程争夺资源，例如CPU.
- 有两种调度空间，一种是`PTHREAD_SCOPE_SYSTEM`，表示内核线程，一种是`PTHREAD_SCOPE_PROCESS`，表示用户进程。
- 大多数`pthread`标准的实现只支持一种调度空间。
- 只可以在线程创建的时候指定。



## 调度策略和优先级

- 最常支持的几种，`SCHED_RR`、`SCHED_FIFO`、`SCHED_OTHER`.
- Round Robin和FIFO正如名称所说。
- `OTHER`线程的优先级通常用于公平的调度器。
- 调度策略和优先级可以在线程创建或者动态地指定。

```c
int pthread_attr_getschedpolicy(pthread_attr_t * attr, int * policy);
int pthread_attr_setschedpolicy(pthread_attr_t * attr, int policy);
int pthread_attr_getschedparam(pthread_attr_t * attr, struct sched_param * param);
int pthread_attr_setschedparam(pthread_attr_t * attr, struct sched_param * param);
int pthread_getschedparam(pthread_t, int * policy, struct sched_param * param);
int pthread_setschedparam(pthread_t, int * policy, struct sched_param * param);
```

```c
int sched_get_priority_max(int policy);
int sched_get_priority_min(int policy);
```

- 用来返回当前调度策略下最高的优先级。
- 支持常用的优先级策略。



## Yield

```c
int sched_yield(void);
```

- `SCHED_FIFO`，`SCHED_RR`仅仅将CPU的控制权交给同等优先级的线程。
- `SCHED_OTHER`有更接近有`UNIX`风格的行为。



## 调度策略继承

```c
int pthread_attr_getinheritsched(pthread_attr_t *, int * inherit);
int pthread_attr_setinheritsched(pthread_attr_t *, int inherit);
```

- PTHREAD_INHERIT_SCHED属性使得创建线程时忽略指定的线程调度属性，使用从父线程继承的调度属性。
- 默认是`PTHREAD_EXPLICIT_SCHED`.



# 同步属性

## 初始化

```c
int pthread_mutexattr_init(pthread_mutexattr_t * attr);
int pthread_mutexattr_destroy(pthread_mutexattr_t * attr);
int pthread_condattr_init(pthread_condattr_t * attr);
int pthread_condattr_destroy(pthread_condattr_t * attr);
```

- 用初始化函数创建的`attr`为默认属性。
- 没有静态的属性初始化器。



## 进程共享

```c
int pthread_mutexattr_getpshared(pthread_mutexattr_t * attr, int * state);
int pthread_mutexattr_setpshared(pthread_mutexattr_t * attr, int state);
int pthread_condattr_getpshared(pthread_condattr_t * attr, int * state);
int pthread_condattr_setpshared(pthread_condattr_t * attr, int state);
```

- `PTHREAD_PROCESS_SHARED`属性表示可以用当前互斥锁或条件变量让多个进程同步。
- 使用者负责将互斥锁或条件变量的内存映射到进程空间。
- 默认为`PTHREAD_PROCESS_PRIVATE`.

*译者注：在初始化mutex或者cond的时候需要传入一个attr指针，这里说的属性就是指这个attr指针，可以将这个attr的shared状态设置为`PTHREAD_PROCESS_SHARED`或者`PTHREAD_PROCESS_PRIVATE`表示锁或条件变量可以进程中共享或进程私有*



## 互斥锁类型

```c
int pthread_mutexattr_gettype(pthread_mutexattr_t * attr, int * type);
int pthread_mutexattr_settype(pthread_mutexattr_t * attr, int type);
```

- 基本的互斥锁类型有`PTHREAD_MUTEXTYPE_RECURSIVE`，`PTHREAD_MUTEXTYPE_DEBUG`等.

*译者注：可以参考[官方文档](https://linux.die.net/man/3/pthread_mutexattr_gettype)，有很多种类型的锁，比如是否检查死锁，比如是否可以在已经上锁的情况下继续上锁等*



```c
int pthread_condattr_gettype(pthread_condattr_t * attr, int * type);
int pthread_condattr_settype(pthread_condattr_t * attr, int type);
```

- 基本的条件变量的种类有`PTHREAD_CONDTYPE_RECURSIVE`等.



# 取消

## 取消线程

```c
int pthread_cancel(pthread_t);
```

- 对一个特定的线程注册取消事件.
- 无法注销取消事件.
- 不需要等待线程终止.
- 对一个被取消的线程使用`join`函数时，会返回`PTHREAD_CANCELLED`状态.



## 控制取消属性

- 一个线程必须处于三种可取消的状态之一：`disabled`、`deferred`、`asynchronous`
- 一个线程总是以`deferred`状态开始其生命周期.

```c
int pthread_setcancelstate(int state, int * old_state);
int pthread_setcanceltype(int type, int * old_type);
```

- 只有两种取消状态(cancellation state)，`PTHREAD_CANCEL_ENABLE`、`PTHREAD_CANCEL_DISABLE`

- 只有两种取消类型，`PTHREAD_CANCEL_ASYNCHRONOUS`、`PTHREAD_CANCEL_ASYNCHRONOUS`



## 延迟取消

- 延迟取消只能在某个特定的时间发生.
- 通常，阻塞当前线程的系统调用是取消当前线程的时间点.
- 互斥锁永远不是一个取消当前线程的时间点。

```c
void pthread_testcancel(void);
```

- `pthread_testcancel`是一个取消当前线程的时间点.
- 如果取消被停用，则这个函数什么都不做.
- 对于一个取消事件，没有无损的测试函数.





