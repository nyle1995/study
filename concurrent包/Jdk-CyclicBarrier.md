# CyclicBarrier

### 使用
就是当有多少个线程都执行完await方法之后，才能都往下走

### await
主要逻辑在dowait方法里, 每次await都加锁  

通过ReentrantLock， Condition实现等待和唤醒 

1. 维护一个count来表示是否await的线程数  
2. 一个generation，表示这个是否坏了  
    interrupted的时候
    barrierCommand抛出异常的时候
    await超时的时候

流程大概是
1. 当count还没有减到0的时候，执行condition的await挂在Condition的等待列表上
2. 当count == 0了，触发runnable，然后执行nextGeneration，唤醒所有condition上的等待线程  
        （当然也可能是在发生错误的时候唤醒，所以唤醒之后要check一下generation，正常释放generation会重新创建一个new Generation（））

```
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```