## Linux流量控制的组件

流量控制元素与Linux组件之间的相关性：

| traditional element                                          | Linux component                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 入队列                                                       | 修订：从用户或网络接收报文                                   |
| [整流](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-shaping) | [class](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-class) 提供了整流的能力 |
| [调度](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-scheduling) | 一个 [`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc) 就是一个调度器。调度器可以是一个简单的FIFO，也可以变得很复杂，包括classes和其他qdiscs，如HTB。 |
| [分类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-classifying) | [`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter) 对象通过一个[`classifier`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-classifier) 对象执行分类。严格上讲，除filter之外的组件不会用到分类器。 |
| [策略](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-policing) | [`policer`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)仅作为[`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter)的一部分而存在。 |
| [丢弃](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-dropping) | [`drop`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-drop) 流量要求使用一个带 [`policer`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)的 [`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter) ，动作为"drop" |
| [标记](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-marking) | `dsmark` [`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc) 用于标记报文。 |
| 入队列;                                                      | 驱动队列位于qdisc和[网络接口控制器(NIC)](https://tldp.org/en/Traffic-Control-HOWTO/ar01s02.html#o-nic)之间。驱动队列给上层(IP栈和流量控制子系统)提供了数据异步入队列的位置(后续由硬件对数据进行操作)。队列的大小由[Byte Queue Limits (BQL)](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-bql)动态设置。 |

#### 4.1 qdisc

简单讲，一个qidsc就是一个调度器。每个出接口都需要某种类型的调度器，默认的调度器为FIFO。Linux下的其他qdisc会根据调度器的规则来重新安排进入调度器队列的报文。

qdisc是构建所有Linux流量控制的主要部件，也被称为排队规则。

 [classful qdiscs](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html) 可以包含[类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-class)，并提供了可以附加到[过滤器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter)的句柄。一个classful qidsc可以不使用子类，但这样通常会消耗CPU周期和其他系统资源，且毫无意义。

[classless qdiscs](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html) 不包含类，也不会附加过滤器。由于一个classless qdisc不包含任何类的子类，因此不能使用[分类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-classifying)，意味着不能附加任何过滤器。

在使用中可能会对术语`root` qdisc 和`ingress` qdisc产生混淆。实际中并不存在真正的排队规则，而是连接流量控制结构的出站(出流量)和入口(入流量)的位置。

每个接口都会包含`root` qdisc 和`ingress` qdisc。最主要和最常用的是egress qdisc，即`root` qdisc，它可以包含任何排队规则([`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc)s)以及潜在的[类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-class)和类结构。大部分文档适用于`root` qdisc及其子qdisc。一个接口传输的流量会经过egress或`root` qdisc。

一个接口上接收到的流量会经过`ingress` qdisc。由于其功能的限制，不允许创建子类，且仅允许存在一个被[过滤器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter) 附加的对象。事实上，ingress qdisc仅仅是一个对象，可以在其上附加[策略器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)来限制网络接口上接收的流量。

总之，由于egress qdisc包含一个真正的qdisc，且具有流量控制系统的全部功能，因此可以使用egress qdisc做很多事情。而一个`ingress` qdisc仅支持一个策略器。除非另有说明，本文后续将主要关注附加到root qdisc的流量控制结构。

### 4.2 类

