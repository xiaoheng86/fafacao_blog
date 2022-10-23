---
title: "Redisson Retry Lock"
subtitle: "从简单分布式锁说起"
date: 2022-10-23T10:53:27+08:00
draft: false
tags: ["分布式", "Redis"]
---

&nbsp;&nbsp; &nbsp;单机场景下的互斥锁，只需要同一个JVM进程即可共享锁监视器。但是在多机分布式环境下，想保 证线程安全性，需要一个系统全局的互斥锁，而Redis正好有这个特征，可以被所有进程访问到，且性能很高，适合高并发场景。

&nbsp;&nbsp;  互斥锁的本质其实就是一个bool值，代表加锁/释放。Redis中提供了一个命令`setnx`，setnx本身便具有互斥性：当key不存在时则set，当key已存在时则返回false。我们就可以利用`setnx`这个命令的特性，实现一个简单的用于保证线程安全的Redis分布式锁，set成功代表成功加锁，往下执行业务逻辑即可，set失败则直接返回，保证不出现不一致的问题。并且在分布式场景下，一定要考虑的问题就是服务宕机的容错，所以我们还需要实现**超时释放**，这个直接利用Redis自带的过期时间即可实现一个简单锁。

<!--more-->

# Redisson的Retry分布式锁

 &nbsp;&nbsp; 最近在学习Redis，发现它在分布式并发安全上用的很多，本篇文章总结一下我对基于Redis的分布式锁的学习。

 &nbsp;&nbsp; 如果您在阅读过程中发现任何问题或不足，欢迎您联系我一起讨论。

</br>

#### 重点问题

1. **锁误删**

​		&nbsp;&nbsp;当业务超时，锁会被Redis的expire自动释放，此时另一线程2获取到锁，超时线程1完成时会把线程2的锁给误释放。

<img src="https://s1.ax1x.com/2022/10/23/xg43h6.png" style="zoom:150%;" />

​	&nbsp;&nbsp;	  解决方法很简单，在加锁时，记录下自己的标识，一个线程只能释放由它自己加的锁。这个线程标识必须是全局唯一的，于是我们可以用UUID（Universal 		Unique Identifier）来记录。

2. **原子操作**

​	&nbsp;&nbsp;&nbsp;	令人感叹，上面的问题用UUID解决了，But，这样我们在释放锁的时候必须要两步操作，先判断UUID再释放，不能保证原子性。



<img src="https://s1.ax1x.com/2022/10/23/xg4G9K.png" style="zoom:150%;" />

​		  &nbsp;&nbsp; 解决方法就是Redis支持的Lua脚本，我们这里的原子性是Redis命令级别的而不是操作系统级别的，可以认为Redis的一条语句就是原子操作。Redis是C语   	 言编写，可以完美集成Lua脚本，所以我们写脚本，把多步操作放到一条Redis命令即可保证原子性。Lua脚本不在本文总结范围内。

</br>

3. **不可重入**

​	  &nbsp;&nbsp;当我们的加上锁以后调用另外的函数，如果被调用函数也需要获取同一锁的话，就会发生死锁。这个解决方法也很简单，锁里加上一条引用计数就完事儿，这	个时候我们锁的数据结构就得用hash。

#### 实现代码在文末

</br>

## Redisson Retry锁

&nbsp;&nbsp;以上简单的分布式还存在很多问题待解决、优化。

&nbsp;&nbsp;目前可以简单的认为Redisson是一个Redis高级客户端，但其实它远不止这些。

&nbsp;&nbsp;更多信息请移步：[Redisson: Redis Java client with features of In-Memory Data Grid](https://redisson.org/)。

[![xg5QKg.png](https://s1.ax1x.com/2022/10/23/xg5QKg.png)](https://imgse.com/i/xg5QKg)



&nbsp;&nbsp;下面我简单总结Reddison RetryLock实现的关键点。

### 简单锁的缺陷

1. 加锁失败不能重试，丢失请求。
2. 超时时间过于死板，没办法动态伸缩，也会丢失请求。

</br>

### Lock与UnLock流程图

[![xgbHij.png](https://s1.ax1x.com/2022/10/23/xgbHij.png)](https://imgse.com/i/xgbHij)



### 流程图关键点

部分注释我写在了流程图里

流程关键点：当`Publish`时，通知`Subscriber`开始抢锁，抢到锁后，判断是否超时，若超时则释放锁并返回false，若未超时，则递归调用TryLock, 并将TTL置为空，空即代表已拿到锁，然后进入leaseTime的判断（-1决定了是否开启watchDog）选择锁的持有时间是固定一段时间还是一直持有，直到调用者主动Unlock。

</br>

1. TTL是TryLock函数接受的参数，限制了获取锁的尝试超时时间。

2. leaseTime为锁的持有超时时间，非-1则与之前简单锁一样是调用者传入的自定义固定时长，当leaseTime为-1（默认值）时，会开启watchDog，它会每到leaseTime三分之一长的时候续租锁，也就是保证业务能执行完再释放锁，而不是暴力截断业务。不要忘了我们设置**超时的目的主要是为了增强宕机容错性**，文章最早就提到过分布式系统设计时随时都要考虑到高可用，所以正常的业务执行超时我们是能接受的，不需要直接暴力中止，所以我们相当于搞了个watchDog来提供业务执行心跳，至于业务中的网络连接等导致的超时过长，并不是我们锁设计者需要考虑的问题，应该把这个问题交给业务层实现人员自检。

3. Reddison通过Sub/Pub模式结合阻塞队列解决了轮询忙等的问题，订阅消息/发布消息+阻塞队列也是消息队列的重要设计之一，这一点启发了我对忙等轮询的改进，后面的项目中可以实践一下捏。

</br>

# Redisson MultiLock

&nbsp;&nbsp;  学了Redis我发现在分布式系统中考虑问题，还需要考虑到集群切换的同步问题。

&nbsp;&nbsp;  如下图，如果Redis主宕机了，lock1还没来得及同步到从，此时failover主从切换，就会丢失锁。

[![xg5u28.png](https://s1.ax1x.com/2022/10/23/xg5u28.png)](https://imgse.com/i/xg5u28)

## 解决方法

​	Redisson提供了MultiLock，在构造函数中你可以指明多个Redis服务进程，每次加锁，**MultiLock会确保每个指定的Redis进程中都加好锁以后再返回成功**。

</br></br>

### 简单分布式锁代码实现

[![xg5kbd.png](https://s1.ax1x.com/2022/10/23/xg5kbd.png)](https://imgse.com/i/xg5kbd)

[![xg5Z5t.png](https://s1.ax1x.com/2022/10/23/xg5Z5t.png)](https://imgse.com/i/xg5Z5t)



## 参考资料

[1] [黑马程序员Redis入门到实战教程]([Redis课程介绍导学_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1cr4y1671t?p=1))

[2] [Redisson Documentation](https://github.com/redisson/redisson/wiki/Table-of-Content)

[3] [利用Redisson实现分布式锁及其底层原理解析 - 黄文博 - 博客园 (cnblogs.com)](https://www.cnblogs.com/nov5026/p/11684632.html)
