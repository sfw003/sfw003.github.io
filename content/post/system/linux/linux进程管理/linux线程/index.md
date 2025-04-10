---
title: "Linux 线程"
description: "Linux 线程"
date: 2025-04-03
slug: "linux-thread"
categories:
    - linux
tags:
    - linux线程
---









## 线程与TCB

>线程（Thread）是进程内的一个执行分支，线程的执行粒度，要比进程更细。

如何理解？

![image-20250410164313828](image/image-20250410164313828.png)



### linux下的线程与pthread库

**linux没有真正意义的线程**，这在很多教材都提过的观点。究其原因，就如上图所示，让n个PCB来管理一个进程地址空间不就有了线程了。区别windows系统专门设置了TCB结构体，linux选择用PCB来模拟TCB，因此linux可以说没有真正意义的线程，而是叫做**轻量级进程**。



你linux搞特殊是吧？我想使用线程，你告诉我说让我使用轻量级进程。这让用户满意吗？于是linux封装了一个库 **pthread**，让用户可以使用到**用户级线程**，并不能叫做**内核级线程**，因为在linux内核中只有轻量级进程。可以通过下图来理解：

![在这里插入图片描述](image/f99a74ee3b55fc2c2ab61eae415b29a7.png)

注：

- 轻量级进程:用户级线程 = 1:1
- 用户使用到的TCB结构体并不在内核空间中。



上图中出现了2个名词：线程栈和线程局部存储

#### 线程栈

每一个线程都有自己的调用链，注定了每个线程都要有调用链对应的栈帧结构，用来存储线程函数中的局部变量、函数参数以及函数调用的返回地址等信息。这一点类似进程的main函数的函数栈帧。这里的线程栈由pthread来维护，有了线程栈，才能在内核里创建执行流，有了新的执行流，才是一个真正的线程。

> pthread是如何创建线程栈？
>
> inux下只有轻量级进程的概念，自然linux会提供轻量级进程的接口，pthread库正是对轻量级进程的接口进行了封装，才在用户层创建了线程的概念。
>
> linux下创建轻量级进程的系统接口是clone
>
> ![在这里插入图片描述](image/4172bee3ea61ac55cc7cf5ad25794b62.png)

如何验证？

```cpp
#include <pthread.h>
#include <unistd.h>
#include <iostream>
#include <vector>
using namespace std;
int g_val = 0;
void *thread_routine(void* arg) {
    int* pi = (int*)arg;
    int i = *pi;
    int j = 0;
    while(j < 5) {
        cout << "thread-" << i << ", j=" << j << ", &j=" << &j << ", g_val=" << g_val << ", &g_val=" << &g_val << endl;
        sleep(2);
        j++, g_val++;
    }
    return NULL;
}
int main() {
    int i = 0;
    std:;vector<pthread_t> tids;
    while(i < 4) {
        pthread_t tid;
        pthread_create(&tid, nullptr, thread_routine, (void*)&i);
        sleep(1);
        tids.push_back(tid);
        i++;
    }
    for(int j = 0; j < 4; j++){
        pthread_join(tids[j], nullptr);
        sleep(1);
    }
    sleep(3);
    return 0;
}
```
![在这里插入图片描述](image/b752d85cc384361884d16c1771508fba.png)


不同的线程，都执行的同一个函数thread_routine，但函数内部的临时变量j，不是共享的，如果是共享的，那么不同的线程的j的地址应该一样。但结果表明，j的地址都不同。这也表明线程有自己独立的栈结构。

对于全局变量g_val，不同线程是共享的。



插入一个问题：线程栈由多大？

在linux系统中使用 `ulimit -a`可以查看

```
root@hcss-ecs-f8b5:/blog# ulimit -a
real-time non-blocking time  (microseconds, -R) unlimited
core file size              (blocks, -c) 0
data seg size               (kbytes, -d) unlimited
scheduling priority                 (-e) 0
file size                   (blocks, -f) unlimited
pending signals                     (-i) 6628
max locked memory           (kbytes, -l) 226728
max memory size             (kbytes, -m) unlimited
open files                          (-n) 65535
pipe size                (512 bytes, -p) 8
POSIX message queues         (bytes, -q) 819200
real-time priority                  (-r) 0
stack size                  (kbytes, -s) 8192
cpu time                   (seconds, -t) unlimited
max user processes                  (-u) 6628
virtual memory              (kbytes, -v) unlimited
file locks                          (-x) unlimited
```