类仅会存在于classful [`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc) (如 [HTB](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#qc-htb) 和 [CBQ](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#qc-cbq))。类非常灵活，可以包含多个子类或单个子qdisc。一个子类本身也可以包含一个classful qdisc，通过这种方式可以实现复杂的流量控制场景。

任何类都可以附加任意多的[过滤器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter)，从而允许选择一个子类或使用过滤器来重新分类或直接丢弃进入特定类的流量。叶子类是qdisc中的终止类，它包含一个qdisc(默认是[FIFO](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-fifo))，且不会包含子类。任何包含子类的类都属于内部类(或root类)，而非叶子类。

### 4.3 过滤器

过滤器是Linux流量控制系统中最复杂的组件，提供了将流量控制的主要元素粘合到一起的机制。过滤器最简单和最明显的角色就是对报文进行分类([Section 3.3, “Classifying”](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-classifying))。Linux过滤器允许用户使用多个或单个过滤器来将报文分类到一个输出队列。

- 一个过滤器必须包含一个[分类器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-classifier)
- 一个过滤器可能包含一个[策略器](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)

过滤器可能附加到classful qdiscs或[类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-class)，但入队列的报文总是首先进入root qdisc。在报文经过的root qdisc上附加的过滤器后，报文可能被重定向到任何子类(子类可以包含自己的过滤器)，后续可能对报文进一步分类。

#### 4.4 分类器

过滤器的对象，可以使用[**tc**](https://tldp.org/en/Traffic-Control-HOWTO/ar01s05.html#s-iproute2-tc)进行操作，且可以使用不同的分类机制，其中最常用的是`u32`分类器。u32分类器允许用户根据报文的属性选择报文。

分类器可以作为过滤器的一部分来标识报文的特征或元数据。Linux分类器对象可以看作是流量控制[分类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-classifying)的基本操作和基本机制。

#### 4.5 策略器

该机制仅作为Linux流量控制中的过滤器的一部分。一个策略器可以在速率超过指定速率时执行一个动作，在速率低于指定速率时执行另一个动作，善用策略可以模拟出一个三色表。参见 [Section 10, “Diagram”](https://tldp.org/en/Traffic-Control-HOWTO/ar01s10.html)。

虽然[策略](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-policing) 和[整流](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-shaping) 都是流量控制中用来限制带宽的基本元素，但使用策略器并不会导致流量延迟。它只会根据特定的准则来执行某个动作。参见[Example 5, “tc `filter`”](https://tldp.org/en/Traffic-Control-HOWTO/ar01s05.html#ex-s-iproute2-tc-filter)。

### 4.6 丢弃

该流量控制机制仅作为策略器的一部分。任何附加到过滤器的策略器都包含一个[drop](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-drop)动作。

> 注：策略器是流量控制系统中唯一可以显式地丢弃报文的地方。策略器可以限制入队列的报文的速率，或丢弃匹配特定模式的所有流量。

流量控制系统中，报文的丢失可能是由某个动作引起的副作用。例如，如果使用的调度器使用和[GRED](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-gred)一样的方法控制流时，报文将被丢弃。

或者，当出现突发或超负荷时，如果整流器或调度器的缓冲用尽，也可能会丢弃报文。

### 4.7 句柄

每个类和classful qdisc([Section 7, “Classful Queuing Disciplines (qdiscs)”](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html))都要求在流量控制结构中存在一个唯一的标识符，该唯一标识符被称为句柄，每个句柄包含两个组成成员，一个主号和一个次号。用户可以根据以下规则随意分配这些号。

**类和qdiscs的句柄号**:

**主号**

- 该参数对内核完全没有意义。用户可能会任意使用一个编号方案，但流量控制结构中具有相同父qdisc的所有对象必须共享一个次句柄号。对于直接附加到root qdisc的对象，传统的编号方案会从1开始。

**次号**

- 如果次号为0，则表明该对象为qdisc，否则表明该对象为一个类。所有共享同一个qdisc的类必须包含一个唯一的次号。

特殊的句柄 `ffff:0` 保留给`ingress` qdisc使用。

句柄作为**tc**过滤器的classid和flowid的目标参数，同时也是用户侧应用使用的标识对象的外部标识符。内核为每个对象维护内部标识符。

### 4.8 txqueuelen

可以使用**ip**或**ifconfig**目录获取当前传输队列的长度。令人困惑的是，这些命令对传输队列长度的命名各部不同：

```shell
$ifconfig eth0

eth0      Link encap:Ethernet  HWaddr 00:18:F3:51:44:10
          inet addr:69.41.199.58  Bcast:69.41.199.63 Mask:255.255.255.248
          inet6 addr: fe80::218:f3ff:fe51:4410/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:435033 errors:0 dropped:0 overruns:0 frame:0
          TX packets:429919 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:65651219 (62.6 MiB)  TX bytes:132143593 (126.0 MiB)
          Interrupt:23
      
$ip link

1: lo:  mtu 16436 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0:  mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:18:f3:51:44:10 brd ff:ff:ff:ff:ff:ff
```

Linux默认的传输队列的长度为1000个报文，这是一个相当大的缓冲(特别是当带宽比较低时)。(为了理解其原因，请参见针对延迟和吞吐量的讨论，特别是[缓冲膨胀](https://tldp.org/en/Traffic-Control-HOWTO/ar01s02.html#o-bufferbloat)。)

更有趣的是，`txqueuelen`仅用作下面排队规则的默认队列长度。

- `pfifo_fast` (Linux的默认队列规则)
- `sch_fifo`
- `sch_gred`
- `sch_htb` (仅用于默认队列)
- `sch_plug`
- `sch_sfb`
- `sch_teql`

txqueuelen参数控制上述QDiscs的队列大小。对于大多数的队列规则，tc命令行中的`limit`参数会覆盖默认的txqueuelen 值。总之，如果没有使用上述的任意一种队列规则或覆盖了默认的队列长度，那么txqueuelen 就没有任何意义。

可以使用**ip**或**ifconfig**命令来配置接口的传输队列长度。

```
ip link set txqueuelen 500 dev eth0
```

注意：ip命令使用`qlen`来表示`txqueuelen`。

### 4.9 驱动队列(即ring buffer)

在IP栈和网络接口控制器之间存在驱动队列。该队列通常使用先进先出的ring buffer来实现(可以认为是一个固定长度的缓冲)。驱动队列不包含任何报文数据，仅包含指向其他数据结构(socket kernel buffers，简称SKBs)的描述符，SKB包含报文数据，并在整个内核中使用。

![](https://img2020.cnblogs.com/blog/1334952/202011/1334952-20201116092208609-1435526002.png)

驱动队列的输入源为保存了完整IP报文的IP栈，这些报文可能是本地的，或当设备作为路由器时接收到的需要从一个NIC路由到另一个NIC的报文。IP栈会将报文添加到驱动队列，并由硬件驱动出队列，在传输时会通过数据总线发送到NIC硬件。

驱动队列存在的原因是为了保证在任何时候，当系统需要传输数据时，NIC会立即传输该数据。即，驱动队列为IP栈和硬件操作提供了一个异步处理数据的位置。一个备选方案是，一旦物理媒介就绪时就向IP栈查询可用的报文。但由于对这类请求的响应不可能是即时的，因此这种设计浪费了宝贵的传输机会，导致吞吐量降低。相反的方案是，IP栈在创建一个报文后会等待硬件就绪，这种方案同样不理想，因为IP栈将无法继续其他工作。

更多关于驱动队列的细节参见[5.5章节](https://tldp.org/en/Traffic-Control-HOWTO/ar01s05.html#s-ethtool)。

### 4.10 Byte Queue Limits(BQL)

Byte Queue Limits (BQL) 是Linux内核(> 3.3.0 )引入的一个新特性，用于尝试自动解决驱动程序队列大小的问题。该特性添加了一层处理，它会根据当前系统的情况计算出避免出现[饥饿](https://tldp.org/en/Traffic-Control-HOWTO/ar01s02.html#o-starv-lat)的最小缓冲大小，以此作为报文进入驱动队列的依据。回顾一下，队列中的数据总量越少，队列中的报文的最大延迟越小。

需要注意的是，BQL不会修改驱动队列的实际长度，相反，它会计算当前时间可以入队列的数据的(字节数)上限。当队列中的数据超过该限制之后，驱动队列的上层需要决定是否保留会丢弃这部分数据。

当发生两种情况时会触发BQL机制：当报文进入驱动队列，或当线路上的传输已经结束。下面给出了一个简单的BQL算法。LIMIT指BQL计算出的值。

```
****
** After adding packets to the queue
****

if the number of queued bytes is over the current LIMIT value then
        disable the queueing of more data to the driver queue
    
```

BQL基于测试设备是否发生了饥饿现象，如果是，则增加LIMIT来允许更多的报文入队列，以此降低饥饿的概率。如果设备繁忙，且后续还有报文持续传输到队列中，当队列中的报文大于当前系统所需要的数量时，会降低LIMIT来限制饥饿。

下面给出一个真实的例子，可以帮助了解BQL能够在多大程度上影响排队的数据量。在一台服务器上，驱动队列的大小默认为256个描述符。由于以太网的MTU为1500字节，意味着驱动队列中的报文最大为256 * 1,500 = 384,000字节(禁用TSO，GSO等)。但此时BQL计算出的限制值为3012字节。如你所见，BQL大大限制了进入队列的数据量。

从名称的第一个单词可以推断出BQL的一个有趣的特点--字节。与驱动队列和其他大多数报文队列不同，BQL操作的是字节。这是因为相比报文数或描述符，字节数与物理媒介的传输时间有着更为直接的关系。

BQL将进入队列的数据量限制到避免饿死所需的最小数量，从而减少了网络延迟。它还有一个非常重要的副作用，那就是将大多数报文排队的点从驱动队列(一个简单的FIFO)移动到排队规则(QDisc)层，从而实现更复杂的排队策略。下一节将介绍Linux的QDisc层。

#### 4.10.1 设置BQL

BQL算法是自适应的，并不需要过多的人为接入。但如果需要关注低比特率下的最佳延迟，则有可能需要覆盖计算出的LIMIT值。可以在/sys目录根据NIC的名称和位置下找到BQL的状态和配置。例如我的一台服务器上的eth0 的目录为：

> 可以使用`ethtool -i <接口名称>`来查看设备的PCI号。

```
/sys/devices/pci0000:00/0000:00:14.0/net/eth0/queues/tx-0/byte_queue_limits
```

该目录中的文件为：

- *hold_time:* 修改LIMIT的间隔时间，单位毫秒
- *inflight:* 队列中还没有传输的字节数
- *limit:* BQL计算出的LIMIT值。如果NIC驱动不支持BQL，则为0
- *limit_max:* 可配置的LIMIT的最大值，减小该值可以优化延迟
- *limit_min:* 可配置的LIMIT的最小值，增大该值可以优化吞吐量

要对可排队的字节数设置上限，请将新值写入limit_max文件：

```
echo "3000" > limit_max
```