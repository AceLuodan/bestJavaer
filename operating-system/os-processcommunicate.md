# 操作系统进程间通信

## 进程间通信

进程是需要频繁的和其他进程进行交流的。例如，在一个 shell 管道中，第一个进程的输出必须传递给第二个进程，这样沿着管道进行下去。因此，进程之间如果需要通信的话，必须要使用一种良好的数据结构以至于不能被中断。下面我们会一起讨论有关 `进程间通信(Inter Process Communication, IPC)` 的问题。

关于进程间的通信，这里有三个问题

* 上面提到了第一个问题，那就是一个进程如何传递消息给其他进程。
* 第二个问题是如何确保两个或多个线程之间不会相互干扰。例如，两个航空公司都试图为不同的顾客抢购飞机上的最后一个座位。
* 第三个问题是数据的先后顺序的问题，如果进程 A 产生数据并且进程 B 打印数据。则进程 B 打印数据之前需要先等 A 产生数据后才能够进行打印。

需要注意的是，这三个问题中的后面两个问题同样也适用于线程

第一个问题在线程间比较好解决，因为它们共享一个地址空间，它们具有相同的运行时环境，可以想象你在用高级语言编写多线程代码的过程中，线程通信问题是不是比较容易解决？

另外两个问题也同样适用于线程，同样的问题可用同样的方法来解决。我们后面会慢慢讨论这三个问题，你现在脑子中大致有个印象即可。

### 竞态条件

在一些操作系统中，协作的进程可能共享一些彼此都能读写的公共资源。公共资源可能在内存中也可能在一个共享文件。为了讲清楚进程间是如何通信的，这里我们举一个例子：一个后台打印程序。当一个进程需要打印某个文件时，它会将文件名放在一个特殊的`后台目录(spooler directory)`中。另一个进程 `打印后台进程(printer daemon)` 会定期的检查是否需要文件被打印，如果有的话，就打印并将该文件名从目录下删除。

假设我们的后台目录有非常多的 `槽位(slot)`，编号依次为 0，1，2，...，每个槽位存放一个文件名。同时假设有两个共享变量：`out`，指向下一个需要打印的文件；`in`，指向目录中下个空闲的槽位。可以把这两个文件保存在一个所有进程都能访问的文件中，该文件的长度为两个字。在某一时刻，0 至 3 号槽位空，4 号至 6 号槽位被占用。在同一时刻，进程 A 和 进程 B 都决定将一个文件排队打印，情况如下

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150240410-825157989.png)

`墨菲法则(Murphy)` 中说过，任何可能出错的地方终将出错，这句话生效时，可能发生如下情况。

进程 A 读到 in 的值为 7，将 7 存在一个局部变量 `next_free_slot` 中。此时发生一次时钟中断，CPU 认为进程 A 已经运行了足够长的时间，决定切换到进程 B 。进程 B 也读取 in 的值，发现是 7，然后进程 B 将 7 写入到自己的局部变量 `next_free_slot` 中，在这一时刻两个进程都认为下一个可用槽位是 7 。

进程 B 现在继续运行，它会将打印文件名写入到 slot 7 中，然后把 in 的指针更改为 8 ，然后进程 B 离开去做其他的事情

现在进程 A 开始恢复运行，由于进程 A 通过检查 `next_free_slot`也发现 slot 7 的槽位是空的，于是将打印文件名存入 slot 7 中，然后把 in 的值更新为 8 ，由于 slot 7 这个槽位中已经有进程 B 写入的值，所以进程 A 的打印文件名会把进程 B 的文件覆盖，由于打印机内部是无法发现是哪个进程更新的，它的功能比较局限，所以这时候进程 B 永远无法打印输出，类似这种情况，**即两个或多个线程同时对一共享数据进行修改，从而影响程序运行的正确性时，这种就被称为竞态条件(race condition)**。调试竞态条件是一种非常困难的工作，因为绝大多数情况下程序运行良好，但在极少数的情况下会发生一些无法解释的奇怪现象。

### 临界区

不仅共享资源会造成竞态条件，事实上共享文件、共享内存也会造成竞态条件、那么该如何避免呢？或许一句话可以概括说明：**禁止一个或多个进程在同一时刻对共享资源（包括共享内存、共享文件等）进行读写**。换句话说，我们需要一种 `互斥(mutual exclusion)` 条件，这也就是说，如果一个进程在某种方式下使用共享变量和文件的话，除该进程之外的其他进程就禁止做这种事（访问统一资源）。上面问题的纠结点在于，在进程 A 对共享变量的使用未结束之前进程 B 就使用它。在任何操作系统中，为了实现互斥操作而选用适当的原语是一个主要的设计问题，接下来我们会着重探讨一下。

避免竞争问题的条件可以用一种抽象的方式去描述。大部分时间，进程都会忙于内部计算和其他不会导致竞争条件的计算。然而，有时候进程会访问共享内存或文件，或者做一些能够导致竞态条件的操作。我们把对共享内存进行访问的程序片段称作 `临界区域(critical region)` 或 `临界区(critical section)`。如果我们能够正确的操作，使两个不同进程不可能同时处于临界区，就能避免竞争条件，这也是从操作系统设计角度来进行的。

尽管上面这种设计避免了竞争条件，但是不能确保并发线程同时访问共享数据的正确性和高效性。一个好的解决方案，应该包含下面四种条件

1. 任何时候两个进程不能同时处于临界区
2. 不应对 CPU 的速度和数量做任何假设
3. 位于临界区外的进程不得阻塞其他进程
4. 不能使任何进程无限等待进入临界区

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150250342-914295354.png)

从抽象的角度来看，我们通常希望进程的行为如上图所示，在 t1 时刻，进程 A 进入临界区，在 t2 的时刻，进程 B 尝试进入临界区，因为此时进程 A 正在处于临界区中，所以进程 B 会阻塞直到 t3 时刻进程 A 离开临界区，此时进程 B 能够允许进入临界区。最后，在 t4 时刻，进程 B 离开临界区，系统恢复到没有进程的原始状态。

### 忙等互斥

下面我们会继续探讨实现互斥的各种设计，在这些方案中，当一个进程正忙于更新其关键区域的共享内存时，没有其他进程会进入其关键区域，也不会造成影响。

#### 屏蔽中断

在单处理器系统上，最简单的解决方案是让每个进程在进入临界区后立即`屏蔽所有中断`，并在离开临界区之前重新启用它们。屏蔽中断后，时钟中断也会被屏蔽。CPU 只有发生时钟中断或其他中断时才会进行进程切换。这样，在屏蔽中断后 CPU 不会切换到其他进程。所以，一旦某个进程屏蔽中断之后，它就可以检查和修改共享内存，而不用担心其他进程介入访问共享数据。

这个方案可行吗？进程进入临界区域是由谁决定的呢？不是用户进程吗？当进程进入临界区域后，用户进程关闭中断，如果经过一段较长时间后进程没有离开，那么中断不就一直启用不了，结果会如何？可能会造成整个系统的终止。而且如果是多处理器的话，屏蔽中断仅仅对执行 `disable` 指令的 CPU 有效。其他 CPU 仍将继续运行，并可以访问共享内存。

