# JVM内存分配

## 堆基本结构

![img](https://gitee.com/dachuant/image/raw/master/picgo/01d330d8-2710-4fad-a91c-7bbbfaaefc0e.png)

目前主流的垃圾收集器都会采用分代回收算法，因此需要将堆内存分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法

> 大部分情况，对象都会首先在 Eden 区域分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 s0 或者 s1，并且对象的年龄还会加 1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。
>
> “Hotspot遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了survivor区的一半时，取这个年龄和MaxTenuringThreshold中更小的一个值，作为新的晋升年龄阈值”。
>
> 
>
> 经过这次GC后，Eden区和"From"区已经被清空。这个时候，"From"和"To"会交换他们的角色，也就是新的"To"就是上次GC前的“From”，新的"From"就是上次GC前的"To"。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，"To"区被填满之后，会将所有对象移动到老年代中。



## 堆内存常见分配策略

1. 对象优先在eden区分配

   > 当 eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC
   >
   > 如果Minor GC时，对象太大导致无法存入 Survivor 空间， **分配担保机制** 会把新生代的对象提前转移到老年代中去

2. 大对象直接进入年老代

   > 大对象就是需要大量连续内存空间的对象（比如：字符串、数组）

3. 长期存活对象将进入年老代

   > 如果对象在 Eden 出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间中，并将对象年龄设为 1，对象在 Survivor 中每熬过一次 MinorGC,年龄就增加 1 岁，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。
   >
   > “Hotspot遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了survivor区的一半时，取这个年龄和MaxTenuringThreshold中更小的一个值，作为新的晋升年龄阈值”。
   >

4. 主要GC区域

   > 针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：
   >
   > 部分收集 (Partial GC)：
   >
   > - 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
   > - 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
   > - 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。
   >
   > 整堆收集 (Full GC)：收集整个 Java 堆和方法区
   >
5. 触发full gc的情况主要有这几种：

（1）System.gc()方法的调用。此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定，但很多情况下它会触发 Full GC,从而增加Full GC的频率，也即增加了间歇性停顿的次数。强烈影响系建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过-XX:+ DisableExplicitGC来禁止RMI（Java远程方法调用）调用System.gc。

（2）老年代空间不足。老年代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出错误：java.lang.OutOfMemoryError: Java heap space 。为避免以上两种状况引起的FullGC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

（3）Permanet Generation空间满了。Permanet Generation中存放的为一些class的信息等，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出错误信息：java.lang.OutOfMemoryError: PermGen space 。为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存。如果发现统计数据说之前Minor GC的平均晋升大小比目前old gen剩余的空间大，则不会触发Minor GC而是转为触发full GC。

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

   > 总结
   >
   > - System.gc()方法的调用 
   > - 老年代空间不足
   > - 永生区空间不足



## 对象已死亡

> 堆中几乎放着所有的对象实例，对堆垃圾回收前的第一步就是要判断那些对象已经死亡（即不能再被任何途径使用的对象）。

![image-20201015002812213](https://gitee.com/dachuant/image/raw/master/picgo/image-20201015002812213.png)

## 垃圾回收算法

1. 标记-清除法

2. 复制算法

3. 标记-整理法

4. 分带收集算法

   > 当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。
   >
   > **比如在新生代中，每次收集都会有大量对象死去，所以可以选择复制算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**

## 垃圾收集器

---

参考

[JVM垃圾回收](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/JVM%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6)



