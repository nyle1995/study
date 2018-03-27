# 使用

```
private static final ThreadLocal<Object> threadlocal = new ThreadLocal<Object>() {
    @Override
    protected Object initialValue() {
        return new Object;
    }
};

value.set(**)

value.get()
```

## 源码分析

1. 每个Thread里有一个包含一个 ThreadLocal.ThreadLocalMap threadLocals = null;
2. 从维护一个Entry的数组，Entry是一个 WeakReference, 弱引用的是这个threadlocal，value 是 set的对象

所以其实就是从 当前Thread的一个map里找这个ThreadLocal为key的对象

```
ThreadLocal : 
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

```

## Entry使用WeakReference的目的

一般定义ThreadLocal， 是使用的static的，也就是说一直会有一个强引用指向这个ThreadLocal  
这个时候，Entry使用弱引用是没有效果的，因为不会被gc  

场景是为了解决这种问题:
当一个线程的一个Threadlocal不再使用的时候，设为null（强引用释放）  
但是该线程还持有这个ThreadLocalMap的时候，会导致该部分内存泄漏  

比如说一个tomcat线程，每次操作都会  new一个ThreadLocal  
然后起多个线程操作这个ThreadLocal  
然后多个线程结束，这个ThreadLocal也不用了  
如果不设置成虚引用的话，这个tomcat主线程里，就不会gc调这个threadlocal，因为当前线程的map里还有这个threadlocal的引用