另一方面，对内核来说，当它在执行更新变量或列表的几条指令期间将中断屏蔽是很方便的。例如，如果多个进程处理就绪列表中的时候发生中断，则可能会发生竞态条件的出现。所以，屏蔽中断对于操作系统本身来说是一项很有用的技术，但是对于用户线程来说，屏蔽中断却不是一项通用的互斥机制。

#### 锁变量

作为第二种尝试，可以寻找一种软件层面解决方案。考虑有单个共享的（锁）变量，初始为值为 0 。当一个线程想要进入关键区域时，它首先会查看锁的值是否为 0 ，如果锁的值是 0 ，进程会把它设置为 1 并让进程进入关键区域。如果锁的状态是 1，进程会等待直到锁变量的值变为 0 。因此，锁变量的值是 0 则意味着没有线程进入关键区域。如果是 1 则意味着有进程在关键区域内。我们对上图修改后，如下所示

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150301165-1150145648.png)

这种设计方式是否正确呢？是否存在纰漏呢？假设一个进程读出锁变量的值并发现它为 0 ，而恰好在它将其设置为 1 之前，另一个进程调度运行，读出锁的变量为0 ，并将锁的变量设置为 1 。然后第一个线程运行，把锁变量的值再次设置为 1，此时，临界区域就会有两个进程在同时运行。

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150309420-2145300109.png)

也许有的读者可以这么认为，在进入前检查一次，在要离开的关键区域再检查一次不就解决了吗？实际上这种情况也是于事无补，因为在第二次检查期间其他线程仍有可能修改锁变量的值，换句话说，这种 `set-before-check` 不是一种 `原子性` 操作，所以同样还会发生竞争条件。

#### 严格轮询法

第三种互斥的方式先抛出来一段代码，这里的程序是用 C 语言编写，之所以采用 C 是因为操作系统普遍是用 C 来编写的（偶尔会用 C++），而基本不会使用 Java 、Modula3 或 Pascal 这样的语言，Java 中的 native 关键字底层也是 C 或 C++ 编写的源码。对于编写操作系统而言，需要使用 C 语言这种强大、高效、可预知和有特性的语言，而对于 Java ，它是不可预知的，因为它在关键时刻会用完存储器，而在不合适的时候会调用垃圾回收机制回收内存。在 C 语言中，这种情况不会发生，C 语言中不会主动调用垃圾回收回收内存。有关 C 、C++ 、Java 和其他四种语言的比较可以参考 **链接**

**进程 0 的代码**

```c
while(TRUE){
  while(turn != 0){
    /* 进入关键区域 */
    critical_region();
    turn = 1;
    /* 离开关键区域 */
    noncritical_region();
  }
}
```

**进程 1 的代码**

```c
while(TRUE){
  while(turn != 1){
    critical_region();
    turn = 0;
    noncritical_region();
  }
}
```

在上面代码中，变量 `turn`，初始值为 0 ，用于记录轮到那个进程进入临界区，并检查或更新共享内存。开始时，进程 0 检查 turn，发现其值为 0 ，于是进入临界区。进程 1 也发现其值为 0 ，所以在一个等待循环中不停的测试 turn，看其值何时变为 1。连续检查一个变量直到某个值出现为止，这种方法称为 `忙等待(busywaiting)`。由于这种方式浪费 CPU 时间，所以这种方式通常应该要避免。只有在有理由认为等待时间是非常短的情况下，才能够使用忙等待。用于忙等待的锁，称为 `自旋锁(spinlock)`。

进程 0 离开临界区时，它将 turn 的值设置为 1，以便允许进程 1 进入其临界区。假设进程 1 很快便离开了临界区，则此时两个进程都处于临界区之外，turn 的值又被设置为 0 。现在进程 0 很快就执行完了整个循环，它退出临界区，并将 turn 的值设置为 1。此时，turn 的值为 1，两个进程都在其临界区外执行。

突然，进程 0 结束了非临界区的操作并返回到循环的开始。但是，这时它不能进入临界区，因为 turn 的当前值为 1，此时进程 1 还忙于非临界区的操作，进程 0 只能继续 while 循环，直到进程 1 把 turn 的值改为 0 。这说明，在一个进程比另一个进程执行速度慢了很多的情况下，轮流进入临界区并不是一个好的方法。

这种情况违反了前面的叙述 3 ，即 **位于临界区外的进程不得阻塞其他进程**，进程 0 被一个临界区外的进程阻塞。由于违反了第三条，所以也不能作为一个好的方案。

#### Peterson 解法

荷兰数学家 T.Dekker 通过将锁变量与警告变量相结合，最早提出了一个不需要严格轮换的软件互斥算法，关于 Dekker 的算法，参考 **链接**

后来， G.L.Peterson 发现了一种简单很多的互斥算法，它的算法如下

```c
#define FALSE 0
#define TRUE  1
/* 进程数量 */
#define N     2													

/* 现在轮到谁 */
int turn;					

/* 所有值初始化为 0 (FALSE) */
int interested[N];											

/* 进程是 0 或 1 */
void enter_region(int process){					
  
  /* 另一个进程号 */
  int other;														
  
  /* 另一个进程 */
  other = 1 - process;				
  
  /* 表示愿意进入临界区 */
  interested[process] = TRUE;						
  turn = process;
  
  /* 空循环 */
  while(turn == process 
        && interested[other] == true){} 
  
}

void leave_region(int process){
  
  /* 表示离开临界区 */
  interested[process] == FALSE;				 
}
```

在使用共享变量时（即进入其临界区）之前，各个进程使用各自的进程号 0 或 1 作为参数来调用 `enter_region`，这个函数调用在需要时将使进程等待，直到能够安全的临界区。在完成对共享变量的操作之后，进程将调用 `leave_region` 表示操作完成，并且允许其他进程进入。

现在来看看这个办法是如何工作的。一开始，没有任何进程处于临界区中，现在进程 0 调用 `enter_region`。它通过设置数组元素和将 turn 置为 0 来表示它希望进入临界区。由于进程 1 并不想进入临界区，所以 enter_region 很快便返回。如果进程现在调用 enter_region，进程 1 将在此处挂起直到 `interested[0]` 变为 FALSE，这种情况只有在进程 0 调用 `leave_region` 退出临界区时才会发生。

那么上面讨论的是顺序进入的情况，现在来考虑一种两个进程同时调用 `enter_region` 的情况。它们都将自己的进程存入 turn，但只有最后保存进去的进程号才有效，前一个进程的进程号因为重写而丢失。假如进程 1 是最后存入的，则 turn 为 1 。当两个进程都运行到 `while` 的时候，进程 0 将不会循环并进入临界区，而进程 1 将会无限循环且不会进入临界区，直到进程 0 退出位置。

#### TSL 指令

现在来看一种需要硬件帮助的方案。一些计算机，特别是那些设计为多处理器的计算机，都会有下面这条指令

```shell
TSL RX,LOCK	
```

称为 `测试并加锁(test and set lock)`，它将一个内存字 lock 读到寄存器 `RX` 中，然后在该内存地址上存储一个非零值。读写指令能保证是一体的，不可分割的，一同执行的。在这个指令结束之前其他处理器均不允许访问内存。执行 TSL 指令的 CPU 将会锁住内存总线，用来禁止其他 CPU 在这个指令结束之前访问内存。

