### 面试必备之TCP协议

TCP协议无论是前端还是后端面试，都会被经常问到，而且在我们日常的开发中，这里面的一些知识对我们了解网络也有重要的意义。所以非常有必要去详细了解一下TCP协议。

#### 什么是TCP

​	TCP是主要的网络协议之一，属于传输层的协议。它使两台主机能够建立连接并交换数据流。TCP 能保证数据的交付，维持数据包的发送顺序。

#### TCP的特点：

1. **TCP是面向连接的传输层的协议**。就是使用TCP协议之前，必须建立连接。在数据传输完成之后，必须释放已经建立的TCP连接。

2. **每一条TCP连接只能有两个端点**，每一条TCP连接只能是点对点（一对一）

3. **TCP提供可靠交付的服务**。通过TCP连接传送数据，不会出差错，不会丢失，不会重复，并且按序到达。

4. **TCP提供全双工通信**。TCP允许通信双方的应用程序在任何时候都能发送数据。TCP连接的两端都设有发送缓冲和接受缓冲，用来临时存放双向通信的数据。在发送的时候，应用程序把数据传送给TCP的缓存后，就可以做自己的事，而TCP会在合适的时候把数据发送出去。在接受的时候，TCP把接受到的数据存放的缓存，上层应用进程在合适的时候读取缓存中的数据。

5. **面向字节流**：TCP中的“流”是指，流入到进程或者从进程流出的字节序列。“面向字节流”的含义：虽然应用程序和TCP的交互是一次一个数据块（大小不等），但是TCP把应用程序交下来的数据仅仅看成是一连串的去结构的字节流。TCP并不知道所传送的字节流的含义。TCP不保证接收方的应用程序所接受的数据块和发送方所发送的数据块大小一样。

#### TCP的连接建立与释放

##### TCP的连接

![image-20220209165159902](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209165159902.png)

一开始, 服务端的TCP服务器进程先创建传输控制块TCB，准备接受客户进程的连接请求。然后服务器进程就处于LISTEN (收听)状态,等待客户的连接请求。如有,即作出响应。

​	客户端的TCP客户进程也是首先创建传输控制模块TCB，然后,在打算建立TCP连接时,向服务端发出连接请求报文段,这时首部中的**同步位SYN = 1,同时选择一个初始序号seq =x，TCP规定, SYN报文段(即SYN =1的报文段)不能携带数据,但要消耗掉一个序号**。这时, **TCP客户进程进入SYN-SENT** (同步已发送)状态.

​	**B收到连接请求报文段后,如同意建立连接,则向A发送确认,在确认报文段中应把SYN位和ACK位都置1,确认号是ack=x+1,同时也为自己选择一个初始序号seq =y。请注意,这个报文段也不能携带数据**，但同样要消耗掉一个序号。**这时TCP服务器进程进入SYN-RCVD** (同步收到)状态。

​	**TCP客户进程收到B的确认后,还要向B给出确认。确认报文段的ACK置1,确认号ack=y+1，而自己的序号seq=x+ 1.** TCP的标准规定, ACK报文段可以携带数据。但如果不携带数据则不消耗序号,在这种情况下,下一个数据报文段的序号仍是seq=x+1.这时, TCP连接已经建立, **客户端进入ESTABLISHED (已建立连接)状态.**当B收到A的确认后,也进入ESTABLISHED状态。

##### TCP的释放

​    ![image-20220209171556729](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209171556729.png)

​	数据传输结束后，通信的双方都可释放连接。现在客户端和服务端都处于ESTABLISHED状态, 客户端的应用进程先向其TCP发出连接释放报文段，并停止再发送数据,主动关闭TCP连接。**客户端把连接释放报文段首部的终止控制位FIN置1，其序号seq=u,它等于前面己传送过的数据的最后一个字节的序号加1,这时客户端进入FIN-WAIT-I (终止等待1)状态,等待服务端的确认。请注意, TCP规定, FIN报文段即使不携带数据,它也消耗掉一个序号。**

