---
title: DirectBuffer垃圾回收
date: 2019-09-10 23:48:49
author: Chopin
tags:   #标签
    - NIO
---

## 前言

本文是笔者在研究DirectByteBuffer垃圾回收过程中引发的学习与探索。众所周知，DirectByteBuffer是一个管理直接内存的引用对象，直接内存不能通过JVM进行垃圾回收，只能通过DirectByteBuffer被回收时，调用相应的JNI方法来释放直接内存。

由于垃圾回收本身成本较高，一般JVM在堆内存未耗尽时，不会进行垃圾回收操作。如果不断分配本地内存，堆内存很少使用，那么JVM就不需要执行GC，DirectByteBuffer对象们就不会被回收，这时候堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError，那程序就直接崩溃了。因此我们希望能够手工回收直接内存，于是对DirectByteBuffer被回收时如何释放直接内存进行研究。

<!-- more -->

```java
    DirectByteBuffer(int cap) {                   // package-private
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
    
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```

前面部分为分配内存地址，回收直接内存的关键在于

```java
cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
```

## Cleaner

我们看一下Cleaner这个类的源码

```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
    
    /**
     * 所有的cleaner都会被加到一个双向链表中去，这样做是为了保证在referent被回收之前
     * 这些Cleaner都是存活的。
     */
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    
    // 用户自定义的一个Runnable对象，
    private final Runnable thunk;

    // 构造的时候把自己加到双向链表中去
    private static synchronized Cleaner add(Cleaner var0) {
        if (first != null) {
            var0.next = first;
            first.prev = var0;
        }

        first = var0;
        return var0;
    }

    // clean方法会调用remove把当前的cleaner从链表中删除。
    private static synchronized boolean remove(Cleaner var0) {
        if (var0.next == var0) {
            return false;
        } else {
            if (first == var0) {
                if (var0.next != null) {
                    first = var0.next;
                } else {
                    first = var0.prev;
                }
            }

            if (var0.next != null) {
                var0.next.prev = var0.prev;
            }

            if (var0.prev != null) {
                var0.prev.next = var0.next;
            }

            var0.next = var0;
            var0.prev = var0;
            return true;
        }
    }

    // 私有有构造函数，保证了用户无法单独地使用new来创建Cleaner。	
    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }

    /**
     * 所有的Cleaner都必须通过create方法进行创建。
     */
    public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null ? null : add(new Cleaner(var0, var1));
    }

    /**
     * 这个方法会被Reference Handler线程调用，来清理资源。
     */
    public void clean() {
        if (remove(this)) {
            try {
                this.thunk.run();
            } catch (final Throwable var2) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        if (System.err != null) {
                            (new Error("Cleaner terminated abnormally", 
                            				var2)).printStackTrace();
                        }

                        System.exit(1);
                        return null;
                    }
                });
            }

        }
    }
}
```

1. Cleaner继承自PhantomReference（虚引用），它本质上仍然是一个Reference。所以它的处理方法与WeakReference，SoftReference十分相似。仍然是由GC标记，Reference Handler线程处理的。

2. Reference的定义里新启的那个线程，它的run方法会专门判断从pending链表上取出来的那个对象是不是Cleaner，如果是就会调用它的clean方法。所以我们知道了，**Cleaner的clean方法是由Reference Handler线程调用的**。 

   ```java
   if (r instanceof Cleaner) {
       ((Cleaner)r).clean();
       continue;
   }
   ```

3. Cleaner本身不带有清理逻辑，所有的逻辑都封装在thunk中，因此thunk是怎么实现的才是最关键。

因此我们接着看DirectByteBuffer自定义的thunk方法 **Deallocator**

```java
    private static class Deallocator implements Runnable {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }
    }
```

1. 使用unsafe根据堆外内存的起始地址释放堆外内存；
2. 根据当前DirectByte的size与cap修改在Bits中的统计信息，Bits类主要就是统计当前堆外内存的分配情况。

只是研究如何手工回收DirectByteBuffer引用的直接内存空间到这里就可以了，DirectByteBuffer实现了DirectBuffer，而DirectBuffer本身是public的，所以通过接口去调用内部的Cleaner对象来做clean方法。

```java
if (byteBuffer.isDirect()) {
    ((DirectBuffer)byteBuffer).cleaner().clean();
}
```

Netty通过引用计数和池化来回收空间以及减少性能消耗，为不同类型的ByteBuf实现了不同的release方法，底层也是unsafe方法，对直接内存进行回收。

**那JVM是如何自动回收直接内存的呢？**

## PhantomReference

上文提到

> Cleaner继承自PhantomReference（虚引用），它本质上仍然是一个Reference。所以它的处理方法与WeakReference，SoftReference十分相似。仍然是由GC标记，Reference Handler线程处理的。

### 引用类型

这里比较一下Java的引用类型：强引用、软引用、弱引用、虚引用。

