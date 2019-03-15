---
title: 《Intro Go》第二部分：核心实现
date: 2019-03-13 21:20:37
tags: golang 
---


# 第二部分 核心实现
《Intro Go》系列文字将从第二部分开始，分三个章节分别介绍Go语言最为重要的三个关键字：

- `channel`
- `select`
- `go`

这三个章节将通过走读部分关键代码的形式，分别介绍它们的核心实现思想。

注：
1. 本文所使用的Go源码版本来源于github，版本号为1.10，你可以从[这里](https://github.com/golang/go/tree/release-branch.go1.10)获取到；
2. 由于篇幅所限，本节所引用的代码可能被部分删减、重组过。你可以在代码块顶部注释处找到本段代码在源码中所处的位置，以便查阅。

## 第一章节 channel

### 1.0 chan内存模型
`chan`在[runtime](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go)中的数据结构被定义为：

```go
/* Ref 2-1-1. Data structure of channel 
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L31 
 */
 
type hchan struct {  
	qcount   uint // total data in the queue  
	dataqsiz uint // size of the circular queue  
  
	buf      unsafe.Pointer // points to an array of dataqsiz elements  
	sendx    uint // send index  
	recvx    uint // receive index  

	elemsize uint16  
	elemtype *_type // element type 
  
	sendq    waitq  // list of send waiters  
	recvq    waitq  // list of recv waiters 
  
	closed   uint32 
  
	lock mutex  
}
```

在`hchan`中，有两个与其关系密切的结构体，`waitq`和`sudog`：

```go
/* Ref 2-1-2. Data structure of waitq
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L52 
 */
 
type waitq struct {  
	 first *sudog  
	 last  *sudog  
}
```

显而易见，`waitq`参与维护双向链表，链表的结点为`sudog`：

```go
/* Ref 2-1-3. Data structure of sudog
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/runtime2.go#L285
 */
 
type sudog struct {  
	// The following fields are protected by the hchan.lock of the  
	// channel this sudog is blocking on. shrinkstack depends on // this for sudogs involved in channel ops.  
	g *g  
  
	// isSelect indicates g is participating in a select, so  
	// g.selectDone must be CAS'd to win the wake-up race.  isSelect bool  
	next     *sudog  
	prev     *sudog  
	elem     unsafe.Pointer // data element (may point to stack)  
  
	// SKIP...
	// SKIP...
	// SKIP...
}
```

`sudog`结构体的中的`g`，可以简单理解为Go语言世界中的`进程`，即Go语言术语`协程（goroutine）`。`sudog`的存在是为了更好的管理`g`，因而包含了用于维护双链表的两根指针，以及一个指向具体数据的`unsafe.Pointer`指针。

可以看到，`hchan`本质上维护了一个**环形队列**，用于**存放数据**；以及两个**双链表**，维护等待读取/存放数据的**协程**。

###  1.1 创建channel
channel的创建遵从`make`关键字的使用语法，通过对参数`size`赋值的不同，以区分buffered及unbuffered的channel：

```go
// Example 2-1-1. The way to create channel

// syntax:
func make(t Type, size ...IntegerType) Type

// create channel:
c := make(chan int) 			// unbuffered
cp := make(chan int, 10)		// buffered
cpp := make(chan int, 1)		// diff w/ unbuffered?
```

根据是否带有数据缓存区，`chan`总体可以分为两大类，三小类：

- **unbuffered**：不缓存区的channel
- **buffered**：带缓存区的channel
	- w/ pointers： 缓存区数据类型是指针的channel
	- w/o pointers： 缓存区数据类型不是指针的channel


在为其分配内存时，三种不同情况需要区别对待。但为了行文方便，除非特殊说明，本文大部分情况下只区分有缓存和无缓存两种情况。

> ***题外话：***
> *`chan`的大小取决于描述符`hchan`的大小，以及`chan`中所存放的数据大小：*
> 
> ```c
> sizeof(chan) = sizeof(chan_descriptor) + sizeof(chan_elem) 
> ```
> 
> *数据大小 = 数据量 × 单个数据大小，很容易理解。但`hchan`的大小呢，尤其当为`hchan`分配内存时需要考虑对齐因素时？ [answer here](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L27)*

#### 1.1.1 不带缓存区的channel
有两种等价的方式创建无缓存的channel：

```go
// Example 2-1-2. Two ways to create unbuffered channel
c := make(chan int) 			// unbuffered

// equal with
c := make(chan int, 0)
```

注意第二种等价的编码方法，`size`被赋值为了0，此时可以被省略。

！@#！@#这里放内存模型图

#### 1.1.2 带缓存区的channel 
对于数据类型为非指针和指针的两种带缓存的`channel`，创建方式也一致：

```go
// Example 2-1-3. The way to create buffered channel w/o pointer
c := make(chan int, 10)			// no pointers 		
```

！@#！@#这里放内存模型图

```go
// Example 2-1-4. The way to create buffered channel w/ pointers
c := make(chan *int, 10) 		// contain pointers
```

！@#！@#这里放内存模型图

带指针/不带指针两种内存模型不一致的原因与垃圾回收机制相关。~~缘分到了的话，以后会把GC这块补上。~~

### 1.2 读写channel
#### 1.2.0 导言
##### 1.2.0.1 `<-` 操作符
Go语言使用`<-`操作符操作`chan`：

```go
// Example 2-1-5. Play with <-
c := make(chan int, 10)

// send
c <- 10

// recv
x := <- c
``` 

##### 1.2.0.2 阻塞模式与非阻塞模式
对于`chan`的读写操作，Go程序员可以选择**阻塞模式**或者**非阻塞模式**。

```go
// Example 2-1-6. Example for blocking send operation
c := make(chan int, 1)

// send
c <- 10			// send one element into channel, which make it full
c <- 20			// cannot send anymore, will be blocked here

// recv
x := <- c		// CAN NOT BE HERE
``` 

*Example 2-1-6*试图通过**阻塞写**的方式发送2个元素（10,20）。由于`c`的容量只有1，因此程序会阻塞在`c <- 20`这一行代码处。

```go
// Example 2-1-7. Example for blocking recv operation
c := make(chan int, 10)

// send
// c <- 10 NONONO

// recv
x := <- c		// nothing to recv, will be blocked here 
``` 

*Example 2-1-7*创建了一个容量为10的`chan`，因为没有“生产者”向channel中写数据，因此程序会挂起在**阻塞读**操作：`x := <- c`。

与之相对的，程序员可以通过使用**select**关键字，实现对**chan**的非阻塞读写操作：

```go
// Example 2-1-8. Example for blocking send operation
c := make(chan int, 10)

select {
	case c <- 1:
		// enough space, send succeed
		fmt.Println("job assigned",)
	default:
		// full, cannot send anything
		fmt.Println("oh shit, too many work todo...")
}	
``` 

```go
// Example 2-1-9. Example for non-blocking recv operation
c := make(chan int, 10)

select {
	case x := <- c:
		// do something with fetched element
		fmt.Println("got something:", x)
	default:
		// do something when there is nothing in channel
		fmt.Println("nothing to do, boring...")
}	
``` 

#### 1.2.1 发送操作实现
本小节将结合源码，介绍向channel中发送数据的实现。
就发送操作而言，分类讨论的主要依据，是否有已经**等待数据的接收者**（因读操作阻塞挂起在当前channel的协程），以及**缓存区**是否满。

##### 1.2.2.1  边界条件检测
由1.0节知晓，在操作`chan`时，实质上是在维护包括环形队列、双链表在内的一系列数据结构。因此，在切实操作这些数据结构之前，需要先进行一些正确性检测，以确定发送操作是可行的。
Go语言规定，**对于关闭了的`chan`，向其发送数据会导致程序panic**；在非阻塞模式下，如果一个没有被关闭的`chan`因为某种原因无法接收数据，则发送操作会被跳过。
在完成了这一系列边界条件检测之后，Go才会对具体的`chan`、数据、协程进行相关操作。

##### 1.2.2.2 有接收者（-->缓存区为空）
当试图向`chan`里写数据时，若此时有接收者，则在代码层面体现为`hchan`中`recvq`不为空。

```go
/* Ref 2-1-4. Receive queue in hchan
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L40
 */
 
type hchan struct {  
	// SKIP...
	recvq    waitq  // list of recv waiters 
	// SKIP...
}
```

在这种情况下，隐含说明了缓存区的数据都已经被消费完，因此没有必要再将数据拷贝到缓冲区，而是:
1. **直接**写入待接收者所指定的数据接收地址;
2.  唤醒这个被阻塞住的协程。

```go
/* Ref 2-1-5-1. Send data to receiver directly. Part I
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L188
 */
 
// step 1:
// dequeue a receiver from queue
if sg := c.recvq.dequeue(); sg != nil {  
    // Found a waiting receiver. We pass the value we want to send  
    // directly to the receiver, bypassing the channel buffer (if any).  
	send(c, sg, ep, func() { unlock(&c.lock) }, 3)  
	return true  
}
```

```go
/* Ref 2-1-5-2. Send data to receiver directly. Part II
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L283
 * line 283 ~ 2923
 */
 
// step 2:
// copy mem direct into someplace specificed by receiver
// NOTE: usually into receiver's stack
sendDirect(c.elemtype, sg, ep)

// SKIP...
// SKIP...
// SKIP...

// step 3:
// wake up receiver
goready(gp, skip+1)
```

！@#！@#！#！#图示

##### 1.2.2.3 无接受者&&缓存区有空余空间
缓存区有空间隐含说明没有等待数据的接受者，此时将数据拷贝到缓存区（环形队列）即可。

```go
/* Ref 2-1-6. Send data to buffer.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L195
 * line 195 ~ 210
 */
 
if c.qcount < c.dataqsiz {  
	// Space is available in the channel buffer. Enqueue the element to send.  
	// SKIP...
	typedmemmove(c.elemtype, qp, ep)  
	c.sendx++  
	if c.sendx == c.dataqsiz {  
		c.sendx = 0  
	}  
	c.qcount++  
    // SKIP...
}
```

！@#！@#！#！#图示

##### 1.2.2.4 无接受者&&缓存区满
在非阻塞模式下，缓存区满会导致发送数据的协程**阻塞并挂起**。从`hchan`的角度来说，即是维护`sendq`双链表：

```go
/* Ref 2-1-7. Send data but will be blocked.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L233
 * line 218 ~ 234
 */
 
// SKIP...
// SKIP...

// step 1:
// enqueue current sudog
c.sendq.enqueue(mysg)

// step 2:
// put the current goroutine into waiting state
// wait for someone to wake me up by calling goready(gp)
goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

// step 3:
// wait for awoken
// SKIP...
```

#！@#！@# 图示

#### 1.2.2 接收操作实现
本小节将结合源码，介绍从channel中接收数据的实现。
就接收操作而言，分类讨论的主要依据，channel是否已经被关闭，以及有无挂起的发送者。

##### 1.2.2.1 边界条件检查
对于读`chan`操作，在实际读取`hchan`中`buf`之前同样有一些检测：在非阻塞条件下，对于

1. **无缓存+无发送者**；
2. **有缓存+无数据**。

这两类情况，由于都无法读取到数据，则直接跳过。

##### 1.2.2.2 已关闭&&无数据
前文已经提到，`chan`在关闭之后，并不会拒绝读操作，而是会拒绝写操作。在有数据的情况下，数据依旧会被正常读取；而在数据被读完之后，会通过清空目标地址空间内存，并返回一个标记的形式给予读取者回馈。

```go
/* Ref 2-1-8. Read a closed channel.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L462
 */
 
// step 1:
// clear target memory
typedmemclr(c.elemtype, ep)

// step 2:
// return a signal
return true, false
```

##### 1.2.2.3 有挂起的发送者
```go
/* Ref 2-1-9. Send queue in hchan.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L41
 */
 
type hchan struct {  
	// ...
	sendq    waitq  // list of send waiters 
	// ...
}
```

有挂起的发送者，即`hchan`中`sendq`双链表不为空，此时有两种情况：`chan`有缓存区，以及`chan`无缓存区。

对于无缓冲区，直接从sender队列队首中读取数据：

```go
/* Ref 2-1-10. Receive data from sender.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L555
 */
 
// read directly from sender
recvDirect(c.elemtype, sg, ep)
```

对于有缓冲区，说明buffer已经被占满了。此时读取buffer中数据，并将在发送队列队首的协程等待发送的数据写入buffer。无论有无缓存区，两种情况最终都会唤醒对应的协程。

```go
/* Ref 2-1-11. Receive data from buffer, and wake up goroutine.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L562
 * line 562 ~ 588
 */
 
// step 1:
// read from the head slot of the buffer
qp := chanbuf(c, c.recvx)  
if ep != nil {  
   typedmemmove(c.elemtype, ep, qp)  
}

// step 2:
// copy data from sender to queue  
typedmemmove(c.elemtype, qp, sg.elem)  
c.recvx++  
if c.recvx == c.dataqsiz {  
	c.recvx = 0  
}  
c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz

// SKIP...
// SKIP...

// step 3:
// awoke that waiting sender
goready(gp, skip+1)
``` 

！@#！@#！@#！#图示

##### 1.2.3.4 没有挂起的发送者
对于没有挂起的发送者这个情况，需要分两类讨论：

##### 1.2.3.4.1 缓冲区有数据
理所当然，读buffer即可

```go
/* Ref 2-1-12. Receive data from buffer, no goroutine to wake up.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L476
 * line 476 ~ 493
 */
 
if c.qcount > 0 {  
	// Receive directly from queue  
	qp := chanbuf(c, c.recvx)  
	
	// SKIP..
	 
	// step 1.
	// copy from data buf
	if ep != nil {  
		typedmemmove(c.elemtype, ep, qp)  
	}  

	// step 2.
	// maintain index
	typedmemclr(c.elemtype, qp)  
	c.recvx++  
	if c.recvx == c.dataqsiz {  
		c.recvx = 0  
	}  
	c.qcount--  
	unlock(&c.lock)  
	return true, true  
}
```

##### 1.2.3.4.2 缓冲区无数据/无缓冲区
无数据，导致对应协程阻塞：

```go
/* Ref 2-1-13. Failed to receive data from buffer, blocked.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L502
 * line 502 ~ 518
 */
 
// no sender available: block on this channel. 
// step 1.
// get current goroutine 
gp := getg()  
mysg := acquireSudog()  

// SKIP...

// step 2.
// block
c.recvq.enqueue(mysg)  
goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)
```

### 1.3 关闭channel
使用close关键字用于关闭一个chan：

```go
// Example 2-1-10. Example to close channel
close(c)
```

在实现层面，`close()`释放并唤醒了所有挂起在`sendq`和`recvq`的协程。

```go
/* Ref 2-1-13. Close a channel.
 * https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go#L325
 * line 325 ~ 395
 */
 
func closechan(c *hchan) {  
	// SKIP...
	c.closed = 1  
  
	// SKIP...
  
    // release all readers  
	for {  
		sg := c.recvq.dequeue()  
		// SKIP...
	}  
  
    // release all writers (they will panic)  
	for {  
		sg := c.sendq.dequeue()  
		// SKIP...
	}  
    
   // Ready all Gs now that we've dropped the channel lock.  
   for glist != nil {  
		gp := glist  
		glist = glist.schedlink.ptr()  
		gp.schedlink = 0  
		goready(gp, 3)  
   }  
}
```

（正文完）

----
**BONUS：**

对channel进行`close`操作，是Go常用的发送一次性信号的手段，常见于处理关闭流程。如果你对这个技巧还不熟悉，请阅读：
> [Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)。

在一次查验docker生产环境（版本号*v17.03.0-ce* ）问题的时候，发现了一个既没有在Github issue区被讨论，也没有被记录在CHANGELOG，但确于17.06版修复掉的bug。其原因来自于编码疏忽，导致在某些情况下没有`close`channel，使得docker（准确的说是libcontainerd）一直阻塞在读取这个`chan`的语句。

```go
/* Ref 2-1-14. The signal channel used in docker.
 * https://github.com/moby/moby/blob/v17.03.0-ce/libcontainerd/remote_unix.go#L60
 */
 
type remote struct {  
	// SKIP... 
	daemonWaitCh         chan struct{}  
    // SKIP
}
```

如*Ref 2-1-14*，`libcontainerd`的`remote`数据结构中，使用`daemonWaitCh`来收取标识*containerd*正常启动的信号。

```go
/* Ref 2-1-15. Close channel to sendout signal.
 * https://github.com/moby/moby/blob/v17.03.0-ce/libcontainerd/remote_unix.go#L415
 * line 415 ～ 434
 */
 
func (r *remote) runContainerdDaemon() error {
	// SKIP...
	// SKIP...
	
	if err := cmd.Start(); err != nil {  
		return err  
	}  
	logrus.Infof("libcontainerd: new containerd process, pid: %d", cmd.Process.Pid)  
	if err := setOOMScore(cmd.Process.Pid, r.oomScore); err != nil {  
	    utils.KillProcess(cmd.Process.Pid)  
	    return err  
	}  
	if _, err := f.WriteString(fmt.Sprintf("%d", cmd.Process.Pid)); err != nil {  
	    utils.KillProcess(cmd.Process.Pid)  
	    return err  
	}  
	  
	r.daemonWaitCh = make(chan struct{})  
	go func() {  
	    cmd.Wait()  
		
		// NOOOOOOOOOOOOOOOOOOOOOOOOOTE HERE!!!!!
	    close(r.daemonWaitCh)  
	}() // Reap our child when needed  
	r.daemonPid = cmd.Process.Pid  
	return nil
}
```
s
```go
/* Ref 2-1-16. Wait on that channel.
 * https://github.com/moby/moby/blob/v17.03.0-ce/libcontainerd/remote_unix.go#L170
 */
 
func (r *remote) handleConnectionChange() {
	// SKIP...
   <-r.daemonWaitCh  
    // SKIP.. 
}
```

注意*Ref 2-1-15*中的第21行，在这个`close`操作之前，`setOOMScore()`和`WriteString()`的异常处理都可能导致`close`操作无法被执行到。因而可能导致*Ref 2-1-16*中，试图获取这个信号的协程永远阻塞在那。表现在生产环境的现象即是，在触发了OOM异常（即`setOOMScore()`失败）之后，docker客户端命令卡死。

在*docker-ce-17.06*版本中，这个bug被修复，方法为将`close`所在的代码位置移到`setOOMScore()`和`WriteString()`之前，保证`close`语句的执行。维护者同时写一下如下注释：

```go
/* Ref 2-1-17. Bug fixed of unclosed channel.
 * https://github.com/docker/docker-ce/blob/17.06/components/engine/libcontainerd/remote_unix.go#L472
 */
 
// unless strictly necessary, do not add anything in between here
// as the reaper goroutine below needs to kick in as soon as possible
// and any "return" from code paths added here will defeat the reaper
// process.
r.daemonWaitCh = make(chan struct{})
go func() {
	cmd.Wait()
	close(r.daemonWaitCh)
}() // Reap our child when needed
```

（本章节完）