stack size                  (kbytes, -s) 8192 约为8M







#### 线程的局部存储
如果我想线程拥有私有的全局变量呢？这就要使用线程的局部存储。
`int g_val = 0  改为 __thread in g_val = 0`
此时不同的线程的变量g_val，都是不同的
![在这里插入图片描述](image/82808e1010e8844a562705b983fe0c6c.png)
在变量前加上 `__thread`， 就可以将该变量改为局部存储。除了这个关键字之外，pthread库中也有一些函数可以实现局部存储。

__thread 是 GCC 中用于实现线程局部存储（Thread Local Storage，TLS）的关键字。它可以用来声明线程局部变量，这些变量对于每个线程是唯一的，不同线程之间的变量不会相互影响。**但是它只能声明内置类型，无法声明自定义类型。**
需要注意的是，__thread 关键字是 GCC 的扩展语法，虽然在大多数情况下可以正常工作，但并不是 C 标准的一部分，因此在一些不支持 GCC 扩展语法的编译器中可能无法使用。在 C11 标准中引入了 _Thread_local 关键字，用于实现线程局部存储，具有类似的功能，而且是标准化的语法。









## 线程的竞争与协作

进程地址空间存在的意义是什么？扩大地址空间，内存保护、进程隔离。这中间最重要一点的就是进程隔离，它可以放在数据被随意修改。现在多个线程共享一个进程地址空间，不可避免的就会出现数据竞争的问题。

看下面这段代码：4个线程同时访问全局变量tickets

```cpp
#include <pthread.h>
#include <unistd.h>
#include <iostream>
#include <vector>
using namespace std;

int tickets = 1000;

void *getTickets(void* arg) {
    int* pi = (int*)arg;
    int i = *pi;
    while(1) {
    //票大于0才抢
        if(tickets > 0) {
            usleep(1000);
            printf("thread-%d, get a ticket: %d\n", i, tickets);
            tickets--;
        }
        else {
            break;
        }
    }
    return nullptr;
}
int main() {
    int i = 0;
    std::vector<pthread_t> tids;
    while(i < 4) {
        pthread_t tid;
        pthread_create(&tid, nullptr, getTickets, (void*)&i);
        tids.push_back(tid);
        i++;
    }
    for(int j = 0; j < 4; j++) {
        pthread_join(tids[j], nullptr);
    }
    sleep(3);
    return 0;
}
```

![在这里插入图片描述](image/220d95bc9366db5f9a606c2a70d4e3e2.png)

按照抢票的逻辑，票>0才抢，<=0就退出，怎么会出现负数呢？一张票应该属于一个线程，怎么出现了相同数字？问题就出在多线程并发访问。

<img src="image/1263aed2829689f36df7f566e583c72e.png" alt="在这里插入图片描述" style="zoom:67%;" />

多线程执行操作共享变量会导致竞争状态，在执行过程中发生了上下文切换，得到了错误的结果。每次运行都可能得到不同的结果，存在不确定性，为了解决这类问题，我们需要引入同步和互斥

### 同步和互斥相关概念

首先了解以下几个概念

- **并发**：指的是多个事情，在同一时间段内同时发生了。

- **并行**：指的是多个事情，在同一时间点上同时发生了。只有在多CPU的情况中，才会出现

- **共享资源**：多个线程之间可以**并发**访问的资源。
- **临界资源**：多个线程**互斥**访问的共享资源。（共享资源 且 同一时间只能由一个线程访问 即为临界资源）
- **临界区**：每个线程内部，访问临界资源的代码，就叫做临界区
- **互斥**：任何时刻，互斥保证有且只有一个执行流进入临界区，访问临界资源，通常对临界资源起保护作用
- **同步**：就是并发进程/线程在一些关键点上可能需要互相等待与互通消息，这种相互制约的等待与互通信息称为进程/线程同步。
- **原子性**：不会被任何调度机制打断的操作，该操作只有两态，要么完成，要么未完成



