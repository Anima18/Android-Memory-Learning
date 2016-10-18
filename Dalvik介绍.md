### 1. android为什么使用Dalvik？  
  Dalvik虚拟机是Google等厂商合作开发的Android移动设备平台的核心组成部分之一。它可以支持已转换为.dex（即Dalvik Executable）格式的Java应用程序的运行，.dex格式是专为Dalvik设计的一种压缩格式，适合内存和处理器速度有限的系统。  

### 2. Dalvik VS JVM  
**区别：**
1. 具有不同的类文件格式以及指令集，Dalvik虚拟机使用的是dex（Dalvik Executable）格式的类文件，而Java虚拟机使用的是class格式的类文件。一个dex文件可以包含若干个类，而一个class文件只包括一个类。
2. Dalvik虚拟机使用的指令是基于寄存器的，而Java虚拟机使用的指令集是基于堆栈的  

**Dalvik优势：**
1. 将多个类文件收集到同一个dex文件中，以便节省空间；

2. 使用只读的内存映射方式加载dex文件，以便可以多进程共享dex文件，节省程序加载时间；

3. 提前调整好字节序（byte order）和字对齐（word alignment）方式，使得它们更适合于本地机器，以便提高指令执行速度；

4. 尽量提前进行字节码验证（bytecode verification），提高程序的加载速度；

5. 需要重写字节码的优化要提前进行。  
    
### 3. Dalvik内存管理 
1. 内存管理  
![image](http://img.blog.csdn.net/20141123013605828?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
-  Dalvik虚拟机用来分配对象的堆划分为两部分，一部分叫做Active Heap，另一部分叫做Zygote Heap。  
-  Android系统的第一个Dalvik虚拟机是由Zygote进程创建的，应用程序进程是由Zygote进程fork出来的。应用程序进程和Zygote进程共享了同一个用来分配对象的堆，Zygote进程或者应用程序进程对该堆进行写操作时，内核就会执行真正的拷贝操作(==一种写时拷贝技术（COW）来复制了Zygote进程的地址空间==)  
-  当Zygote进程在fork第一个应用程序进程之前，会将已经使用了的那部分堆内存划分为一部分，还没有使用的堆内存划分为另外一部分。前者就称为Zygote堆，后者就称为Active堆。以后无论是Zygote进程，还是应用程序进程，当它们需要分配对象的时候，都在Active堆上进行。这样就可以使得Zygote堆尽可能少地被执行写操作。
-  在Zygote堆里面分配的对象其实主要就是Zygote进程在启动过程中预加载的类、资源和对象了。这意味着这些预加载的类、资源和对象可以在Zygote进程和应用程序进程中做到长期共享。
-  分配对象的堆划分为Active堆和Zygote堆，既能减少内存需求，也能减少拷贝操作。  

2. 创建对象  
![image](http://img.blog.csdn.net/20141203015902705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  

- 调用函数dvmHeapSourceAlloc在Java堆上分配指定大小的内存。如果分配成功，那么就将分配得到的地址直接返回给调用者了。 

- 如果上一步内存分配失败，这时候就需要执行一次GC了。调用函数gcForMalloc来执行一次GC了，参数false表示不要回收软引用对象引用的对象。  

- GC执行完毕后，再次调用函数dvmHeapSourceAlloc尝试轻量级的内存分配操作。如果分配成功，那么就将分配得到的地址直接返回给调用者了。  

- 如果上一步内存分配失败，这时候就得考虑先将Java堆的当前大小设置为Dalvik虚拟机启动时指定的Java堆最大值，再进行内存分配了。  

- 如果调用函数dvmHeapSourceAllocAndGrow分配内存成功，则直接将分配得到的地址直接返回给调用者了。  

- 如果上一步内存分配还是失败，这时候就得出狠招了。再次调用函数gcForMalloc来执行GC。参数true表示要回收软引用对象引用的对象。  

- GC执行完毕，再次调用函数dvmHeapSourceAllocAndGrow进行内存分配。这是最后一次努力了，成功与事都到此为止。


3. 垃圾回收机制  
- GC定义  
每一种类型GC使用一个GcSpec结构体来描述，它的定义如下所示：  
    ```
    struct GcSpec {  
      /* If true, only the application heap is threatened. */  
      bool isPartial;  
      /* If true, the trace is run concurrently with the mutator. */  
      bool isConcurrent;  
      /* Toggles for the soft reference clearing policy. */  
      bool doPreserve;  
      /* A name for this garbage collection mode. */  
      const char *reason;  
    };  
    ```  
    GcSpec结构体的各个成员变量的含义如下所示：

    isPartial: 为true时，表示仅仅回收Active堆的垃圾；为false时，表示同时回收Active堆和Zygote堆的垃圾。

    isConcurrent: 为true时，表示执行并行GC；为false时，表示执行非并行GC。

    doPreserve: 为true时，表示在执行GC的过程中，不回收软引用引用的对象；为false时，表示在执行GC的过程中，回收软引用引用的对象。

    reason: 一个描述性的字符串。  
    
- GC类型  
    Davlik虚拟机定义了四种类的GC，如下所示：  
    ```
    /* Not enough space for an "ordinary" Object to be allocated. */  
    extern const GcSpec *GC_FOR_MALLOC;  
      
    /* Automatic GC triggered by exceeding a heap occupancy threshold. */  
    extern const GcSpec *GC_CONCURRENT;  
      
    /* Explicit GC via Runtime.gc(), VMRuntime.gc(), or SIGUSR1. */  
    extern const GcSpec *GC_EXPLICIT;  
      
    /* Final attempt to reclaim memory before throwing an OOM. */  
    extern const GcSpec *GC_BEFORE_OOM; 
    ```  
    它们的含义如下所示：

    GC_FOR_MALLOC: 表示是在堆上分配对象时内存不足触发的GC。

    GC_CONCURRENT: 表示是在已分配内存达到一定量之后触发的GC。

    GC_EXPLICIT: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。

    GC_BEFORE_OOM: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC  
    
- 触发GC条件  
    1. 分配对象内存失败，执行GC_FOR_MALLOC
    2. 已分配内存达到一定量之后触发GC_CONCURRENT
    3. 显式调用Runtime.getRuntime().gc()，执行GC_EXPLICIT
    4. 准备抛OOM异常之前，执行GC_BEFORE_OOM
