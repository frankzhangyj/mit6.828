# Lab 2: Memory Management

## Part 1: Physical Page Management

## ==虚拟内存注意点：==

> - **虚拟内存空间可以大于内存空间大小**(也可以映射到外存) **每个进程拥有一个独立的虚拟内存空间** 确保可以连续使用空闲空间(若没有虚拟内存 那么进程在申请一个连续空闲空间很难)
>
> - **每个进程都有各自独立的4G 字节的虚拟地址空间**。4G的进程空间分为两部分，**0~3G-1 为用户空间**，**3G~ 4G-1 为内核空间**。
>
> - 给每个进程分配一个4G(对32位系统来说)的虚拟地址空间。进程直接操作虚拟地址空间，**读写数据时，才给它调拨物理存储器。**
>
>
> - **用户程序中使用的都是虚拟地址空间中的地址，永远无法直接访问实际物理地址。**
>
> - 一个 x86 页表就是一个包含 2^20  (1,048,576）条*页表条目*（PTE）的数组(虚拟内存大小为2^32B 即4GB)  **高10位为页目录共2^10个** 每个页目录对应一个页表(**次高10位为页表** **总共也就是2^10 * 2^10**) 2^20  每个页对应的**物理页框**大小为**4KB** **所以总共就占2^20 * 2^12 = 4GB** 所以**就可以将虚拟内存通过页表映射到物理内存的所有位置**
> - **==一个页表具有2^10个PTE 一个PTE对应一个物理页框 一个物理页框占4KB 其中用来存储指令和数据==**
> - ==**虚拟地址中的一个页也就是一个PTE对应一个pageinfo结构体指针 一个pageinfo结构体存放物理地址 读取权限等信息 占8B**==

- **从虚拟地址到物理地址转换过程**：
- 1. 需要先通过**高10位**找到对应信息在page directory页目录中的**PDE** 用其中的PPN确定一个**page table页表** 
  2. 然后再用虚拟地址**次高10位**在找到的page table中找到其对应的**PTE** 用其中的**PPN**确定一个**物理地址前20位** 
  3. 将之前在**page table中找到的PPN** 和 **虚拟地址后12位**组合 即可组成一个**物理地址**


<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240130202253860.png" alt="image-20240130202253860" style="zoom:50%;" />

#### **Exercise 1.**

 **补充boot_alloc()         mem_init()  (only up to the call to check_page_free_list(1))                                               	   page_init()          page_alloc()           page_free() 这几个函数**

- ==**boot_alloc()**== 这个函数类似于malloc函数 用于开辟一段参数大小的空间 返回这个开辟空间的首地址

利用ROUNDUP这个宏来开辟

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240130210632740.png" alt="image-20240130210632740" style="zoom:50%;" />

- ==**mem_init()**== 中的check_page_free_list(1)之前只有一处需要补充：

分配一个存放内核代码虚拟地址的数组 共npages个页 每个单位都是一个结构体 里面有隐射物理地址的信息PageInfo

然后将所有空间中的地址初始化为0

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240130212529744.png" alt="image-20240130212529744" style="zoom: 67%;" />

page_init() 根据不同位置的页是否空闲来进行初始化 0表示空闲 1表示被占用

1、[0, 0]对应于第一个页，占用内存4KB，物理地址实际上是[0, 4KB]。这部分内存是被占用不能分配的，用来保存real-mode IDT和BIOS。

2、[1, npages_basemem)对应了接下来的一段空闲内存，映射到物理地址是[PGSIZE, npages_basemem * PGSIZE)。

3、[IOPHYSMEM/PGSIZE, EXTPHYSMEM/PGSIZE).由于地址的连续性，所以IOPHYSMEM==npages_basemem * PGSIZE。注意IO空洞，绝对不能使用。

4、[EXTPHYSMEM/PGSIZE, ...)这一区间比较麻烦。有些内存被占用，有些空闲。

利用 ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE得到额外内存中非空闲的页大小 其中boot_alloc(0)得到额外内存中的空闲空间起始地址 KERNBASE为虚拟地址转换位物理地址的差值

![image-20240130224418053](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240130224418053.png)

- ==**page_alloc()**== 将page_free_list中的一个空闲节点取出 然后将头节点向后移动一个 并将取出的空闲页对应的物理地址赋值为0

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201102135487.png" alt="image-20240201102135487" style="zoom:67%;" />

- ==**page_free()**== 将参数节点添加到page_free_list中 头插法

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201102659661.png" alt="image-20240201102659661" style="zoom:67%;" />

## Part 2: Virtual Memory

### Virtual, Linear, and Physical Addresses

#### **Exercise 2.** 

**了解虚拟地址通过查询页目录和页表找到映射关系 进而找到物理地址** 

低12位是页偏移，然后高10位是页表的偏移，再高10位是页目录的索引。 每个页目录有2^10条PTE ；每个页表也有2^10条PTE 所以共2^10 * 2^10 = 2^20条PTE 每个PTE都有一个20位的PPN对应物理地址中的高20位

虚拟地址由高20位Selector和低12位Offset组成 由段处理器将Virtual地址转换为Linear地址 再由页处理器将Linear地址转换为Physical地址

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201104543051.png" alt="image-20240201104543051" style="zoom:67%;" />

在内核加载到内存之前 BIOS设置了一个GDT 但是段基地址设置为0 所以Virtual地址和Physical地址完全重合 没有经过Linear地址

之后会将物理地址`[0, 4MB)`映射到了虚拟地址`[0xf0000000, 0xf0000000 + 4MB)`的地方了。相当于建立了一个页表。

#### **Exercise 3.** 

**使用 QEMU 监视器中的 xp 命令和 GDB 中的 x 命令检查相应物理和虚拟地址的内存，并确保看到相同的数据。**

- 如何进入qemu monitor:

  首先在lab目录下make qemu启动qemu

  然后用鼠标点击屏幕后先点击ctrl+a(鼠标会消失) 然后输入c即可进入monitor

  ![image-20240201120825741](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201120825741.png)

- 退出qemu类似进入monitor ctrl+a后 按下x即可退出

**在qemu monitor中所采用的地址是物理地址 使用xp 命令查看物理地址为0x100000处后5个指令**

![image-20240201121050979](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201121050979.png)

**而gdb对程序员可见的地址是逻辑地址，因此在使用gdb的命令时，需要在高位增加字段：**

使用x 命令查看虚拟地址为0xf0100000处的后5个指令

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201121329943.png" alt="image-20240201121329943" style="zoom:67%;" />

在jos内核中**C程序运行在虚拟地址上** 需要通过MMU来转换成物理地址，通常会有两种指针类型(但他们都是unit32_t整型): **uintptr_t**(不透明类型 即**虚拟地址指针**) 和 **physaddr_t**(**物理地址指针**)

#### **Question**1.

1. 以下x变量是哪种类型的指针 

   因为在C程序中的指针类型都是虚拟地址指针 而且对physaddr_t的强制类型转换时，会将物理地址视作虚拟地址来进行类型转换（这样就出错了）

   ```
   	mystery_t x;  // uintptr_t
   	char* value = return_a_pointer();
   	*value = 10;
   	x = (mystery_t) value;
   ```

内核对应的物理地址需要加上0xf0000000后映射对应的虚拟地址

### Page Table Management

#### **Exercise 4.** 

**补全kern/pmap.c中的pgdir_walk() boot_map_region() page_lookup() page_remove() page_insert()这几个函数**

注：

1. `page2pa`函数将给定的物理页面指针转换为对应的物理地址。它接受一个`struct PageInfo`类型的指针参数，该结构表示一个物理页面的信息，包括物理地址等。`page2pa`函数会提取出物理页面结构中的物理地址，并返回该物理地址。
2. `pa2page`函数则相反，它将给定的物理地址转换为对应的物理页面指针。它接受一个物理地址作为参数，并根据物理地址查找或计算得到对应的物理页面结构的指针。这样，可以通过该指针来访问和操作相应的物理页面。

**==pgddir_walk()==** 

- 这个函数是来根据参数va这个虚拟地址先通过高10位先找到其在页目录中的偏移量 然后通过计算得到的page dir entry找到页目录中对应的页表
- 如果这个页表不存在 那么就创建一个
- 然后再通过va次高10位通过计算得到其在页表中的偏移量 
- 返回这个页表中偏移量对应的地址即page table entry(PTE)

### ![image-20240201171805201](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201171805201.png)

**==boot_map_region()==** 

这个函数主要是将虚拟地址va通过其在页表中的偏移量 建立页表保存的映射 来映射物理地址

扫描一遍区间[va, va+size)，每一个虚拟地址通过页表映射到物理地址空间[pa, pa+size)上，这样页的地址可以通过二级页表的页偏移找到 而它保存的数值就变成了物理地址。这样就建立了逻辑地址到物理地址的映射。(PGSIZE 就是一页所占的大小)

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201173314725.png" alt="image-20240201173314725" style="zoom:67%;" />

**==page_lookup()==** 

这个函数返回指定的**虚拟地址va通过映射关系得到的物理页帧PPN**(PTE_ADDR()函数从PTE中提取物理页框地址)  

利用pgdir_walk()查找到page table entry之后，将找到的entry放到pte_store中，然后返回PTE中的PPN对应的物理页指针

PTE_ADDR(pte): 提取PTE或 PDE中的物理页帧PPN
    return pte & 0xFFFFF000 // 提取出高20位 屏蔽低12位

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201175011258.png" alt="image-20240201175011258" style="zoom:67%;" />

**==page_remove()==**

**取消虚拟地址与物理地址的关联**

将虚拟地址对应页中的信息全部清空 并且将TLB中的对应映射关系删除 最后将找到虚拟地址对应的PTE设置为0代表为空

==**注意**==: 释放了虚拟内存与物理内存的映射之后。**并没有直接把相应的物理内存直接放到链表里面**。这主要是因为，**可能存在多个虚拟内存映射到同一个物理内页面的情况。**虽然这个虚拟内存不在与这个物理内存发生联系了。但是其他的虚拟地址还是有可能继续与这个物理内存关联并且还在使用的。

**TLB**: 是一种高速缓存机制 将虚拟地址映射到物理地址。当CPU访问虚拟地址时，首先检查TLB中是否存在对应的转换结果。如果存在，TLB会直接提供物理地址，从而避免了访问页表或段表的延迟。如果TLB中不存在对应的转换结果，CPU会继续访问页表或段表来进行转换，并将结果存入TLB以供以后的访问。(类似Inoodb存储引擎中的插入缓冲 将频率高的直接存到缓冲中 cpu查询时先在缓冲中找 然后没找到再去磁盘中找 然后将这个数据插入到缓冲中)

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201181144944.png" alt="image-20240201181144944" style="zoom:67%;" />

**==page_insert()==**

**这个函数将一个物理页面与给定的虚拟地址建立映射，并设置相应的权限**

首先根据虚拟地址得到相应的PTE 然后将这个物理页面中计数信息加一 表示新加一个映射关系

如果这个虚拟地址已经具有映射关系 那么就将之前的页通过page_remove()删除

将物理页面 `pp` 的地址与给定的权限 `perm` 设置到页表项 `entry` 中。这里是page2pa将pp物理页转换为对应的物理页地址 然后修改这个地址

并且将相应的权限 `perm` 设置到页目录项 `pgdir[PDX(va)]` 中。

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201212342825.png" alt="image-20240201212342825" style="zoom:67%;" />

## Part 3: Kernel Address Space

JOS将处理器32位线性地址空间分为用户部分和内核部分，下半部分为用户 上半部分为系统内核 分界线由ULIM任意定义 为内核保留了大约256MB的虚拟地址空间 所以在lab1中内核的物理地址到虚拟地址映射需要加0xf0000000 保证为用户留有足够空间

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201214350376.png" alt="image-20240201214350376" style="zoom: 50%;" />

### Permissions and Fault Isolation

> `UTOP`代表用户地址空间的最高地址。它定义了**用户程序可以访问的最高虚拟地址**。
>
> `ULIM`代表用户地址空间的上限。它是**用户程序可以访问的最高有效内存地址**。在此地址之上的内存区域被视为无效，任何尝试访问该区域的操作都将被操作系统阻止。

所以在[UTOP, ULIM) 这个范围内的地址空间 内核和用户都可以访问 但不能写入这个地址空间 这个地址空间向用户公开某些只读的内核数据结构

### Initializing the Kernel Address Space

#### **Exercise 5.**

在mem_init()将内核虚拟地址映射到物理地址

**将[UTOP, ULIM)这部分中的空间映射到物理地址中 设置为只读权限** UPAGES是需要映射的起始虚拟地址 PTSIZE是页数目 PADDR是得到pages的物理地址 PTE_U是只读

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201220643500.png" alt="image-20240201220643500" style="zoom:67%;" />

**将[KSTACKKTOP - KSTKSIZE, KSTACKTOP)这部分内核栈虚拟地址映射到从bootstack开始的物理地址中 设置为只能写入**

==注== 在[KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE)范围内的虚拟地址是不会映射到物理地址中的 **设置这部分空间是为了避免内核栈溢出覆盖内存而设置的** 被称为"guard page"

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201222721081.png" alt="image-20240201222721081" style="zoom:67%;" />

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201221429100.png" alt="image-20240201221429100" style="zoom:67%;" />

**将[KERNBASE, 2^32)这部分内核空间映射从0开始的物理地址中 设置为只写入权限**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240201222812986.png" alt="image-20240201222812986" style="zoom:67%;" />

**由此也可以看出虚拟地址中是连续的 但是映射到物理地址中就不一定连续了**

#### **Question2**.

1. **填写下表**： 这就是一个页目录 一共有2^10个PDE 每个PDE都有4MB大小的PTE

![image-20240202114655261](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240202114655261.png)

2. **将内核和用户存放在同一个虚拟内存中，但用户不能读写内核中的信息所采用的是什么保护机制？**

   > 虚拟内存通过**ULIM和UTOP**分段了。其中**[0, UTOP]**只允许**用户进程读写**，**(UTOP, ULIM]是用户和内核都可读**，**(ULIM, 4GB)只允许内核读写**。他们之间的访问控制通过**permission**位来进行控制。

3. **这个操作系统支持的最大物理内存是多少？为什么？**

   > 也就是在问所有页表中的页对应多少的物理内存(也就是**所有的PTE对应的所有物理页框所占的大小**)
   >
   > 由上面的虚拟地址示意图可以知道 虚拟地址用**4MB**的空间来存储所有的页的**PageInfo结构体信息** 每个结构体的大小为**8B**，所以一共可以存放**512KB个PageInfo结构体** 所以一共可以出现**512KB个物理页**，每个物理页大小为4KB，自然总的物理内存占**2GB**。

   ![image-20240202180819736](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240202180819736.png)

4. **如果我们现在的物理页达到最大，那么管理这些内存所需要的额外空间开销有多少？**

   > pageinfo需要4MB 
   >
   > 页目录共2^10个 每个都是一个指针占4B 所以所有页目录需要占4KB
   >
   > 页表共2^10 * 2^10个 每个页表项也是一个指向pageinfo的指针占4B 所以就是4MB
   >
   > 所以总共占4MB + 4MB + 4KB

5. **重新访问kern/entry.S和kern/entrypgdir.C中的页表设置 为什么EIP既可以在高位KERNBASE上运行 又可以在低位上运行？**

   > 在entrypgdir.C中有这样一段描述：在加载内核到内存中前会将虚拟地址中**[KERNBASE, KERNBASE + 4MB)和[0, 4MB)映射到物理地址(0, 4MB]处** 所以当eip在物理地址[0, 4MB)上时，它既在低位的[0, 4MB)，也在高位[KERNBASE, KERNBASE+4MB)。这是一个**过渡的状态**，过一会儿页索引会被加载进来而虚拟地址[0, 4MB)就被舍弃了。
   >
   > <img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240202225130383.png" alt="image-20240202225130383" style="zoom:67%;" />

对于challenage无力完成

**以上对于Lab2就总结完成！**
