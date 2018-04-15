# java多线程（一） - volatile&notify 

java多线程基本的类总结一下要点

## volatile


1. 各个线程获取时都要从主存里获取，不存线程的工作内存中获取
1. 和final不能同时使用


#### volatile 和 static的区别
一开始很疑惑和static的区别是什么，因为static在多个变量获取的时候，是同用一份的。而且多线程使用发现也是修改能被其他线程感知到。

直到。。。。看了这篇博客，实验了一下 [https://dzone.com/articles/java-volatile-keyword-0](https://dzone.com/articles/java-volatile-keyword-0)

多线程如果对static的值写入过于频发的话，jvm可能会优化拷贝一份内存到工作内存中，而导致其他线程感知不到
也就是说static不保证多线程读到的值是同一份！

## wait - notify - notifyall

这几个函数的具体含义就不总结了，可以去看看线程的状态类型转换

注意点：

1. 永远在循环里调用
1. 必须要在 wait的 object的 synchronized里

    因为没有锁的话wait和notify有可能会产生竞态条件

1. notifyall 是唤醒所有 wait线程； notify是唤醒一个

#### notify 容易造成死锁
比如生产者消费者模型，如果存在多个生产者和多个消费者的情况，如果使用notify容易造成死锁
具体可以看下这篇博客 [http://blog.csdn.net/tayanxunhua/article/details/20998449](http://blog.csdn.net/tayanxunhua/article/details/20998449)