很重要的一点是锁住内存总线和禁用中断不一样。禁用中断并不能保证一个处理器在读写操作之间另一个处理器对内存的读写。也就是说，在处理器 1 上屏蔽中断对处理器 2 没有影响。让处理器 2 远离内存直到处理器 1 完成读写的最好的方式就是锁住总线。这需要一个特殊的硬件（基本上，一根总线就可以确保总线由锁住它的处理器使用，而其他的处理器不能使用）

为了使用 TSL 指令，要使用一个共享变量 lock 来协调对共享内存的访问。当 lock 为 0 时，任何进程都可以使用 TSL 指令将其设置为 1，并读写共享内存。当操作结束时，进程使用 `move` 指令将 lock 的值重新设置为 0 。

这条指令如何防止两个进程同时进入临界区呢？下面是解决方案

```assembly
enter_region:
			| 复制锁到寄存器并将锁设为1
			TSL REGISTER,LOCK              
			| 锁是 0 吗？
  		CMP REGISTER,#0						 		
  		| 若不是零，说明锁已被设置，所以循环
  		JNE enter_region					 		
  		| 返回调用者，进入临界区
  		RET												 
       
leave_region:

			| 在锁中存入 0
			MOVE LOCK,#0			      
      | 返回调用者
  		RET												 
```

我们可以看到这个解决方案的思想和 Peterson 的思想很相似。假设存在如下共 4 指令的汇编语言程序。第一条指令将 lock 原来的值复制到寄存器中并将 lock 设置为 1 ，随后这个原来的值和 0 做对比。如果它不是零，说明之前已经被加过锁，则程序返回到开始并再次测试。经过一段时间后（可长可短），该值变为 0 （当前处于临界区中的进程退出临界区时），于是过程返回，此时已加锁。要清除这个锁也比较简单，程序只需要将 0 存入 lock 即可，不需要特殊的同步指令。

现在有了一种很明确的做法，那就是进程在进入临界区之前会先调用 `enter_region`，判断是否进行循环，如果lock 的值是 1 ，进行无限循环，如果 lock 是 0，不进入循环并进入临界区。在进程从临界区返回时它调用 `leave_region`，这会把 lock 设置为 0 。与基于临界区问题的所有解法一样，进程必须在正确的时间调用 enter_region 和 leave_region ，解法才能奏效。

还有一个可以替换 TSL 的指令是 `XCHG`，它原子性的交换了两个位置的内容，例如，一个寄存器与一个内存字，代码如下

```assembly
enter_region:
		| 把 1 放在内存器中
		MOVE REGISTER,#1	
    | 交换寄存器和锁变量的内容
		XCHG REGISTER,LOCK			
    | 锁是 0 吗？
		CMP REGISTER,#0		
    | 若不是 0 ，锁已被设置，进行循环
		JNE enter_region					
    | 返回调用者，进入临界区
		RET														
	
leave_region:				
		| 在锁中存入 0 
		MOVE LOCK,#0	
    | 返回调用者
		RET														
```

XCHG 的本质上与 TSL 的解决办法一样。所有的 Intel x86 CPU 在底层同步中使用 XCHG 指令。

### 睡眠与唤醒

上面解法中的 Peterson 、TSL 和 XCHG 解法都是正确的，但是它们都有忙等待的缺点。这些解法的本质上都是一样的，先检查是否能够进入临界区，若不允许，则该进程将原地等待，直到允许为止。

这种方式不但浪费了 CPU 时间，而且还可能引起意想不到的结果。考虑一台计算机上有两个进程，这两个进程具有不同的优先级，`H` 是属于优先级比较高的进程，`L` 是属于优先级比较低的进程。进程调度的规则是不论何时只要 H 进程处于就绪态 H 就开始运行。在某一时刻，L 处于临界区中，此时 H 变为就绪态，准备运行（例如，一条 I/O 操作结束）。现在 H 要开始忙等，但由于当 H 就绪时 L 就不会被调度，L 从来不会有机会离开关键区域，所以 H 会变成死循环，有时将这种情况称为`优先级反转问题(priority inversion problem)`。

现在让我们看一下进程间的通信原语，这些原语在不允许它们进入关键区域之前会阻塞而不是浪费 CPU 时间，最简单的是 `sleep` 和 `wakeup`。Sleep 是一个能够造成调用者阻塞的系统调用，也就是说，这个系统调用会暂停直到其他进程唤醒它。wakeup 调用有一个参数，即要唤醒的进程。还有一种方式是 wakeup 和 sleep 都有一个参数，即 sleep 和 wakeup 需要匹配的内存地址。

#### 生产者-消费者问题

作为这些私有原语的例子，让我们考虑`生产者-消费者(producer-consumer)` 问题，也称作 `有界缓冲区(bounded-buffer)` 问题。两个进程共享一个公共的固定大小的缓冲区。其中一个是`生产者(producer)`，将信息放入缓冲区， 另一个是`消费者(consumer) `，会从缓冲区中取出。也可以把这个问题一般化为 m 个生产者和 n 个消费者的问题，但是我们这里只讨论一个生产者和一个消费者的情况，这样可以简化实现方案。

如果缓冲队列已满，那么当生产者仍想要将数据写入缓冲区的时候，会出现问题。它的解决办法是让生产者睡眠，也就是阻塞生产者。等到消费者从缓冲区中取出一个或多个数据项时再唤醒它。同样的，当消费者试图从缓冲区中取数据，但是发现缓冲区为空时，消费者也会睡眠，阻塞。直到生产者向其中放入一个新的数据。

这个逻辑听起来比较简单，而且这种方式也需要一种称作 `监听` 的变量，这个变量用于监视缓冲区的数据，我们暂定为 count，如果缓冲区最多存放 N 个数据项，生产者会每次判断 count 是否达到 N，否则生产者向缓冲区放入一个数据项并增量 count 的值。消费者的逻辑也很相似：首先测试 count 的值是否为 0 ，如果为 0 则消费者睡眠、阻塞，否则会从缓冲区取出数据并使 count 数量递减。每个进程也会检查检查是否其他线程是否应该被唤醒，如果应该被唤醒，那么就唤醒该线程。下面是生产者消费者的代码

```c
/* 缓冲区 slot 槽的数量 */
#define N 100						
/* 缓冲区数据的数量 */
int count = 0										
  
// 生产者
void producer(void){
  int item;
  
  /* 无限循环 */
  while(TRUE){				
    /* 生成下一项数据 */
    item = produce_item()				
    /* 如果缓存区是满的，就会阻塞 */
    if(count == N){
      sleep();									
    }
    
    /* 把当前数据放在缓冲区中 */
    insert_item(item);
    /* 增加缓冲区 count 的数量 */
    count = count + 1;					
    if(count == 1){
      /* 缓冲区是否为空？ */
      wakeup(consumer);					
    }
  }
}

// 消费者
void consumer(void){
  
  int item;
  
  /* 无限循环 */
  while(TRUE){
    /* 如果缓冲区是空的，就会进行阻塞 */
  	if(count == 0){							
      sleep();
    }
    /* 从缓冲区中取出一个数据 */
   	item = remove_item();			
    /* 将缓冲区的 count 数量减一 */
    count = count - 1
    /* 缓冲区满嘛？ */
    if(count == N - 1){					
      wakeup(producer);		
    }
    /* 打印数据项 */
    consumer_item(item);				
  }
  
}
```