​	服务端收到连接释放报文段后即发出确认,**确认号是ack=u+1,而这个报文段自己的序号是v,等于服务端前面已传送过的数据的最后一个字节的序号加1.然后服务端进入CLOSE-WAIT (关闭等待)状态.** **TCP服务器进程这时应通知高层应用进程,因而从客户端到服务端这个方向的连接就释放了**,这时的TCP连接处于半关闭(half-close)状态,即客户端已经没有数据要发送了,但服务端若发送数据, 客户端仍要接收。也就是说,从服务端到客户端这个方向的连接并未关闭,这个状态可能会持续一段时间。

​	服务端收到来自客户端的确认后,就进入FIN-WAIT-2 (终止等待2)状态,等待B发出的连接释放报文段。

​	**若服务端已经没有要向客户端发送的数据,其应用进程就通知TCP释放连接。这时服务端发出的连接释放报文段必须使FIN=1.现假定服务端的序号为w(在半关闭状态服务端可能又发送了一些数据), 服务端还必须重复上次已发送过的确认号ack =u+ 1.这时B就进入LAST-ACK (最后确认)状态,等待客户端的确认。**

客户端收到服务端的连接释放报文后，必须对此发出确认。在确认报文段把ACK置1，确认号ack = w+1,而自己的序号是seq =u+1 (根据TCP标准,前面发送过的FIN报文段要消耗一个序号),然后进入到TIME-WAIT (时间等待)状态。**请注意,现在TCP连接还没有释放掉。必须经过时间等待计时器(TIME-WAIT timer)设置的时间2MSL后, 客户端才进入到CLOSED状态。**

以上就是关于TCP的连接与释放，看了以上的过程，对那些标志位是不是很模糊，接下来看一下TCP的报文格式，来进一步了解TCP的工作原理

#### TCP报文段的首部格式：

TCP虽然是面向字节流，但TCP传送的数据单元是报文段。一个TCP报文段分为首部和数据两部分，而TCP的全部功能都体现在它首部中各个字段的作用。

![1290987-20190615111551823-1173939851](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/1290987-20190615111551823-1173939851.png)

首部固定部分个字段的意义

1. 源端口号和目的端口号：各占两个字节。

2. 序号：在TCP传送数据中，每一个字节都按顺序编号。首部中的序号是指本报文段所发送的数据的第一个字节的序号。例子：假设一报文的序号是301，并有100字节数据，那么如果有下一个报文，那么他的序号就是从401开始。

3. 确认号：有四个字节，是期望收到对方下一个报文段的第一个数据字节的序号。例如：B正确收到了A发送过来的一个报文段，其序号值为501，而数据的长度是200字节，这表明B正确收到了A发送的序列号700为止的数据。因此，B期望收到A的下一个数据的序列号为701，于是B在发送给A的确认报文中把确认号置为701。

4. 数据偏移：TCP报文段的首部的长度。

5. 保留：保留6位，留作今后使用。

6. URG：当URG = 1，表明紧急指针字段有效。它告诉系统，这段报文中含有紧急数据，应尽快传送，而不要按照原来排队顺序发送。

7. **ACK：当仅ACK = 1时，确认号字段才有效。若为0，确认号无效。TCP规定，在建立连接之后，所有的传送的报文的ACK都必须为1。**

8. PSH：推送PSH (PuSH) 当两个应用进程进行交互式的通信时,有时在一端的应用进程希望在键入一个命令后立即就能够收到对方的响应。在这种情况下, TCP就可以使用推送(push)操作。这时,发送方TCP把PSH置1,并立即创建一个报文段发送出去。接收方TCP收到PSH-1的报文段,就尽快地(即“推送”向前)交付接收应用进程,而不再等到整个缓存都填满了后再向上交付。虽然应用程序可以选择推送操作,但推送操作很少使用。

9. RST：当RST = 1时，表明TCP连接中出现严重差错，必须释放连接，然后重新连接。

10. **SYN：同步，在建立连接的时候用同步序号，当SYN=1而ACK=0的时候，这表明这是一个连接请求报文。若对方同意建立连接，则在响应的报文中将SYN = 1和ACK = 1。**

