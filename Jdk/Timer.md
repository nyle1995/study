# java多线程（二） - Timer 源码阅读

在JDK5ScheduledThreadPoolExecutor 之前。使用Timer定时器使用

Timer在调用shedulue TimerTask的时候，会把这个task放到一个队列里

```
  private final TaskQueue queue = new TaskQueue();
```

在Timer new出来的时候，会启动一个Thread  
这个线程会循环的从queue中取Task出来执行， 一个一个执行；
·
每一个task会存一个应该执行的时间；
```
class TimerTask implements Runnable {
    ...
    long nextExecutionTime;
    ...
}
```

1. 如果时间到了，就立即执行
1. 如果时间没到，会整个timer wait

```
while(true) {
    ...
    task = queue.getMin();
    ...
    bool taskFired = (executionTime<=currentTime)

    if (!taskFired) // Task hasn't yet fired; wait
        queue.wait(executionTime - currentTime);
    else {
        task.run();
        ...
    }
    ...
}
```

#### shedule和scheduleAtFixedRate 

当这个task执行完之后，设定下次一个要执行的task的时间上有区别；

1. shedule 下次task的执行时间 = 这个task执行完的时间 + 间隔时间
1. cheduleAtFixedRate 下次执行task的时间 = 这个task应该执行的时间 + 间隔时间

```
while(true) {
    ...
    executionTime = task.nextExecutionTime;

    // scheduleAtFixedRate 的period是用正数标示，schedule 则是用负数标示
    queue.rescheduleMin(task.period<0 ? currentTime   - task.period : executionTime + task.period);
    ...
}

```

也就是说：

1. 如果一段时间内本应该执行很多task，但超时了，那么如果使用cheduleAtFixedRate，这短时间内本应该执行到次数最后还是会补上，并且会很快的补完
1. 一个Timer不会同时执行
1. Timer没有try机制，如果抛出了异常，就不会再执行后面的任务了·