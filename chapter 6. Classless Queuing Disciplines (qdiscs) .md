## Classless Queuing Disciplines (qdiscs)

本文涉及的队列规则(Qdisc)都可以作为接口上的主qdisc，或作为一个[classful qdiscs](https://tldp.org/en/Traffic-Control-HOWTO/ar01s07.html)的叶子类。这些是Linux下使用的基本调度器。默认的调度器为[`pfifo_fast`](https://tldp.org/en/Traffic-Control-HOWTO/ar01s06.html#qs-pfifo_fast)。

### 6.1 FIFO，先进先出(pfifo和bfifo)

