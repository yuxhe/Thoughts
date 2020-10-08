socket网络部分

[https://github.com/incrediblejustin/LinuxKernelNetworkProtocolStack/tree/master/socket](https://github.com/incrediblejustin/LinuxKernelNetworkProtocolStack/tree/master/socket)

物理内存的直接映射是从ffff880000000000地址开始的，最大64TB

IA-32系统起始于0x08048000，在text段的起始地
址与最低的可用地址之间有大约128 MiB的间距，用于捕获NULL指针  ?

strace 跟踪系统调用的工具，网络助手netassit； epoll_wait唤醒惊群
nginx是多进程处理方式，非多线程，防止相互干扰，每次请求无关联，相互无干扰；
nginx CAS锁 ;

nginx 线程池  落地日志 及 其他访问文件，异步落地非阻塞，里面用到队列，for循环获取队列中数据写盘；

wrk压测软件，打击nginx

net.ipv4.tcp_mem=最小页 默认页  最大页

net.ipv4.tcp_wmem=最少字节 默认字节  最大字节

net.ipv4.tcp_rmem=最少字节 默认字节  最大字节

数据库异步写

普通的静态访问最大并发数

nginx  worker支持的最大并发数

worker_connections * worker_processes / 2
作为反向代理的话

worker_connections * worker_processes / 4


自旋锁
基于原子操作，Nginx实现了一个自旋锁。自旋锁是一种非睡眠锁，也就是说，某进程如果试图获得自旋锁，当发现锁已经被其他进程获得时，那么不会使得当前**进程进入睡眠**状态，而是始终保持进程在可执行状态。

要解决的问题：进程对共享资源的访问时间非常短，否则，不适合。

自旋锁使用场景：使用锁的进程不太希望自己进入睡眠状态

用户可以从锁的使用时间长短角度来选择使用哪一种锁:

当锁的使用时间很短时，使用自旋锁非常合适，尤其是对于现在普遍存在的多核处理器来说，这样的开销最小。 如果锁的使用时间很长时，那么一旦进程拿不到锁就不应该再执行任何操作了，这时应该使用睡眠锁将系统资源释放给其他进程使用

轻量级锁的目标是，减少无实际竞争情况下，使用**重量级锁产生的性能消耗**，包括系统调用引起的内核态与用户态切换、线程阻塞造成的线程切换等

轻量级锁有指向栈的指针哈；**否指向当前线程的栈帧**，否则升级为重量级锁，重量级锁有指向重量级锁的指针

![](https://img-blog.csdn.net/20170420102716139?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenF6X3pxeg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



当然，由于轻量级锁天然瞄准不存在锁竞争的场景，如果存在锁竞争但不激烈，仍然可以用自旋锁优化，自旋失败后再膨胀为重量级锁

偏向锁--->轻量级锁 ---> 重量级锁分配和膨胀的详细过程见后

![](https://img-blog.csdn.net/20170419215511634?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenF6X3pxeg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


**轻量级状态下引入的自旋锁**

终端设置编码可解决乱码问题 utf-8

--------------------------------------------解压到某目录
tar zxvf protobuf-cpp-3.12.4.tar.gz -C /usr/local 

./configure
 
make 
 
sudo make install

重新加载动态库
sudo ldconfig

find /  -name  查找的名字

编译参数

g++ -o code_test code_test.cpp IM.BaseDefine.pb.cc IM.Login.pb.cc -lprotobuf -lpthread  --std=c++11

g++ -o pb_speed pb_speed.cpp IM.BaseDefine.pb.cc IM.Login.pb.cc -lprotobuf -lpthread  --std=c++11

-- **共享库目录配置**

ldconfig命令的用途, 主要是在默认搜寻目录(/lib和/usr/lib)以及动态库配置文件/etc/ld.so.conf内所列的目录下

共享库文件安装到了/usr/local/lib(很多开源的共享库都会安装到该目录下)

cat /etc/ld.so.conf
include ld.so.conf.d/*.conf

**仓库地址源**

https://distfiles.macports.org/protobuf3-java/



-lpthread  动态库符号连接 编译阶段有用

g++ -o queue_stl queue_stl.cpp -std=c++11 -lpthread

无锁队列性能高哈

**布隆过滤器**（Bloom Filter）是1970年由布隆提出的
它实际上是一个很长的**二进制向量**和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中
优点：
可以**高效地进行查询**，可以用来告诉你“某样东西一定不存在或者可能存在”
可以高效的进行插入
相比于传统的List、Set、Map等数据结构，它占用**空间更少**，因为其本身并不存储任何数据（重点）
缺点：
其返回的结果是概率性（**存在误差**）的
一般不提供删除操作
布隆过滤器一般使用在数据量特别大的场景下，一般不会使用
常用的使用场景：
使用word文档时，判断某个单词是否拼写正确。例如我们在编写word时，某个单词错误那么就会在单词下面显示红色波浪线
网络爬虫程序，不去爬相同的url页面
垃圾邮件的过滤算法
缓存崩溃后造成的缓存击穿
集合重复元素的判别
查询加速（比如基于key-value的存储系统，如redis等）

针对于“baidu”这个元素，我们调用**三个哈希函数**，将其映射到bit向量的三个位置（分别为1、4、7），并且将对应的位置置为1 ；**多哈希函数**
**查询一个不存在的元素，并且确定其肯定不存在**；
去查询“baidu”这个元素，不能判断其百分百存在；

可以做初步筛查

![](https://img-blog.csdnimg.cn/20200528231307761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNDUzMjg1,size_16,color_FFFFFF,t_70)


哈希函数越多、插入元素越少，误判率越低

n：布隆过滤器最大处理的元素的个数
P：希望的误差率
m：布隆过滤器的bit位数目
k：哈希函数的个数

Jenkins，还有许多其他的持续集成工具，包括Strider和Drone.io这种直接利用Docker的工具，这些工具都是真正基于Docker的


压缩原理其实很简单，就是找出那些重复出现的字符串，然后用更短的符号代替， 从而达到缩短字符串的目的

**Deflate压缩算法**
deflate压缩算法用来很多地方：
例如其是zip压缩文件的默认算法、在zip文件中，在7z， xz 等其他的压缩文件中都用
gzip压缩算法、zlib压缩算法等都是对defalte压缩算法的封装（下面会介绍）
gzip、zlib等压缩程序都是无损压缩，因此对于文本的压缩效果比较好，对视频、图片等压缩效果不是很好(视频一般都是采用有损压缩算法)，所以对于视频、图片这种已经是二进制形式的文件可以不需要压缩，因为效果也不是很明显
实际上deflate只是一种压缩数据流的算法。 任何需要流式压缩的地方都可以用
**Deflate压缩算法=LZ77算法+哈夫曼编码**

LZ77 压缩算法采用字典的方式进行压缩， 是一个简单但十分高效的数据压缩算法。其方式就是把数据中一些可以**组织成短语(最长字符)的字符加入字典**，然后再有相同字符出现采用标记来代替字典中的短语，如此通过标记代替多数重复出现的方式以进行压缩

哈夫曼设计了一个贪心算法来构造最优前缀码，被称为哈夫曼编码（Huffman code）， 其正确性证明依赖于贪心选择性质和最优子结构。哈夫曼编码可以很有效的压缩数据，具体压缩率依赖于数据本身的特性
这里我们先介绍几个概念：
码字：每个字符可以用一个唯一的二进制串表示，这个二进制串称为这个字符的码字
码字长度：这个二进制串的长度称为这个码字的码字长度
定长编码：码字长度固定就是定长编码。
变长编码：码字长度不同则为变长编码。





