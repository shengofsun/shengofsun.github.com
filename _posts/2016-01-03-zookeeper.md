---
layout: post
title: "zookeeper使用笔记"
categories: 技术问题
tags: zookeeper
---

最近在公司的分布式项目pegasus中用到了zookeeper的c客户端，在此记录下使用zookeeper c client时的一些心得。

## zookeeper c api的介绍和部分坑

这里先推荐两处别人写的blog,分别介绍了c api接口的[使用方法](), 和一些[要注意的坑]()．后面的内容主要依据自己在使用zk时的心得体会,对这两篇blog中的内容做补充.

## zookeeper client的会话状态

使用zookeeper的c client端访问服务的初始化逻辑如下图所示:

```C
//初始化会话
zhandle_t* handle = zookeeper_init(hosts, watcher_callback ... ); 
...
// 等待zookeeper client会话的状态转换为connected
... 
// 其他的zookeeper read/write操作
zookeeper_close(handle);
```

总的来说，zk C client的api是非常简洁的：首先创建一个会话用以和服务器列表进行通信；然后便开始调用zk的读写api。但就细节上而言，有一点是非常值得注意的：客户端会话是有“连接状态”这一概念的，只有状态是“connected”的会话，才能和服务器进行正常的通信。

会话的状态是zk客户端中一个比较容易混淆的概念。下图给出了client会话一个简要的状态转移图(基于3.4.6的c client代码，没有考虑zk ACL层)：
![ ](/blogimg/zookeeper/zookeeper-state.png)

上图中几个状态的说明如下：

* not_connected 最开始，未和服务器进行通讯的时候
* connecting 客户端正在开始和zookeeper集群服务中的某一台服务器进行连接
* associating 客户端已经和某个服务器建立tcp连接,在等待服务器根据客户端上传的会话信息进行"匹配"
* connected 客户端已和服务器联系上，此时可以和服务器发送读写请求
* expired 会话已经超时不可用，客户端此时应关闭本会话，自杀或者重新发起连接

客户端在创建handle后，等待会话状态变为connected的方法一般有以下两种：

1. 在调用zookeeper_init生成handle后，开始不断用zoo_state的api轮询handle的状态，并根据返回的状态值做对应的动作。
2. 在调用zookeeper_init时，用户需要注册一个回调函数。在会话状态的每一次改变时，回调函数就会被调用一次。所以客户端的使用者可以根据
回调函数所传入的不同状态执行不同的动作。

在zk会话的整个生命周期中，其状态可能在connected和connecting之间做多次切换。使用客户端库的业务代码需要注意这两种情况，并避免在connecting
的时候对zookeeper进行读写。

## 会话状态转换的进一步说明

造成client session状态转换的原因主要有两种：

* zk服务器集中一台或多台机器的crash
* client和server的网络分区

在发生以上情况时，client会和server失去连接。在这种情况下，clientlib会关掉和当前服务器的socket句柄，开始对集合中的服务器进行轮询。在重新连上某一台服务器
后，clientlib会把本地保存的会话签名(也就是c client库中的clientid_t结构体)发送给服务器；根据该签名，服务器可以判断该会话是否已经超时。客户端再根据服务器返回的结果，
在回调函数中返回重新连接上，或者是session expired。

由此可知zk一个非常重要的特性：客户端会话是否超时是由<B>服务器</B>决定的。考虑到zookeeper是由多个机器组成的一个集群服务，这样的语义其实
是非常合理的：只有把会话生命周期的决定权交给服务器，才能比较好的实现服务在不同机器间的“平滑切换”。

其实对于zk状态转换和会话超时做何种处理，是完全取决于使用zk的业务场景的。如果仅仅把zk当做一个保存元数据的服务，那么只要在connected的时候做读写即可。
而对于zk最常见的分布式任务的“仲裁者”的功能，在处理客户端会话的状态上就要小心很多。

比如很常见的分布式抢锁的场景(比如：主从分布式系统中的Master)：

有一个任务T需要进行持续处理。因为任务比较重要，我们给任务派了A，B，C三个候选机器；这样，

<B>(a)一旦有一个机器发生宕机，就会有另一个候选者接棒。</B>