11. **FIN：用来释放连接。当FIN=1时，表明这个报文的发送方的数据已经发送完毕，并要求释放连接。**

12. 窗口：表明允许对方发送的数据量，是动态变化的。

13. 校验和：与UDP协议的一样。

14. 紧急指针：紧急数据的字节数。

15. 选项：长度可变，最大可达40字节。



#### TCP是怎么实现可靠传输的

##### 1.以字节为单位的滑动窗口。（以A为客户端，B为服务端）

发送方A的发送窗口。发送窗口表示：在没有收到B确认的情况下，A可以把连续把窗口内的数据发送出去。**凡是已经发送出去的数据，在没有收到确认之前都必须暂时保留，以便在超时的时候使用。**

发送窗口里面的序号表示允许发送的序号。显然窗口越大，发送方就可以在收到确认序号之前发送更多的数据，从而提高传输效率。

<center style="color:#C0C0C0;text-decoration:underline">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209173452215.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">A的发送窗口</div>
</center>

现在假设A发送了29，30，31的数据，发送窗口的位置没有改变。

<center style="color:#C0C0C0;text-decoration:underline">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209173348329.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">A的发送窗口</div>
</center>

可以看出描述一个发送窗口需要三个指针。各个指针的运算的意义在上图可以看出。

<center style="color:#C0C0C0;text-decoration:underline">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209173405177.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">B的接收窗口</div>
</center>

看一下B接收窗口大小为7，在上图中，B收到了30，31这两个数据，这些数据没有按序到达，因为序号29没有收到，这时候B只能对按序收到的的数据中的最高序号给出确认，因此B发送的确认报文中确认序号还是29（即期望收到的序号）。

现在假设B已经接受到29序号，并且把29，30，31交付给主机，因此B可以不保留这些数据，然后B窗口删除这些数据，接着向前移动3个序号，同时给A发送确认序号，其中窗口值还是为7，这时候确认号为32。我们来看看A发送窗口的收到确认后的窗口变化。

<center style="color:#C0C0C0;text-decoration:underline">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209173414864.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">B的接收窗口</div>
</center>

若A继续发送序号32到38的数据，指针P2于P3重合，发送窗口的序号全部用完，但是还没有收到确认，这时候，A必须停止发送。

<center style="color:#C0C0C0;text-decoration:underline">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209173423658.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">A的发送窗口</div>
</center>



强调以下三点：

- 虽然A的窗口时根据B的接收窗口来确定的，但是同一时刻，A的发送窗口并不总是等于B的接收窗口。因为网络有一定时间的滞后性。A还会根据网络的拥塞情况来减小自己的窗口值。
- 对于不按顺序到达的数据如何处理，TCP并没有明确的规定。通常情况下TCP接收窗口对不按顺序到达的数据先缓存到接收窗口，等到接受到缺失的数据之后，在交给上层的应用程序。
- TCP要求接收方必须有积累确认的功能。接受方可以在合适的时候发送确认。但是接收方不应该过分推迟发送确认，否则会导致发送方不必要的重传，确认推迟的时间不应该超过0.5秒。

##### 2.超时重传时间的选择

如果TCP的发送方在规定的时间内没有收到确认，就要重传已发送的报文段。重传时间的选择是TCP最复杂的问题之一。

有一种情况：

发出一个报文，到了重传时间，没有收到确认，于是重新发一份报文，经过一段时间后，收到确认报文，那么收到的报文是先发送的确认，还有后发送的报文的确认？

![image-20220209180117454](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209180117454.png)

解决方法：

报文每重传一次，就把RTO增大一些，具体做法就是新的重传时间设为就的重传时间的2倍。



#### TCP的流量控制：

**1.利用滑动窗口来实现流量控制。流量控制就是要发送方的发送速率不要太快，要让接收方来得及接收**。一个例子：

​    ![image-20220209180336721](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209180336721.png)

