## golang channel 要点
本文的代码基于 golang 1.8.3

## channel 的日常生活

(img)

为了把channel解释的更易懂，我们虚拟一个家庭，家里有爸爸、妈妈和三个孩子，他们喜欢吃苹果，也喜欢睡觉。爸爸削苹果给孩子们吃：
1. 同步模型：爸爸削完，双手被占着，什么也干不了，只能停下来什么等孩子把苹果拿走，他一停下来，就是去睡觉了。
2. 异步模型：爸爸削完苹果，放盘子里，然后去干其他工作了，孩子从盘子里拿着吃。但是如果盘子被放满，他也只能等盘子里有苹果被取走，不然手被占着，什么也做不了。
如果爸爸削的慢，孩子们如果拿不到苹果，会在盘子边排队等待。孩子们一停下来，也是去睡觉。

基于channel 的goruntinue 同步异步模型内部是统一的，同步的情况，可以认为盘子大小为0。channel /盘子维护了一个等待和发送队列，发送方/爸爸如果发现没有buffer（接收方/孩子们发现buffer中数据为空），则被挂在 c.sendq （c.recvq）上，等待另一方的唤醒。

在创建channel 时候，注意以下差别的
> c := make(chan int)  没有 buffer，用作同步。
> c := make(chan int, 0)  同上
> c : = make(chan int, 1) 生成有一个buffer 的channel，它有一个空间，所以在放一个时候不会被阻塞。

c<- chansend 内部关键代码如下（chan.go:128）：
'' c.sendq.enqueue(mysg)
'' goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)
<- c chanrecv 内部关键代码如下（chan.go:410）：
'' c.recvq.enqueue(mysg)
'' goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)
其中 goparkunlock 会将当前的 goroutine 置为wating 状态，该goroutine 可以被goready(gp)唤醒，以下是goparkunlock里设置为wating 的代码片段（proc.go:2262）：
'' casgstatus(gp, _Grunning, _Gwaiting) // 将 gp 置为新状态。
'' setMNoWB(&_g_.m.curg.m, nil) // 分离 goruntinue 与 M, 置空_g_.m.curg.m
'' setGNoWB(&_g_.m.curg, nil)  // 分离 goruntinue 与 P，置空_g_.m.curg.m
'' schedule() // 重新调度
schedule() 会触发重新调度，重新调度不再执行状态为stop的goruntinue。 调度原理参见这里【5】，讲解的很形象。

## 被对方唤醒
上一小节讲到，goruntinue 在会挂在channel上等待。当爸爸削好苹果后，叫醒起孩子们去盘子里拿，或者直接从他手里取；当孩子们来取苹果时，会叫醒爸爸，让他继续削苹果，填满盘子。
精确的说是： A 生产快，盘子被占用，则 A 等待， B 有唤醒 A的责任。B 消费快，盘子被清空，则 B 等待， A 有唤醒 B的责任。挂在 channel 等待队列 c.sendq 或者 c.recvq 里的 goruntinue 会被被唤醒。

以下是被唤醒的关键代码：
chan.go:410
'' func chanrecv(t *chantype, c *hchan, ep unsafe.Pointer, block bool) 
'' {
'' 	   if sg := c.sendq.dequeue(); sg != nil {
'' 	     // Found a waiting sender. If buffer is size 0, receive value
'' 	     // directly from sender. Otherwise, receive from head of queue
'' 	     // and add sender's value to the tail of the queue (both map to
'' 	     // the same buffer slot because the queue is full).
'' 	     goready(gp, 4)
''          }
'' }
'' 

goready 将设置状态为\_Grunnable，随后将其置入运行队列。
''  casgstatus(gp, _Gwaiting, _Grunnable)
''  runqput(_g_.m.p.ptr(), gp, next)


## 单向限定有什么用？
声明 channel 变量时候可以使用<-来限定channel的方向。还拿盘子为例，限定盘子只能从中取，不能放入；或者盘子只能放入，不能取；如果没有指定方向，那么盘子可取可放。默认情况 channel是双向的，既可以接收数据，也可以从其取数据。单向限定是在编译时候确定的，如果违反使用规则，在编译时候编译器就报错。

ch5 := <-chan int(ch4)   // 单向读
ch6 := chan<- int(ch4)   // 单向写

单向限定的应用场景是对参数进行限制，就像C里的const函数参数，可以避免程序员出错。
''  func worker(id int, jobs chan <-int, results chan <- int)  {
'' 	for j:= range jobs {
'' 		fmt.Println("workder", id, "started job", j)
'' 	}
'' }

