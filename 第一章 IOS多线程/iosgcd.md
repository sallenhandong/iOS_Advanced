![](https://user-gold-cdn.xitu.io/2018/8/23/1656712e9712aac8?w=5184&h=3456&f=jpeg&s=2882507)
>`Grand Central Dispatch（GCD）`是异步执行任务的技术之一。一般将应用程序中记述的线程管理用的代码在系统级中实现。开发者只需要定义想执行的任务并追加到适当的`Dispatch Queue`中，`GCD`就能生成必要的线程并计划执行任务。由于线程管理是作为系统的一部分来实现的，因此可统一管理，也可执行任务，这样就比以前的线程更有效率。  

本文会以图文并茂的形式介绍GCD的常用api基础及线程安全相关，篇幅会比较长。
### GCD常用函数
GCD中有2个用来执行任务的函数  
#### 同步方式
`dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);`
#### 异步方式
`dispatch_async(dispatch_queue_t queue, dispatch_block_t block);`  
### 什么是同步?
在当前线程中执行任务，不具备开启新线程的能力
### 什么是异步?
在新的线程中执行任务，具备开启新线程的能力
### 什么是并发?
多个任务并发（同时）执行
### 什么是串行?
一个任务执行完毕后，再执行下一个任务
### 如下表格所示
|         | 并发队列   |  串行队列  |  主队列  |
| --------   | -----:  | :----:  | :----:  |
| 同步(sync) | 没有开启新线程 |  没有开启新线 |  没有开启新线  |
|            | `串行`执行任务 |  `串行`执行任务    | `串行`执行任务   |
| 异步(async)|  开启新线程   |  开启新线程  |没有开启新线程   |
|            | `并发`执行任务 |  `串行`执行任务  | `串行`执行任务 |
### 下面是本文会讲到的内容
#### GCD的API
- ## `Dispatch Queue ` 
>DispatchQueue manages the execution of work items. Each work item submitted to a queue is processed on a pool of threads managed by the system.

![](https://user-gold-cdn.xitu.io/2018/8/23/1656603a5320e9a9?w=624&h=202&f=png&s=15180)
| Dispatch Queue 有两种|  |
| --------   | -----:  |
| `Serial Dispatch Queue` | 等待现在执行中处理结束 |
| `Concurrent Dispatch Queue` | 不等待现在执行中处理结束 |


![](https://user-gold-cdn.xitu.io/2018/8/23/165661158c656771?w=472&h=202&f=png&s=12153)
![](https://user-gold-cdn.xitu.io/2018/8/23/165661165fc9f384?w=472&h=202&f=png&s=14879)
`Serial Dispatch Queue` 使用一个线程,我们通过代码来看一下。
```
dispatch_queue_t serialQueue = dispatch_queue_create("com.slim.www", DISPATCH_QUEUE_SERIAL);
dispatch_async(serialQueue, ^{
NSLog(@"1");
});
dispatch_sync(serialQueue, ^{
NSLog(@"2");
});
dispatch_async(serialQueue, ^{
NSLog(@"3");
});
```
因为不用等待执行中的处理结束，所以会依次向下打印。结果为1、2、3
为了再次证实串行队列中只有一个线程执行任务，我们再来看一段代码
```
dispatch_queue_t serialQueue = dispatch_queue_create("com.slim.www", DISPATCH_QUEUE_SERIAL);
for (int i = 0; i< 10;i++){
dispatch_async(serialQueue, ^{
NSLog(@"%@--%d",[NSThread currentThread], i);
});
}
```
打印结果为：
```
2018-08-23 17:32:17.490879+0800 gcdTest[1447:249018] <NSThread: 0x6000004605c0>{number = 3, name = (null)}--0
2018-08-23 17:32:17.491318+0800 gcdTest[1447:249018] <NSThread: 0x6000004605c0>{number = 3, name = (null)}--1
2018-08-23 17:32:17.492612+0800 gcdTest[1447:249018] <NSThread: 0x6000004605c0>{number = 3, name = (null)}--2
2018-08-23 17:32:17.492777+0800 gcdTest[1447:249018] <NSThread: 0x6000004605c0>{number = 3, name = (null)}--3
.....
```
我们可以发现thread的地址是一样的，那就证实了`serial queue`只有一个线程执行任务  
#### 关于Serial Dispatch Queue生成个数的注意事项
- 当生成多个`Serial Dispatch Queue`时，各个`Serial Dispatch Queue`将并行执行。
- 如果生成多个`Serial Dispatch Queue`，那么就会消耗大量内存，引起大量的上下文切换，从而影响性能。
如下图所示：

![](https://user-gold-cdn.xitu.io/2018/8/23/16566d4d1d780624?w=503&h=301&f=png&s=24687)

![](https://user-gold-cdn.xitu.io/2018/8/23/16566d7c927f5cfb?w=504&h=371&f=png&s=18686)
`Concurrent Dispatch Queue` 使用多个线程同时执行多个处理，并行执行的处理数量由当前系统的状态决定。
```
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.slim.www", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrentQueue, ^{
NSLog(@"1");
});
dispatch_sync(concurrentQueue, ^{
NSLog(@"2");
});
dispatch_async(concurrentQueue, ^{
NSLog(@"3");
});
```
打印结果为：
```

2018-08-23 17:38:54.330096+0800 gcdTest[1519:258011] 2
2018-08-23 17:38:54.330096+0800 gcdTest[1519:258048] 1
2018-08-23 17:38:54.330441+0800 gcdTest[1519:258048] 3
```
我们再来证实一下
```
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.slim.www", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i< 10;i++){
dispatch_async(concurrentQueue, ^{
NSLog(@"%@--%d",[NSThread currentThread], i);
});
}
```
打印结果为：
```
<NSThread: 0x6040002754c0>{number = 3, name = (null)}--0
2018-08-23 17:44:00.487048+0800 gcdTest[1585:265970] <NSThread: 0x604000275580>{number = 6, name = (null)}--3
2018-08-23 17:44:00.487104+0800 gcdTest[1585:265971] <NSThread: 0x604000275540>{number = 5, name = (null)}--2
2018-08-23 17:44:00.487112+0800 gcdTest[1585:265968] <NSThread: 0x600000461240>{number = 4, name = (null)}--1
....
```
可以看出并发队列中是有多个线程执行任务的。
- ## `dispatch_queue_create `
- `dispatch_queue_create`函数可生成`Dispatch Queue`
```
dispatch_queue_t queue = dispatch_queue_create("read_queue", DISPATCH_QUEUE_CONCURRENT);
```
- 第一个参数为queue名称，第二个参数为类型
- ## `Main Dispatch Queue`/`Global Dispatch Queue`
- `Main Dispatch Queue`主线程，追加在`Main Dispatch Queue`的处理在主线程的runloop进行。所以要将界面更新等必须在主线程执行的处理追加在`Main Dispatch Queue`使用。

![](https://user-gold-cdn.xitu.io/2018/8/23/16566df89cecf2ce?w=630&h=161&f=png&s=14376)
-  `Global Dispatch Queue` 是所有程序都能使用的`Conncurrent Dispatch Queue`，有四个优先级，（High）、（Default）、（Low）、（Background）`Conncurrent Dispatch Queue`的线程不能保证实时性。  

- ## `dispatch_after`
- 我们经常会遇到在某个时间之后执行某个方法，那我们可以可以用`dispatch_after`这个函数来实现。
```
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
[self.navigationController popToRootViewControllerAnimated:YES];
});
```
比如上面2s返回到rootViewController,并不是在指定的2s后执行处理，而只是在指定时间追加到Dispatch Queue。因为Main Dispatch Queue在主线程中执行，所以runloop如果有正在处理执行的处理，那么这个时间会延迟，那么这个方法会在2s + x后执行，如果Dispatch Queue有大量处理追加线程或者主线程处理本身有延迟时，这个时间会更长。

### 多线程的安全隐患
- 多个线程可能会访问同一块资源，比如多个线程访问同一个对象、同一个变量、同一个文件
- 当多个线程访问同一块资源时，很容易引发数据错乱和数据安全问题
如图所示
![](https://user-gold-cdn.xitu.io/2018/8/23/16564d0b1f21b82f?w=1148&h=638&f=png&s=74484)

![](https://user-gold-cdn.xitu.io/2018/8/23/16565a22ccd4aa5c?w=169&h=242&f=png&s=14233)
### 多线程安全隐患的解决方案
常见的线程同步技术是：加锁
![](https://user-gold-cdn.xitu.io/2018/8/23/16564d225f6b2259?w=1248&h=766&f=png&s=95617)
### ios中的线程同步方案
-  ## `os_unfair_lock`
- os_unfair_lock用于取代不安全的OSSpinLock ，从iOS10开始才支持。
- 从底层调用看，等待os_unfair_lock锁的线程会处于休眠状态，并非忙等
- 需要导入头文件#import <os/lock.h>
```
os_unfair_lock lock = OS_UNFAIR_LOCK_INIT;
os_unfair_lock_trylock(&lock);
os_unfair_lock_lock(&lock);
os_unfair_lock_unlock(&lock);
```
- ## `pthread_mutex`
- mutex叫做”互斥锁”，等待锁的线程会处于休眠状态
- 需要导入头文件#import <pthread.h>
```
//初始化锁的属性
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);
//初始化锁
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);
//尝试加锁
pthread_mutex_trylock (&mutex);
//加锁
pthread_mutex_lock (&mutex); //解锁
pthread_mutex_unlock(&mutex);
//销毁相关资源
pthread_mutexattr_destroy(&attr);
pthread_mutex_destroy (&mutex);
```
### `pthread_mutex` 条件
```
//初始化锁
pthread_mutex_t mutex;
//NULL代表默认属性
pthread_mutex_init(&mutex, NULL);
//初始化条件
pthread_cond_t condition;
pthread_cond_init(&condition, NULL);
//等待条件 （进入休眠，放开mutex锁，被唤醒后，会再次对mutex加锁）
pthread_cond_wait(&condition, &mutex);
// 激活一个等待该条件的线程
pthread_cond_signal(&condition);
//激活所有等待该条件的线程
pthread_cond_broadcast(&condition);
//销毁资源
pthread_mutex_destroy(&mutex);
pthread_cond_destroy(&condition);
```
- ## `dispatch_semaphore`
- semaphore叫做”信号量”
- 信号量的初始值，可以用来控制线程并发访问的最大数量
- 信号量的初始值为1，代表同时只允许1条线程访问资源，保证线程同步
```
//信号量的初始值
int value = 1;
//初始化信号量
dispatch_semaphore_t semaphore = dispatch_semaphore_create(value);
//如果信号量的值<=0，当前线程就会进入休眠等待（直到信号量的值>0）
//如果信号量的值>0，就减1，然后往下执行后面的代码
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
//让信号量的值加1
dispatch_semaphore_signal(semaphore);
```
- ## `dispatch_queue(DISPATCH_QUEUE_SERIAL)`
这个在上面已经提到了
- ## `NSLock`、`NSRecursiveLock`
- NSLock是对mutex普通锁的封装
- NSRecursiveLock也是对mutex递归锁的封装，API跟NSLock基本一致

```
- (void)lock;
- (void)unlock;
- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate *)limit;
NSLock *lock = [[NSLock alloc]init];
```
- ## `NSCondition`
- NSCondition是对mutex和cond的封装
```
- (void)wait;
- (BOOL)waitUntilDate:(NSDate *)limit;
- (void)signal;
- (void)broadcast;
```
- ## `NSConditionLock`
- NSConditionLock是对NSCondition的进一步封装，可以设置具体的条件值
```
- (void)lockWhenCondition:(NSInteger)condition;
- (BOOL)tryLock;
- (BOOL)tryLockWhenCondition:(NSInteger)condition;
- (void)unlockWithCondition:(NSInteger)condition;
- (BOOL)lockBeforeDate:(NSDate *)limit;
- (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;
```
- ## `@synchronized`
- @synchronized是对mutex递归锁的封装
- @synchronized(obj)内部会生成obj对应的递归锁，然后进行加锁、解锁操作
```
@synchronized(obj){
//任务
}
```
下面我们来比较一下他们的性能
对他们分别进行了加锁解锁1000次操作
这是日志结果：
```
2018-08-23 20:04:50.368758+0800 gcdTest[2700:390931] @synchronized: 218.018055 ms
2018-08-23 20:04:50.409929+0800 gcdTest[2700:390931] NSLock: 40.930986 ms
2018-08-23 20:04:50.446109+0800 gcdTest[2700:390931] NSLock + IMP: 35.949945 ms
2018-08-23 20:04:50.482748+0800 gcdTest[2700:390931] NSCondition: 36.401987 ms
2018-08-23 20:04:50.591528+0800 gcdTest[2700:390931] NSConditionLock: 108.524919 ms
2018-08-23 20:04:50.650369+0800 gcdTest[2700:390931] NSRecursiveLock: 58.606982 ms
2018-08-23 20:04:50.678437+0800 gcdTest[2700:390931] pthread_mutex: 27.842999 ms
2018-08-23 20:04:50.700001+0800 gcdTest[2700:390931] os_unfair_lock: 21.309972 ms
```
| 方法       | 耗时   |
| :----------------   | ------: |
| synchronized     | 218.018055 ms |
| NSLock        |  40.930986 ms |
| NSLock + IMP        | 35.949945 ms|
| NSCondition        | 36.401987 ms|
| NSConditionLock        |108.524919 ms|
| NSRecursiveLock       | 58.606982 ms|
| pthread_mutex        |  27.842999 ms|
| os_unfair_lock       |   21.309972 ms|

- 耗时方面：
- `os_unfair_lock`耗时最少;
- `pthread_mutex`其次。
- `@synchronized`和`NSConditionLock`效率较差。
如果考虑性能可以使用os_unfair_lock，如果不考虑性能，只是图个方便的话，那可以使用@synchronized。

## 如何实现ios读写安全方案?
- 同一时间，只能有1个线程进行写的操作
- 同一时间，允许有多个线程进行读的操作
- 同一时间，不允许既有写的操作，又有读的操作  
#### ios实现的方案有
- `pthread_rwlock`：读写锁
```
//初始化锁
pthread_rwlock_t lock;
pthread_rwlock_init(&lock, NULL);
//读加锁
pthread_rwlock_rdlock(&lock);
//写加锁
pthread_rwlock_wrlock(&lock);
//解锁
pthread_rwlock_unlock(&lock);
//销毁
pthread_rwlock_destroy(&lock);
```
- `dispatch_barrier_async`：异步栅栏调用
- 这个函数传入的并发队列必须是自己通过`dispatch_queue_cretate`创建的
- 如果传入的是一个串行或是一个全局的并发队列，那这个函数便等同于`dispatch_async`函数的效果
```
dispatch_queue_t queue = dispatch_queue_create("read_queue", DISPATCH_QUEUE_CONCURRENT);
//读
dispatch_async(queue, ^{

});
//写
dispatch_barrier_sync(queue, ^{

});
```
## 为何会产生死锁？
#### 定义
>所谓死锁，通常指有两个线程T1和T2都卡住了，并等待对方完成某些操作。T1不能完成是因为它在等待T2完成。但T2也不能完成，因为它在等待T1完成。于是大家都完不成，就导致了死锁（DeadLock）。  

![](https://user-gold-cdn.xitu.io/2018/8/23/16565a71b39167c2?w=162&h=182&f=png&s=9256)

#### 产生死锁的四个必要条件：
>- 互斥条件：一个资源每次只能被一个进程使用。
>- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
>- 不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。
>-  循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

这四个条件是死锁的必要条件，只要系统发生死锁，这些条件必然成立，而只要上述条件之一不满足，就不会发生死锁。  
#### 来看这段代码：
```
dispatch_sync(dispatch_get_main_queue(), ^(void){
NSLog(@"这里死锁了");
});
```
执行这个`dispatch_get_main_queue`队列的是主线程。执行了`dispatch_sync`函数后，将`block`添加到了`main_queue`中，同时调`dispatch_syn`这个函数的线程被阻塞，等待block执行完成，而执行主线程队列任务的线程正是主线程，此时他处于阻塞状态，所以block永远不会被执行，因此主线程一直处于阻塞状态。因此这段代码运行后不是在block中无法返回，而是无法执行到这个`block`。

#### 小结
在实际开发中，我们遇到的情况会比较多，大家根据实际情况选择，本文就不一一列举，欢迎有问题留言讨论。
> * Follow: https://github.com/sallenhandong
> * blog: slimsallen.com
