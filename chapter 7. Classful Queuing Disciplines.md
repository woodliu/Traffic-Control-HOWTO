## Classful Queuing Disciplines

可以使用classful qdisc的代理来解锁Linux流量控制的灵活性和控制力。classful qdisc可以附加过滤器，允许将报文重定向到特定的类和子队列。

有几个常见的术语用来描述直接附加到`root` qdisc和终止类的类。附加到`root` qdisc的类称为根类，一般为内部类。任何特定qdisc中的终止类称为叶类，类似于树形结构的类。除了把结构比喻成一棵树外，通常也会使用家庭关系进行比喻。

### 7.1. HTB, 层级令牌桶

HTB是Linux中CBQ(参阅第7.4章)qdisc的一种更易理解和直观的替换品。CBQ和HTB可以控制给定链路上的出站带宽。这两种方式都可以使用一个物理链路来模拟多个较慢的链接，并将不同的链路发送到不同的模拟链路上。在这两种情况下，必须指定如何将物理链路划分为模拟链路，以及确定要发送的报文使用哪个模拟链路。

HTB使用了令牌和桶的概念，并使用了基于类的系统和过滤器对流量进行复杂和细粒度的控制。通过一个复杂的[借用](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#qc-htb-borrowing)模型，HTB可以实现各种复杂的流量控制技术。另一种最简单的方式是在整流时使用HTB。

通过理解令牌和桶或掌握HTB的功能可以了解到，HTB仅仅是一个逻辑上的步骤。该qdisc允许用户定义令牌和桶的特性，并允许用户任意嵌套这些桶。当与分类方案结合使用时，可以以非常细粒度的方式控制流量。

例11. HTB的tc用法：

```
Usage: ... qdisc add ... htb [default N] [r2q N]
 default  minor id of class to which unclassified packets are sent {0}
 r2q      DRR quantums are computed as rate in Bps/r2q {10}
 debug    string of 16 numbers each 0-3 {0}

... class add ... htb rate R1 burst B1 [prio P] [slot S] [pslot PS]
                      [ceil R2] [cburst B2] [mtu MTU] [quantum Q]
 rate     rate allocated to this class (class can still borrow)
 burst    max bytes burst which can be accumulated during idle period {computed}
 ceil     definite upper class rate (no borrows) {rate}
 cburst   burst but for ceil {computed}
 mtu      max packet size we create rate map for {1600}
 prio     priority of leaf; lower are served first {0}
 quantum  how much bytes to serve from leaf at once {use r2q}

TC HTB version 3.3
```

#### 7.1.1. 软件要求

不同于前面讨论的几乎所有软件，HTB是一个新的qdisc，现有的发行版可能缺少使用HTB所需要的所有软件和能力。内核版本2.4.20及其以后的版本会支持HTB，早期的内核版本需要打补丁。为了在用户空间支持HTB，参见[HTB](http://luxik.cdi.cz/~devik/qos/htb/)。

#### 7.1.2. 整流

HTB的最常见应用之一是将传输的流量调整到特定速率。

所有的整流都发生在叶类上。内部或根类上不会发送整流，这些类仅会在借用模型中给出如何分配可用的令牌。

#### 7.1.3. 借用

HTB的一个基本功能是借用机制。当子类超速率之后会借用父类的令牌。在达到[*`ceil`*](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#vl-qc-htb-params-ceil)(此时子类会有数据包排队，等待传输，直到有更多可用的令牌为止。)之前，子类会持续尝试借用父类的令牌。由于使用HTB仅可以创建两种主要的类(叶子类和内部类)，因此下面的表和图区分了借用机制的各种可能的状态和行为。

表2.HTB类型状态和和潜在的action令牌

| type of class | class state            | HTB internal state | action taken                                                 |
| ------------- | ---------------------- | ------------------ | ------------------------------------------------------------ |
| leaf          | < *`rate`*             | *`HTB_CAN_SEND`*   | 叶子类会出站队列中的数据，不能大于突发的报文。               |
| leaf          | > *`rate`*, < *`ceil`* | *`HTB_MAY_BORROW`* | 叶子类会尝试从父类借用令牌(tokens/ctokens)。如果有可用的令牌，则以*`quantum`*增量的方式借出，且叶子类的出站速率不能大于`cburst`字节。 |
| leaf          | > *`ceil`*             | *`HTB_CANT_SEND`*  | 没有出队列的报文。会导致报文延迟，并增加延迟以满足所需的速率。 |
| inner, root   | < *`rate`*             | *`HTB_CAN_SEND`*   | 内部类会给子类借用令牌                                       |
| inner, root   | > *`rate`*, < *`ceil`* | *`HTB_MAY_BORROW`* | 叶子类会尝试从父类借用令牌(tokens/ctokens)，父类会以*`quantum`*递增的方式将令牌借给竞争的子类。 |
| inner, root   | > *`ceil`*             | *`HTB_CANT_SEND`*  | 内部类不会尝试从父类借用令牌，且父类不会将令牌借给子类。     |

下图展示了借用令牌的流程，被借用的令牌会计入父类。为了借用模型能够正常工作，每个类都必须精确计算自身和子类使用的令牌。基于这种原则，子类或叶子类使用的令牌会计入父类中，直到到达root类。

任何想要借用令牌的子类会从其父类中请求一个令牌，如果父类也达到了`rate`的限制，它将会向自己的父类借用令牌，以此类推，直到找到一个可用的令牌或达到root类为止。因此借用的令牌会流向叶子类，而对令牌的统计则会流向root类。

![](https://img2020.cnblogs.com/blog/1334952/202011/1334952-20201123135802141-66635116.png)

注意：上图中有好几个HTB root类，每个root类都可以模拟一个虚拟回路。

#### 7.1.4. HTB 类参数

*`default`*

每个HTB qdisc对象可选的参数，默认的default为0，这样未分类的流量会使用硬件速度出队列，不会经过附加到root qdisc的所有类。

*`rate`*

用于设置限制传输流量的最小速度。可以认为其相当于信息的提交速率，或为一个给定的叶子类保证的带宽。

*`ceil`*

用于设置限制传输流量的最大速度。借用模型应该说明如何使用该参数。可以认为其等同于"突发的带宽"。

*`burst`*

`rate`的桶大小。HTB在等待更多令牌前可以入队列`burst`字节的数据。

*`cburst`*

[*`ceil`*](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#vl-qc-htb-params-ceil) 的桶大小。HTB在等待更多ctokens前可以入队列`cburst `字节的数据。

*`quantum`*

这是HTB用于控制借用的关键参数。通常HTB会计算一个正确的*`quantum`* ，用户无需指定。修改该值会对竞争下的借用和整流造成巨大的影响(因为它会在流量超过rate(且低于ceil)的子类间对流量进行分割，并传输这些子类的报文)。

*`r2q`*

提供给用户使用，用于帮助优化特定类的`quantum`。

*`mtu`*

*`prio`*

在轮询处理中，可以优先处理具有最低优先级字段数值的类对应的报文。强制字段。

*`prio`*

类在层次结构中的位置。如果一个类直连到一个qdisc而不是另一个类，则可以省略minor(见下)。强制性字段。

*`prio`*

与qdisc类型，也可以命名类。major号必须等于其附加到的qdisc的major号。该字段是可选的，但在类包含子类时需要配置该字段。

#### 7.1.5. HTB root 参数

一个HTB qdisc类树的root包含三个字段：

`parent major:minor | root`

该强制参数确定了HTB实例的位置：接口的root位置还是位于一个已存在的类中。

`handle major:`

与其他的qdisc类似，可以给HTB分配一个句柄，该句柄应该包含一个major号，后跟一个冒号。可选字段，但如果类将在这个qdisc中生成，则非常有用。

`default minor-id`

未分类的流量会使用该minor-id发送到类。

#### 7.1.6. 规则

以下是从http://www.docum.org/docum.org/和[(new) LARTC mailing list](http://www.spinics.net/lists/lartc/) (也可以参见 [(old) LARTC mailing list archive](http://mailman.ds9a.nl/mailman/listinfo/lartc/))中挑选的使用HTB的一般准则。这些准则可以方便初学者在最大程度上了解HTB。

- HTB的整流仅发生在叶子类上。

- 由于HTB不会在除叶子类的类上进行整流，因此叶子类的*`rate`*s 之和不能大于父类的`ceil`。理想情况下子类的*`rate`*s 之和应该与父类的`ceil`相匹配，允许父类将剩余的带宽(ceil - rate)分配给子类。

  在使用HTB时，会多次重复这个关键概念。只有叶子类才会真正进行整流；报文只会在这些叶子类上延迟。内部类(到root类路径上的类)定义了如何进行借入/借出(参见[Section 7.1.3, “Borrowing”](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html#qc-htb-borrowing))。

- 只有在一个类大于`rate`但低于`ceil`时才会用到*`quantum`* 。

- *`quantum`* 应该设置为等于或大于MTU的值。即使在*`quantum`* 很小的情况下，HTB也会至少给每个服务一次报文入队列的机会。这种情况下，HTB将无法精确计算真实使用的带宽。

- 父类以增量为*`quantum`*的方式将令牌提供给子类，以便获得最大的粒度和最均匀的瞬时带宽分布，*`quantum`* 应该尽量小，但不能小于MTU。

- tokens和ctokens的不同点仅对叶子类有意义，因为非叶子的类仅会借给子类令牌。

- 对HTB借用的更精确的描述应该是"使用"(并不会归还)

#### 7.1.7. 分类

如前面所述，一个HTB实例可能会包含很多类，每个类都包含一个qdisc，默认为[tc-pfifo](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-fifo)。当入队列一个报文时，HTB会从root类开始，使用多种方式来决定哪个类去接收该数据。在没有特殊配置选项的情况下，处理会相当简单。在树的每个节点上查找一条指令，然后转到指令指向的类。如果找到的类是一个叶子类，则将报文入队列到此处，如果不是一个叶子节点，则从该节点开始重复上述工作。

在访问的每个节点上会执行以下操作，直到发送到另一个节点(子节点)或终止该过程为止。

- 查询附加到类的过滤器。如果发送到一个叶节点，则工作完成。否则，重新启动。
- 如果上述操作没有返回指令，则在该节点上将报文入队列。

这种算法会确保报文总是在某个地方结束。

### 7.2. HFSC, 分层公平服务曲线(Hierarchical Fair Service Curve)

HFSC classful qdisc会对延迟敏感的流量和吞吐量敏感的流量进行平衡。当处于拥塞或挤压状态下时，HFSC排队规则会在需要时根据服务曲线定义穿插处理对延迟敏感的流量。

### 7.3. PRIO, 优先级调度器

PRIO classful qdisc的工作原理非常简单。当它需要入队列一个报文时，会检查第一个类，如果该类包含报文，则将该报文入队列，否则检查下一个类，直到最后一个类。PRIO是一个不会延迟报文的调度器，它是一个连续工作的qdisc(尽管包含在类中的qdisc可能不是连续工作的)。

#### 7.3.1. 算法

当使用tc qdisc add命令创建PRIO时，会创建固定数目的bands(与pfifo类似)。每个band就是一个类(虽然不能使用tc class add添加类)。创建qdisc的时候创建的band的数目是固定的。

当入队列报文时，总会优先检查band 0。如果band 0中没有报文，则PRIO会检查band 1，以此类推。具有最大可靠性的报文会进入band 0，最小延迟的报文会进入band 1，其余进入band 2。

由于PRIO本地包含minor号0，band 0实际上就是major:1，band 1为major:2，等等。对于major，可以用handle参数替换'tc qdisc add'上分配给qdisc的major号。

#### 7.3.2. 用法

```
$ tc qdisc ... dev dev ( parent classid | root) [ handle major: ] prio [bands bands ] [ priomap band band band...  ] [ estimator interval time‐constant ]
```

#### 7.3.3. 分类

有三种方式决定一个报文入队列时的band：

- 在用户空间中，具有足够特权的进程可以直接使用SO_PRIORITY对目标类进行编码。
- 通过编程，附加到root qdisc的tc过滤器可以将任何流量直接指向一个类。
- 通常，参考priomap，报文的优先级是从分配给报文的服务类型(ToS)派生出来的。

只有此qdisc指定了priomap。

#### 7.3.4. 可配置的参数

- bands：不同band的数目，如果该值不是默认值3，则需要更新priomap。

- priomap：附加到root qdisc的tc过滤器，可以将流量直接指向一个类。

一个*priomap* 指定了该qdisc如何将一个报文映射到一个特定的band。对报文的映射基于其TOS的值。

```
   0     1     2     3     4     5     6     7
+-----+-----+-----+-----+-----+-----+-----+-----+
|   PRECEDENCE    |          ToS          | MBZ |   RFC 791
+-----+-----+-----+-----+-----+-----+-----+-----+

   0     1     2     3     4     5     6     7
+-----+-----+-----+-----+-----+-----+-----+-----+
|    DiffServ Code Point (DSCP)     |  (unused) |   RFC 2474
+-----+-----+-----+-----+-----+-----+-----+-----+
```

RFC 791 和RFC 2474对(4个比特位的)TOS的定义稍微有所不同，后者取代了前者在格式上的定义，但不是所有的软件，系统和术语都能及时跟上这种变化。因此，报文分析程序通常会使用Type of Service (ToS)而不是DiffServ Code Point (DSCP)。

 **[RFC 791](https://tools.ietf.org/rfc/rfc791.txt)** 对IP的TOS首部的解析如下：

| Binary | Decimal | Meaning                      |
| ------ | ------- | ---------------------------- |
| 1000   | 8       | Minimize delay (md)          |
| 0100   | 4       | Maximize throughput (mt)     |
| 0010   | 2       | Maximize reliability (mr)    |
| 0001   | 1       | Minimize monetary cost (mmc) |
| 0000   | 0       | Normal Service               |

由于这4个比特位的右边还有一个比特位，因此TOS字段的实际值为TOS比特值的两倍。运行 **tcpdump -v -v**可以显示整个TOS字段的值，而不仅仅4个比特位的值。

下表展示了如何将TOS值映射到priomap band：

| ToS Field | ToS Bits | Meaning                      | Linux Priority | Band |
| --------- | -------- | ---------------------------- | -------------- | ---- |
| 0x0       | 0        | Normal Service               | 0 Best Effort  | 1    |
| 0x2       | 1        | Minimize Monetary Cost (mmc) | 1 Filler       | 2    |
| 0x4       | 2        | Maximize Reliability (mr)    | 0 Best Effort  | 1    |
| 0x6       | 3        | mmc+mr                       | 0 Best Effort  | 1    |
| 0x8       | 4        | Maximize Throughput (mt)     | 2 Bulk         | 2    |
| 0xa       | 5        | mmc+mt                       | 2 Bulk         | 2    |
| 0xc       | 6        | mr+mt                        | 2 Bulk         | 2    |
| 0xe       | 7        | mmc+mr+mt                    | 2 Bulk         | 2    |
| 0x10      | 8        | Minimize Delay (md)          | 6 Interactive  | 0    |
| 0x12      | 9        | mmc+md                       | 6 Interactive  | 0    |
| 0x14      | 10       | mr+md                        | 6 Interactive  | 0    |
| 0x16      | 11       | mmc+mr+md                    | 6 Interactive  | 0    |
| 0x18      | 12       | mt+md                        | 4 Int. Bulk    | 1    |
| 0x1a      | 13       | mmc+mt+md                    | 4 Int. Bulk    | 1    |
| 0x1c      | 14       | mr+mt+md                     | 4 Int. Bulk    | 1    |
| 0x1e      | 15       | mmc+mr+mt+md                 | 4 Int. Bulk    | 1    |

第二列包含相关的四个ToS位的值，以及它们的含义。例如，15表示想要最小货币成本，最大可靠性，最大吞吐量以及最小延迟。

第四列列出了Linux内核解析TOS比特位的方式，展示了TOS映射到的优先级。

最后一列展示了默认的priomap值。在命令行中，默认的priomap为：`1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1`，其对应的优先级为4，对应的band为1。priormap允许更高的优先级(>7)，这类优先级并不对应TOS的映射，表示其他的含义。

#### 7.3.5. 类

无法对PRIO类进行进一步的配置——它们在附加PRIO qdisc时自动创建。但每个类可以包含更多的qdisc。

#### 7.3.6. Bugs

当低band中包含了大量流时，可能会导致高band饥饿。可以给这些band附加一个整流器，以确保这些band不会占用大部分链路。

### 7.4. CBQ, 基于类的队列 (CBQ)

CBQ是一个流量控制系统的类实现。CBQ是一个classful qdisc，它可以在类层次结构中共享链路。它包含整流元素以及优先级功能。通过对入队列事件的链路空闲时间和对底层链路带宽的了解来实现整流。

#### 7.4.1. 整流算法

通过计算链路空闲时间以及设置当结果偏离了设定的限定值时所采取的动作来实现整流。

当一个10mbit/s连接到1mbit/s时，该链路上90%的时间都是空闲的。如果不是，则需要对其进行限流，使其90%的时间处于空闲状态。

从内核的角度很难对流量进行衡量，因此，CBQ会根据设备驱动程序请求数据之间的毫秒数计算空闲时间。结合报文的大小，可以近似知道链路满或空的程度。

应当谨慎应对这种结果，因为并不是每次都能计算出合适的结果。在不太真实的网络设备（例如以太网上的PPP或TCP / IP上的PPTP）的情况下，对物理链路带宽的定义可能不正确。

在操作时，有效空闲时间是用指数加权移动平均(EWMA)来进行测量的。这种针对空闲状态计算出的最近的报文数是以前的报文数的指数倍。EWMA是一种有效的计算方法，可以解决系统处于活动状态或非活动状态的问题。例如，UNIX系统的平均负载就采用了这种计算方式。

从EWMA测得的值中减去计算出的空闲时间，结果称为`avgidle`(平均空闲时间)。一个完美加载的链路的`avgidle`应该为0：接收的报文间隔等于计算的结果。

一个过载的链路的`avgidle`为负值，如果负值过大，则CBQ会限流。相反，如果一个空闲的链路的`avgidle`过大，则在一段时间的静默后，可能会允许无限的带宽。为了防止发生这种情况，将`avgidle`的上限是`maxidle`。

从理论上讲，如果超出限制，CBQ会严格限制计算出的用于放通报文的时间，然后放通一个报文，然后再进行限流。由于定时器精度的限制，这种方式可能不可行，参见下图的`minburst`参数。

#### 7.4.2. 分类

在CBQ qdisc下可能存在很多类。每个类会包含其他qdisc，默认为[tc-pfifo](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-fifo)。

当入队列一个报文时，CBQ会从root开始，使用多种方法来确定使用哪个类来接收该数据。如果做出判定，则对接收的类重复此过程，该类可能有进一步的方法来将流量分类到子类(如果有的话)。CBQ可以使用如下方法将一个报文分类给任何子类：

- skb->priority类编码。可以由设置了SO_PRIORITY setsockopt的用户空间的应用进行设置。skb->priority类编码只适用于当skb->priority持有此qdisc中现有类的major:minor句柄的情况。
- 附加到该类的tc过滤器
- 一个类的defmap，由split和defmap参数设置。defmap 可能包含针对每个可能的Linux报文优先级的指令。

每个类也有一个级别。连接到类层次结构底部的叶节点的级别为0。

#### 7.4.3. 分类算法

分类是一个循环，当达到叶子类时终止。循环中的任一点都可能跳到一个回退算法中。循环包含如下步骤：

- 如果报文是本地生成的，且在skb->priority中编码了一个有效的classid，则选择该类，并终止循环。
- 查询附加到该子类的tc过滤器(如果存在)。如果返回的类不是叶子类，则从返回的类上重新执行循环。如果返回的类是叶子类，则选择该类并终止循环。
- 如果tc过滤器没有返回类，也没有返回类的有效引用，则将引用的minor 号作为优先级，然后从该类的defmap中检索一个此优先级的类。如果没有此优先级的类，则查询该类的defmap，检视一个BEST_EFFORT 类。如果这是一个向上引用，或者没有定义BEST_EFFORT 类，则进入回退算法。如果发现一个有效的类，且非叶节点，则从该类重启循环；如果是一个叶节点，则选择该类并终止循环。如果根据classid提取的优先级或BEST_EFFORT优先级均未发现相关的类，则进入回退算法。

回退算法位于循环之外，遵循如下规则：

- 查询要回退的跳转发生在哪个类的defmap，如果该defmap包含一个与该类的优先级(与TOS字段相关)相同的类，则选择该类并终止循环。
- 查询BEST_EFFORT 优先级的类，如果找到，则选择该类并终止循环。
- 选择发生回退算法的类。 终止循环。

当任意一种算法终止时，报文被加入到所选择的类中。因此，报文可以不在叶节点入队列，而在层次结构的中间入队列。

#### 7.4.4. 链路共享算法

当入队列发送到网络驱动的报文时，CBQ决定哪个类可以发送报文。CBQ使用一个基于权重的轮询处理，每个类中的报文都有机会按照顺序发送出去。WRR会从具有最高优先级的类中处理报文，直到这些类中没有任何数据，然后处理低优先级的类。

由于每个类都不允许以长度发送数据，因此只能在每轮中取出可配置数量的数据。

如果一个类即将超出限制，且不受限制，它将尝试从没有隔离的兄弟节点中借用avgidle，从下向上重复这个过程。如果一个类无法借用到足够的avgidle来发送报文，则这个类会限流，并且不要求报文等待足够的时间来使avgidle增加到零以上。

#### 7.4.5. Root 参数

CBQ的root qdisc有如下参数:

`parent` *`root`* | *`major:minor`*:

强制参数，确定CBQ实例的位置，即接口的root类还是一个现有的类中。

`handle` *`major:`*

与其他类的qdisc类似，CBQ可以分配一个句柄。应该包含一个使用冒号分割的主号。可选参数。

`avpkt` *`bytes`*

为了计算，平均报文大小必须事先可知。默认至少为MTU的2/3。强制参数。

`bandwidth` *`rate`*

底层可用的带宽，用于确定空闲时间，CBQ必须知道带宽：A)目标带宽；B)底层物理接口或C)父qdisc。这是一个至关重要的参数，后面会详细介绍。强制参数。

`cell` *`size`*

cell大小确定了报文传输时间计算的粒度，必须是2的整数次幂，默认为8。

`mpu` *`bytes`*

0大小的报文可能也会占用传输的时间。这个值是报文传输时间计算的下限，小于该值的报文仍然被认为是这个大小。默认值为0。

`ewma` *`log`*

CBQ使用指数加权移动平均值（EWMA）来计算空闲状态，该方法可以以平滑测量的方式轻松应对短时间的突发。`log`值决定了平滑发送的数量。更低的值意味着更高的灵敏度。必须为0~31之间的数值，默认为5。

一个CBQ qdisc不会自行整流。它需要知道有关底层链路的某些参数。实际的整流是在类中完成的。

#### 7.4.6. 类参数

类有很多参数类配置其操作：

`parent` *`major:minor`*

层级结构中类的位置。如果直接附加到一个qdisc，而不是一个类，则可以忽略minor。强制参数。

`classid` *`major:minor`*

与qdisc类似，class是可命名的。majob号必须等于其属于的qdisc的major号。可选参数，但如果一个类包含子类时必选。

`weight` *`weightvalue`*

当入队列到底层时，将以循环方式使用类处理流量。通常具有更高权重的qdisc的类在一轮循环中可以发送更多的流量，因此该类也可以入队列更多的流量。由于一个类下的所有权重都被归一化了，因此只有比率才是重要的。默认使用配置的速率，除非该类的优先级是最大的，在这种情况下，它的优先级设置为1。

`allot` *`bytes`*

`allot` 指定了在每个轮询中一个qdisc可以入队列的字节数据。可以使用上面描述的权重对该参数进行加权。

`priority` *`priovalue`*

在轮询处理中，具有最低优先级字段值的类会优先处理报文。强制字段。

`rate` *`bitrate`*

这个类(包括子类)能够传输的最大聚合速率。bitrate 使用tc的方式指定速率(如1544kbit)。强制字段。

`bandwidth` *`bitrate`*

该参数与创建CBQ qdisc时指定的带宽不同。CBQ类的`bandwidth` 参数仅用于确定maxidle和offtime，而该参数仅在指定maxburst或minburst时才计算。因此，该参数仅在指定maxburst或minburst时使用。

`maxburst` *`packetcount`*

该报文数用于计算maxidle，这样在avgidle达到maxidle时，且在avgidle 将到0前，允许平均报文的突发。增加该值可以允许更大的突发。用户无法直接设置maxilde，只能通过该参数设置。

`minburst` *`packetcount`*

如前面所述，CBQ需要设置限制来防止发生超额。理想的解决方案是精确计算出空闲时间，然后传递1个报文。然而，Unix内核通常很难调度小于10ms的事件，因此最好能在一段比较长的时间内限流，然后一次性传递minburst的报文，然后使minburst 时间更长。等待的时间称为offtime。从长远角度看，更大的minburst 值会获得更精确的整流效果，但在毫秒时间尺度上会产生更大的突发。

`minidle` *`microseconds`*

*minidle*：如果avgidle小于0，则说明此时已经超额，需要等到avgidle能够发送一个报文为止。为了防止底层链路长时间中断导致的突发，如果avgidle值过低，则会重置为minidle。minidle 使用负值表示，因此10表示avgidle的上限为-10us。

`bounded` | `borrow`

指定了一个借用策略。即要么类会从兄弟节点借用带宽，要么将自己视为受限的。 二者是互斥的。

`isolated` | `sharing`

指一个共享策略。这个类要么对它的兄弟类使用共享策略，要么认为自己是孤立的。二者是互斥的。

*split major:minor and defmap bitmap[/bitmap]:* 如果查询附加到类的过滤器之后没有给出结论，CBQ可以根据报文的优先级进行分类。有16个优先级，从0到15。defmap指定了类期望接收的优先级，优先级使用位图进行表示。最低有效位对应的优先级为零。split参数告诉CBQ必须在哪个类上做出决定，这个类应该是要添加的类的(祖)父类。

如， 'tc class add ... classid 10:1 cbq .. split 10:0 defmap c0' 配置类10:0发送优先级为6和7到10:1的报文。

可替代的写法为： 'tc class add ... classid 10:2 cbq ... split 10:0 defmap 3f'，将发送所有的优先级为0, 1, 2, 3, 4 和5 的报文到10:1。

*estimator interval timeconstant:* CBQ可以测量每个类使用的带宽，以及哪些tc过滤器可以用来对报文进行分类。为了确定带宽，它使用了一个非常简单的评估器，每隔一微秒测量通过了多少流量。这也是一个EWMA，它的时间常数可以指定，同样是以微秒为单位。时间常数对应于测量的迟缓性，或者相反，对应平均值对短脉冲的灵敏度。更高的值表示更低的灵敏度。

### 7.5. WRR, 基于权重的轮询

该qdisc不包含在标志的内核中。

WRR qdisc使用加权轮询方案在其类之间分配带宽。它与CBQ qdisc类似，包含可以插入任意qdisc的类。具有足够需求的所有类都将获得与类相关的权重成比例的带宽。可以通过tc程序手动设置权重。但是对于传输大量数据的类，也可以使它们自动减少。

该qdisc有一个内置的分类器，可以将来自或发送到不同机器的数据包分配给不同的类(使用MAC或IP以及源或目的地址)。当Linux作为桥时会使用MAC地址。这些类会根据所看到的报文自动分配给机器。

该qdisc在许多无关的人共享Internet连接的站点上时非常有用。WRR发行版的关键部分是一组为此类站点设置相关行为的脚本。