设A向B发送数据。在连接建立时, B告诉了A: "我的接收窗口rwnd =400" (这里rwnd表示receiver window),因此,发送方的发送窗口不能超过接收方给出的接收窗口的数值。**请注意, TCP的窗口单位是字节,不是报文段**。TCP连接建立时的窗口协商过程在图中没有显示出来。再设每一个报文段为100字节长,而数据报文段序号的初始值设为1(见图中第一个箭头上面的序号seq= 1，图中右边的注释可帮助理解整个过程)，**请注意图中箭头上面大写ACK表示首部中的确认位ACK，小写ack表示确认字段的值。**

​	我们应注意到,接收方的主机B进行了三次流量控制。**第一次把窗口减小到rwnd =300，第二次又减到rwnd=100，最后减到rwnd=0，即不允许发送方再发送数据了。**这种使发送方暂停发送的状态将持续到主机B重新发出一个新的窗口值为止。我们还应注意到, **B向A发送的三个报文段都设置了ACK=1，只有在ACK=1时确认号字段才有意义。**

现在我们考虑一种情况。在图5-22中, B向A发送了零窗口的报文段后不久, B的接收缓存又有了一些存储空间。于是B向A发送了rwnd = 400的报文段。然而这个报文段在传送过程中丢失了，A一直等待收到B发送的非零窗口的通知，而B也一直等待A发送的数据。如果没有其他措施,这种互相等待的死锁局面将一直延续下去。

​	为了解决这个问题, TCP为每一个连接设有一个持续计时器(persistence timer),只要TCP连接的一方收到对方的零窗口通知,就启动持续计时器。若持续计时器设置的时间到期，就发送一个零窗口探测报文段(仅携带1字节的数据) ，而对方就在确认这个探测报文段时给出了现在的窗口值。如果窗口仍然是零，那么收到这个报文段的一方就重新设置持续计时器。如果窗口不是零,那么死锁的僵局就可以打破了。

#### TCP拥塞控制

##### 拥塞控制的一般原理。

​	拥塞：在计算机网络中。带宽，交换节点中的缓存和处理机等都是网络的资源，在某段时间，若对网络的某一资源的需求超过该资源所能提供的可用部分，网络的性能就会变差，这种情况就是拥塞。

​	拥塞控制就是：防止过多的数据注入网络中，这样可以使网络中的路由器或者链路不至于过载。**拥塞控制所要做的都有一个前提，就是网络能够承载现有的网络负荷**，拥塞控制是一个全局性的过程，涉及所有的主机，路由器，以及降低网络性能的所有因素。但TCP连接的端点只要迟迟不能收到对方的确认消息，就猜想当前网络某处可能发生了拥塞，但是不能知道拥塞发生在网络的哪处，也无法知道具体原因。

##### TCP的拥塞控制的方法

​	**TCP进行拥塞控制的算法有四种,即慢开始(slow-start)、拥塞避免(congestionavoidance)、快重传(ast retransmit)和快恢复(fast recovery) ,下面就介绍这些算法的原理。**为了集中精力讨论拥塞控制,我们假定:

​	(1)数据是单方向传送的,对方只传送确认报文。

​	(2)接收方总是有足够大的缓存空间，因而发送窗口的大小由网络的拥塞程度来决定。

**(1)慢开始和拥塞避免**

​	下面讨论的拥塞控制也叫做基于窗口的拥塞控制。为此,发送方维持一个叫做拥塞窗口cwnd(congestion window)的状态变量。拥塞窗口的大小取决于网络的拥塞程度,并且动态地在变化。发送方让自己的发送窗口等于拥塞窗口。

​	**发送方控制拥塞窗口的原则是:只要网络没有出现拥塞,拥塞窗口就可以再增大一些,以便把更多的分组发送出去,这样就可以提高网络的利用率。但只要网络出现拥塞或有可能出现拥塞,就必须把拥塞窗口减小一些,以减少注入到网络中的分组数,以便缓解网络出现的拥塞。**

​	发送方又是如何知道网络发生了拥塞呢?我们知道,当网络发生拥塞时,路由器就要丢弃分组。因此只要发送方没有按时收到应当到达的确认报文,也就是说,只要出现了超时,就可以猜想网络可能出现了拥塞。现在通信线路的传输质量一般都很好,因传输出差错而丢弃分组的概率是很小的(远小于1%)，因此,**判断网络拥塞的依据就是出现了超时。**

