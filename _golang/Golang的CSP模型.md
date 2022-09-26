---
title: Golang 的CSP模型
author: 罗松涛
category: golang
layout: post
---

go语言最大两个亮点，一个goroutine，一个是chan。二者合体的典型应用CSP，基本就是大家认可的并行开发神器，简化了并行程序的开发难度。

## 1. 什么是CSP

CSP（Commuicating Sequential Process）直译通信顺序进程，是一种并发编程模型。

> 用于描述两个独立的并发实体通过共享的通讯channel进行通信的并发模型。
{: .block-tip }

相对于Actor模型，CSP中channel是第一类对象，它并不关注发送消息的实体，而关注与发送消息时使用的channel

> #### tips
> 
> Actor是一个概念模型，用于处理并发计算，一个Actor指的是一个最基本的计算单元。他能接收一个消息并且给予其执行计算。
> 
> Actor一大重要特征在于actors之间相互隔离, 它们并不互相共享内存，即一个actor能维持一个私有的状态，并且这个状态不可能被另一个actor改变
{: .block-tip }

严格来说，CSP是一门**形式语言**，用于描述并发系统中的互动模式，也因此成为一众面向并发的变成语言的理论源头，并衍生出了Occam/Limbo/Golang...

而具体到编程语言，如golang，其实只用到了CSP的很小一部分，即理论中的Process/Channel(对应到语言中的goroutine/channel)：这两个并发原语之间没有从属关系，Process可以订阅任意个Channel，Channel也并不关心是哪个Process在利用它进行通；Process围绕Channel进行读写，形成一套有序阻塞和可预测的并发模型。

## 2. Golang的CSP

与主流语言通过共享内存来进行并发控制方式不同，Go语言采用了CSP模式。

Golang就是借用CSP模型的一些概念为之实现并发进行理论支持，但从实际角度出发，golang并么有完全实现CSP模型的所有理论，仅仅是借用了Process和Channel两个概念。

> Process在golang上表现就是goroutine，是实际的并发执行的实体，每个实体之间通过Channel通讯来实现数据共享
>
> Golang的协程Goroutine：是一种轻量线程，它不是操作系统的线程，而是一个将操作系统线程分段使用，通过调度器实现协作式调度。
> 它是一种绿色线程、微线程，它与Coroutine协程也有区别，能够在发现阻塞后启动新的微线程。
>
> 通道Channel：类似Unix的Pipe，用于协程之间通讯和同步。协程之间虽然解耦，但是它们和Channel有着耦合。
{: .block-tip }


## 3. Channel

Goroutine和Channel是Golang并发编程的两大基石。

Goroutine用于执行并发任务，Channel用于Goroutine之间的通信和同步。

Channel在Goroutine间架起了一条管道，在管道里传输数据，实现Goroutine的通信；由于它是线程安全的，所以用起来非常方便；Channel还提供"FIFO"的特性；它还能影响Goroutine的阻塞和唤醒。

Golang的并发哲学：
> Do not communicate by sharing memory; instead, share memory by communicating
{: .block-warning }

### 3.1 Channel 实现CSP

Channel是Golang中一个非常重要的类型，是Golang的第一对象。通过Channel，Golang实现了通过通信来实现内存共享。

Channel是在多个Goroutine之间传递数据和同步的重要手段。

使用原子函数、读写锁可以保证资源的共享访问安全，但使用Channel更优雅。

Channel字面意义是“通道”，类似与Unix中的管道。Golang声明Channel的语法：
```golang
chan T // 声明一个双向通道
chan<- T // 声明一个只能用于发送的通道
<-chan T // 声明一个只能用于接收的通道
```

单向通道的声明，用`<-`来表示


### 缓冲和非缓冲Channel

因为 Channel 是一个引用类型，所以在它被初始化之前，它的值是 `nil`，Channel 使用 `make`函数进行初始化。

初始化时，通过传递一个 `int` 值，代表 Channel 缓冲区的大小（容量），构造出来的是一个缓冲型的 Channel；不传或传 `0` 的，构造的就是一个非缓冲型的 channel。

- 非缓冲型 channel 无法缓冲元素，对它的操作一定顺序是 “发送 -> 接收 -> 发送 -> 接收 -> ......”。如果想连续向一个非缓冲 `chan` 发送 2 个元素，并且没有接收的话，第一次一定会被阻塞
对不带缓冲的 channel 进行的操作实际上可以看作 “同步模式”
- 带缓冲的则称为 “异步模式”

同步模式下，发送方和接收方要同步就绪，只有在两者都 ready 的情况下，数据才能在两者间传输（后面会看到，实际上就是内存拷贝）。否则，任意一方先行进行发送或接收操作，都会被挂起，等待另一方的出现才能被唤醒。

异步模式下，在缓冲槽可用的情况下（有剩余容量），发送和接收操作都可以顺利进行。否则，操作的一方（如写入）同样会被挂起，直到出现相反操作（如接收）才会被唤醒。

小结：**同步模式下，必须要使发送方和接收方配对，操作才会成功，否则会被阻塞；异步模式下，缓冲槽要有剩余容量，操作才会成功，否则也会被阻塞。**


简单来说：CSP模型由并发执行的实体（线程、进程或协程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用就是通道，Channel。

CSP模型的关键是关注Channel，而不关注发送消息的实体。Golang实现了CSP部分理论，Goroutine对应CSP中并发执行的实体，Channel也就对应着CSP中的Channel。


## 4. Goroutine

Goroutine是实际并发的实体，它底层是使用协程(coroutine)实现并发，coroutine是一种运行在用户态的用户线程，类似于greenthread，golang底层选择使用coroutine的出发点是因为，它具有以下特点：
> - 用户空间：避免了内核态和用户态的切换带来的成本
> - 可以由语言和框架层进行调度
> - 更小的栈空间允许创建大量的实例
{: .block-tip }

Goroutine是在Golang层面提供了调度器，并且对网络IO库进行了封装，屏蔽了复杂的细节，对外提供统一的语法关键字支持，简化了并发程序编写的成本

