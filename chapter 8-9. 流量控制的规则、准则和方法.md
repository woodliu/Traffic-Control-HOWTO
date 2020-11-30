## 8. 流量控制的规则、准则和方法

### 8.1. Linux流量控制的通用规则

可以使用如下通用规则来学习Linux流量控制。可以使用[**tcng**](https://tldp.org/en/Traffic-Control-HOWTO/ar01s05.html#s-tcng) 或 [tc](https://tldp.org/en/Traffic-Control-HOWTO/ar01s05.html#s-iproute2-tc)进行初始化配置Linux下的流量控制结构。

- 任何执行整流功能的路由器都应该成为链路的瓶颈，并且应该调整为略低于最大可用的链路带宽。通过整流可以防止在其他路由器中形成队列，从而最大程度地控制到整流设备的报文的延迟/延期。
- 一个设备可以对其传输的流量进行调整。由于已经在输入接口上接收到流量，因此无法调整这类流量。 解决此问题的传统方法是使用ingress策略。
- 每个接口必须包含一个[`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc)。当没有明确附加其他qdisc时会使用默认的qdisc( [`pfifo_fast`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-pfifo_fast) qdisc)。
- 如果一个接口附加了[classful qdiscs](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html) ，但没有任何子类，这种情况会无意义地消耗CPU。
- 任何新创建的类都包含一个 [FIFO](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-fifo)。可以使用任何qdisc来替换这个qdisc。当一个子类附加到该类时，会隐式地删除FIFO qdisc。
- 直接附加到root qdisc的类可以模拟虚拟电路。
- 可以在类或任意一种[classful qdiscs](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html)上附加过滤器。

### 8.2. 处理已知带宽的链路

当一个链路已知带宽时，HTB是一个理想的qdisc，因为可以给最内层的（最根）类设置给定链接上的最大可用带宽。流量后续会被分割到子类中，用于保证特定类型的流量的带宽或允许优先选择特定类型的流量。

### 8.3. 处理已知可变带宽的链路

理论上，PRIO是一个理想的用于处理可变带宽的调度器，因为它是一个连续工作的qdisc（这意味着它不提供整流）。在带宽未知或波动的链路中，PRIO 调度器更喜欢优先将具有最高优先级的band中的所有报文出队列，然后再处理较低优先级的队列中。

### 8.4. 基于流的带宽共享和分割

在多种类型的网络带宽竞争中，这通常是比较容易处理的竞争类型之一。通过使用SFQ，特定队列中的流量可以分为多条流，然后公平地处理该队列中的每条流。表现良好的应用程序（和用户）会发现，使用SFQ和ESFQ足以满足大多数共享需求。

这些公平排队算法的致命弱点是无法抵御行为不端的用户或应用程序同时打开很多连接(例如，eMule, eDonkey, Kazaa)。通过创建大量独立的流，应用程序可以控制公平排队算法中的时间间隙。重申一下，公平排队算法不知道单个应用程序会生成大多数流，并且不会惩罚用户。 需要依赖其他方法。

### 8.5. 基于IP的带宽共享和分割

对于许多管理员来说，这是在用户之间划分带宽的理想方法。不幸的是，没有简单的解决方案，而且随着共享网络连接的机器数量的增加，它变得越来越复杂。

为了在N个IP地址之间公平地分配带宽，必须有N个类。

## 9. 使用QoS/流量控制的脚本

### 9.1. wondershaper

更多参见 [wondershaper](http://lartc.org/wondershaper/).

### 9.2. ADSL Bandwidth HOWTO script (`myshaper`)

更多参见[myshaper](http://www.tldp.org/HOWTO/ADSL-Bandwidth-Management-HOWTO/implementation.html).

### 9.3. `htb.init`

更多参见[`htb.init`](http://sourceforge.net/projects/htbinit/).

### 9.4. `tcng.init`

更多参见 [`tcng.init`](http://linux-ip.net/code/tcng/tcng.init).

### 9.5. `cbq.init`

更多参见 [`cbq.init`](http://sourceforge.net/projects/cbqinit/).