**慢开始算法**的思路是这样的:当主机开始发送数据时,由于并不清楚网络的负荷情况,所以如果立即把大量数据字节注入到网络,那么就有可能引起网络发生拥塞。经验证明,较好的方法是先探测一下,即由小到大逐渐增大发送窗口,也就是说,**由小到大逐渐增大拥塞窗口数值。**

慢开始规定,在每收到一个对新的报文段的确认后,可以把拥塞窗口增加最多一个SMSS的数值。更具体些,就是

拥塞窗口cwnd每次的增加量=min (N, SMSS)

其中N是原先未被确认的、但现在被刚收到的确认报文段所确认的字节数。不难看出,当N

​	用这样的方法逐步增大发送方的拥塞窗口cwnd,可以使分组注入到网络的速率更加合理。

​	下面用例子说明慢开始算法的原理。请注意,虽然实际上TCP是用字节数作为窗口大小的单位。但为叙述方便起见,我们用报文段的个数作为窗口大小的单位,这样可以使用较小的数字来阐明拥塞控制的原理.

​	在一开始发送方先设置cwnd-1,发送第一个报文段M,,接收方收到后确认M.发送方收到对M,的确认后,把cwnd从1增大到2,于是发送方接着发送M2和M,两个报文段。接收方收到后发回对M2和M3的确认,发送方每收到一个对新报文段的确认(重传的不算在内)就使发送方的拥塞窗口加1,因此发送方在收到两个确认后, cwnd就从2增大到4,并可发送M4-M,共4个报文段,**因此使用慢开始算法后,每经过一个传输轮次(transmission round),拥塞窗口cwnd就加倍。**

![image-20220209182213034](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/image-20220209182213034.png)

为了防止拥塞窗口cwnd增长过大引起网络拥塞,还需要设置一个**慢开始门限 sshresh**状态变量(如何设置ssthresh,后面还要讲),慢开始门限ssthresh的用法如下:

当cwnd < ssthresh时,使用上述的慢开始算法

当cwnd > ssthresh时,停止使用慢开始算法而改用拥塞避免算法。

当cwnd =ssthresh时,既可使用慢开始算法,也可使用拥塞避免算法。

**拥塞避免算法的思路是让拥塞窗口cwnd缓慢地增大,即每经过一个往返时间RTT就把发送方的拥塞窗口cwnd加1**,而不是像慢开始阶段那样加倍增长。因此在拥塞避免阶段就有“加法增大” Al (Additive Increase)的特点。这表明在拥塞避免阶段,拥塞窗口cwnd按**线性规律缓慢增长**,比慢开始算法的拥塞窗口增长速率缓慢得多。

![2562458_1534140701413_37C803982B45FFC1856D88A1BA3CF4A8](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/2562458_1534140701413_37C803982B45FFC1856D88A1BA3CF4A8.png)

当拥塞窗口cwnd =16时(图中的点4),出现了一个新的情况,就是发送方一连收到3个对同一个报文段的重复确认(图中记为3-ACK),关于这个问题要解释如下。

​	有时个别报文段会在网络中丢失,但实际上网络并未发生拥塞。如果发送方迟迟收不到确认,就会产生超时,就会误认为网络发生了拥塞。这就导致发送方错误地启动慢开始,把拥塞窗口cwnd又设置为1,因而降低了传输效率。

​	采用**快重传**算法可以让发送方尽早知道发生了个别报文段的丢失。**快重传算法首先要求接收方不要等待自己发送数据时才进行捎带确认,而是要立即发送确认,**即使收到了失序的报文段也要立即发出对已收到的报文段的重复确认。如图5-26所示,接收方收到了M,和M2后都分别及时发出了确认。现假定接收方没有收到M3;但却收到了M4。本来接收方可以什么都不做。但按照快重传算法,接收方必须立即发送对M2的重复确认，以便让发送方及早知道接收方没有收到报文段M3。发送方接着发送M5和M6。接收方收到后也仍要再次分别发出对M2的重复确认。这样,发送方共收到了接收方的4个对M2的确认,其中后3个都是重复确认。**快重传算法规定,发送方只要一连收到3个重复确认,就知道接收方确实没有收到报文段M3,因而应当立即进行重传(即“快重传”),这样就不会出现超时,**发送方也不就会误认为出现了网络拥塞。使用快重传可以使整个网络的吞吐量提高约20%。