为了在 C 语言中描述像是 `sleep` 和 `wakeup` 的系统调用，我们将以库函数调用的形式来表示。它们不是 C 标准库的一部分，但可以在实际具有这些系统调用的任何系统上使用。代码中未实现的 `insert_item` 和 `remove_item` 用来记录将数据项放入缓冲区和从缓冲区取出数据等。

现在让我们回到生产者-消费者问题上来，上面代码中会产生竞争条件，因为 count 这个变量是暴露在大众视野下的。有可能出现下面这种情况：缓冲区为空，此时消费者刚好读取 count 的值发现它为 0 。此时调度程序决定暂停消费者并启动运行生产者。生产者生产了一条数据并把它放在缓冲区中，然后增加 count 的值，并注意到它的值是 1 。由于 count 为 0，消费者必须处于睡眠状态，因此生产者调用 `wakeup` 来唤醒消费者。但是，消费者此时在逻辑上并没有睡眠，所以 wakeup 信号会丢失。当消费者下次启动后，它会查看之前读取的 count 值，发现它的值是 0 ，然后在此进行睡眠。不久之后生产者会填满整个缓冲区，在这之后会阻塞，这样一来两个进程将永远睡眠下去。

引起上面问题的本质是 **唤醒尚未进行睡眠状态的进程会导致唤醒丢失**。如果它没有丢失，则一切都很正常。一种快速解决上面问题的方式是增加一个`唤醒等待位(wakeup waiting bit)`。当一个 wakeup 信号发送给仍在清醒的进程后，该位置为 1 。之后，当进程尝试睡眠的时候，如果唤醒等待位为 1 ，则该位清除，而进程仍然保持清醒。

然而，当进程数量有许多的时候，这时你可以说通过增加唤醒等待位的数量来唤醒等待位，于是就有了 2、4、6、8 个唤醒等待位，但是并没有从根本上解决问题。

### 信号量

信号量是 E.W.Dijkstra 在 1965 年提出的一种方法，它使用一个整形变量来累计唤醒次数，以供之后使用。在他的观点中，有一个新的变量类型称作 `信号量(semaphore)`。一个信号量的取值可以是 0 ，或任意正数。0 表示的是不需要任何唤醒，任意的正数表示的就是唤醒次数。

Dijkstra 提出了信号量有两个操作，现在通常使用 `down` 和 `up`（分别可以用 sleep 和 wakeup 来表示）。down 这个指令的操作会检查值是否大于 0 。如果大于 0 ，则将其值减 1 ；若该值为 0 ，则进程将睡眠，而且此时 down 操作将会继续执行。检查数值、修改变量值以及可能发生的睡眠操作均为一个单一的、不可分割的 `原子操作(atomic action)` 完成。这会保证一旦信号量操作开始，没有其他的进程能够访问信号量，直到操作完成或者阻塞。这种原子性对于解决同步问题和避免竞争绝对必不可少。

> 原子性操作指的是在计算机科学的许多其他领域中，一组相关操作全部执行而没有中断或根本不执行。

up 操作会使信号量的值 + 1。如果一个或者多个进程在信号量上睡眠，无法完成一个先前的 down 操作，则由系统选择其中一个并允许该程完成 down 操作。因此，对一个进程在其上睡眠的信号量执行一次 up 操作之后，该信号量的值仍然是 0 ，但在其上睡眠的进程却少了一个。信号量的值增 1 和唤醒一个进程同样也是不可分割的。不会有某个进程因执行 up 而阻塞，正如在前面的模型中不会有进程因执行 wakeup 而阻塞是一样的道理。

#### 用信号量解决生产者 - 消费者问题

用信号量解决丢失的 wakeup 问题，代码如下

```c
/* 定义缓冲区槽的数量 */
#define N 100
/* 信号量是一种特殊的 int */
typedef int semaphore;
/* 控制关键区域的访问 */
semaphore mutex = 1;
/* 统计 buffer 空槽的数量 */
semaphore empty = N;
/* 统计 buffer 满槽的数量 */
semaphore full = 0;												

void producer(void){ 
  
  int item;  
  
  /* TRUE 的常量是 1 */
  while(TRUE){			
    /* 产生放在缓冲区的一些数据 */
    item = producer_item();		
    /* 将空槽数量减 1  */
    down(&empty);	
    /* 进入关键区域  */
    down(&mutex);	
    /* 把数据放入缓冲区中 */
    insert_item(item);
    /* 离开临界区 */
    up(&mutex);	
    /* 将 buffer 满槽数量 + 1 */
    up(&full);														
  }
}

void consumer(void){
  
  int item;
  
  /* 无限循环 */
  while(TRUE){
    /* 缓存区满槽数量 - 1 */
    down(&full);
    /* 进入缓冲区 */	
    down(&mutex);
    /* 从缓冲区取出数据 */
    item = remove_item();	
    /* 离开临界区 */
    up(&mutex);	
    /* 将空槽数目 + 1 */
    up(&empty);	
    /* 处理数据 */
    consume_item(item);											
  }
  
}
```

为了确保信号量能正确工作，最重要的是要采用一种不可分割的方式来实现它。通常是将 up 和 down 作为系统调用来实现。而且操作系统只需在执行以下操作时暂时屏蔽全部中断：**检查信号量、更新、必要时使进程睡眠**。由于这些操作仅需要非常少的指令，因此中断不会造成影响。如果使用多个 CPU，那么信号量应该被锁进行保护。使用 TSL 或者 XCHG 指令用来确保同一时刻只有一个 CPU 对信号量进行操作。

使用 TSL 或者 XCHG 来防止几个 CPU 同时访问一个信号量，与生产者或消费者使用忙等待来等待其他腾出或填充缓冲区是完全不一样的。前者的操作仅需要几个毫秒，而生产者或消费者可能需要任意长的时间。

上面这个解决方案使用了三种信号量：一个称为 full，用来记录充满的缓冲槽数目；一个称为 empty，记录空的缓冲槽数目；一个称为 mutex，用来确保生产者和消费者不会同时进入缓冲区。`Full` 被初始化为 0 ，empty 初始化为缓冲区中插槽数，mutex 初始化为 1。信号量初始化为 1 并且由两个或多个进程使用，以确保它们中同时只有一个可以进入关键区域的信号被称为 `二进制信号量(binary semaphores)`。如果每个进程都在进入关键区域之前执行 down 操作，而在离开关键区域之后执行 up 操作，则可以确保相互互斥。

现在我们有了一个好的进程间原语的保证。然后我们再来看一下中断的顺序保证

1. 硬件压入堆栈程序计数器等

2. 硬件从中断向量装入新的程序计数器

3. 汇编语言过程保存寄存器的值

4. 汇编语言过程设置新的堆栈

5. C 中断服务器运行（典型的读和缓存写入）

6. 调度器决定下面哪个程序先运行

7. C 过程返回至汇编代码

