
# ReentrantLock 
通过state 维护锁逻辑， 维护一个线程lock的次数（支持同线程重入）

公平锁 FairSync      
非公平锁 NonfairSync   

都继承自 内部类 Sync，Sync 继承自 AQS

### 加锁的逻辑
都是调用AQS 的aquire（非公平锁是先尝试加锁， 再调用aquire）  
aquire就是调用tryquire的里的方法

#### 1. tryAcquire逻辑
基本就是基于cas修改state，成功相当于枷锁成功  
公平锁， 需要先判断是不是在head上，是的话才尝试cas获取锁 （hasQueuedPredecessors） 
比较复杂的逻辑在AQS里

数据结构Node（双向链表 prev, next）  
包含  
nextWaiter ：使用在condition中  
线程  
维护一个waitStatus,标记是独占锁还是共享锁  

```
总的流程就是
获取一个锁
加锁失败的话，cas的方式把自己挂在CLH队列的最后
然后把前一个节点的状态修改SIGNAL
然后把自己挂起

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        // 1. 尝试加锁 - tryacuire这个由继承类自己实现
        // 加锁失败的话
        //  首先 addWaiter 一个读占类型的 Node
        //          addWaiter的方式： 如果tail不为空 cas把这个node设置为tail，然后放到上一个tail的next节点后
        //                          tail为空 或者cas失败 ，调用enq方法
        //                                                  enq ： 循环检测：
                                                                    1. 如果没有初始化过，先设置cas head为一个new出来的node，成功的话，tail = head
                                                                    2. 初始化过了的话， cas设置node为tail，成果把node放到tail的后面
        //    然后调用acquireQueued
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 返回true的话，标志失败了
// 一个for循环，获取这个node的前一个节点
        如果前一个节点就是head节点（正在执行的节点，或者初始节点，就tryAcquire一下，tryAcquire成功了，就执行）
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //不是head或者tryAcquire失败了的话
            // 先调用shouldParkAfterFailedAcquire
                    如果前一个节点没有check过，把前一个节点的状态设为SIGNAL，返回false。再循环检查一次
                    如果前一个节点已经被关掉了，把前面的被关掉的节点删掉，node.pre 连接到最前面一个正常节点
                    如果这个节点的状态已经是SIGNAL，返回true
            // true了之后，调用parkAndCheckInterrupt ，使用LockSupport来阻塞
            // 然后根据是否interrupted来确定是否是否把interrupted设置为true
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
             // 如果失败了，关掉这个node
                    状体设置为cancel，
                        如果node是tail， 把tail设置为node前一个有效节点
                        不是tail的话， 当 前一个节点不是头节点 && （前一个节点是SIGNAL状态 || (初始状态，且变成了SIGNAL状态)） && 前一个线程的thrad不是null， 这种情况下，把后一个和前一个连接起来
                        否则，唤醒node后面的一个节点（为空，或者后一个节点cancel了，就从tail开始往前找到最前面的该被唤醒的节点）
            cancelAcquire(node);
    }
}

```


### 释放锁 
调用AQS里的 release

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // tryRelease 子类中实现
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 头节点就是正在运行的节点，如果头节点的status不是0，就唤醒下一个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

unparkSuccessor唤醒之后节点的第一个有效节点

1.  这里本来觉得有一个问题： 就是出现在unparkSuccessor下一个线程，但是那个线程还没有parkAndCheckInterrupt的情况下：

当 unpark可能在 park之前。出现在release的时候，检查有tail，但是tai的线程还没有park的情况下。  
后来测试发现， LockSuport支持 先unpark在park.  
所以，在release中，先unpark，再park的情况是可以接受的。  




## Condition
没有什么很复杂的操作（因为不会有多线程操作，这里是lock之后操作）

维护一个firstWaiter， 和一个lastWaiter 的节点
一个列表里面的状态只能为  
Canceled或者condition

#### await
await 挂在 condition的列表里
```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // addNode到末尾    
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //  是否还挂在Condition的等待list里
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }

    // 重新加锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

#### signalAll 
把整个队列从condition的队列里删掉，然后挂到AQS的队列后面（状态从 Condition -> 0）



## 备注

1. CLH队列是什么？   
维护head，tail  
每一个节点监控前一个节点的状态，来监控是否获取锁。  
AQS 这里加了变种  
每一个node，在自旋锁的过程中，会把自己挂起。然后node执行完之后，会唤醒node的next节点。  