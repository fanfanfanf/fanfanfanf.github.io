[TOC]
## 1.线程
#### 1. 线程号
> pthread_t pthread_self();

返回本线程的线程号  

#### 2. 创建线程
``` C
int pthread_create(
        pthread_t *thread,
        const pthread_attr_t *attr,
        void *(*start_routine)(void *),
        void *arg
);
```
创建一个新线程  
thread        用于线程号的回填  
attr          线程属性  
start_routine 回调函数  
arg           回调函数参数  

#### 3. 主线程
1. 主线程的参数是argc，argv，普通线程参数只能是void*
2. 主线程在默认堆栈上生长，普通线程堆栈受限

#### 4. joinable和detached
[属性设置](####11.-线程属性)见11.
1. joinable：  
能被其他线程杀死和回收资源；  
线程结束后一直占用资源，直到被其他线程回收
2. detached：  
终止时资源立刻被系统收回  
注意结束前释放互斥量、动态内存等。 
3. 线程连接
```
int pthread_join(pthread_t thread, void **retval);
```
连接pid线程，阻塞当前线程，直到pid线程结束，rval用于回填返回值
4. 线程分离
```
int pthread_detach(pthread_t thread);
```

#### 5. 结束线程
`return;`结束线程及其子线程  
`pthread_exit(void *rval);` 退出当前线程  

#### 6. 线程取消
``` C++
int pthread_cancel(pthread_t thread);
```
发送终止信号给thread线程，如果成功则返回0，否则为非0值。发送成功并不意味着thread会终止。
``` C++
int pthread_setcancelstate(int state,   int *oldstate)
```
设置本线程对Cancel信号的反应  
state有两种值：  
PTHREAD_CANCEL_ENABLE：（缺省）   
PTHREAD_CANCEL_DISABLE： 会导致**延迟**所有的取消请求  
old_state如果不为NULL则存入原来的Cancel状态以便恢复。
```
int pthread_setcanceltype(int type, int *oldtype)
```
type有两种取值：  
PTHREAD_CANCEL_DEFFERED: 到下一个取消点退出  
PTHREAD_CANCEL_ASYCHRONOUS：立即退出  
**取消点**:  
* `void pthread_testcancel(void)`专门用作取消点的函数  
* 阻塞函数和一些标准库函数等

#### 7. 线程清理
线程在占用临界资源时如果被取消，则临界资源得不到释放  
可将占用和解除占用临界资源的代码段写在如下两个函数之间：
```
void pthread_cleanup_push(void (*routine) (void *), void *arg)
void pthread_cleanup_pop(int execute)
```
· 如果线程在这两个之间退出或被取消，则会执行`routine(arg)`函数。  
· 如果execute != 0， 运行到`pthread_cleanup_pop`时也会执行`routine(arg)`函数。

#### 8. 信号
1. 发送信号
```
int pthread_kill(pthread_t thread, int sig);
```
类比进程中的信号`kill(pid, sig);`  
sig如果是0，则不发送信号，只返回是否成功，用于检测线程是否存活  
2. 处理信号
```
int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
```
(进程通信中的信号处理也是用这个函数)  
signum: 信号代码  
act: 信号处理相关结构体，其中`act.sa_mask`是信号屏蔽字，`act.sa_handler`为信号处理程序
3. 相关函数
```
int sigemptyset(sigset_t *set);  // 清空信号集
int sigfillset(sigset_t *set     // 将所有信号加入信号集
int sigaddset(sigset_t *set, int signum);    // 增加一个信号到信号集
int sigdelset(sigset_t *set, int signum);    // 删除一个信号到信号集
int sigismember(const sigset_t *set, int signum);
```
4. 信号屏蔽
```
int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset); // 线程
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);     // 进程
```
how =   
SIG_BLOCK:   向当前的信号掩码中添加set，set表示要阻塞的信号组。  
SIG_UNBLOCK: 向当前的信号掩码中删除set  
SIG_SETMASK: 将当前的信号掩码替换为set  
· 子线程会继承父线程的信号掩码  
· SIGKILL和SIGSTOP不能忽略  
* 不同线程共用一个sigaction处理函数；但是不共用一个pthread_sigmask。  

#### 9. 线程同步
##### 1. 互斥量  
相关函数：
```C
// 创建和删除互斥量
// 静态分配互斥量，可以将其置为常量PTHREAD_MUTEX_INITIALIZER，或使用pthread_mutex_init初始化
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;  // 互斥量的声明
// 动态分配的互斥量必需使用以下两个函数初始化和释放
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);

// 加锁和解锁
int pthread_mutex_lock(pthread_mutex_t *mutex);     // 如果互斥量已被锁，则会阻塞
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
使用流程:
```C
// main thread
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);

// other thread
pthread_mutex_lock(&mutex);
/******  code of using parameter ******/
pthread_mutex_unlock(&mutex);
```
* 锁是对于线程的，不是对于变量的：如果一个线程通过`pthread_mutex_lock()`上锁，那么另一个线程执行`pthread_mutex_lock()`时必会阻塞，无论第二个线程是否使用临界资源；如果第二个不执行`pthread_mutex_lock()`，即使第一个线程上锁了，依旧可以访问临界资源。

##### 2. 读写锁  
读写锁可以允许多个线程同时读取一个变量
* 读写锁的三个状态:  
读模式：多个线程可同时占有读写锁；其他线程可进行**读加锁**；线程此时进行**写加锁**，会阻塞随后的**读加锁**请求，并阻塞自己直到锁被释放  
写模式：仅一个线程可占有读写锁；其他线程进行**读加锁**或**写加锁**请求都会阻塞  
不加锁。
* 读写锁适合读次数大于写次数的数据  
```
// 创建读写锁
pthread_rwlock_t rwlock;