8. 汇编语言过程开始运行新的当前进程

在使用`信号量`的系统中，隐藏中断的自然方法是让每个 I/O 设备都配备一个信号量，该信号量最初设置为0。在 I/O 设备启动后，中断处理程序立刻对相关联的信号执行一个 `down` 操作，于是进程立即被阻塞。当中断进入时，中断处理程序随后对相关的信号量执行一个 `up`操作，能够使已经阻止的进程恢复运行。在上面的中断处理步骤中，其中的第 5 步 `C 中断服务器运行` 就是中断处理程序在信号量上执行的一个 up 操作，所以在第 6 步中，操作系统能够执行设备驱动程序。当然，如果有几个进程已经处于就绪状态，调度程序可能会选择接下来运行一个更重要的进程，我们会在后面讨论调度的算法。

上面的代码实际上是通过两种不同的方式来使用信号量的，而这两种信号量之间的区别也是很重要的。`mutex` 信号量用于互斥。它用于确保任意时刻只有一个进程能够对缓冲区和相关变量进行读写。互斥是用于避免进程混乱所必须的一种操作。

另外一个信号量是关于`同步(synchronization)`的。`full` 和 `empty` 信号量用于确保事件的发生或者不发生。在这个事例中，它们确保了缓冲区满时生产者停止运行；缓冲区为空时消费者停止运行。这两个信号量的使用与 mutex 不同。

### 互斥量

如果不需要信号量的计数能力时，可以使用信号量的一个简单版本，称为 `mutex(互斥量)`。互斥量的优势就在于在一些共享资源和一段代码中保持互斥。由于互斥的实现既简单又有效，这使得互斥量在实现用户空间线程包时非常有用。

互斥量是一个处于两种状态之一的共享变量：`解锁(unlocked)` 和 `加锁(locked)`。这样，只需要一个二进制位来表示它，不过一般情况下，通常会用一个 `整形(integer)` 来表示。0 表示解锁，其他所有的值表示加锁，比 1 大的值表示加锁的次数。

mutex 使用两个过程，当一个线程（或者进程）需要访问关键区域时，会调用 `mutex_lock` 进行加锁。如果互斥锁当前处于解锁状态（表示关键区域可用），则调用成功，并且调用线程可以自由进入关键区域。

另一方面，如果 mutex 互斥量已经锁定的话，调用线程会阻塞直到关键区域内的线程执行完毕并且调用了 `mutex_unlock` 。如果多个线程在 mutex 互斥量上阻塞，将随机选择一个线程并允许它获得锁。

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150342072-865527119.png)

由于 mutex 互斥量非常简单，所以只要有 TSL 或者是 XCHG 指令，就可以很容易地在用户空间实现它们。用于用户级线程包的 `mutex_lock` 和 `mutex_unlock` 代码如下，XCHG 的本质也一样。

```assembly
mutex_lock:
			| 将互斥信号量复制到寄存器，并将互斥信号量置为1
			TSL REGISTER,MUTEX
      | 互斥信号量是 0 吗？
			CMP REGISTER,#0	
      | 如果互斥信号量为0，它被解锁，所以返回
			JZE ok	
      | 互斥信号正在使用；调度其他线程
			CALL thread_yield	
      | 再试一次
			JMP mutex_lock	
      | 返回调用者，进入临界区
ok: 	RET														

mutex_unlcok:
			| 将 mutex 置为 0 
			MOVE MUTEX,#0	
      | 返回调用者
			RET														
```

mutex_lock 的代码和上面 enter_region 的代码很相似，我们可以对比着看一下

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150352444-1754865350.png)

上面代码最大的区别你看出来了吗？

* 根据上面我们对 TSL 的分析，我们知道，如果 TSL 判断没有进入临界区的进程会进行无限循环获取锁，而在 TSL 的处理中，如果 mutex 正在使用，那么就调度其他线程进行处理。所以上面最大的区别其实就是在判断 mutex/TSL 之后的处理。

* 在（用户）线程中，情况有所不同，因为没有时钟来停止运行时间过长的线程。结果是通过忙等待的方式来试图获得锁的线程将永远循环下去，决不会得到锁，因为这个运行的线程不会让其他线程运行从而释放锁，其他线程根本没有获得锁的机会。在后者获取锁失败时，它会调用 `thread_yield` 将 CPU 放弃给另外一个线程。结果就不会进行忙等待。在该线程下次运行时，它再一次对锁进行测试。

上面就是 enter_region 和 mutex_lock 的差别所在。由于 thread_yield 仅仅是一个用户空间的线程调度，所以它的运行非常快捷。这样，`mutex_lock` 和 `mutex_unlock` 都不需要任何内核调用。通过使用这些过程，用户线程完全可以实现在用户空间中的同步，这个过程仅仅需要少量的同步。

我们上面描述的互斥量其实是一套调用框架中的指令。从软件角度来说，总是需要更多的特性和同步原语。例如，有时线程包提供一个调用 `mutex_trylock`，这个调用尝试获取锁或者返回错误码，但是不会进行加锁操作。这就给了调用线程一个灵活性，以决定下一步做什么，是使用替代方法还是等候下去。

#### Futexes

随着并行的增加，有效的`同步(synchronization)`和`锁定(locking)` 对于性能来说是非常重要的。如果进程等待时间很短，那么`自旋锁(Spin lock)` 是非常有效；但是如果等待时间比较长，那么这会浪费 CPU 周期。如果进程很多，那么阻塞此进程，并仅当锁被释放的时候让内核解除阻塞是更有效的方式。不幸的是，这种方式也会导致另外的问题：它可以在进程竞争频繁的时候运行良好，但是在竞争不是很激烈的情况下内核切换的消耗会非常大，而且更困难的是，预测锁的竞争数量更不容易。

有一种有趣的解决方案是把两者的优点结合起来，提出一种新的思想，称为 `futex`，或者是 `快速用户空间互斥(fast user space mutex)`，是不是听起来很有意思？

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150401556-853982062.png)

futex 是 `Linux` 中的特性实现了基本的锁定（很像是互斥锁）而且避免了陷入内核中，因为内核的切换的开销非常大，这样做可以大大提高性能。futex 由两部分组成：**内核服务和用户库**。内核服务提供了了一个 `等待队列(wait queue)` 允许多个进程在锁上排队等待。除非内核明确的对他们解除阻塞，否则它们不会运行。

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150610838-825682646.png)

对于一个进程来说，把它放到等待队列需要昂贵的系统调用，这种方式应该被避免。在没有竞争的情况下，futex 可以直接在用户空间中工作。这些进程共享一个 32 位`整数(integer)` 作为公共锁变量。假设锁的初始化为 1，我们认为这时锁已经被释放了。线程通过执行原子性的操作`减少并测试(decrement and test)` 来抢占锁。decrement and set 是 Linux 中的原子功能，由包裹在 C 函数中的内联汇编组成，并在头文件中进行定义。下一步，线程会检查结果来查看锁是否已经被释放。如果锁现在不是锁定状态，那么刚好我们的线程可以成功抢占该锁。然而，如果锁被其他线程持有，抢占锁的线程不得不等待。在这种情况下，futex 库不会`自旋`，但是会使用一个系统调用来把线程放在内核中的等待队列中。这样一来，切换到内核的开销已经是合情合理的了，因为线程可以在任何时候阻塞。当线程完成了锁的工作时，它会使用原子性的 `增加并测试(increment and test)` 释放锁，并检查结果以查看内核等待队列上是否仍阻止任何进程。如果有的话，它会通知内核可以对等待队列中的一个或多个进程解除阻塞。如果没有锁竞争，内核则不需要参与竞争。

