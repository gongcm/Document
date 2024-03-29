# 内存

说到内存就必须从mmu开始讲起，对于linux内存一般都是采用物理地址映射到虚拟地址，
通过虚拟地址来管理物理地址，所以就需要分为以下几个步骤学习：
  1. mmu 如何建立虚拟到物理地址映射？
  2. linux 内存如何管理？

## MMU 内存管理

MMU 是一种硬件，用来实现物理地址到虚拟地址的转换。

通过内核源码中了解到对于不同的虚拟地址空间内核建立了不同级别的映射方式，PGD/PUD/PMD/PTE
等来保存虚拟和物理地址之间的关系。并且规定了最小管理单元页，一般内核页大小为4K。

内存访问一般： 基地址 + 偏移量，对于多级页表项也是。

1. 伙伴系统(Buddy system)

2. slab系列

## 内存分配

1. kmalloc
  分配连续的物理内存.
  最大分配内存 32M，基于slab缓存分配。

  注意： 容易造成内存碎片化

2. vmalloc
  分配非连续的物理内存大小

3. kmem_cache_create 和 kmem_cache_alloc
  分配object size的大小的slab缓存，然后在从slab 缓存中分配object size大小的内存。

4. alloc_pages

  分配连续页的内存。

## 内存回收 LRU

内核存在一个内核线程对内存进行回收。
在系统内存不足时，LRU 算法回收内存非活跃的内存。

