## ReentrantReadWriteLock

读写锁  
通过CAS 修改state, 来加锁  
共享锁：高16位表示共享锁占用的的数量  
独占锁：用后16位表示独占锁的数量  

总的来说，就是读锁获取，如果失败了（只有当前面有写锁的时候，且自己没有持有锁的时候）, 挂一个共享SHARE node挂在CLH的。  
写锁失败了，挂一个排他锁放到CLH后面。  
唤醒的时候，发现如果是SHARE的情况，就继续唤醒后面的节点。  

```
/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

### 读锁-ReadLock

维护
1. firstReader  第一个读线程
2. firstReaderHoldCount， 第一个读线程，读锁的hold数量
3. readHolds 一个ThreadLocal， 线程对应的读锁获取的数量
4. cachedHoldCounter 最后一个获取到读锁的线程的 counter

lock的时候调用  
sync.acquireShared(1);

```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

最后会尝试加锁的时候，调用Sync里的tryAcquireShared
1. 获取独占锁, 如果独占锁被其他线程取了，return false
2. 获取共享锁的锁数量
3. 先执行readerShouldBlock
    公平锁：前面有一个不是自己线程的节点, 返回true
    非公平锁：头节点后面一个是一个独占锁的节点，返回true
4. 如果不需要block
    - CAS state，高16位 + 1
    - CAS 成功，则表示加读锁成功。
5. 如果需要block 或者，cas加读锁失败，调用fullTryAcquireShared

```
 protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}

返回 < 0的情况 ：
1. 独占锁，且独占锁不是自己
2. CLH队列前面有其他节点且当前节点没有获取到过读锁，返回-1

其他情况，cas的修改state

final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            // 如果有独占锁，且不是自己占的独占锁，return -1
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            if (firstReader == current) {
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        // 获取自己这个线程的 holds，如果为0，总readHolds中剔除
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    // 检查一下自己的线程，是否获取过读锁，没获取过，返回-1
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

如果尝试加锁的返回值 <0,再执行doAcquireShared
```
这样的话，就会挂一个Node节点放到CHL队列上
```

### 写锁

```
 public void lock() {
    sync.acquire(1);
}
 返回false的情况 ：
 1. 如果有读锁
 2. 如果有写锁且写锁的占有者不是自己
 3. 如果没有写锁
 4. 需要阻塞
 5. 或者不需要阻塞，但是CAS加锁失败
 protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 表示当前线程占过了写锁，直接return true，加锁成功， state += acquires
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        // 如果需要阻塞，直接return true
        // 尝试不需要，尝试CAS加锁，失败再阻塞
        return false;
    // 走到这里表示加锁不需要阻塞 且， CAS 加锁成功，设置exclusiveOwnerThread 为自己
    setExclusiveOwnerThread(current);
    return true;
}

writerShouldBlock：
公平锁 ： 前面有其他节点，需要阻塞
非公平锁： 一直return false

```