#### Pthreads 中的互斥量

Pthreads 提供了一些功能用来同步线程。最基本的机制是使用互斥量变量，可以锁定和解锁，用来保护每个关键区域。希望进入关键区域的线程首先要尝试获取 mutex。如果 mutex 没有加锁，线程能够马上进入并且互斥量能够自动锁定，从而阻止其他线程进入。如果 mutex 已经加锁，调用线程会阻塞，直到 mutex 解锁。如果多个线程在相同的互斥量上等待，当互斥量解锁时，只有一个线程能够进入并且重新加锁。这些锁并不是必须的，程序员需要正确使用它们。

下面是与互斥量有关的函数调用

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150633956-1982090400.png)

向我们想象中的一样，mutex 能够被创建和销毁，扮演这两个角色的分别是 `Phread_mutex_init` 和 `Pthread_mutex_destroy`。mutex 也可以通过 `Pthread_mutex_lock` 来进行加锁，如果互斥量已经加锁，则会阻塞调用者。还有一个调用`Pthread_mutex_trylock` 用来尝试对线程加锁，当 mutex 已经被加锁时，会返回一个错误代码而不是阻塞调用者。这个调用允许线程有效的进行忙等。最后，`Pthread_mutex_unlock` 会对 mutex 解锁并且释放一个正在等待的线程。

除了互斥量以外，`Pthreads` 还提供了第二种同步机制： `条件变量(condition variables)` 。mutex 可以很好的允许或阻止对关键区域的访问。条件变量允许线程由于未满足某些条件而阻塞。绝大多数情况下这两种方法是一起使用的。下面我们进一步来研究线程、互斥量、条件变量之间的关联。

下面再来重新认识一下生产者和消费者问题：一个线程将东西放在一个缓冲区内，由另一个线程将它们取出。如果生产者发现缓冲区没有空槽可以使用了，生产者线程会阻塞起来直到有一个线程可以使用。生产者使用 mutex 来进行原子性检查从而不受其他线程干扰。但是当发现缓冲区已经满了以后，生产者需要一种方法来阻塞自己并在以后被唤醒。这便是条件变量做的工作。

下面是一些与条件变量有关的最重要的 pthread 调用

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150642772-1483454099.png)

上表中给出了一些调用用来创建和销毁条件变量。条件变量上的主要属性是 `Pthread_cond_wait` 和 `Pthread_cond_signal`。前者阻塞调用线程，直到其他线程发出信号为止（使用后者调用）。阻塞的线程通常需要等待唤醒的信号以此来释放资源或者执行某些其他活动。只有这样阻塞的线程才能继续工作。条件变量允许等待与阻塞原子性的进程。`Pthread_cond_broadcast` 用来唤醒多个阻塞的、需要等待信号唤醒的线程。

>需要注意的是，条件变量（不像是信号量）不会存在于内存中。如果将一个信号量传递给一个没有线程等待的条件变量，那么这个信号就会丢失，这个需要注意

下面是一个使用互斥量和条件变量的例子

```c
#include <stdio.h>
#include <pthread.h>

/* 需要生产的数量 */
#define MAX 1000000000										
pthread_mutex_t the_mutex;
/* 使用信号量 */
pthread_cond_t condc,condp;								
int buffer = 0;

/* 生产数据 */
void *producer(void *ptr){								
  
  int i;
  
  for(int i = 0;i <= MAX;i++){
    /* 缓冲区独占访问，也就是使用 mutex 获取锁 */
    pthread_mutex_lock(&the_mutex);				
    while(buffer != 0){
      pthread_cond_wait(&condp,&the_mutex);
    }
    /* 把他们放在缓冲区中 */
    buffer = i;			
    /* 唤醒消费者 */
    pthread_cond_signal(&condc);	
    /* 释放缓冲区 */
    pthread_mutex_unlock(&the_mutex);			
  }
  pthread_exit(0);
  
}

/* 消费数据 */
void *consumer(void *ptr){								
  
  int i;
  
  for(int i = 0;i <= MAX;i++){
    /* 缓冲区独占访问，也就是使用 mutex 获取锁 */
    pthread_mutex_lock(&the_mutex);				
    while(buffer == 0){
      pthread_cond_wait(&condc,&the_mutex);
    }
    /* 把他们从缓冲区中取出 */
    buffer = 0;	
    /* 唤醒生产者 */
    pthread_cond_signal(&condp);
    /* 释放缓冲区 */
    pthread_mutex_unlock(&the_mutex);			
  }
  pthread_exit(0);
  
}							  
```

### 管程

为了能够编写更加准确无误的程序，Brinch Hansen 和 Hoare 提出了一个更高级的同步原语叫做 `管程(monitor)`。他们两个人的提案略有不同，通过下面的描述你就可以知道。管程是程序、变量和数据结构等组成的一个集合，它们组成一个特殊的模块或者包。进程可以在任何需要的时候调用管程中的程序，但是它们不能从管程外部访问数据结构和程序。下面展示了一种抽象的，类似 Pascal 语言展示的简洁的管程。不能用 C 语言进行描述，因为管程是语言概念而 C 语言并不支持管程。

```pascal
monitor example
	integer i;
	condition c;
	
	procedure producer();
  ...
	end;	
	
	procedure consumer();
	.
	end;
end monitor;
```

管程有一个很重要的特性，即在任何时候管程中只能有一个活跃的进程，这一特性使管程能够很方便的实现互斥操作。管程是编程语言的特性，所以编译器知道它们的特殊性，因此可以采用与其他过程调用不同的方法来处理对管程的调用。通常情况下，当进程调用管程中的程序时，该程序的前几条指令会检查管程中是否有其他活跃的进程。如果有的话，调用进程将被挂起，直到另一个进程离开管程才将其唤醒。如果没有活跃进程在使用管程，那么该调用进程才可以进入。

进入管程中的互斥由编译器负责，但是一种通用做法是使用 `互斥量(mutex)` 和 `二进制信号量(binary semaphore)`。由于编译器而不是程序员在操作，因此出错的几率会大大降低。在任何时候，编写管程的程序员都无需关心编译器是如何处理的。他只需要知道将所有的临界区转换成为管程过程即可。绝不会有两个进程同时执行临界区中的代码。

即使管程提供了一种简单的方式来实现互斥，但在我们看来，这还不够。因为我们还需要一种在进程无法执行被阻塞。在生产者-消费者问题中，很容易将针对缓冲区满和缓冲区空的测试放在管程程序中，但是生产者在发现缓冲区满的时候该如何阻塞呢？