同时，因为任务涉及全局状态的改变。为了保证状态的一致，我们必须保证

<B>(b)在任意时刻只能有至多一个的候选机器在处理任务T。</B>

对于这种场景，zk有几乎“模板化”的解决方案，所以就不再搬运了:-)。这里只是想强调一下客户端会话的状态在分布式抢锁中的作用。
为此，我们先大体复现下A，B，C利用zk抢锁完成任务T的一个流程：

1. A、B、C都和zk集群Q创建会话。
2. 通过zk的仲裁，A抢得处理任务T的资格。
3. zk在断定A会话超时后，会用某种形式让B和C得到通知；并在B和C之间重新做仲裁。

为了满足前面的(b)，我们在流程3中需要处理一点细节：在zk断定A超时之前，A自己必须停止对任务T的处理。这种要求使得A<B>绝对不能</B>在收到zk服务端的session_expired
消息后才停止自己的工作。而是应该在之前的某个时间点时就把自己停下来。

而这个时间点，其实就应该是状态由connected变为connecting的时刻。

## 会话状态怎样会变为connecting

就实现细节而言，有两种事件会导致zk会话变为connecting

1. client和某个server间的tcp socket不可用(poll失败，或者send/recv失败)
2. client端租约超时

其中client端的租约用的是最常见的租约技术：

1. client定期给zk server定期发送一个ping的消息，记周期为T1。
2. 如果client有一段时间没收到zk server端的数据，则视为client的租约超期；记超期的时长T2。

在具体的实现中，ping消息的发送并非单纯的以T1为周期的数据包，而是和zk数据请求混在一起的心跳机制。具体来说，如果距上次发送数据包
的时间已经过了T1，那么就会发送一个ping。在3.4.6的客户端中，T1的值为recv_timeout/3。

相应的，client租约也是随着不停的收到数据包而不停的延长。在3.4.6的c client代码中，T2的值为recv_timeout*2/3。

依据这些实现细节，我们可以看出对于前面讨论的“分布式抢锁”，作为zk client的A，在会话变为connecting后将自己的任务T停下来是合理的。
一方面而言，无论是socket不可用还是租约超时，client都有理由怀疑自身在zk server那边会有可能发生会话超时。另一方面而言，这种服务不可用
只是一种暂时性的不确定的“怀疑”。随着client对zk server列表中其他机器的连接成功，这种暂时性的不确定性就很快被打破，回到可用的状态或者
变成确定的“过期”状态。

有一点需要注意的是，如果client端无法连接到zk server列表中的任何一台机器，那么client端就会陷入到无限的等待中去。就正确性而言，
这并无影响。但事实上而言，这样的状态还是有一些不太好的：

1. 因为客户端无法打破这种不确定性，所以它无法依靠zk的某个确定事件来执行自杀的逻辑。
2. 在无限的循环等待时，client事实上在做不停的轮询，这带来的最直接的后果就是——费电。

所以在具体的使用上，client最好在收到connecting的事件后自己触发一个定时器，用以在合适的时候做自杀的工作。

另外，有一个很简单的实验可以证明客户端的session expire是取决于zk server的：

1. 启动一个单机版的zk服务，并写一个简单的客户端连接上它。
2. 手动停掉zk服务，这时候可以观察到客户端变为connecting
3. 过很长的时间(至少超过和server协商的recv_timeout)，再启动起来服务；可以看到客户端变为connected，而非expire。

原因也很简单，在关掉zk服务后，server的整个时间流其实是“停止”的。再重新启动起来后，server端其实只是过了很短的一个时间段，自然不会
把client判定为超时。

这其实是一个比较好的特性。可以试想，如果因为某种原因，zk集群发生了整体性的宕机(断电？)，那么在集群重启之后，所有依赖zk的服务在
理论上是可以原样启动起来的。

## zk api的返回值问题

不得不承认,程序员大部分的时间精力,其实都集中在了程序只有在很少情况下才会触发的一些异常逻辑上．所以在使用某个库的时候,我们自然而然就得多花精力
去关注api那些表征异常的返回值是什么含义.

zk api返回值,在异常情况下基本上可以分为三大类:

1. 运行时环境错误,比如内存分配失败等等
2. 输入参数不合法
3. tcp连接异常而导致的错误

这里重点强调下3．具体来说,会话异常可能会导致zk api返回三种错误码:

1. invalid_state:　当前session的状态不是connected,而调用zk的读写api
2. connection_loss: 在进行异步的读写操作时,socket不可用
3. operation_timeout: 在进行异步的读写操作时,client session发生租约超时

2和3所代表的错误含义，在前面分析connecting_state的时候已经给出了说明．这里更多的想说明以下返回这些错误的具体场景．为此，我们先看一下zk c client
的程序结构：

![](/blogimg/zookeeper/zookeeper_structure.png)

如上图所示，zk client内部共有两个线程：

* io_thread, 主要负责socket的读写
* completion_thread, 主要负责异步api返回状态的分发

另一方面，zk client内部一共维护了四个队列：

* to_send: 裸的memory_buffer, client线程调用api的时候, 请求被序列化成memory buffer放到to_send尾部
* sent_request:　和to_send中的memory_buffer一一对应，存放一个异步api的回调函数地址，上下文等
* to_process: 裸的memory_buffer, 收到的服务端发来的消息时,io_thread将其按序入队
* completions_to_process: 根据服务端发来的数据(数据请求响应,watcher), io_thread把本地保存的回调函数上下文等一系列东西和之对应后放入该队列；然后
等待competion_thread对该队列中的内容进行dispatch

在知道zk client的结构后，我们可以得出一些结论：

* zk api的回调全部在completion thread的上下文中执行．所以如果有全局变量在多个回调函数中执行，不用担心竞争条件的发生．
* 会话变为connecting的处理逻辑是这样的: 
  1. io thread把会话异常的消息放入到completions_to_process队列中
  2. io_thread清空to_send和to_process
  3. io_thread遍历sent_request队列，设置返回值为connection_loss(operation_timout)，放入completions_to_process队列．所以，一旦会话变为connecting，已经加入发送队列中的消息旋即都会随后返回，而不做任何重发的cache．
* 如果遇到了zk会话暂时不可用，合理的方法是等待状态变为了connected了之后再开始重试；无脑的重试只会导致cpu空转．

## 封装zk的细节是不是一个好的主意

在最开始对zookeeper的c api进行封装的时候, 想法是屏蔽掉zk会话状态和超时重传的细节, 直接提供一个简单的接口供上层使用．后来随着对zk了解的深入,才渐渐觉得这样的
封装其实不是非常合适．从会话状态的角度看, 其重要性已在前面解释过; 而对于超时重传, 是不是要把该功能封装起来其实也是值得商榷的．

这一点其实是取决于使用zk的逻辑是如何的. 如果zk client的使用者就是数据的产生者, 那么这样做其实并无不可. 而如果使用者也是一个二道贩子, 那么这么做就不是很合适了.

在pegasus项目中, 存到zk上的数据就是由二道贩子转手过的. 正常的流程是这样的:

1. Server A(Partition Server)把数据发送给Server B(meta server), 如果数据满足一定的要求, 再由B把数据存到zk上; 
2. 存放成功后, B再给A发响应. 其中Server A和Server B的通信采用的是无连接的rpc通信, A在超时后会进行重传. 

这时候，如果在Server B和zk之间实现一层cache(cache的状态对A是不可见的), 那么很显然的, 一旦B和zk之间出现了connecting state, B就会很有可能给zk发重复请求．
更进一步而言, 长的缓存队列, 对整个系统的实时性可能是会造成影响的( 请允许我拿这个极端的例子信口开河胡说八道:-) )．

此外，对需要重传的消息进行cache也得注意一些小问题，比如：

* 如果对已经发送成功的消息进行重传，那么需要注意错误处理(重复create, 第二次会返回失败)
* 对于基于状态的带有保序特性的连接，添加中间cache往往意味着会破坏保序性(第一条失败，第二条成功，第一条重传成功)

## transaction不是万能的

较新版本的zk引入了transaction的接口, 可以批量的向zk服务器发送请求, 并保证操作的原子性. 对于这个特性, 有一点需要补充: 接口不能支持较大数据量的传送, 如果一次传输过多
会导致会话超时.