![2020-06-23-19-34-48](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/2020-06-23-19-34-48-16444023530171.png)

因此,在图5-25中的点4,发送方知道现在只是丢失了个别的报文段。于是不启动慢开始,而是执行快恢复算法。这时,发送方调整门限值ssthresh =cwnd /2=8,同时设置拥塞窗口cwnd =ssthresh =8 (见图5-25中的点5),并开始执行拥塞避免算法.

![1099419-20180130152329250-2090977971](%E9%9D%A2%E8%AF%95%E5%BF%85%E5%A4%87%E4%B9%8BTCP%E5%8D%8F%E8%AE%AE.assets/1099419-20180130152329250-2090977971.png)



#### 经典的TCP面试题

**为什么要进行3次握手，两次不行吗？**

三次握手的目的是“**为了防止已经失效的连接请求报文段突然又传到服务端，因而产生错误**”，这种情况是：一端(client)A发出去的第一个连接请求报文并没有丢失，而是因为某些未知的原因在某个网络节点上发生滞留，导致延迟到连接释放以后的某个时间才到达另一端(server)B。本来这是一个早已失效的报文段，但是B收到此失效的报文之后，会误认为是A再次发出的一个新的连接请求，于是B端就向A又发出确认报文，表示同意建立连接。如果不采用“三次握手”，那么只要B端发出确认报文就会认为新的连接已经建立了，但是A端并没有发出建立连接的请求，因此不会去向B端发送数据，B端没有收到数据就会一直等待，这样B端就会白白浪费掉很多资源。如果采用“三次握手”的话就不会出现这种情况，B端收到一个过时失效的报文段之后，向A端发出确认，此时A并没有要求建立连接，所以就不会向B端发送确认，这个时候B端也能够知道连接没有建立。



**为什么建立连接协议是三次握手，而关闭连接却是四次握手呢？**

这是因为服务端的LISTEN状态下的SOCKET当收到SYN报文的建连请求后，**它可以把ACK和SYN（ACK起应答作用，而SYN起同步作用）放在一个报文里来发送。**但关闭连接时，当收到对方的FIN报文通知时，它仅仅表示对方没有数据发送给你了；但未必你所有的数据都全部发送给对方了，所以你可以未必会马上会关闭SOCKET,也即你可能还需要发送一些数据给对方之后，再发送FIN报文给对方来表示你同意现在可以关闭连接了，所以它这里的ACK报文和FIN报文多数情况下都是分开发送的。

**为什么A在TIME-WAIT状态必须等待2MSL的时间呢?这有两个理由。**

​	第一,为了保证A发送的最后一个ACK报文段能够到达B.这个ACK报文段有可能丢失，因而使处在LAST-ACK状态的B收不到对已发送的FIN + ACK报文段的确认。B会超时重传这个FIN + ACK报文段,而A就能在2MSL时间内收到这个重传的FIN + ACK报文段。接着A重传一次确认,重新启动2MSL计时器。最后, A和B都正常进入到CLOSED状态。如果A在TIME-WAIT状态不等待一段时间,而是在发送完ACK报文段后立即释放连接,那么就无法收到B重传的FIN + ACK报文段,因而也不会再发送一次确认报文段。这样, B就无法按照正常步骤进入CLOSED状态。

​	第二,防止上一节提到的“已失效的连接请求报文段”出现在本连接中.A在发送完最后一个ACK报文段后,再经过时间2MSL,就可以使本连接持续的时间内所产生的所有报文段都从网络中消失。这样就可以使下一个新的连接中不会出现这种旧的连接请求报文段。



**参考《计算机网络》谢希仁第七版**