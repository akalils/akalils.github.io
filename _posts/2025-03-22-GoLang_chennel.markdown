---
layout:     post
title:      "GoLang chennel详解"
subtitle:   " GoLang "
date:       2025-03-22 12:00:00
author:     "Lils"
header-img: "img/post-bg-2015.jpg"
tags:
    - GoLang
---

## 参考资料
本文核心思路参考了 **小徐先生** 的博客文章，在此表示感谢：  
- **作者**：小徐先生  
- **原文标题**：[Golang Channel 实现原理](https://mp.weixin.qq.com/s/QgNndPgN1kqxWh-ijSofkw)

## 前置知识补充

1. 构造channel
	- `ch := make(chan int)` 这样构造出来的chennel是**无缓冲**类型
	- `ch := make(chan int, 10` **有缓冲**，如果缓冲区已满我再往里面去进行写操作的话会陷入**阻塞**
2. 读
	- `val := <- ch`
	- `<- ch`
	- `val, ok := <- ch` 读成功的话ok的值为true，如果这个ok的值是false代表的是我读到的是一个已关闭的channel
3. 写
	- `ch <- data`
4. 关闭
	- `close(ch)`
	- 如果关闭了channel之后再尝试往这个channel去读数据，这个时候不管这个channel当中有没有数据，这个读操作都不会被阻塞，倘若有数据，那我就会把chennel当中剩余的这部分数据读取到。
	- 倘若这个chennel本身是空的我再尝试去读的话，此时会从里面读到我当前这个类型的一个零值。
	- 如果我往一个关闭的channel去写数据的话会发生一个panic
5. Channel 的三种形态
	- 无缓冲Channel（同步Channel）：必须读写配对，否则阻塞。  
	- 有缓冲Channel（异步Channel）：缓冲区满/空时阻塞。  
	- 已关闭Channel：读操作不阻塞，写操作panic。  
6. 多路复用select
	- select支持同时去监听多个分支，哪一个分支有事件我就打破阻塞接着往下执行。
```go
select {
case <-ch1:
	// do some logic
case <-ch2:
	// do some logic
case ch3 <- data:
	// do some logic
default:
	// 放行
}
```

## 核心数据结构
![](/img/FlowChart/1_1.jpeg)
### hchan
channel 数据结构
```go
type hchan struct {
    qcount   uint           // 当前 channel 中存在多少个元素
    dataqsiz uint           // 当前 channel 能存放的元素容量
    buf      unsafe.Pointer // channel 中用于存放元素的环形缓冲区，可以复用数组的地址空间，同时也能保证这部分内存地址是连续的
    elemsize uint16 // channel 元素类型的大小
    closed   uint32 // 标识 channel 是否关闭
    elemtype *_type // channel 元素类型
    sendx    uint   // 写入元素的位置
    recvx    uint   // 读取元素的位置
    recvq    waitq  // 因接收而陷入阻塞的协程队列
    sendq    waitq  // 因发送而陷入阻塞的协程队列
    
    lock mutex
}
```
### waitq

阻塞的协程队列，是一个**双向链表**
```go
type waitq struct {
	// 指向首部和尾部节点的指针
	first *sudog
	last *sudog
}
```

### sudog
用于包装协程的节点
```go
type sudog struct {
    g *g // goroutine，协程

    next *sudog // 队列中的下一个节点
    prev *sudog // 队列中的前一个节点
    elem unsafe.Pointer // data element (may point to stack)

    isSelect bool   // 标识当前协程是否处在 select 多路复用的流程中

    c        *hchan // 回指向所属的channel
}
```
## 构造器函数
![[img/FlowChart/1_6.jpeg]]
```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // mem是估算出来的缓冲区的大小
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
        panic(plainError("makechan: size out of range"))
    }

    var c *hchan
    // 这三种类型对应的是上图中的三种类型
    switch {
    case mem == 0: // 可能是无缓冲区类型，也可能是缓冲区大小为0的类型，如go中struct缓冲区大小就是0
        c = (*hchan)(mallocgc(hchanSize, nil, true)) // 分配当前channel除了缓冲区外需要的一个大小空间
        // Race detector uses this location for synchronization.
        c.buf = c.raceaddr()
    case elem.ptrdata == 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true)) // 分配一个channel所需要的内存大小hchanSize再加上缓冲区的大小mem
        c.buf = add(unsafe.Pointer(c), hchanSize) // 偏移一定的大小
    default: // 指针类型
        // Elements contain pointers.
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true) // 分两次分配，不是连续的内存空间，有一个空间地址上的隔离
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size) // 缓冲区的总大小
    
    lockInit(&c.lock, lockRankHchan)

    return
}
```
## 写流程
### 两类异常情况处理

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

// 尝试往一个没有初始化过的一个channel当中去写数据时，写入操作会引发死锁
// 没有初始化指的是var ch chan int，没有通过make来分配内存空间
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
	    // 因为这是一个nil channel，所以永远都不会有人往里面去读数据所以我当前挂起的这个协程永远不会被唤醒，出现死锁
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2) // 被动阻塞
        throw("unreachable")
    }

    lock(&c.lock)

	// 如果我们往一个已经被关闭的chennel当中去写数据时会引发 panic。
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    
    // ...
```

### 写时存在阻塞读协程
![[img/FlowChart/1_2.jpeg]]
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...

    lock(&c.lock)

    // ...

	// 从阻塞读协程队列中取出一个 goroutine 的封装对象 sudog
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        // 在 send 方法中，会基于 memmove 方法，当无缓冲区时会直接将元素拷贝交给 sudog 对应的读 goroutine，有缓冲区但已满时写协程阻塞，读协程从缓冲区读数据，在 send 方法中会完成解锁动作
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    
    // ...
}
```