// 初始化读写锁
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
              const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);

// 读模式加锁
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);

// 写模式加锁
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);

// 解锁
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```
使用流程类似互斥量  

##### 3. 条件变量
（当不满足条件时，）释放锁，并阻塞，直到检测到满足的条件信号，上锁，结束wait。（pthread_cond_wait()的作用）
```
// 创建条件变量
pthread_cond_t cond;

// 初始化和销毁
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;       // 静态初始化
int pthread_cond_init(pthread_cond_t *restrict cond,  // 动态初始化
              const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);

// 等待条件变量信号
int pthread_cond_wait(pthread_cond_t *restrict cond,   // 条件变量和互斥锁组合使用
              pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,  // 加入了timeout(绝对时间)
              pthread_mutex_t *restrict mutex,
              const struct timespec *restrict abstime);

// 发送条件变量信号
int pthread_cond_signal(pthread_cond_t *cond);  // 向一个线程发送(系统决定)
int pthread_cond_broadcast(pthread_cond_t *cond);  // 向所有线程发送
```

#### 10. 一次性初始化
为了防止互斥量、读写锁、条件变量等被多次初始化
```
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control,
              void (*init_routine)(void));
```
定义一个初始化函数，并作为第二个参数传入`pthread_once()`，则在多个位置调用`pthread_once()`时，只执行一次。

#### 11. 线程属性
```
pthread_attr_t attr;

int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
```
包括三个属性：
  * detached：线程的分离状态
  * guardsize：线程栈末尾的警戒区域大小（字节数）
  * stacksize：线程栈的最低地址
  * stacksize：线程栈的大小（字节数）

1. 分离状态
```
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(pthread_attr_t *attr, int *detachstate);
```

使用流程
```
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED); // 或PTHREAD_CREATE_JOINABLE
pthread_create(&tid, &attr, thread_fun, NULL);
```

2. 栈属性-起始地址和大小
```
// 改变栈的起始地址和大小
int pthread_attr_setstack(pthread_attr_t *attr,
                          void *stackaddr, size_t stacksize);
int pthread_attr_getstack(pthread_attr_t *attr,
                          void **stackaddr, size_t *stacksize);
```
stackaddr是栈的最低地址，如果栈是从高往低的，则stackaddr是栈的结尾。  
stacksize是栈的大小，不能小于PTHREAD_STACK_MIN(limit.h)；可在线程创建后修改。
```
// 只修改栈的大小
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_getstacksize(pthread_attr_t *attr, size_t *stacksize);
```
* 编译阶段可通过_POSIX_THREAD_ATTR_STACKADDR和_POSIX_THREAD_ATTR_STACKSIZE符号来检查系统是否支持线程栈属性(使用#IFDEF)，在`/usr/include/bits/posix_opt.h`中。  
3. 栈属性-警戒区  
guardsize控制线程末尾以后用以避免栈溢出的扩展内存大小，默认是PAGESIZE个字节
```
int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
int pthread_attr_getguardsize(pthread_attr_t *attr, size_t *guardsize);
```

#### 12. 互斥量属性
```
pthread_mutexattr_t attr;

int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);
```
1. 进程属性：  
PTHREAD_PROCESS_PRIVATE:默认值，同一进程中的多个线程访问同一同步对象  
PTHREAD_PROCEXX_SHARED: 可以多个进程同步
```
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr,
              int pshared);
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *
              restrict attr, int *restrict pshared);
```
* 检测宏_POSIX_THREAD_PROCESS_SHARED检测系统是否支持
2. 类型属性：  

| 互斥量类型 | 未解锁时加锁 | 未加锁时解锁 | 已解锁时解锁 |
|:-----:|:-----:|:-----:|:-----:|
| PTHREAD_MUTEX_NORMAL | 死锁 | 未定义 | 未定义 |
| PTHREAD_MUTEX_ERRORCHEK | 返回错误 | 返回错误 | 返回错误 |
| PTHREAD_MUTEX_RECURSIVE | 允许 | 返回错误 | 返回错误 |
| PTHREAD_MUTEX_DEFAULT | 未定义 | 未定义 | 未定义 |
PTHREAD_MUTEX_RECURSIVE只允许在同一线程内多次加锁
```
int  pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
int  pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict
       attr, int *restrict type);
```

#### 13. 读写锁和条件变量属性
读写锁:
```
pthread_rwlockattr_t attr;

int pthread_rwlockattr_init(pthread_rwlockattr_t *attr);
int pthread_rwlockattr_destroy(pthread_rwlockattr_t *attr);

// 只有进程属性，参考互斥锁
int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t *
              restrict attr, int *restrict pshared);
int pthread_rwlockattr_setpshared(pthread_rwlockattr_t *attr,
              int pshared);
```

条件变量：
```
pthread_condattr_t attr;

int pthread_condattr_init(pthread_condattr_t *attr);
int pthread_condattr_destroy(pthread_condattr_t *attr);

int pthread_condattr_setpshared(pthread_condattr_t *attr,
              int pshared);
int pthread_condattr_getpshared(const pthread_condattr_t *restrict
       attr, int *restrict pshared);
```

#### 14. 线程私有数据
```
pthread_key_t key;

int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));
int pthread_key_delete(pthread_key_t key);

int pthread_setspecific(pthread_key_t key, const void *value);
void *pthread_getspecific(pthread_key_t key);
```
不同的线程可使用pthread_key_create()，将同一个key和不同的value关联，当调用pthread_getspecific(key)时，返回的是不同的value的值。

