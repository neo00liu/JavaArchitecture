<!-- TOC -->

- [垃圾收集器与内存分配策略](#垃圾收集器与内存分配策略)
    - [1.概述](#1概述)
    - [2.哪些内存需要回收](#2哪些内存需要回收)
        - [2.1 如何确定对象是否需要回收？](#21-如何确定对象是否需要回收)
            - [2.1.1 引用计数法](#211-引用计数法)
            - [2.1.2 可达性分析算法](#212-可达性分析算法)
        - [2.2 引用](#22-引用)
        - [2.3 生存还是死亡](#23-生存还是死亡)
        - [2.4 回收方法区](#24-回收方法区)
    - [3.垃圾回收算法](#3垃圾回收算法)
        - [3.1 标胶-清除算法](#31-标胶-清除算法)
        - [3.2 复制算法](#32-复制算法)
        - [3.3 标记-整理算法](#33-标记-整理算法)
        - [3.4 分代收集算法](#34-分代收集算法)
    - [4.HotSpot的算法实现](#4hotspot的算法实现)
        - [4.1 枚举根节点](#41-枚举根节点)
    - [5.垃圾收集器](#5垃圾收集器)

<!-- /TOC -->

# 垃圾收集器与内存分配策略
##  1.概述
垃圾回收(Grabage Collection,GC):        
1. 哪些内存需要回收？
2. 什么时候回收？
3. 如何回收？

**垃圾回收基本概念**   
java内存运行时区域中：程序计数器，虚拟机栈，本地方法3个区域生命周期同线程相同。这几个区域的内存分配和回收具有确定性，在这几个区域内就不需要多考虑回收的问题，因为线程结束或方法结束，内存就自然回收了。     
java堆和方法区则不一样，只有在程序运行时才能知道会创建哪些对象，这部分内存的分配和回收都是动态的，垃圾收集器所关注的就是这部分内存。        

所以：**java堆和方法区中的内存需要回收**。      

##  2.哪些内存需要回收   
## # 2.1 如何确定对象是否需要回收？
这个即判断对象的死活，需要回收死去的对象。      
通常有以下几种方法确定对象的死活：
1. 引用计数法
2. 可达性分析法

## ##  2.1.1 引用计数法
----**WHAT1:什么是引用计数法**（JVM中没有采用此方法）----      
给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。       

----**WHY1:为什么JVM不用引用计数法**----        
最主要的原因是它很难解决对象之间相互循环引用的问题。        


## ##  2.1.2 可达性分析算法
----**WAHT2:什么是可达性分析算法**----      
通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到**GC Roots**没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。     

----**WAHT2:什么是GC ROOTS**----        
虚拟机栈中引用的对象。      
方法区中类静态属性引用的对象。      
本地方法栈中JNI（即一般说的Native方法）引用的对象。     


## # 2.2 引用 
----**WHAT2:什么是引用**----     
JDK1.2之前：           
如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用。     
JDK1.2以后：        
引用分为强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）、虚引用（Phantom Reference）4种，这4种引用强度依次逐渐减弱。

----**WHY2:为什么出现引用的强弱之分？**----     
因为我们可能会出现这种情况：我们希望在内存空间中进行垃圾回收时会考虑剩余空间的容量，若剩余空间还足够那么久可以保存那些不太需要回收的对象，若剩余空间紧张则回收一切可以回收的对象，这时就可以根据引用的强弱来判断。  

**强引用**    
就是指在程序代码中普遍存在的，类似“Object obj = new Object()”这类引用，只要强引用还存在，GC就不会回收这些对象。     

**软引用**      
用来描述一些还有用但非必须的对象。对于软引用关联的对象，在系统将要发生内存溢出之前，就会把这些对象进行二次回收。SoftReference类来实现。     

**弱引用**      
也是用来描述非必须对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次GC发生之前。WeakReference类来实现。   

**虚引用**      
是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。      


## # 2.3 生存还是死亡
可达性分析算法中不可达的对象，至少要经历两次标记过程：      
1. 第一次标记并且进行一次删选，删选的条件是此对象是否有必要执行finalize()方法。
    1. 当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，这时都会被视为“没必要执行”。
    2. 有必要执行finalize()方法，gc时就会去执行其finalize()方法。
2. 第二次进行标记时，对象将被移除“即将回收”的集合。         


## # 2.4 回收方法区   
Java虚拟机规范中确实说过可以不要求虚拟机在方法区实现垃圾收集，而且在方法区中进行垃圾收集的“性价比”一般比较低：在堆中，尤其是在新生代中，常规应用进行一次垃圾收集一般可以回收70%～95%的空间，而永久代的垃圾收集效率远低于此。        

永久代的垃圾收集主要回收两部分内容：        
**废弃常量**    
**无用的类**        


**回收废弃常量**        
与回收java堆中的对象非常类似。当一个常量没有再被引用时，这时也正好发生了内存回收，而且必要的话，那么此常量将会被回收。常量池中的其他类，方法、字段的符号引用也是这样。      

**回收无用的类**        
首先需要判断类的无用，需要满足是哪个条件：      
1. 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。
2. 加载该类的ClassLoader已经被回收。
3. 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。


##  3.垃圾回收算法       
## # 3.1 标胶-清除算法    
分为标记和清除两个阶段：    
首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。        

**缺点**        
1. 效率问题，标记和清除两个过程的效率都不高
2. 空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

## # 3.2 复制算法     
为了解决效率问题，一种称为“复制”（Copying）的收集算法出现了，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。 

**优点**        
内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。   
**缺点**             
算法的代价是将内存缩小为了原来的一半。  

**改进的复制算法**      
现在的商业虚拟机都采用这种收集算法来回收新生代。将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。

HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%（80%+10%），只有10%的内存会被“浪费”。当然，98%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活，当Survivor空间不够用时，需要依赖其他内存（这里指老年代）进行分配担保（Handle Promotion）。


## # 3.3 标记-整理算法        
----**WHEN1:什么时候使用标记整理**----          
复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。     

----**WHERE:此算法用在哪里**----            
老年代中一般采用此算法。

----**WHAT:什么是标记清理算法**----     
标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。       


## # 3.4 分代收集算法     
----**WHAT:什么是分代算法**----         
根据对象的存活周期，把对象分为新生代和老年代，然后根据各个代的特点采用适当的算法。一般新生代采用复制算法，而老年代采用标记-整理算法。         

当前商业虚拟机的垃圾收集都采用“分代收集”（Generational Collection）算法。
   

##  4.HotSpot的算法实现      
## # 4.1 枚举根节点       


##  5.垃圾收集器         
垃圾收集器是内存回收的具体实现。        
![垃圾收集器](http://img1.imgtn.bdimg.com/it/u=199418516,2033934188&fm=214&gp=0.jpg)        

