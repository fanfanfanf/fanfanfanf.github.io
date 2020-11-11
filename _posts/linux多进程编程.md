[toc]
## 1.进程运行
#### 1. 进程创建与退出  
创建：
```
pid_t getpid(void);
pid_t fork(void);  // 复制一个完全相同的程序，子进程从此句开始执行。  
pid_t vfork(void); // 执行子进程时阻塞父进程；共享父进程的栈内容。
```
退出：
```
void exit(int status)　// 退出进程,退出时会调用atexit注册的函数，先注册的后调用
                       // exit函数还会按需调用fclose函数关闭打开的文件流
int atexit(void(*func)(void))    //  为进程注册退出时调用的函数
void _exit(int status)     　　  // 直接退出进程
```
`fork()`和`vfork()`的区别：  
todo：  


#### 2. 加载可执行文件映像

```
int execl(const char *path,const char *arg,...);
int execle(const char *path,const char *arg,...,char *const envp[]);  
int execlp(const char *file,const char *arg,...);                      
int execv(cosnt char *path,char *const argv[]);
int execve(const char *path,char *const argv[],char *const envp[]);
int execvp(const char *file,char *const argv[]);
```
l表示命令行参数为以0结束的多个字符串组成  
v表示命令行参数为以0结束的字符串数组组成  
e表示指定环境表量，原来的环境变量不起作用  
p表示可执行映像文件在环境变量path路径中查找  
例如：
```
execl("bin/ls", "ls", "/home", "NULL");
```
此句后面的语句不再执行；不产生新的pid，即只搬代码

#### 3. 等待子进程结束
```
pid_t wait(int * status)   //挂起进程，直到一个子进程结束,返回结束进程的pid，status存放返回值
pid_t waitpid(pid_t pid,int *status,int options)  //等待指定子进程结束
```
waitpid中：  
pid的意义：  
pid<-1     等待pid所代表的进程组中的进程  
pid=-1     等待任何子进程  
pid=0      等待与该进程同组的进程  
pid>0      等待的进程标识  
options的意义：  
WNOHANG      //表示不阻塞  
WUNTRACED   //当有子进程结束时返回


## 2. 进程间通信
#### 1. 无名管道 pipe
1. 无名管道的特点  
   1. 只能用于具有亲缘关系的进程之间，父子进程，兄弟进程之间通信，因为只有子进程才能继续父进程的文件描述符。
   2. 半双共通信（同一时刻只能对管道进行一种操作（读操作或者写操作）），具有读端口和写端口。
   3. 管道是一种特殊的文件，可以使用文件io函数（read,write...）来操作，但不能使用lseek函数来定位操作。
   4. 管道是在内存中，不用我们主动区删除。
   5. 管道是基于队列实现的，有大小限制。
2. pipe函数  
`int pipe(int pipefd[2]);`
   1. pipe函数创建了一个单向数据通道，这个通道可以用来在进程之间通信。
   2. pipefd: pipefd数组用来返回两个文件描述符，这两个文件描述符代表了通道的两端。pipefd[0]代表通道的读取端，pipefd[1]代表了通道的写入段。在读取端读取数据，在输写入端口写入数据。读取空内容或写满都会阻塞。
   3. 在用pipefd[0]来读数据时，要先关闭pipefd[1];在用pipefd[1]来写入数据时，要关闭pipefd[0]。
3. 返回值  
函数调用成功返回0  
调用失败返回-1，这时errno存储错误码。   

#### 2. 有名管道 fifo
有名管道可以不同进程访问
```
int mkfifo(const char *pathname, mode_t mode);
```
创建一个fifo文件，文件名为pathname，fifo文件只能以WRONLY或RDONLY方式打开

#### 3. 信号 signal

```
int kill(pid_t pid, int sig);  // 向pid发送信号sig

typedef void (*sighandler_t)(int);     // 信号处理函数的类型，参数是信号sig
sighandler_t signal(int signum, sighandler_t handler);  // 注册信号处理函数
// handler也可以取SIG_IGN(忽略)，SIG_DFL(内核处理)。
```

#### 4. 消息队列

#### 5. 共享内存

#### 6. 信号量
信号量表示某个资源（如文件）的可访问性，类似互斥锁  
可以理解成计数器，信号量会有初值（>0），每当有进程申请使用信号量，通过一个P操作来对信号量进行-1操作，当计数器减到0的时候就说明没有资源了，其他进程要想访问就必须等待，当该进程执行完这段工作（我们称之为临界区）之后，就会执行V操作来对信号量进行+1操作。
1. 创建信号量
```
int semget(key_t key, int nsems, int semflg);  // 获取信号量集合的标识符
key_t ftok(const char *pathname, int proj_id);  // 创建信号量键值
```
key：信号量的键值  
nsems：创建信号量集合的个数  
semflg：IPC_CREAT 和 IPC_EXCL  
2. 操作信号量
```
int semop(int semid, struct sembuf *sops, unsigned nsops);
int semtimedop(int semid, struct sembuf *sops, unsigned nsops,
                      struct timespec *timeout);
```
semid：由semget()得到的信号量集合  
nsops：操作的信号量个数  
sops：每个信号量的操作内容:  

   
3. 信号量初始化
```
int semctl(int semid, int semnum, int cmd, ...);
```

#### 7. 套接字