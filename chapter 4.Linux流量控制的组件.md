## Linux流量控制的组件

流量控制元素与Linux组件之间的相关性：

| traditional element                                          | Linux component                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 入队列                                                       | 修订：从用户或网络接收报文                                   |
| [整流](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-shaping) | [class](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-class) 提供了整流的能力 |
| [调度](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-scheduling) | 一个 [`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc) 就是一个调度器。调度器可以是一个简单的FIFO，也可以变得很复杂，包括classes和其他qdiscs，如HTB。 |
| [分类](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-classifying) | [`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter) 对象通过一个[`classifier`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-classifier) 对象执行分类。严格上讲，除filter之外不会使用分类器。 |
| [策略](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-policing) | [`policer`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)仅作为[`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter)的一部分存在。 |
| [丢弃](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-dropping) | [`drop`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-drop) 流量要求使用一个带 [`policer`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-police)的 [`filter`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-filter) ，动作为"drop" |
| [标记](https://tldp.org/en/Traffic-Control-HOWTO/ar01s03.html#e-marking) | `dsmark` [`qdisc`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-qdisc) 用于标记报文。 |
| 入队列;                                                      | 位于驱动队列上的qdisc和[网络接口控制器(NIC之间)](https://tldp.org/en/Traffic-Control-HOWTO/ar01s02.html#o-nic)。驱动队列给了上层(IP栈和流量控制子系统)将输入异步入队列的位置(后续由硬件对数据进行操作)。队列的大小由[Byte Queue Limits (BQL)](https://tldp.org/en/Traffic-Control-HOWTO/ar01s04.html#c-bql)动态设置。 |

#### 4.1 qdisc