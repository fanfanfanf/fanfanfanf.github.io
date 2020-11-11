[toc]

### 1. std::thread
创建线程
```
{
  explicit thread( Function&& f, Args&&... args );  // 构造函数
  thread(const thread&) = delete;
}
std::thread t1(f, n);    // 创建线程
t1.joinable();           // 检查线程是否可合并，即潜在地运行于平行环境中
t1.get_id();             // 返回线程的 id 
native_handle_type tmp = t1.native_handle();      // 返回实现定义的底层线程句柄。
unsigned int n = std::thread::hardware_concurrency();  // 返回实现所支持的并发线程数。应该只把值当做提示。
t1.join();      // 阻塞当前线程，直至t1 所标识的线程完成其执行。
t1.detach();    // 从 thread 对象分离执行的线程，允许执行独立地持续。一旦线程退出，则释放所有分配的资源。
                // 调用 detach 后， *this 不再占有任何线程。
t1.swap(t2);    // 互换二个 thread 对象的底层句柄。
native_handle_type native_handle();  // 返回实现定义的底层线程句柄。
```
[在类中创建](http://www.cnblogs.com/c4isr/p/9205164.html)

### 2. std::this_thread
#### yield
请求调度，让出CPU。提供提示给实现，以重调度线程的执行，允许其他线程运行。  
此函数的准确行为依赖于实现，特别是使用中的 OS 调度器机制和系统状态。例如，先进先出实时调度器（ Linux 的 SCHED_FIFO ）将悬挂当前线程并将它放到准备运行的同优先级线程的队列尾（而若无其他线程在同优先级，则 yield 无效果）。
```
// 时间到达end前，一直让出CPU
void little_sleep(std::chrono::microseconds us)
{
    auto start = std::chrono::high_resolution_clock::now();
    auto end = start + us;
    do {
        std::this_thread::yield();
    } while (std::chrono::high_resolution_clock::now() < end);
}
```

#### sleep_for
阻塞当前线程执行，至少经过指定的 sleep_duration 。  
此函数可能阻塞长于 sleep_duration ，因为调度或资源争议延迟。  
标准库建议用稳定时钟度量时长。若实现用系统时间代替，则等待时间亦可能对始终调节敏感。  
```
auto start = std::chrono::high_resolution_clock::now();
std::this_thread::sleep_for(2s);
auto end = std::chrono::high_resolution_clock::now();
std::chrono::duration<double, std::milli> elapsed = end-start;
std::cout << "Waited " << elapsed.count() << " ms\n";
```

#### sleep_until
阻塞当前线程，直至抵达指定的 sleep_time 。  
使用联倾向于 sleep_time 的时钟，这表示时钟调节有影响。从而在调用时间点后，阻塞的时长可能小于，但不会多于 sleep_time - Clock::now() 。函数亦可能阻塞长于抵达 sleep_time 之后，由于调度或资源争议延迟。

### 3. 缓存大小访问
std::hardware_destructive_interference_size(Cpp17): 避免假共享的最小偏移 std::hardware_constructive_interference_size(C++17):促使真共享的最大偏移 

### 4. 互斥锁和[条件变量](https://blog.csdn.net/tgxallen/article/details/73522233)  

```C
std::mutex mut_;                 // 创建互斥锁
std::condition_variable cond_;   // 创建条件变量
std::atomic<bool> is_stop_;       // 类型的变量可以在多线程中不加锁操作。

// std::unique_lock相对std::lock_guard更灵活的地方在于等待中的线程，
// 可以解锁mutex，并在之后重新将其锁定。而std::lock_guard却不具备这样的功能，也就是不能用于条件变量
std::unique_lock<std::mutex> lk1(mut_);     // 对互斥锁mut_上锁，自动解锁
std::lock_guard<std::mutex> lk2(mut_);      // 对互斥锁mut_上锁，自动解锁，占用空间更小

mut_.lock();     // 对mut_上锁，需要手动解锁
                 // 若另一线程已锁定互斥，则到 lock 的调用将阻塞执行，直至获得锁。
mut_.trylock();  // 尝试锁定互斥，若互斥不可用则返回false
mut_.unlock();   // 解锁

cond_.wait(lk1, [this]{return !que_.empety();});  // 等待条件变量，只能用lk1！
cond_.wait_for(...)  // 指定时限时长
cond_.wait_until(...)  // 指定抵达时间点 
cond_.notify_one();   // 通知一个等待的线程
cond_.notify_all();   // 通知所有等待的线程 

std::timed_mutex mut_;  // 类似普通mutex，多了以下操作
try_lock_for();     // 尝试锁定互斥，若互斥在指定的时限时期中不可用则返回 
try_lock_until();   // 尝试锁定互斥，若直至抵达指定时间点互斥不可用则返回 

std::recursive_mutex   // 提供能被同一线程递归锁定的互斥设施
std::shared_mutex      //(C++14) 用于多个读线程能同时访问同一资源而不导致数据竞争，但只有一个写线程能访问的情形。
std::shared_lock<std::shared_mutex> lock(mutex_);  // 读加锁
std::unique_lock<std::shared_mutex> lock(mutex_);  // 写加锁

// wait 函数可以接受任何 lockable参数，而 std::condition_variable只能接受 std::unique_lock类型的参数
std::condition_variable_any cond2_;  

scoped_lock();  // (C++17)多互斥的免死锁 RAII 封装器(待补充)
```

### 5. 单次调用
当有多个线程调用的是同一个函数时，只调用一次，也就是只有一个线程会执行。（其实也不一定是同一个函数，只要flag是同一个就行）
- 若在调用 call_once 的时刻， flag 指示**已经调用**了 f （通过flag判断），则 call_once 立即返回（称这种对 call_once 的调用为消极）。
- **否则**， call_once 以参数 std::forward<Args>(args)... 调用(f) （如同用 std::invoke ）。不同于 std::thread 构造函数或 std::async ，不移动或复制参数，因为不需要转移它们到另一执行线程（称这种对 call_once 的调用为积极）。
- 若该调用**抛出异常**，则传播异常给 call_once 的调用方，并且**不翻转 flag** ，以令其他调用将得到尝试（这种对 call_once 的调用被称为异常）。
- 若该调用**正常返回**（这种对 call_once 的调用被称为返回），则**翻转 flag** ，并保证以同一 flag 对 call_once 的其他调用为消极。
```
void call_once( std::once_flag& flag, Callable&& f, Args&&... args );   // 函数原型
std::once_flag flag1, flag2;  // 创建flag
```

### 2. 访问异步操作结果
使用std::async创建线程，可返回线程的执行结果。  
std::async的原型async(std::launch::async | std::launch::deferred, f, args...)，第一个参数是线程的创建策略，有两种策略，默认的策略是立即创建线程：
* std::launch::async：在调用async就开始创建线程。
* std::launch::deferred：延迟加载方式创建线程。调用async时不创建线程，直到调用了future的get或者wait时才创建线程。
```
std::future<int> f1 = std::async(std::launch::async, [](){ 
        return 8;  
    }); 

cout<<f1.get()<<endl; //output: 8

std::future<int> f2 = std::async(std::launch::async, [](){ 
        cout<<8<<endl;
    }); 

f2.wait(); //output: 8

std::future<int> future = std::async(std::launch::async, [](){ 
        std::this_thread::sleep_for(std::chrono::seconds(3));
        return 8;  
    }); 
 
    std::cout << "waiting...\n";
    std::future_status status;        // future_status有三种状态
    do {
        status = future.wait_for(std::chrono::seconds(1));
        if (status == std::future_status::deferred) {    // deferred：异步操作还没开始
            std::cout << "deferred\n";
        } else if (status == std::future_status::timeout) {   // timeout：异步操作超时
            std::cout << "timeout\n";
        } else if (status == std::future_status::ready) {    // ready：异步操作已经完成
            std::cout << "ready!\n";
        }
    } while (status != std::future_status::ready); 
 
    std::cout << "result is " << future.get() << '\n';
可能的结果：
waiting...
timeout
timeout
ready!
result is 8
```
[std::async](https://blog.csdn.net/yockie/article/details/50595958)封装了std::promise和std::packaged_task，他们内部都封装了future以便访问异步操作结果，std::packaged_task包装的是一个异步操作，而std::promise包装的是一个值，都是为了方便异步操作的，因为有时我需要获取线程中的某个值，这时就用std::promise，而有时我需要获一个异步操作的返回值，这时就用std::packaged_task。
1. std::promise
```
    std::promise<int> pr;
    std::thread t([](std::promise<int>& p){ p.set_value_at_thread_exit(9); },std::ref(pr));
    std::future<int> f = pr.get_future();
    auto r = f.get();
```
2. std::packaged_task
```
    std::packaged_task<int()> task([](){ return 7; });
    std::thread t1(std::ref(task)); 
    std::future<int> f1 = task.get_future(); 
    auto r1 = f1.get();
```

### 死锁的调试方式
[调试](https://www.cnblogs.com/zhuyp1015/p/3618863.html)