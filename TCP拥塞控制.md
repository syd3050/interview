## TCP 拥塞控制

> TCP拥塞控制是传输控制协议（Transmission Control Protocol）避免网络拥塞的算法，是互联网上主要的一个拥塞控制措施。 它使用一套基于**线增积减**模式的多样化网络拥塞控制方法（包括慢启动和拥塞窗口等模式）来控制拥塞。 

从定义可以看出，TCP拥塞控制主要的目的在于“避免网络拥塞”。你可能会问：什么叫“网络拥塞”？

> 拥塞的主要成因是**数据发送方**投放到网络中的数据超出了网络的承载能力，网络设备接口往往都有**缓冲区**，超量的数据无法从瓶颈路径发送，只好在缓存区排队，而缓冲区的大小有**限制**，积压的数据量超出缓存区大小，就会发生丢弃。

所以我们知道，如果要避免拥塞，就需要发送方按照接收网络方的流量承载能力发送数据，那究竟有什么方法可以做到这点呢？发送端是怎么获得接收网络的流量承载能力的呢？进一步，发送端能否动态监测接收网络的承载能力，从而动态调整发送速率呢？当然有办法。以下就是常用的拥塞控制方法的组成：
1.慢启动；2.避免拥塞；3.快速重传；4.快速恢复；

 **1. 慢启动**
慢启动其实就是试探试增长并调整发送速率（也就是**发送窗口**，“窗口”一词在中文情境下比较难理解，但在英文中有“机会”的意思。如果把它理解为“时机，机会”的话可以更容易理解它和“速率”的关系。假设发送窗口初始化为cwnd）。开始时，cwnd初始化为1个**MSS**（Maximum Segment Size， 最大报文长度，是TCP数据包每次能够传输的最大数据分段）大小，然后随着ACK线性递增，每当经过一个**RTT**（Round Trip Time， 也就是一个数据包从发出去到回来的时间） 又指数上升，最后进入拥塞避免阶段。
详情如下：
<1>. 连接建好的开始先初始化cwnd = 1MSS，表明可以传一个MSS大小的数据
<2>. 每当收到一个ACK，cwnd++; 呈线性上升
<3>. 每当过了一个RTT，cwnd = cwnd*2; 呈指数让升
<4>. 当cwnd >= ssthresh（慢启动门限）时，就会进入“拥塞避免算法”

**2.拥塞避免算法**
通过上面对“慢启动”的介绍，我们知道，发送窗口cwnd不能无限制的增长，否则接收网络将无法承受。所以在上面最后一步中，当cwnd达到慢启动门限ssthresh后，将需要进行拥塞避免调控。
这一算法的基本思想是改指数增长为线性增长。到这个阶段，每当收到一个RTT，窗口递增1。但也不是无限递增，如果发现出现了网络拥塞（基本上是丢包），这时候将调整ssthresh（ssthresh = ssthresh/2）和cwnd(cwnd = 1)，重新进入慢启动阶段。

**3.快速重传**
快速重传为什么可以避免拥塞呢？回答这个问题之前，先来看两个问题：
***<1>重传是什么？  
<2>快速是什么意思？***

**<1>重传**
在一个复杂多变的网络环境中，数据包丢失是无法避免的。很容易想到，如果要保证数据传输的有效性，丢失的数据包必需重新发送，这就是所谓的数据包重传。假设发送方发送数据包M3，然后启动一个计时器T3，发送方设定，如果在T3计时器到时之后还没有都到接收方对数据包M3的确认，它就认为接收方没有收到M3，M3在网络中丢失了，这时发送方会重新发送数据包M3。
    但仔细想想你会发现，T3计时器不太好设定。值小了的话，在不稳定的网络环境中特别容易超时，而一旦超时就会重新发送数据包M3（其实接收方可能已经收到了M3，但接收方发回的确认ACK还没到发送方），从而很容易造成网络拥塞；值大了的话，重传变慢，包M3都丢老半天了才重传，效率低。
    
**<2>快速**
    这样说来，等待超时重传并不是特别好的办法，那怎么办呢？想象一下，发送方按顺序发送M1-6这6个数据包，发送方的M1,M2包都收到了接收方发回的ACK确认，但M3丢失。因为M3的超时计时器T3未到时，所以发送方不会立即重传M3，这时继续发送M4,M5,M6。对于接收方来讲，无论是M4，M5或者M6，它都不会返回对于的ACK确认，因为它发现序号不对，它认为它只收到过M1,M2，接下来它应该收到M3才对，你发送方发过来的M4,M5,M6老子不要，所以它继续不断给发送方返回M2的ACK确认，目的是告知发送方它只收到M2。
   对于发送方，当收到3个M2的ACK确认之后，即便M3超时计时器T3还没到期，它也认为不能再等了，不然接收方该火了，于是它重传数据包M3，并重置超时器T3。
   由此可见，这样的机制的确可以有效的利用网络并控制拥塞。

**4.快速恢复**
快速恢复与快速重传搭配，具体细节如下：
<1>当发送方连续收到三个重复确认，就执行“乘法减小”算法，把慢开始门限ssthresh减半。这是为了预防网络发生拥塞。请注意：接下去不执行慢开始算法。
 <2>由于发送方现在认为网络很可能没有发生拥塞，因此与慢开始不同之处是现在不执行慢开始算法（即拥塞窗口cwnd现在不设置为1），而是把cwnd值设置为慢开始门限ssthresh减半后的数值，然后开始执行拥塞避免算法（“加法增大”），使拥塞窗口缓慢地线性增大。
