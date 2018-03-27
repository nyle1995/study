# ConcurrentHashMap

主要总结一下 ConcurrentHashMap(JDK 1.8 )的技术点：

1.  和HashMap一样，通过hash到值，放到一个Node<K,V>[] table里。 每一个节点下挂着一个list或者一个红黑树
2.  当一个节点下的数量达到一个数量，会扩容成一个红黑树
3.  当写操作时，如果发现正在扩容，会协助扩容。  
        当扩容一个节点，或者对一个节点进行写操作的时候（插入，删除）的时候，会对这个节点进行加锁处理

### 扩容
1. 会扩容到一个新的list里Node<K,V>[] nextTable;
2. 逐步把之前table里的每一个节点的头节点都替换（迁移了这个节点之后完后替换）成 ForwardingNode（维护nextTable的指针）
3. sizeCtl  
通过维护一个sizeCtl, 这个值来表示共同扩容的线程数量（一个特殊值维护在高16位，下16位为共同操作的线程数）  
最后一个线程，会再检查一遍，再把table替换掉
4. transferIndex  
逆序迁移，从n-1开始  
不同线程进行处理时，会先申请节点（cas处理）进行处理。每次申请的节点数量会根据cpu和节点数量计算（最小16个）
```
while (advance) {
    int nextIndex, nextBound;
    if (--i >= bound || finishing)
        advance = false;
    else if ((nextIndex = transferIndex) <= 0) {
        i = -1;
        advance = false;
    }
    else if (U.compareAndSwapInt
                (this, TRANSFERINDEX, nextIndex,
                nextBound = (nextIndex > stride ?
                            nextIndex - stride : 0))) {
        bound = nextBound;
        i = nextIndex - 1;
        advance = false;
    }
}
```
5. 申请到了之后  
    就开始对一个节点进行扩容。  
    这里对于一个列表，会计算到最后一批hash & newSize 值一样的（重新分配也在一个节点上的）。  
    然后再遍历一遍，分成两个列表，最后一批一样的节点就不处理了 
    扩容结束后，把头节点替换成ForwardingNode

### 计数

使用一个baseCount 和 CounterCell[] counterCells;  
计数的时候取和  

当更新baseCount失败的情况，会尝试修改counterCells。  
而且counterCells只会扩容，不会减少。  

### 红黑树注意点
首节点是一个TreeBin来进行维护，同时维护一个红黑树和一个双向列表  
通过维护一个violate lockState 来支持一个读写锁  
（因为在写操作前，已经对首节点加锁了，所以不会有写操作的多线程竞争）  

锁的状态：
```
001 - static final int WRITER = 1; // set while holding write lock   
010 - static final int WAITER = 2; // set when waiting for write lock
100 - static final int READER = 4; // increment value for setting read lock， 可以叠加，每次加4相当于
```

#### 红黑树-读：  
当发现有写线程执行或者等待的时候，转为链表读取  
没有的话，加读锁，然后通过红黑树查找  
```
final Node<K,V> find(int h, Object k) {
    if (k != null) {
        for (Node<K,V> e = first; e != null; ) {
            int s; K ek;
            if (((s = lockState) & (WAITER|WRITER)) != 0) {
                // 锁的状态  WAITER|WRITER 11
                // 表示是写锁，或者等待写锁的状态
                // 用链表超找
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                e = e.next;
            }
            else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                            s + READER)) {
                // lockState + READER , 修改成功表示加读锁成功
                TreeNode<K,V> r, p;
                try {
                    p = ((r = root) == null ? null :
                            r.findTreeNode(h, k, null));
                } finally {
                    Thread w;
                    if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                        (READER|WAITER) && (w = waiter) != null)
                        // 读取完成, 读锁 - 4
                        // 如果释放读锁之后，发现是等待状态，唤醒等待的线程
                        LockSupport.unpark(w);
                }
                return p;
            }
        }
    }
    return null;
}
```

#### 红黑树-写逻辑
1. 如果没有读锁，直接执行
2. 如果有读锁，把自己挂起，设置waiter线程为自己，等待读锁结束后把自己唤醒
```
private final void contendedLock() {
    boolean waiting = false;
    for (int s;;) {
        // ~WAITER表示反转WAITER，当没有线程持有读锁时，该条件为true
        // ～WAITER = 111..101
        // lockState 的 第1位和第3位都不能是1
        if (((s = lockState) & ~WAITER) == 0) {
            if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                //没有任何线程持有读写锁时，尝试让当前线程获取写锁，同时清空waiter标识位
                if (waiting)
                    waiter = null;
                return;
            }
        }
        else if ((s & WAITER) == 0) {  
             // 当前线程持有读锁
             // 并且当前线程不是WAITER状态时
            if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {   //尝试占据WAITER标识位
                waiting = true;    //表明自己处于waiter状态
                waiter = Thread.currentThread();
            }
        }
        else if (waiting)  //当前线程持有读锁，并且当前线程处于waiter状态时，该条件为true
            LockSupport.park(this);  //阻塞自己
    }
}
```