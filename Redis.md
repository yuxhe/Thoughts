redis   好的特性、解决的问题、同时可能遇到新问题

低⼀致性：本地缓存

Twemproxy 为twitter开源的代理分片机制，硬伤无法平滑地扩容/缩容

codis 国内定制的redis-server 硬伤：非官方血统

proxy 价值可监控、分析，可优雅的读写分离，大key有风险

弹分片最大QPS可达9.8w,99.99%的响应在4ms以内
单分片QPS在5w以内，99.99%的响应在1ms以内


**扩展性-数据分⽚** ；增删节点时，数据需要重新均衡

Redis**扩容**太痛苦，**海量**数据在内存，**成本**太⾼，**多机房**部署；

**异步复制**，读写分离时，可能**读到旧数据**；

**事务-原⼦性提交**

事务-并发安全，数据分⽚后，⽆法使⽤跨节点使⽤事务；

解决方案：**业务的代码，需要解决这些问题**

Titan 
• 分布式NoSQL存储

• QPS可扩展，低延迟（20ms内）

• 完整的ACID事务⽀持，强⼀致

• Raft协议多副本共识，容忍⼩于半数节点挂掉

Redis扩展困难，不⽀持分布式事务

Titan⾼可⽤可扩展强⼀致，⽀持分布式事务



策略模式 --实现多个同一接口 方便扩展 针对不同的情况或对象或业务处理 ;假如参数不一样 此时如何处理呢 ？

模板模式：定义一个操作中的算法的骨架，而将一些步骤延迟到子类中

swagger是一个方便后端编写接口文档的开源项目，并提供界面化测试

Feign是一个声明式的Web Service客户端

还整合了Ribbon和Eureka来提供均衡负载的HTTP客户端实现

REST服务，这个服务一定会有心跳检测、健康检查和客户端缓存等机制

Config分布式统一配置中心，支持放在远程Git仓库中。在spring cloud config 组件中，分两个角色，一是config server，二是config client；

Ribbon是一个基于HTTP和TCP客户端的负载均衡器；

Sleuth链路跟踪，---- 配合 zipkin 的使用

HyStrilx服务熔断降级方式 断路器模式 `hystrilx`不单独使用，此组件一般和`ribbon`、`feign`结合使用

ThreadLocal  每个线程对其进行访问的时候访问的是自己线程的变量，值是根据线程做为key获得的，所以和其他线程并没有关系，不会发生并发冲突

join简介
此线程加入当前线程，当前线程让进入阻塞队列，等待此线程完成

volatile的两点内存语义能保证可见性和有序性，但是能保证原子性

CPU使用抢占式调度模式在多个线程间随机高速的切换；

**锁的专业名称叫监视器 monitor**,其实 java 为每个对象都自带内置了一个锁(监视器 monitor),
当某个线程执行到了某代码快时就会自动得到这个对象的锁,那么其他线程就无法执行该代码块了,一直要等到之前那个线程停止(释放锁)   未解释清楚

当数组大小已经超过64并且链表中的元素个数超过默认设定（8个）时，将链表转化为红黑树

调用wait()方法的线程没有实现获取该对象的监视器锁，则调用wait()方法时线程会抛出IllegalMonitorStateException异常

//设置中断标志
      threadOne.interrupt();


notify()会唤醒被阻塞到该共享变量上的一个线程，notifyAll()会唤醒所有在该共享变量上由于调用wait系列方法而被挂起的线程

ThreadLocalRandom保证每个线程都维护一个种子变量，每个线程根据自己老的种子生成新的种子，避免了竞争问题，大大提高了并发性能ThreadLocalRandom保证每个线程都维护一个种子变量，每个线程根据自己老的种子生成新的种子，避免了竞争问题，大大提高了并发性能

// 等待子线程执行完毕
        threadOne.join();

JUC包中的并发List只有CopyOnWriteArrayList  数组

park() 挂起  ；参与调度

LockkSupport

AbstractQueuedSynchronizer抽象同步队列简称AQS，是实现同步器的基础组件

AQS类并没有提供tryAcquire和tryRelease方法的实现，因为AQS是一个基础框架，这两个方法需要由子类自己实现来实现自己的特性


独占方式下获取和释放资源的方法为： 
> void acquire(int arg)
> void acauireInterruptibly(int arg)
> boolean release(int arg)

共享方式下获取和释放资源的方法为：
> void acauireShared(int arg) 
> void acauireSharedInterruptibly(int arg)
> boolean releaseShared(int arg)

join方法不够灵活，所以开发了CountDownLatch

// 阻塞直到被interrupt或计数器递减至0
    countDownLatch.await();

CountDownLatch 也是aqs的应用

CyclicBarrier基于ReentrantLock实现，本质上还是基于AQS的


如果在synchronized代码块外部调用wait()方法，JVM会抛出IllegalMonitorStateException异常

lock 它是基于Lock接口和实现它的类（如ReentrantLock）


CountDownLatch: CountDownLatch 类是Java语言提供的一个机制，它允许线程等待多个操作的完结。
  * CyclicBarrier: CyclicBarrier 类是又一个java语言提供的机制，它允许多个线程在同一个点同步。

Semaphore是一个控制访问多个共享资源的计数器

