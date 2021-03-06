# JVM 学习总结

## 内存区域

1. 程序计数器
    - 线程私有，每个线程都有一个
    - 较小
    - 表示当前线程执行的字节码的指示器
2. 栈
    - 线程私有
    - 存储局部变量表，操作数栈，动态连接，方法出口等信息
    - 栈异常

        -  请求栈深度大于虚拟机允许的深度，抛出StackOverflowError
        -  如果扩展栈的长度时,没有足够的内存了，也会抛出OutOfMemoryError
3. 本地方法栈
    - 类似与栈
4. Java 堆
    - 线程共享
    - 区域

        1. 新生代 （Eden， Survivor1， Survivor2）
        2. 老年代
5. 方法区
    - 线程共享
    - 类静态变量 & 运行时常量池
6. 直接内存
    - NIO 使用, 如果堆分配的太大，导致没有机器内存分配给直接内存，也会抛OOM

## 内存分配

1. java通过 空闲列表，标记哪些内存配分配了，哪些空闲
2. 通过CAS保证原子性更新
3. 部分使用本地线程分配缓冲（TLAB）, 即每个线程预先分配一块内存

## 对象的内存布局

1. 对象头

    Hash码， GC信息，锁相关的信息  
    类型指针，指向class的指针
2. 实际数据
3. 对象填充

    因为HotSpot VM的内存管理系统要求对象的地址必须是8字节的正式倍，所以起一个占位符的作用

## 对象的访问定位

1. 通过句柄

    句柄维护堆中实际指针的位置, GC的时候，只需改变句柄中的指针就好了

2. 直接指针

    直接指向堆中的Object， 好处是速度更快

## 垃圾收集器

1. 一般都是使用的可达性分析算法  
    从GC ROOT 节点向下搜索
2. 不同引用的区别  
    - 强引用   
        只要引用还在，就不能被GC
    - 软引用 SofeReference  
        在系统OOM之前，会对这些对象进行回收
    - 弱引用 WeakReference  
        生存到下一次GC之前
    - 虚引用  
        唯一目的就是能就是在对象被垃圾回收的时候收到一个系统通知
3. 算法
    - 标记-清除
    - 标记-整理
    - 复制
    - 分代收集
4. 安全点  
    只有到达了安全点，才能停顿下来GC
5. 垃圾收集器
    - 新生代  
        1. Serial 单线程收集
        2. Parnew 多线程收集
        3. Paralled Scavenge 收集器  
            与Parnew的区别：  
            - 自适应调节
            - 目的是达到一个可控制的吞吐量（CMS更关注减少停顿时间）  
                - 停顿时间短是更适合与用户交互的程序，提升用户体验
                - 高吞吐量是为了高效利用CPU时间，适合后台运算
    - 老年代
        1. Serial Old 单线程收集
        2. Parallel Old 多线程收集（Paralled Scavenge收集器的老年版本）
        3. CMS  
            - 只能与Serial 和 Parnew 新生代配和
            - 流程  
                - 先Stop world, 标记GC ROOT 连接的Object
                - 然后恢复用户线程，同时并发GC线程开始标记
                - Stop world， 重新标记，把用户线程修改过的Object重新标记一遍
                - 恢复用户线程，并发清理Object
            - 缺点
                - 对CPU资源比较敏感
                - 无法处理浮动垃圾(用户线程并行时产生的垃圾)， 而且需要预留一部分给空间给用户线程使用
                - 空间碎片

        


## JVM 相关工具

1. jps  
    列出正在运行的虚拟机线程
2. jstat  
    jvm统计信息监视工具，可以查看各堆的状况，gc情况，类装载的情况
3. jinfo  
    查看jvm的各种配置
4. jmap  
    dump 堆快照
5. jhat  
    堆快照的分析工具
6. jstack  
    堆栈的跟踪工具

## JVM调优思路

1. 由于自动扩展堆导致慢  
    将 -Xms(最小)  -Xmx(最大) 设置成一样，可以避免堆自动扩展
2. -XX: +HeapDumpOnOutOfMemoryError

    可以在出现OOM的时候dump当前快照
3. 方法区或者常量池溢出  
    比如String存的太多，或者class类型很多  
    -XX:PermSize , -XX:MaxSize 限制方法区的大小
4. 直接内存溢出  
    直接内存只会在老年代full gc的时候，顺便清理。
5. 临时变量多还是老年代多，可以调新生代老年代比例  
    -XX:SurvivorRatio
6. 分配超大堆的问题  
    - full gc的时间会较长， 所以一般如果是高性能硬件，一般采用多个虚拟机来利用硬件资源
7. 等待的线程和Socket连接很多也会导致虚拟机崩溃

## 类加载

1. 双亲委派模型
    - Bootstrap ClassLoader  
        jdk的一些jar
    - Extensions ClassLoader  
        加载java的扩展库（jre/ext/*.jar路径下的内容
    - Application ClassLoader
        classpath 加载，java应用都是由他完成加载
2. 过程
    - 加载  
        文件，网络，或者自定义都可以(动态代理，cglib)  
        注意：  
            - 通过子类引用父类的静态字段，不会导致子类初始化  
            - 通过数组定义来引用类，不会出发类的初始化  
            - 常量在编译过程中，会存入调用类的常量池中，因此不会出发定义常量类的初始化。 
    - 验证  
        验证class对不对
    - 准备  
        分配内存空间(方法区)
    - 解析  
        虚拟机常量池内的符号引用替换为直接引用
    - 初始化
3. 热部署原理
    - 在类加载器中，java类只能被加载一次
    - 所以销毁该自定义ClassLoader，然后更新更新class类文件，再创建新的ClassLoader去加载更新后的class类文件

其他 ：
1. HotSpot VM 是范围最广的JVM

