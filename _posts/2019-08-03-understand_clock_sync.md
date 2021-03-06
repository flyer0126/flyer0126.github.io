---
layout: post
title: "理解'时钟同步'"
author: "flyer0126"
---

# 理解“时钟同步”



## 一、问题产生
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/01.png)

时间是一个绝对量，而实体计算机的时间是相对量

1. 物理天地本身导致的时间不一致，地球自转、闰年、闰秒
2. 现实的不能绝对一致性，A机器时间同步至B机器，网络传输时间是不确定性的，AB存在绝对不一致性

如上图，computer A在2144  Tick点执行分布式任务 create output.o，注意2144是A的绝对计算量、而此时的集群computer B也许出于2143 Tick点，即使B也运气恰到好处的出于2144 Tick，A任务同步至B消耗的Tick是不确定的。获取是2144也有可能是2143，倘若如上图，ouput.c created在B小于created 2144时间点，时间戳make问题就出现了。

## 二、逻辑时钟
1. Berkeley算法：分布式服务器定时轮询所有服务器，各台服务器依据同步结果，决定本台服务器的Tick快慢同步是时间
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/02.png)


三台服务器明显不一致，3:00仲机器在轮询其他机器时候，分别有2:50,3:25，那么他们是分别加10分钟和减20分钟来同步时间吗？显然不行，各台机器本身是存在事务日志的，如果本身时间进行人为修改，事务时间戳就会出现混乱。此时应该是各台服务器调慢或调快各自时间Tick已达到三台时间Tick一致。

2. Lamport算法时钟同步
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/03.png)

在红箭头时间点，从理想主义角度而言，始终应该是一致的，但现实是明显不一致的，两条偏移线要向Perfect看齐
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/04.png)

T2的时间传播到T1，T1时间Tick慢，T4在应答时间戳上要进行同步了，解决上述问题
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/05.png)

保证逻辑的一致性就需要计算出偏移量，偏移量用途
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/06.png)

P1发起分布式任务到P2大于任务时间戳6，继续传播到P3小于时间戳40，应答回传56，时间戳小了，如果继续回传P1 54，更小了，于是需要计算偏移量机型时间一致性

如：
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/07.png)


偏移量调整时间过程示意
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/08.png)

Perfact解决方案
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/09.png)

不使用同步算法，导致的结果是灾难性的，update1和update2的任务在1和2上的执行顺序显然不能混乱，并且需要连续性。就好比银行是先转账还是先计息，也许，或者可能这是业务需要考虑的问题，ok，那么1和2能够一致性执行update1和update2就是分布式需要解决的技术问题。

解决

```
1、在执行update1时候进行广播，”你们都给听好了，我要执行update1操作”  
2、收到广播“好的，你执行，我也开始执行了”
```

于是update1的执行保障了一致性。当2故障或者2没有收到update1或update2，广播应答结果就是“NO，你别执行哈，执行要出乱子的”，ok，connection refused or time out。


3. 因果一致性
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/10.png)

P0执行任务，写入时间戳100，广播P1、P2，P2执行相同的分布式任务，P2这是发起了分布式任务写入时间戳110,100和110广播到P2时候，P2执行100时候，发现时间戳有问题，告诉P2不能执行110和000, 因为100尚未执行，那么按照时间戳排序执行。
锁的因果一致性
![image](https://flyer0126.github.io/assets/imgs/20190803clock_sync/11.png)

1执行3并取得3 locker，2需要执行3，不好意思，3已经被locker，进入队列，当3被释放才有2的执行权。


<br>

_The end_