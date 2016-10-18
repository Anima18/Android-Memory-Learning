### ART和Dalvik区别  
Dalvik是依靠一个Just-In-Time (JIT)编译器去解释字节码。开发者编译后的应用代码需要通过一个解释器在用户的设备上运行，这一机制并不高效，但让应用能更容易在不同硬件和架构上运行。  

ART则完全改变了这套做法，在应用安装时就预编译字节码到机器语言，这一机制叫Ahead-Of-Time (AOT）编译。在移除解释代码这一过程后，应用程序执行将更有效率，启动更快。  

ART优点：  
1. 系统性能的显著提升。
2. 应用启动更快、运行更快、体验更流畅、触感反馈更及时。
3. 更长的电池续航能力。
4. 支持更低的硬件。

ART缺点：
1. 更大的存储空间占用，可能会增加10%-20%。
2. 更长的应用安装时间。

总的来说ART的功效就是“空间换时间”。  

### ART内存管理
1. 内存管理
![image](http://img.blog.csdn.net/20150104012422401?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  

    1. ART运行时堆划分为四个空间，分别是Image Space、Zygote Space、Allocation Space和Large Object Space。其中，Image Space、Zygote Space、Allocation Space是在地址上连续的空间，称为Continuous Space，而Large Object Space是一些离散地址的集合，用来分配一些大对象，称为Discontinuous Space。  
    
    2. 在Image Space和Zygote Space之间，隔着一段用来映射system@framework@boot.art@classes.oat文件的内存。而Image Space空间就包含了那些需要预加载的系统类对象。如果系统没有升级，那么以后每次系统启动只需要将文件system@framework@boot.art@classes.dex直接映射到内存即可，省去了创建各个类对象的时间。  
    
    3. Zygote Space和Allocation Space与Dalvik虚拟机垃圾收集机制中的Zygote堆和Active堆的作用是一样的。Zygote Space在Zygote进程和应用程序进程之间共享的，而Allocation Space则是每个进程独占的。 
    
    
2. 创建对象  
![image](http://img.blog.csdn.net/20141203015902705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
可以发现，ART运行时和Dalvik虚拟机为新创建对象分配内存的过程几乎是一模一样的，它们的区别仅仅是在于垃圾收集的方式和策略不同。


3. 垃圾回收机制
