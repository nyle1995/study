# Semaphore

也自己实现了两套Sync
1. FairSync
2. NonfairSync

### acquire
首先调用Sync中的tryAcquireShared，当返回的小于0，则进行后续阻塞等处理
```
 if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
```

tryAcquireShared：
通过一个for循环的CAS来修改state，
（公平信号量的区别是，如果不是头节点，则直接返回-1）
```
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

加一个SHARED节点放到尾上
```
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
和之前的AQS操作差不多  
共享锁的区别是：setHeadAndPropagate发现还有剩余资源的话，会继续唤醒下一个线程。独占锁则是不会  
释放的时候，tryReleaseShared成功了就唤醒下一个进行check


