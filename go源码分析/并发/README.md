1 为什么Golang需要调度器？
       Goroutine的引入是为了方便高并发程序的编写。 一个Goroutine在进行阻塞操作（比如系统调用）时，会把当前线程中的其他Goroutine移交到其他线程中继续执行， 从而避免了整个程序的阻塞。

       由于Golang引入了垃圾回收（gc），在执行gc时就要求Goroutine是停止的。通过自己实现调度器，就可以方便的实现该功能。 通过多个Goroutine来实现并发程序，既有异步IO的优势，又具有多线程、多进程编写程序的便利性。

       引入Goroutine，也意味着引入了极大的复杂性。一个Goroutine既要包含要执行的代码， 又要包含用于执行该代码的栈和PC、SP指针。

2 调度器解决了什么问题？
2.1 栈管理
       既然每个Goroutine都有自己的栈，那么在创建Goroutine时，就要同时创建对应的栈。 Goroutine在执行时，栈空间会不停增长。 栈通常是连续增长的，由于每个进程中的各个线程共享虚拟内存空间，当有多个线程时，就需要为每个线程分配不同起始地址的栈。 这就需要在分配栈之前先预估每个线程栈的大小。如果线程数量非常多，就很容易栈溢出。

       为了解决这个问题，就有了Split Stacks技术： 创建栈时，只分配一块比较小的内存，如果进行某次函数调用导致栈空间不足时，就会在其他地方分配一块新的栈空间。 新的空间不需要和老的栈空间连续。函数调用的参数会拷贝到新的栈空间中，接下来的函数执行都在新栈空间中进行。

       Golang的栈管理方式与此类似，但是为了更高的效率，使用了连续栈 （Golang连续栈） 实现方式也是先分配一块固定大小的栈，在栈空间不足时，分配一块更大的栈，并把旧的栈全部拷贝到新栈中。 这样避免了Split Stacks方法可能导致的频繁内存分配和释放。

2.2 抢占式调度
       Goroutine的执行是可以被抢占的。如果一个Goroutine一直占用CPU，长时间没有被调度过， 就会被runtime抢占掉，把CPU时间交给其他Goroutine。

3 调度器的设计
       Golang调度器引入了三个结构来对调度的过程建模：

G 代表一个Goroutine；
M 代表一个操作系统的线程；
P 代表一个CPU处理器，通常P的数量等于CPU核数（GOMAXPROCS）。
       三者都在runtime2.go中定义，他们之间的关系如下：

G需要绑定在M上才能运行；
M需要绑定P才能运行；
程序中的多个M并不会同时都处于执行状态，最多只有GOMAXPROCS个M在执行。
       早期版本的Golang是没有P的，调度是由G与M完成。 这样的问题在于每当创建、终止Goroutine或者需要调度时，需要一个全局的锁来保护调度的相关对象(sched)。 全局锁严重影响Goroutine的并发性能。 (Scalable Go Scheduler)

       通过引入P，实现了一种叫做work-stealing的调度算法：

每个P维护一个G队列；
当一个G被创建出来，或者变为可执行状态时，就把他放到P的可执行队列中；
当一个G执行结束时，P会从队列中把该G取出；如果此时P的队列为空，即没有其他G可以执行， 就随机选择另外一个P，从其可执行的G队列中偷取一半。
       该算法避免了在Goroutine调度时使用全局锁。

4 调度器的实现
4.1 schedule()与findrunnable()函数
       Goroutine调度是在P中进行，每当runtime需要进行调度时，会调用schedule()函数， 该函数在proc1.go文件中定义。

       schedule()函数首先调用runqget()从当前P的队列中取一个可以执行的G。 如果队列为空，继续调用findrunnable()函数。findrunnable()函数会按照以下顺序来取得G：

调用runqget()从当前P的队列中取G（和schedule()中的调用相同）；
调用globrunqget()从全局队列中取可执行的G；
调用netpoll()取异步调用结束的G，该次调用为非阻塞调用，直接返回；
调用runqsteal()从其他P的队列中“偷”。
       如果以上四步都没能获取成功，就继续执行一些低优先级的工作：

如果处于垃圾回收标记阶段，就进行垃圾回收的标记工作；
再次调用globrunqget()从全局队列中取可执行的G；
再次调用netpoll()取异步调用结束的G，该次调用为阻塞调用。
       如果还没有获得G，就停止当前M的执行，返回findrunnable()函数开头重新执行。 如果findrunnable()正常返回一个G，shedule()函数会调用execute()函数执行该G。 execute()函数会调用gogo()函数（在汇编源文件asm_XXX.s中定义，XXX代表系统架构），gogo() 函数会从G.sched结构中恢复出G上次被调度器暂停时的寄存器现场（SP、PC等），然后继续执行。