## channel 关闭后，goruntinue 还会继续执行么？
channel关闭的方法：close(jobs)。close channel  后，会将channel 上挂载的goruntinue队列全部释放，同时，将各个等待goruntinue置为ready状态，释放及参见如下：chan.go:320
''  	// release all readers
'' 	for {
'' 		sg := c.recvq.dequeue()
'' 		if sg == nil {
'' 			break
'' 		}
'' 		各种引用置空释放
'' 		gp.schedlink.set(glist)
'' 		glist = gp
'' 	}
'' 
'' 	// release all writers (they will panic)
'' 	for {
'' 		sg := c.sendq.dequeue()
'' 		if sg == nil {
'' 			break
'' 		}
'' 		各种引用置空释放
'' 		gp.schedlink.set(glist)
'' 		glist = gp
'' 	}
'' 	// Ready all Gs now that we've dropped the channel lock.
'' 	for glist != nil {
'' 		gp := glist
'' 		glist = glist.schedlink.ptr()
'' 		gp.schedlink = 0
'' 		goready(gp, 3)
'' 	}

以下示例展示了在close(c) 执行后，等待中的 goruntine 将继续执行，打印出 " continue after close"
'' func main() {
'' 	c := make(chan int)
'' 	go func() {
'' 		<-c
'' 		fmt.Println("continue after close")
'' 	} ()
'' 	go func() {
'' 		fmt.Println("before close")
'' 		close(c)
'' 	} ()
'' 	time.Sleep(time.Second * 5)
'' }
'' 

## for range channel 为何会一直等待？

这个我还没搞明白，它是如何等待的，等搞明白了，再更新。
for range 需要对应 close，否则会一直等待。

## 崩溃了，all goroutines are asleep!
“fatal error: all goroutines are asleep - deadlock!” 使用channel 时，如果代码有问题，会经常遇到这个运行时错误，这是 runtime 在进行死锁检测。

goroutine 在等待时，期望从管道中获得一个数据，而这个数据必须是其他goroutine 放入管道的。但是其他goroutine 都已经执行完了(all goroutines are asleep)，那么就永远不会有数据放入管道。 所以，goroutine线在等一个永远不会来的数据，那整个程序就永远等下去了。 这显然是没有结果的，所以这个程序就说“算了吧，不坚持了，我自己自杀掉，报一个错给代码作者，我被deadlock了”

检测时机有好几处，其中最重要的是在sysmon，sysmon是一个地位非常高的后台任务，整个函数体是一个死循环的形式，目前主要处理两个事件：对于网络的epoll以及抢占式调度的检测，retake 会间接调用checkdead：
proc.go:3767
'' func sysmon() {
'' 	delay := uint32(0)
'' 	for {
'' 		if idle == 0 { 
'' 			delay = 20
'' 		} else if idle > 50 { 
'' 			delay *= 2
'' 		}
'' 		if delay > 10*1000 { // up to 10ms
'' 			delay = 10 * 1000
'' 		}
'' 		usleep(delay)  // 检测间隔 从 20 us - 10 ms
''                  retake(now)  // 检测在这里发生。
''           }
'' }
checkdead 基于查看sched 维护的是否有运行中的 M，如果运行中的 M 个数为0，则报出该运行时错误。其关键代码如下：
proc.go:3683
'' func checkdead() {
'' 	// -1 for sysmon
'' 	run := sched.mcount - sched.nmidle - sched.nmidlelocked - 1
'' 	if run > 0 {
'' 		return
'' 	}
'' 
'' 	throw("all goroutines are asleep - deadlock!")
'' }
此runtime error 会在 main goroutine里发生，其他线程看不到是因为主线程结束后，整个程序就退出了，其他所有线程也会被结束掉，就不会再检测，这也是在goruntinue里以下代码并不会打印输出的原因。
''  func main() {
'' 	go func() {
'' 		fmt.Println("output")
'' 	} ()
'' }
 