了解完概念，下面的问题是如何实现线程的同步和互斥？



### 同步和互斥的实现

操作系统提供实现线程协作的措施和方法，主要的方法有两种：

- 锁
- POSIX信号量



#### 互斥锁和条件变量

一个线程进入临界区之前，需要**申请锁**，只有拿了锁，才能执行临界区的代码。锁只有一把，故当一个线程拿了锁后，其他线程在申请锁时就会被阻塞，只有当持有锁的线程执行完临界区代码、进行**解锁**后，其他线程才可以申请锁。此时便实现了多个线程互斥的访问共享资源。

> 锁有2中基本类型：互斥锁（互斥量）和自旋锁，下面以互斥锁为例。

![image-20250410225111994](image/image-20250410225111994.png)



单纯加锁已经解决了数据资源竞争问题，但是又引出了锁资源竞争的问题。不同线程对锁的竞争能力是不同，这可能导致大部分时间，锁都在某一个线程上，这将导致线程的**饥饿问题**

![image-20250410225938165](image/image-20250410225938165.png)

为此，我们需要引入一个同步机制，比如条件变量

> 条件变量（Condition Variable）是一种线程同步机制，需要与互斥锁（Mutex）结合使用，用于在线程间传递某个条件的状态并实现线程的等待和唤醒。条件变量允许一个或多个线程在满足特定条件之前进入等待状态，并在条件被满足时被唤醒。

![image-20250410230214278](image/image-20250410230214278.png)



##### 互斥锁和条件变量的使用（以下采用linux原生API)





##### 互斥锁的原理

锁可以保护临界区被线程互斥访问。但锁同时也是所有线程共享，属于共享资源，那锁自身的互斥问题呢？那必须将申请锁设计成**原子的**。（不会被任何调度机制打断的操作，该操作只有两态，要么完成，要么未完成）
首先我们要有一个共识：**一条汇编语句是原子的**。
但加锁的过程不可能仅仅只是一条汇编语句。那是如何实现加锁过程是原子的呢？

为了实现互斥锁操作,大多数体系结构都提供了swap或exchange指令,该指令的作用是把寄存器和内存单元的数据相交换,由于只有一条指令,保证了原子性,即使是多处理器平台,访问内存的 总线周期也有先后,一个处理器上的交换指令执行时另一个处理器的交换指令只能等待总线周期。

```
lock:
        movb $0, %al
        xchgb %al, mutex
        if(al寄存器的内容〉0) {
        	return 0;
        } else
	        挂起等待;
        goto lock;
unlock:
		movb $l,mutex
		唤醒等待Mutex的线程；
		return 0;
```

<img src="image/efdc12726f6487bd1c677f9e9ae7e1c1.gif" alt="请添加图片描述" style="zoom:150%;" />



#### 信号量



## 锁 *



### 锁的种类

#### 互斥锁和自旋锁

**互斥锁和自旋锁是并发编程中最基础的两种锁**，其核心区别在于**等待锁时的行为**

- **互斥锁**：通过**休眠**让出CPU，减少资源占用，但引入上下文切换开销
- **自旋锁**：通过**忙等待**避免切换，适合短临界区，但可能浪费CPU周期





#### 读写锁

读写锁由两把锁组成，读锁和写锁。它的应用场景：**能明确区分读操作和写操作，且读操作 多于 写操作**。







### 死锁 *

什么是死锁？比如当两个线程为了保护两个不同的共享资源而使用了两个互斥锁，那么这两个互斥锁应用不当的时候，可能会造成两个线程都在等待对方释放锁，在没有外力的作用下，这些线程会一直相互等待，就没办法继续运行，这种情况就是发生了死锁。



**死锁有四个必要条件**

- **互斥条件**：多个线程不能同时使用同一个资源。
- **请求与保持条件**：线程因请求资源而阻塞时，对已获得的资源保持不放
- **不剥夺条件**：线程已获得的资源，在末使用完之前，不能强行剥夺
- **循环等待条件**：多个线程之间形成一种头尾相接的循环等待资源的关系



如何破坏死锁？打破4个条件的其中一个就行，最常见的就是打破**循环等待条件**，核心就是理清**资源的获取和释放顺序**。



## CP模式