解决的办法是引入`条件变量(condition variables)` 以及相关的两个操作 `wait` 和 `signal`。当一个管程程序发现它不能运行时（例如，生产者发现缓冲区已满），它会在某个条件变量（如 full）上执行 `wait` 操作。这个操作造成调用进程阻塞，并且还将另一个以前等在管程之外的进程调入管程。在前面的 pthread 中我们已经探讨过条件变量的实现细节了。另一个进程，比如消费者可以通过执行 `signal` 来唤醒阻塞的调用进程。

>Brinch Hansen 和 Hoare 在对进程唤醒上有所不同，Hoare 建议让新唤醒的进程继续运行；而挂起另外的进程。而 Brinch Hansen 建议让执行 signal 的进程必须退出管程，这里我们采用 Brinch Hansen 的建议，因为它在概念上更简单，并且更容易实现。

如果在一个条件变量上有若干进程都在等待，则在对该条件执行 signal 操作后，系统调度程序只能选择其中一个进程恢复运行。

顺便提一下，这里还有上面两位教授没有提出的第三种方式，它的理论是让执行 signal 的进程继续运行，等待这个进程退出管程时，其他进程才能进入管程。

条件变量不是计数器。条件变量也不能像信号量那样积累信号以便以后使用。所以，如果向一个条件变量发送信号，但是该条件变量上没有等待进程，那么信号将会丢失。也就是说，**wait 操作必须在 signal 之前执行**。

下面是一个使用 `Pascal` 语言通过管程实现的生产者-消费者问题的解法

```pascal
monitor ProducerConsumer
		condition full,empty;
		integer count;
		
		procedure insert(item:integer);
		begin
				if count = N then wait(full);
				insert_item(item);
				count := count + 1;
				if count = 1 then signal(empty);
		end;
		
		function remove:integer;
		begin
				if count = 0 then wait(empty);
				remove = remove_item;
				count := count - 1;
				if count = N - 1 then signal(full);
		end;
		
		count := 0;
end monitor;

procedure producer;
begin
			while true do
      begin 
      			item = produce_item;
      			ProducerConsumer.insert(item);
      end
end;

procedure consumer;
begin 
			while true do
			begin
						item = ProducerConsumer.remove;
						consume_item(item);
			end
end;
```

读者可能觉得 wait 和 signal 操作看起来像是前面提到的 sleep 和 wakeup ，而且后者存在严重的竞争条件。它们确实很像，但是有个关键的区别：sleep 和 wakeup 之所以会失败是因为当一个进程想睡眠时，另一个进程试图去唤醒它。使用管程则不会发生这种情况。管程程序的自动互斥保证了这一点，如果管程过程中的生产者发现缓冲区已满，它将能够完成 wait 操作而不用担心调度程序可能会在 wait 完成之前切换到消费者。甚至，在 wait 执行完成并且把生产者标志为不可运行之前，是不会允许消费者进入管程的。

尽管类 Pascal 是一种想象的语言，但还是有一些真正的编程语言支持，比如 Java （终于轮到大 Java 出场了），Java 是能够支持管程的，它是一种 `面向对象`的语言，支持用户级线程，还允许将方法划分为类。只要将关键字 `synchronized` 关键字加到方法中即可。Java 能够保证一旦某个线程执行该方法，就不允许其他线程执行该对象中的任何 synchronized 方法。没有关键字 synchronized ，就不能保证没有交叉执行。

下面是 Java 使用管程解决的生产者-消费者问题

```java
public class ProducerConsumer {
  // 定义缓冲区大小的长度
  static final int N = 100;
  // 初始化一个新的生产者线程
  static Producer p = new Producer();
  // 初始化一个新的消费者线程
  static Consumer c = new Consumer();		
  // 初始化一个管程
  static Our_monitor mon = new Our_monitor(); 
  
  // run 包含了线程代码
  static class Producer extends Thread{
    public void run(){												
      int item;
      // 生产者循环
      while(true){														
        item = produce_item();
        mon.insert(item);
      }
    }
    // 生产代码
    private int produce_item(){...}						
  }
  
  // run 包含了线程代码
  static class consumer extends Thread {
    public void run( ) {											
   		int item;
      while(true){
        item = mon.remove();
				consume_item(item);
      }
    }
    // 消费代码
    private int produce_item(){...}						
  }
  
  // 这是管程
  static class Our_monitor {									
    private int buffer[] = new int[N];
    // 计数器和索引
    private int count = 0,lo = 0,hi = 0;			
    
    private synchronized void insert(int val){
      if(count == N){
        // 如果缓冲区是满的，则进入休眠
        go_to_sleep();												
      }
      // 向缓冲区插入内容
			buffer[hi] = val;					
      // 找到下一个槽的为止
      hi = (hi + 1) % N; 				
      // 缓冲区中的数目自增 1 
      count = count + 1;											
      if(count == 1){
        // 如果消费者睡眠，则唤醒
        notify();															
      }
    }
    
    private synchronized void remove(int val){
      int val;
      if(count == 0){
        // 缓冲区是空的，进入休眠
        go_to_sleep();												
      }
      // 从缓冲区取出数据
      val = buffer[lo];				
      // 设置待取出数据项的槽
      lo = (lo + 1) % N;					
      // 缓冲区中的数据项数目减 1 
      count = count - 1;											
      if(count = N - 1){
        // 如果生产者睡眠，唤醒它
        notify();															
      }
      return val;
    }
    
    private void go_to_sleep() {
      try{
        wait( );
      }catch(Interr uptedExceptionexc) {};
    }
  }
      
}
```

上面的代码中主要设计四个类，`外部类(outer class)` ProducerConsumer 创建并启动两个线程，p 和 c。第二个类和第三个类 `Producer` 和 `Consumer` 分别包含生产者和消费者代码。最后，`Our_monitor` 是管程，它有两个同步线程，用于在共享缓冲区中插入和取出数据。

在前面的所有例子中，生产者和消费者线程在功能上与它们是相同的。生产者有一个无限循环，该无限循环产生数据并将数据放入公共缓冲区中；消费者也有一个等价的无限循环，该无限循环用于从缓冲区取出数据并完成一系列工作。

程序中比较耐人寻味的就是 `Our_monitor` 了，它包含缓冲区、管理变量以及两个同步方法。当生产者在 insert 内活动时，它保证消费者不能在 remove 方法中运行，从而保证更新变量以及缓冲区的安全性，并且不用担心竞争条件。变量 count 记录在缓冲区中数据的数量。变量 `lo` 是缓冲区槽的序号，指出将要取出的下一个数据项。类似地，`hi` 是缓冲区中下一个要放入的数据项序号。允许 lo = hi，含义是在缓冲区中有 0 个或 N 个数据。

Java 中的同步方法与其他经典管程有本质差别：Java 没有内嵌的条件变量。然而，Java 提供了 wait 和 notify 分别与 sleep 和 wakeup 等价。

**通过临界区自动的互斥，管程比信号量更容易保证并行编程的正确性**。但是管程也有缺点，我们前面说到过管程是一个编程语言的概念，编译器必须要识别管程并用某种方式对其互斥作出保证。**C、Pascal 以及大多数其他编程语言都没有管程**，所以不能依靠编译器来遵守互斥规则。