## channel 工作常用场景
摘自【3】effective go 等，有必要继续添加更多常用场景。
同步：
'' c := make(chan int)  // Allocate a channel.
'' // Start the sort in a goroutine; when it completes, signal on the channel.
'' go func() {
''     list.Sort()
''     c <- 1  // Send a signal; value does not matter.
'' }()
'' doSomethingForAWhile()
'' <-c   // Wait for sort to finish; discard sent value.
''  
用作信号量：
'' var sem = make(chan int, MaxOutstanding)
'' 
'' func handle(r *Request) {
''     process(r)  // May take a long time.
'' }
'' 
'' func Serve(queue chan *Request) {
''     for req := range queue {
''         sem <- 1
''         go func(req *Request) {
''             process(req)
''             <-sem
''         }(req)
''     }
'' }
'' 
工作池【4】：
'' func worker(id int, jobs <-chan int, results chan<- int) {
''     for j := range jobs {
''         fmt.Println("worker", id, "started  job", j)
''         time.Sleep(time.Second)
''         fmt.Println("worker", id, "finished job", j)
''         results <- j * 2
''     }
'' }
'' 
'' func main() {
'' 
''     // In order to use our pool of workers we need to send
''     // them work and collect their results. We make 2
''     // channels for this.
''     jobs := make(chan int, 100)
''     results := make(chan int, 100)
'' 
''     // This starts up 3 workers, initially blocked
''     // because there are no jobs yet.
''     for w := 1; w <= 3; w++ {
''         go worker(w, jobs, results)
''     }
'' 
''     // Here we send 5 `jobs` and then `close` that
''     // channel to indicate that's all the work we have.
''     for j := 1; j <= 5; j++ {
''         jobs <- j
''     }
''     close(jobs)
'' 
''     // Finally we collect all the results of the work.
''     for a := 1; a <= 5; a++ {
''         <-results
''     }
'' }

## 被唤醒后，执行顺序有迹可寻么：
之前小节介绍，goruntinue 通过channel通信时，是要挂在channel 的queue上。那么问题来了，queue的特性是明确的先进先出，goruntinue 在channel 资源被满足后，执行的顺序是否有迹可循？

(img)

答案是否定的，虽然队列的进出是固定的，参见【5】对“地鼠偷砖”算法的讲解。但是每个goruntinue是一块砖，每块砖被搬运的时机由“地鼠偷砖”算法决定。以下摘自文章里对“地鼠偷砖”算法的讲解：
1. runqget, 地鼠(M)试图从自己的小车(P)取出一块砖(G)，当然结果可能失败，也就是这个地鼠的小车已经空了，没有砖了。
2. findrunnable, 如果地鼠自己的小车中没有砖，那也不能闲着不干活是吧，所以地鼠就会试图跑去工场仓库取一块砖来处理；工场仓库也可能没砖啊，出现这种情况的时候，这个地鼠也没有偷懒停下干活，而是悄悄跑出去，随机盯上一个小伙伴(地鼠)，然后从它的车里试图偷一半砖到自己车里。如果多次尝试偷砖都失败了，那说明实在没有砖可搬了，这个时候地鼠就会把小车还回停车场，然后睡觉休息了。如果地鼠睡觉了，下面的过程当然都停止了，地鼠睡觉也就是线程sleep了。
3. wakep, 到这个过程的时候，可怜的地鼠发现自己小车里有好多砖啊，自己根本处理不过来；再回头一看停车场居然有闲置的小车，立马跑到宿舍一看，你妹，居然还有小伙伴在睡觉，直接给屁股一脚，“你妹，居然还在睡觉，老子都快累死了，赶紧起来干活，分担点工作。”，小伙伴醒了，拿上自己的小车，乖乖干活去了。有时候，可怜的地鼠跑到宿舍却发现没有在睡觉的小伙伴，于是会很失望，最后只好向工场老板说——”停车场还有闲置的车啊，我快干不动了，赶紧从别的工场借个地鼠来帮忙吧。”，最后工场老板就搞来一个新的地鼠干活了。
4. execute，地鼠拿着砖放入火种欢快的烧练起来。

然而，下面的小程序，仍然可以是依据先进先出的原理是有迹可循的，两个goruntinue执行的工作非常简单，它的输出大概率是12，而不是10：
'' func main() {
'' 	ch := make(chan int, 2)
'' 	sum := 4
'' 	go func() {
'' 		<-ch
'' 		sum += 2
'' 	}()
'' 	go func() {
'' 		<-ch
'' 		sum *= 2
'' 
'' 	}()
'' 
'' 	time.Sleep(time.Second)
'' 	ch <- 1
'' 	ch <- 1
'' 	time.Sleep(time.Second)
'' 	fmt.Println(sum)
'' }

附录
【1】[https://www.zhihu.com/question/20862617]
【2】[https://garbagecollected.org/2017/02/22/go-range-loop-internals/]
【3】[https://golang.org/doc/effective\_go.html#channels]
【4】[https://gobyexample.com/worker-pools]
【5】[http://studygolang.com/wr?u=http%3a%2f%2fskoo.me%2fgo%2f2013%2f11%2f29%2fgolang-schedule]
