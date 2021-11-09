1. 什么是TCMalloc？

   Google开发的内存分配算法库，最开始作为Google性能工具库[perftools](https://github.com/gperftools/gperftools)的一部分，用于替代传统的malloc内存分配函数。

   具有减少内存碎片，适用于多核场景，更好的并行性支持等特性。

   其中TC是Thread Cache的缩写。

   提供了很多优化：

   - 采用固定大小的page执行内存获取、分配等操作【与Linux物理内存页的划分的逻辑是否具有类似特性？】
   - 采用固定大小的Span（如8KB、16KB）用于特定大小对象的内存分配，简化内存获取和释放等操作
   - 利用缓存常用对象提升获取内存的速度
   - 支持基于线程或CPU设置缓存大小
   - 支持基于线程设置缓存分配策略，减少多线程之间锁竞争

2. 概览

![img](https://img2020.cnblogs.com/blog/650581/202010/650581-20201024150152393-1271038616.png)

Front-end：内存缓存，提供快速分配和重分配内存，包含Per-thread cache和Per-CPU cache

Middle-end：为Front-end提供缓存，即当Front-end缓存内存不够用时，从Middle-end申请内存，核心是Central free list

Back-end：负责从操作系统获取内存，并给Middle-end提供缓存使用，主要涉及Page Heap

从整体上看，TCMalloc将整个虚拟内存空间划分为n个同等大小的Page，若干个连续的page连接在一起组成一个Span

Page Heap 负责向操作系统申请内存，申请的span可能只有1个page，也可能有多个page

3. 概念解析

   **Page**：操作系统对内存管理的单位，但TCMalloc中Page大小是操作系统中页的倍数关系（默认为8KB）

   **Span**：Page Heap中管理内存页的单位，由一组连续的Page组成，多个相同大小的Span用链表来管理

   **ThreadCache**：每个线程独立拥有的cache，一个cache包含多个空闲内存链表（size classes），每个链表有各自的object（同一个size class下object大小相同）

   ![img](https://pic4.zhimg.com/80/v2-05a8740554bedf4dc0a6912c6e8551db_1440w.png)

   **CentralCache**：当ThreadCache内存不足时，提供内存给ThreadCache，当ThreadCache内存过多时，可以放回CentralCache。保持的是空闲块链表，链表数量与ThreadCache数量相同。

   CentralCache中的CentralFreeList负责从Page Heap取出部分Span，然后按照预定大小将其拆分为固定大小的object，在需要时提供给TreadCache使用

   **PageHeap**：CentralCache内存不足时，可以从PageHeap获取Span，然后将Span划分成object。保存若干个链表，链表中保存的是Span。Page Heap申请内存时按照页申请，管理已分配的Page的基本单位是Span。

   ![img](https://img2020.cnblogs.com/blog/650581/202010/650581-20201024150307005-574478853.png)

   ![img](https://img2020.cnblogs.com/blog/650581/202010/650581-20201024150319921-1498446946.png)

   ![img](https://img2020.cnblogs.com/blog/650581/202010/650581-20201024150338957-130722097.png)

4. 工作原理

   （1）小对象内存分配（小于256K）

   TCMalloc会根据申请内存大小映射到某个size-class中

   申请0-8个字节的大小时，会被映射到size-class1中，分配8个字节大小

   申请9-16个字节大小时，会被映射到size-class2中，分配16个字节大小

   ...

   当ThreadCache中free list为空时，从CentralCache的CentralFreeList中获取若干个object到ThreadCache对应的size-class中，然后取出一个object返回

   当CentralFreeList为空时，从PageHeap申请一连串由Span组成的页面，并将申请的页面切分为一系列的object后，再将部分object转移给ThreadCache

   当PageHeap不够用时，则向操作系统申请内存

   多级缓存，无须加锁，速度较快

   （2）大对象内存分配

   此时不再通过ThreadCache分配，而是由Page Heap直接分配，多个连续的Page组成一个Span，在Span中记录起始Page的编号，以及Page的数量。

   ![img](https://pic2.zhimg.com/80/v2-1fd6397abdf7d40799d6e1bd4435de91_1440w.png)

   依旧使用多种定长Page的方式实现变长Page的分配，初始时只有128Page的Span，如果此时需要分配一个Page的Span，则将128Page的Span分裂成1+127，再将127记录下来。Span回收时需要考虑合并问题，否则会只剩下很小的Span即外部碎片。

   即涉及到合并相邻Page减少外部碎片，内存申请时，需要分裂Span，内存释放时需要合并Span。

   这需要解决两个问题

   【1】如何知道前后的Span在哪里？

   因为Span中记录了起始Page，也就完成了Span到Page到映射，所以只需要解决从Page到Span到映射就可以知道前后的Span是什么了【即PageMap】。

   最简单的一种方式，用一个数组记录每个Page所属的Span，Page ID作为数组索引。【Page较少时空间浪费严重】

   使用RadixTree（compact trie，保证最坏O(k)，32位系统使用2层RadixTree，64位系统使用3层 RadixTree），使用较少的空间开销+可以接受的速度来完成。

   【2】如何实现从地址到Span到映射？

   实现时，使用一定空间换取时间即减少层数，每层是一个数组，用一个地址的前1/3bit索引数组，剩下的bit对下一层进行寻址。

5. 使用隐式FreeList进行对象分配：链表指针直接分配在待分配内存中，不需要额外开销

   ![img](https://pic2.zhimg.com/80/v2-8627f1c08819b6c8bd03d0b74935ba19_1440w.png)

   分配定长记录【分配变长记录可以归结为分配多种定长记录中】

   ![img](https://pic2.zhimg.com/80/v2-2c7c35cc567510ab8187b18297217b81_1440w.png)

定长记录的大小并非按照2的幂级数划分，这是为了避免比如说分配65字节实际上需要分配128字节造成接近50%的内存碎片。一共创建了86个size class