与管程和信号量有关的另一个问题是，这些机制都是设计用来解决访问共享内存的一个或多个 CPU 上的互斥问题的。通过将信号量放在共享内存中并用 `TSL` 或 `XCHG` 指令来保护它们，可以避免竞争。但是如果是在分布式系统中，可能同时具有多个 CPU 的情况，并且每个 CPU 都有自己的私有内存呢，它们通过网络相连，那么这些原语将会失效。因为信号量太低级了，而管程在少数几种编程语言之外无法使用，所以还需要其他方法。

### 消息传递

上面提到的其他方法就是 `消息传递(messaage passing)`。这种进程间通信的方法使用两个原语 `send` 和 `receive` ，它们像信号量而不像管程，是系统调用而不是语言级别。示例如下

```c
send(destination, &message);

receive(source, &message);
```

send 方法用于向一个给定的目标发送一条消息，receive 从一个给定的源接受一条消息。如果没有消息，接受者可能被阻塞，直到接受一条消息或者带着错误码返回。

#### 消息传递系统的设计要点

消息传递系统现在面临着许多信号量和管程所未涉及的问题和设计难点，尤其对那些在网络中不同机器上的通信状况。例如，消息有可能被网络丢失。为了防止消息丢失，发送方和接收方可以达成一致：一旦接受到消息后，接收方马上回送一条特殊的 `确认(acknowledgement)` 消息。如果发送方在一段时间间隔内未收到确认，则重发消息。

现在考虑消息本身被正确接收，而返回给发送着的确认消息丢失的情况。发送者将重发消息，这样接受者将收到两次相同的消息。

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150730689-723957941.png)

对于接收者来说，如何区分新的消息和一条重发的老消息是非常重要的。通常采用在每条原始消息中嵌入一个连续的序号来解决此问题。如果接受者收到一条消息，它具有与前面某一条消息一样的序号，就知道这条消息是重复的，可以忽略。

消息系统还必须处理如何命名进程的问题，以便在发送或接收调用中清晰的指明进程。`身份验证(authentication)` 也是一个问题，比如客户端怎么知道它是在与一个真正的文件服务器通信，从发送方到接收方的信息有可能被中间人所篡改。

#### 用消息传递解决生产者-消费者问题

现在我们考虑如何使用消息传递来解决生产者-消费者问题，而不是共享缓存。下面是一种解决方式

```c
/* buffer 中槽的数量 */
#define N 100													

void producer(void){
  
  int item;
  /* buffer 中槽的数量 */
  message m;													
  
  while(TRUE){
    /* 生成放入缓冲区的数据 */
    item = produce_item();						
    /* 等待消费者发送空缓冲区 */
    receive(consumer,&m);							
    /* 建立一个待发送的消息 */
    build_message(&m,item);						
    /* 发送给消费者 */
    send(consumer,&m);								
  }
  
}

void consumer(void){
  
  int item,i;
  message m;
  
  /* 循环N次 */
  for(int i = 0;i < N;i++){						
    /* 发送N个缓冲区 */
    send(producer,&m);								
  }
  while(TRUE){
    /* 接受包含数据的消息 */
    receive(producer,&m);							
    /* 将数据从消息中提取出来 */
  	item = extract_item(&m);					
    /* 将空缓冲区发送回生产者 */
    send(producer,&m);								
    /* 处理数据 */
    consume_item(item);								
  }
  
}
```

假设所有的消息都有相同的大小，并且在尚未接受到发出的消息时，由操作系统自动进行缓冲。在该解决方案中共使用 N 条消息，这就类似于一块共享内存缓冲区的 N 个槽。消费者首先将 N 条空消息发送给生产者。当生产者向消费者传递一个数据项时，它取走一条空消息并返回一条填充了内容的消息。通过这种方式，系统中总的消息数量保持不变，所以消息都可以存放在事先确定数量的内存中。

如果生产者的速度要比消费者快，则所有的消息最终都将被填满，等待消费者，生产者将被阻塞，等待返回一条空消息。如果消费者速度快，那么情况将正相反：所有的消息均为空，等待生产者来填充，消费者将被阻塞，以等待一条填充过的消息。

消息传递的方式有许多变体，下面先介绍如何对消息进行 `编址`。

* 一种方法是为每个进程分配一个唯一的地址，让消息按进程的地址编址。
* 另一种方式是引入一个新的数据结构，称为 `信箱(mailbox)`，信箱是一个用来对一定的数据进行缓冲的数据结构，信箱中消息的设置方法也有多种，典型的方法是在信箱创建时确定消息的数量。在使用信箱时，在 send 和 receive 调用的地址参数就是信箱的地址，而不是进程的地址。当一个进程试图向一个满的信箱发送消息时，它将被挂起，直到信箱中有消息被取走，从而为新的消息腾出地址空间。

### 屏障

最后一个同步机制是准备用于进程组而不是进程间的生产者-消费者情况的。在某些应用中划分了若干阶段，并且规定，除非所有的进程都就绪准备着手下一个阶段，否则任何进程都不能进入下一个阶段，可以通过在每个阶段的结尾安装一个 `屏障(barrier)` 来实现这种行为。当一个进程到达屏障时，它会被屏障所拦截，直到所有的屏障都到达为止。屏障可用于一组进程同步，如下图所示

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150743423-1807513724.png)

在上图中我们可以看到，有四个进程接近屏障，这意味着每个进程都在进行运算，但是还没有到达每个阶段的结尾。过了一段时间后，A、B、D 三个进程都到达了屏障，各自的进程被挂起，但此时还不能进入下一个阶段呢，因为进程 B 还没有执行完毕。结果，当最后一个 C 到达屏障后，这个进程组才能够进入下一个阶段。

### 避免锁：读-复制-更新

最快的锁是根本没有锁。问题在于没有锁的情况下，我们是否允许对共享数据结构的并发读写进行访问。答案当然是不可以。假设进程 A 正在对一个数字数组进行排序，而进程 B 正在计算其平均值，而此时你进行 A 的移动，会导致 B 会多次读到重复值，而某些值根本没有遇到过。

然而，在某些情况下，我们可以允许写操作来更新数据结构，即便还有其他的进程正在使用。窍门在于确保每个读操作要么读取旧的版本，要么读取新的版本，例如下面的树

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150754818-1433977551.png)

上面的树中，读操作从根部到叶子遍历整个树。加入一个新节点 X 后，为了实现这一操作，我们要让这个节点在树中可见之前使它"恰好正确"：我们对节点 X 中的所有值进行初始化，包括它的子节点指针。然后通过原子写操作，使 X 称为 A 的子节点。所有的读操作都不会读到前后不一致的版本

![](https://img2020.cnblogs.com/blog/1515111/202003/1515111-20200303150805555-1073956369.png)

在上面的图中，我们接着移除 B 和 D。首先，将 A 的左子节点指针指向 C 。所有原本在 A 中的读操作将会后续读到节点 C ，而永远不会读到 B 和 D。也就是说，它们将只会读取到新版数据。同样，所有当前在 B 和 D 中的读操作将继续按照原始的数据结构指针并且读取旧版数据。所有操作均能正确运行，我们不需要锁住任何东西。而不需要锁住数据就能够移除 B 和 D 的主要原因就是 `读-复制-更新(Ready-Copy-Update,RCU)`，将更新过程中的移除和再分配过程分离开。