引用对象是对JVM内存heap中Java对象的引用，通过软引用、弱引用、虚引用可以和GC做简单的交互。

heap中对象有强可及对象、软可及对象、弱可及对象、虚可及对象和不可到达对象。引用的强弱顺序是强、软、弱、虚。对于对象是属于哪种可及的对象，由他的最强的引用决定。如下：

```java
String str = new String("abc"); //1   
SoftReference<String> softRef = new SoftReference<String>(str); //2   
WeakReference<String> weakRef = new WeakReference<String>(str); //3   
str = null; //4   
softRef.clear(); //5
```

第一行在heap对中创建内容为"abc"的对象，并建立abc到该对象的强引用,该对象是强可及的。

第二行和第三行分别建立对heap中对象的软引用和弱引用，此时heap中的对象仍是强可及的。

第四行之后heap中对象不再是强可及的，变成软可及的。同样第五行执行之后变成弱可及的。

#### 强引用

强引用是使用最普遍的引用。如果一个对象具有强引用，那垃圾回收器绝不会回收它。当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。  

```java
Object o = new Object();   
Object o1 = o;  
```

第一句是在heap中创建新的Object对象通过o引用这个对象，第二句是通过o建立o1到new Object()这个heap堆中的对象的引用，这两个引用都是强引用。只要存在对heap中对象的强引用，GC就不会收集该对象。

#### 软引用

软引用主要用于内存敏感的高速缓存。在JVM报告内存不足之前会清除所有的软引用，这样以来GC就有可能收集软可及的对象，可能解决内存吃紧问题，避免内存溢出。什么时候会被收集取决于GC的算法和GC运行时可用内存的大小。 

以上面的代码为例，回收软可及对象步骤如下：

1. 首先将softRef的referent设置为null，不再引用heap中的new String("abc")对象。
2. 将heap中的new String("abc")对象设置为可结束的(finalizable)。
3. 当heap中的new String("abc")对象的finalize()方法被运行而且该对象占用的内存被释放， softRef被添加到它的ReferenceQueue中。

使用示例：

```java
Object obj = new Object();
SoftRefenrence sr = new SoftReference(obj);
obj = null

// 如果GC还未回收软引用
if(sr != null){
    obj = sr.get();
}
// 如果GC已回收软引用
else {
    obj = new A();
    sr = new SoftReference(obj);
}
```

#### 弱引用

如果一个对象只具有弱引用，那就类似于可有可无的生活用品。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存，然后Java虚拟机会把这个弱引用加入到与之关联的引用队列中。 不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 

使用示例：

```java
Object obj = new Object();
WeakReference wr = new WeakReference(obj);
obj = null;

//等待一段时间，heap中new Object()对象就会被垃圾回收
...

if (wr.get() == null) { 
    System.out.println("obj 已经被清除了"); 
} else { 
    System.out.println("obj 尚未被清除，其信息是 " + obj.toString());
}
```

`WeakHashMap`和`ThreadLocal`都用了弱引用

#### 虚引用

虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。虚引用主要用来跟踪对象被垃圾回收的活动。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

与软引用和弱引用不同，回收虚可及对象时，先把PhantomRefrence对象添加到它的ReferenceQueue中，然后再释放虚可及的对象。

JVM回收直接内存就用到了虚引用。

### Reference

Cleaner继承PhantomRefrence，PhantomRefrence继承Reference，Reference类的静态方法中启动了一个handler线程，代码如下

```java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```

首先创建handler线程类，然后设置优先级，设置为守护线程，然后启动。 ReferenceHandler源码如下：

```java
    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {
                Reference<Object> r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        pending = r.discovered;
                        r.discovered = null;
                    } else {
                        // The waiting on the lock may cause an OOME because it may try to allocate
                        // exception objects, so also catch OOME here to avoid silent exit of the
                        // reference handler thread.
                        //
                        // Explicitly define the order of the two exceptions we catch here
                        // when waiting for the lock.
                        //
                        // We do not want to try to potentially load the InterruptedException class
                        // (which would be done if this was its first use, and InterruptedException
                        // were checked first) in this situation.
                        //
                        // This may lead to the VM not ever trying to load the InterruptedException
                        // class again.
                        try {
                            try {
                                lock.wait();
                            } catch (OutOfMemoryError x) { }
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue<Object> q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }
```

1. 首先看pending是否有值，pending是JVM进行赋值的，当对象可达性变为不可达时会赋值到pending上；
2. 如果pending有值，将pending的值赋给r，discoverd是下一个不可达的对象，赋值给pending；如果pending不存在值，等待pending有值；
3. 判断r是不是Cleaner对象，是的话执行clean方法；
4. 再将虚引用加入引用队列。



参考链接：

https://www.jianshu.com/p/6806ad95ed14

https://zhuanlan.zhihu.com/p/29454205

https://www.jianshu.com/p/825cca41d962

https://www.jianshu.com/p/f86d3a43eec5