### 写时无阻塞读协程但环形缓冲区仍有空间
![[img/FlowChart/1_3.jpeg]]
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...
    lock(&c.lock)
    // ...
    // qcount: channel中已有数据大小
    // 缓冲区仍有空间
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx) // 拿到对应缓冲区的凹槽
        typedmemmove(c.elemtype, qp, ep) // 把当前尝试去写的数据给拷贝到对应的凹槽当中去
        c.sendx++ // 写的index++
        if c.sendx == c.dataqsiz { // this is a 环形数组
            c.sendx = 0
        }
        c.qcount++ // 已有元素个数++
        unlock(&c.lock)
        return true
    }

    // ...
}
```
### 写时无阻塞读协程且环形缓冲区无空间
![[img/FlowChart/1_8.jpeg]]
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...
    lock(&c.lock)

    // ...
    gp := getg() //GMP当中的G
    mysg := acquireSudog() // 构造封装当前 goroutine 的 sudog 对象
    mysg.elem = ep
    mysg.g = gp
    mysg.c = c
    gp.waiting = mysg
    c.sendq.enqueue(mysg) // 把 sudog 添加到当前 channel 的阻塞写协程队列中
    
    atomic.Store8(&gp.parkingOnChan, 1)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2) // park 当前协程，代码会阻塞在这一行

	// 倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被读协程取走）
    gp.waiting = nil
    closed := !mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

### 写流程整体串联
![[img/FlowChart/1_10.jpeg]]
## 读流程
### 4.1读空 channel
如果尝试读的channel是一个空channel
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c == nil {
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2) // park 挂起，引起死锁
        throw("unreachable")
    }
    // ...
}
```
### 4.2channel 已关闭且内部无元素

如果读的是一个已关闭的channel且这个channel已经没有元素了
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  
    lock(&c.lock)

    if c.closed != 0 { // channel已经被关
        if c.qcount == 0 { // 里面的剩余元素是0
            unlock(&c.lock)
            if ep != nil {
                typedmemclr(c.elemtype, ep) // 返回0值
            }
            return true, false
        }
        // The channel has been closed, but the channel's buffer have data.
    } 
    // ...
```
### 4.3读时有阻塞的写协程（缓冲区已满）
![[img/FlowChart/1_11.jpeg]]
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   
    lock(&c.lock)

	// 从阻塞写协程队列中获取到一个写协程
	// 倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程；
	// 倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写协程；
    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
     }
     // ...
}
```

### 4.4读时无阻塞写协程且缓冲区有元素
![[img/FlowChart/1_3.jpeg]]
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
    lock(&c.lock)
    // ...
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp) //  获取到 recvx 对应位置的元素
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++ // 移动读的指针
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    // ...
```

### 4.5读时无阻塞写协程且缓冲区无元素
![[img/FlowChart/1_5.jpeg]]
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   // ...
   lock(&c.lock)
   // ...
   // 构造封装当前 读goroutine 的 sudog 对象
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    gp.waiting = mysg
    mysg.g = gp
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 把读 goroutine 添加到当前 channel 的阻塞读协程队列中
    atomic.Store8(&gp.parkingOnChan, 1)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// 倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被写入）
    gp.waiting = nil
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, success
}
```

### 读流程整体串联

我理解的时候自己动手画了一个流程图，下面也附上原作者的流程图。
![[img/FlowChart/IMG_4C1C997D1152-1.jpeg]]
![[img/FlowChart/1_7.jpeg]]
## 阻塞与非阻塞模式
### 非阻塞模式逻辑区别

非阻塞模式下，读/写 channel 方法通过一个 bool 型的响应参数，用以标识是否读取/写入成功.

- 所有需要使得当前 goroutine 被挂起的操作，在非阻塞模式下都会返回 false去取代挂起操作；
- 所有使得当前 goroutine 会进入死锁的操作，在非阻塞模式下都会返回 false去取代死锁操作；
- 所有能立即完成读取/写入操作的条件下，非阻塞模式下会返回 true.
### 何时进入非阻塞模式

默认情况下，读/写 channel 都是阻塞模式，只有在 select 语句组成的多路复用分支中，与 channel 的交互会变成非阻塞模式。

## 两种读 channel 的协议

之前我们说过如果去读一个已关闭的channel同时这个chennel中已经没有元素了此事会返回0值，**那我们怎么判断这个0值是因为channel已经被关闭了还是因为chennel中本身就存放的是0值**。
```go
if val, ok := <- ch; ok{
	// open
} else {
	// close
}
```

通过上述代码这种方式就能判断0值的来源。
## 关闭
![[img/FlowChart/1_9.jpeg]]
```go
func closechan(c *hchan) {
	// 不能关闭一个未初始化的chennel否则会panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    // 重复关闭一个已关闭的chennel也会panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    c.closed = 1

	// 阻塞读goroutine队列和阻塞写goroutine队列两者当中一定有一个不为空
	// 换句话说下面两个for循环只有一个有效
	// 将阻塞读协程队列中的协程节点统一添加到 glist
    var glist gList
    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }

	// 将阻塞写协程队列中的协程节点统一添加到 glist
    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }
    unlock(&c.lock)

	// 唤醒 glist 当中的所有协程
    // Ready all Gs now that we've dropped the channel lock.
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
    }
}
```
在关闭channel之前一定要保证此时没有在阻塞写操作的goroutine存在，否则会抛出异常

## 鸣谢与参考
本文在整理过程中参考了以下资料
特此致谢： 
1. **小徐先生** - [Golang Channel 实现原理](https://mp.weixin.qq.com/s/QgNndPgN1kqxWh-ijSofkw)
2. Go 官方文档
