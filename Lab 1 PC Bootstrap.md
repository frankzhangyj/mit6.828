# lab 1 

## Part 1: PC Bootstrap

### The PC's Physical Address Space

 PC 的物理地址空间通过硬连线具有以下总体布局:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230913215342024.png" alt="image-20230913215342024" style="zoom: 80%;" />

第一批 PC 基于 16 位 Intel 8088 处理器，只能寻址 1MB 物理内存。PC 的物理地址空间将从 0x00000000 开始，但以 0x000FFFFF 结束，而不是 0xFFFFFFFF

从 0x000A0000 到 0x000FFFFF 的 384KB 区域，该保留区域中最重要的部分是基本输入/输出系统（BIOS）BIOS 负责执行基本的系统初始化，例如激活显卡和检查安装的内存量。执行此初始化后，BIOS 从某个适当的位置（例如软盘、硬盘、CD-ROM 或网络）加载操作系统，并将机器的控制权传递给操作系统。

PC 架构师仍然保留了低 1MB 物理地址空间的原始布局，以确保向后兼容现有软件。因此，现代 PC 在物理内存中存在一个从 0x000A0000 到 0x00100000 的“漏洞”，将 RAM 分为“低端”或“常规内存”（前 640KB）和“扩展内存”（其他所有内存）。

### ROM BIOS

一些常用的操作系统以及cpu的专有名词

X86：这是对于Intel 8086及后续CPU产品的统一称呼（也称为**80x86** [[2\]](https://en.wikipedia.org/wiki/X86#cite_note-2)或**8086 系列**[[3\]](https://en.wikipedia.org/wiki/X86#cite_note-3)[ ）](https://en.wikipedia.org/wiki/Complex_instruction_set_computer)

X64：兼容X86CPU的64-bit CPU（也称为**x64**、**x86_64**、**AMD64**和**Intel 64**）

IA-32：Intel Architecture-32的缩写，是一种不向下兼容的Intel 32-bit CPU 架构

IA-64：Intel Architecture-64的缩写，是一种不向下兼容的Intel 64-bit CPU 架构

### ==在进行初次调试qemu时出现的问题:==

1. **先打开一个terminal进入到lab目录中 再进行make qemu-gdb 然后再重新打开一个新terminal进入到lab目录中 进行make gdb出现报错:**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916230926685.png" alt="image-20230916230926685" style="zoom: 67%;" />

**原因:**

为了避免出现一些问题，`gdb` 不会再主动的执行任何文件。

**解决方法:**

  	需要在用户目录下声明，`lab`这个仓库的 `.gdbinit` 文件是安全的

通过 `echo add-auto-load-safe-path /home/frank/6.828/6.828mit/lab/.gdbinit > ~/.gdbinit` 命令，解决这个报错问题。

**2.在进行make gdb时 qemu自动退出问题**

先执行`make qemu-gdb` 后执行`make gdb`时qemu自动退出:

![image-20230916231807585](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916231807585.png)

在使用`make gdb`的terminal中报错:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916232023816.png" alt="image-20230916232023816" style="zoom:67%;" />

**原因:** 

其中一个teminal中执行 `make qemu-gdb` 打开了qemu的gdb调试模式 qemu在处理器执行第一条指令之前将停止并等到GDB的调试连接 

另一个终端执行 `make gdb` 命令，该命令使用已提供的.gdbinit文件来设置GDB，.gdbinit文件确保GDB能在早期引导期间进行16位代码调试工作，并引导它附属到监听的qemu

**解决方法:**

先执行make gdb:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916233249556.png" alt="image-20230916233249556" style="zoom:67%;" />

然后等待一小会直到出现:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916233408269.png" alt="image-20230916233408269" style="zoom:67%;" />

然后在另一个terminal中执行make qemu-gdb 打开qemu的gdb调试模式 然后会在make gdb terminal中显示:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230916233730029.png" alt="image-20230916233730029" style="zoom:67%;" />

至此就可以调试qemu。

以下行：

```
[f000:fff0] 0xffff0: ljmp $0xf000,$0xe05b
```

是GDB对要执行的第一条指令的反汇编

- IBM PC 从物理地址 0x000ffff0 开始执行，该地址位于为 ROM BIOS 保留的 64KB 区域的最顶部。

- PC 以`CS = 0xf000`和`IP = 0xfff0`开始执行。

- 第一条执行的指令是`jmp`指令，它跳转到分段地址 `CS = 0xf000`和`IP = 0xe05b`。

  ```
  16 * 0xf000 + 0xfff0 # 十六进制乘以 16 
     = 0xf0000 + 0xfff0 # 简单——只需追加一个 0。
     = 0xffff0
  ```

  `0xffff0`是 BIOS 末尾 ( `0x100000` ) 之前的 16 个字节。这就是顶层64KB留给BIOS的空间。

### **1. 什么是CS和IP**

#### 实模式下的分段地址到物理地址转换:

CS和IP是8086CPU中两个关键的寄存器，它们指示了CPU当前要读取指令的地址。

CS : 代码段寄存器；IP : 指令指针寄存器。在8086机中，任意时刻，CPU将CS:IP指向的内容当作指令来执行。

地址加法器完成：物理地址 = 段地址 x 16 + 偏移地址(8086提供20位地址 在确定物理地址时将段地址左移4位加上偏移地址)

#### 保护模式下的分段地址到物理地址转换:

通过分段机制和分页机制相结合来实现分段地址到线性地址的转换，再通过页表将线性地址转换为物理地址。32位保护模式具有更大的内存寻址能力、内存保护和虚拟内存等功能。

#### 段存储器在实模式和保护模式下的区别：

在实模式中 段存储器可以用于定位内存中的物理地址 即段寄存器和偏移地址结合 CS（代码段寄存器），DS（数据段寄存器），SS（堆栈段寄存器），ES（附加段寄存器）

在保护模式下 段寄存器不再直接存储段的起始地址 而是存储一个描述符的选择子（Selector）选择子通过索引到全局描述符表（GDT）或局部描述符表（LDT）中的描述符，从而确定段的属性和位置。通过选择子和偏移地址的组合，可以计算出线性地址，然后再通过分页机制将线性地址转换为物理地址。

### exercise 2: 

**利用单步调试命令 si(step instruction) 进行调试操作**

在执行完`[f000:fff0] 0xffff0: ljmp $0xf000,$0xe05b`后跳转到物理地址为`0xfe05b`后进行`si`调试

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231204175646779.png" alt="image-20231204175646779" style="zoom:50%;" />

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231204175739986.png" alt="image-20231204175739986" style="zoom: 50%;" />

## Part 2: The Boot Loader

扇区是磁盘最小传送单元（粒度）每次读或写操作都必须一扇区（或多于一扇区）为大小且以扇区边界对齐好。

主引导扇区: 0 面 0 道 1 扇区 ROM-BIOS将读取主引导扇区的内容，将它加载到内存地址0x0000 : 0x7c00 处(也就是物理地址0x07c00) 对应的物理地址是 0x7c00 ~ 0x7dff

通常 主引导扇区的功能是继续从硬盘的其他部分读取更多的内容加以执行

然后用 `jmp` 指令把 CS:IP 设置到 0000:7c00，把控制权交到启动引导程序

启动引导程序由一汇编源程序（boot/boot.S）与一C语言程序（boot/main.c）组成

boot/boot.S:

- boot loader把模式从16位实模式切到了32位保护模式。因为只有在这种模式下软件才可以访问1MB+以上的内存空间。
- 在转换过程中使用Bootstrap GDT(引导全局描述符表)引导加载程序阶段使用的临时GDT，用于启动操作系统并进行保护模式的切换。GDT中的描述符定义了内存段的起始地址、大小、访问权限等信息.为了启动操作系统并切换到保护模式，引导加载程序需要设置一个最小化的GDT(Bootstrap GDT)，以确保正确的内存访问和保护模式的切换。
- 设置控制寄存器，进入保护模式
- 按照保护模式的内存寻址方式继续执行

(在boot/boot.S切换到保护模式后调用main)boot/main.c:

首先执行readsect函数，这个函数主要做了三件事情：

等待磁盘（waitdisk）：在控制器空闲和驱动器就绪同时成立时才会结束等待

输出扇区数目及地址信息到端口（out）：主要是是把扇区计数、扇区LBA地址等信息输出到端口1F2-1F6，然后将0x20命令写到1F7，表示要进行读扇区的操作。

读取扇区数据（insl）：从0x1F0端口连续读128个dword（即512个字节，也就是一个扇区的字节数）到目的地址。其中，0x1F0是数据寄存器，读写硬盘数据都必须通过这个寄存器。

### exercise 3: 

通过对比将**boot/boot.S** 和**boot/main.C**与 **obj/boot.asm**(这个文件记录了从0x7c00开始执行的所有命令的地址及其汇编指令 先从boot.S开始转换模式 然后进入bootmain函数 将加载引导扇区内容加载到内存中 然后运行内核)对比找到每个指令执行的地址，**bootmain函数目的是在磁盘的第二个扇区开头找到内核程序**

1. **处理器什么时候开始执行 32 位代码？究竟是什么原因导致从 16 位模式切换到 32 位模式？**

`movl    %eax, %cr0`  cr0寄存器用于控制处理器的各种特性和操作模式。这个指令将 `%eax` 寄存器的值复制到 CR0 寄存器中，以便对处理器的控制和配置进行相应的更改。

`# Jump to next instruction, but in 32-bit code segment.`

`# Switches processor into 32-bit mode.`  

`ljmp    $PROT_MODE_CSEG, $protcseg`  用一个长跳转来进入到32位的代码并能够执行32位的代码 通过 `ljmp` 指令的执行，程序将跳转到给定选择子和偏移量所指定的代码段，并在跳转时改变特权级别，以便在新的特权级别下执行代码。

2. **引导加载程序执行的*最后一条*指令 是什么，刚刚加载的内核的*第一条指令是什么？***

引导加载程序执行*最后一条*指令：`((void (*)(void)) (ELFHDR->e_entry))();` 即boot/main.c 中bootmain函数的最后一个代码

加载的内核的第一条指令：`movw    $0x1234,0x472                   # warm boot` 即kern/entry.S的第一条指令

3. **引导加载程序如何决定必须读取多少个扇区才能从磁盘获取整个内核？它在哪里找到这些信息？**

boot/main.c：

```
void bootmain(void)
{
        struct Proghdr *ph, *eph;
        // 这里把ELF文件的头读到内存里面。
        readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

        // is this a valid ELF?
        // 检查一下ELF头文件的magic是不是与指定的ELF的格式相等。
        if (ELFHDR->e_magic != ELF_MAGIC)
                goto bad;

        // load each program segment (ignores ph flags)
        // ELF header里面指明了第一个program section header的位置。
        // 也指明了最后一个位置在哪里
        // [ph, end_of_program_header)
        // 这里面表明了每个程序段的大小以及位置。
        ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;
        for (; ph < eph; ph++)
                // p_pa是需要被加载的地址。
                // p_memsz指的是需要的物理内存的大小
                // p_offset指的是在逻辑上相对于整个文件头的偏移量。
                // 虽然这里p_memsz表示的时候需要占用的内存的大小。
                // 实际上也是磁盘上需要读取的数据量的大小。
                readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

        // 当把引导扇区内容加载到内存里面之后。开始从这里开始执行第一条指令。
        ((void (*)(void)) (ELFHDR->e_entry))();

bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);
        while (1)
                /* do nothing */;
}

// pa表示的是内存地址开始放磁盘内容的地方。
// count表示需要读的内容的长度。
// offset表示相对程序文件头的偏移量。这个传进来的参数还没有考虑到
// kernel在磁盘便移量。
void readseg(uint32_t pa, uint32_t count, uint32_t offset)
{
        uint32_t end_pa;
        // pa是开始放内容的内存起始地址
        // end_pa是指末尾地址
        end_pa = pa + count;
        // 操作的时候，起始地址取一个512byte对齐的边界
        // 比如，如果起始地址addr =256, 那么内存开始放的地址就是addr = 0
        pa &= ~(SECTSIZE - 1);

        // 注意，这里把offset修正过了。由于第一个扇区放的是
        // boot loader。所以内存实际上是从第2个扇区开始放的。
        // 如果代码要更加友好一点。应该改成。
        // kernel_start_sector = 1;
        // offset = (offset / SECTSIZE) + kernel_start_sector;
        // 这样代码就比较容易理解了。
        offset = (offset / SECTSIZE) + 1;

        // 本来这里可以用不着一个扇区一个扇区读的。
        // 可以读得快一点的。linux kernel里面就是利用较
        // 复杂的代码来加速了这个读取过程。
        while (pa < end_pa) {
                // 这里把磁盘扇区，注意，这个时候offset表示的是哪个扇区
                // 把这个扇区的内容读到内存地址pa处。
                readsect((uint8_t*) pa, offset);
                // 读完之后，移动内存地址以及扇区。
                pa += SECTSIZE;
                offset++;
        }
}
```

简单而言，通过**ELF文件的头**(**program header**)中的信息得到每个程序段的开始和结束位置(**目标物理地址**)，通过一个循环 调用readseg函数 根据函数中的参数`uint32_t offset`确定读取的扇区位置 循环结束后 执行内核程序(**BIOS将引导扇区加载到 0x7c00内存中 然后将控制器转交给引导加载程序 然后引导加载程序运行内核**)引导加载器将内核拷贝到的低地址正是分页硬件最终会映射的物理地址。

### Loading the Kernel

```
objdump -h obj/kern/kernel 利用该命令查看内核可执行文件中所有部分的名称、大小和链接地址的完整列表
objdump -x obj/kern/kernel 显示所有可用的信息 例如程序头 部分 符号链接表
```

其中包括：

- `.text`：程序的可执行指令。

- `.rodata`：只读数据，例如 C 编译器生成的 ASCII 字符串常量。（不过，我们不会费心设置硬件来禁止写入。）

- `.data`：数据部分保存程序的初始化数据，例如使用`int x = 5 等初始化器声明的全局变量；`。

  引导加载程序:

  <img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213154537528.png" alt="image-20231213154537528" style="zoom: 67%;" />

  

  内核:

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231212180523682.png" alt="image-20231212180523682" style="zoom: 67%;" />

其中的VMA为链接地址(为该部分期望执行的内存地址) LMA为加载地址(为该部分加载到内存中的地址)

通常VMA和LMA是相同的(这是由于被加载进BIOS的时候，没有页表与段表可用。)，ELF对象中需要加载到内存中的区域是那些被标记为“LOAD”的区域。

program header是给加载程序方用的。section是给写程序的人以及与编译器看的。

`objdump -p obj/kern/kernel 查看程序头的详细信息`

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213153459021.png" alt="image-20231213153459021" style="zoom: 67%;" />

- filesz 这个程序段所在位置起始位置。注意，这里是相对于文件头而言。 ！！

  ### exercise 5: 

  **如果更改boot/Makerag文件中的-Ttext里面的起始地址。后面程序加载地址就会发生变化，运行报错。**

  <img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213170549869.png" alt="image-20231213170549869" style="zoom:67%;" />

  ### exercise 6: 

  **理解从boot loader到kernel的链接过程:**

  通过`objdump -h obj/kern/kernel`可以看到.text的链接地址（虚拟内存地址）VMA是f0100000，被映射到加载地址LMA是100000。

  通过objdump -h obj/boot/boot.out可以看到boot开始加载的地址是0x7c00，而内核入口调用在0x7d6b

  ​						下图为obj/boot.asm 文件中bootmain执行的最后一个指令 之后运行内核

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213155618069.png" alt="image-20231213155618069" style="zoom: 67%;" />

找到boot开始加载的地址是0x7c00，而内核入口调用在0x7d6b，在这之间boot中加载引导程序ELF信息肯定已经被装载到0x100000中了。

测试: 

在0x7c处设置断点 监视内存地址0x100000处值的变化

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213164046986.png" alt="image-20231213164046986" style="zoom: 67%;" />

在加载引导程序运行结束时 将其ELF信息加载到了内核起始地址0x100000处

![image-20231213164209473](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213164209473.png)

若把kern/kernel.ld文件中的内核入口信息更改，运行就会报错

kern/kernel.ld文件：**链接器脚本**，链接器 ld 将按照脚本内的指令链接多个.o 文件以生成可执行文件

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213164740096.png" alt="image-20231213164740096" style="zoom:67%;" />

更改后运行就会报错。

## Part 3: The Kernel

**在将内核加载到内存之前 由于比如页表没有建立起来 物理地址与虚拟地址完全重合**

kernel在编译的时候是并不知道会被加载到哪里的 通过链接的时候kern/kernel.ld链接脚本可以指定被加载到的物理地址。但是程序的入口地址仍然需要告知ELF。

在boot/main.c中执行最后一个指令跳转到内核的代码：

![image-20231224105536238](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231224105536238.png)

这里面`e_entry`就指向`_start`值。由于从`boot loader`跳转到的内核的时候，还在物理地址与虚拟地址完全重合的情况。并且也没有开启分页。所以这个时候必须在kern/entry.S里面把`_start`地址改造成物理地址

​												kernel/entry.S

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231224105305896.png" alt="image-20231224105305896" style="zoom: 67%;" />

这时逻辑地址0xf0100000就转换为了物理地址0x00100000(这个物理地址也是1M的位置 BIOS上面的地址空间)

### 	exercise 7: 

1.使用 QEMU 和 GDB 跟踪 JOS 内核并在`movl %eax, %cr0`. 检查 0x00100000 和 0xf0100000 处的内存。现在，使用 GDB 命令单步执行该指令stepi。再次检查 0x00100000 和 0xf0100000 处的内存。确保您了解刚刚发生的事情。

​													kern/entry.S

![image-20231224110732598](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231224110732598.png)

在分页前看0xf0100000地址时，内容为0，因为这部分虚拟地址是没有内容的

在分页后查看地址时，`0xf0100000`与`0x00100000`内容就完全一样了。这是因为把`[0, 4MB)`映射到了`[0xf0000000, 0xf0000000 + 4MB)`的地方了。相当于建立了一个页表。

开始跳转：

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231224111454003.png" alt="image-20231224111454003" style="zoom:67%;" />

- 1. CPU跑在物理地址空间上，而不是虚拟地址空间上。（尽管CS:IP会被翻译到真正的地址。）
- 1. C语言认为是自己是跑在虚拟地址空间。

通过跳转到虚拟地址中 cpu就可以通过页表取址 寻址到物理地址 c程序在虚拟地址上运行

2.*建立新映射后，*如果映射不到位将无法正常工作的 第一条指令是什么？注释掉`kern/entry.S``movl %eax, %cr0`中的 内容，跟踪它，看看你是否正确。

若将movl %eax, %cr0注释掉 那么后面通过页表寻址的指令就会报错

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231224112221079.png" alt="image-20231224112221079" style="zoom:67%;" />

那么在`movl $(bootstacktop), %esp`这里就立马出错了。无法将这个虚拟地址转换为实际的物理地址

### Formatted Printing to the Console

### **Exercise 8.**

**我们省略了一小段代码 - 使用“%o”形式的模式打印八进制数所需的代码。找到并填写此代码片段。对照16进制完成8进制**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240119173934369.png" alt="image-20240119173934369" style="zoom: 67%;" />

1. `解释printf.c`和 `console.c`之间的接口。`具体来说， console.c`导出什么函数 ？`printf.c`如何使用这个函数 ？

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240119180847267.png" alt="image-20240119180847267" style="zoom:50%;" />

调用关系：

```
cprintf -> vcprintf -> vprintfmt -> putch -> cputchar
```

printf.c中**cprintf**函数调用**vcprintf**函数(在printfmt.c文件中) 先利用putch函数输出普通字符 然后再在**vcprintf**中格式化 putch函数调用cputchar函数(console.c文件中)

```
console.c:
void
cputchar(int c)
{
	cons_putc(c);
}

console.c:
// output a character to the console
static void
cons_putc(int c)
{
	serial_putc(c);
	lpt_putc(c);
	cga_putc(c);
}

// 在显示之前 初始化 选定特定的屏幕 初始化的时候，需要设定光标的位置
static void
cga_init(void)
{
	volatile uint16_t *cp;
	uint16_t was;
	unsigned pos;

	cp = (uint16_t*) (KERNBASE + CGA_BUF);
	was = *cp;
	*cp = (uint16_t) 0xA55A;
	if (*cp != 0xA55A) {
		cp = (uint16_t*) (KERNBASE + MONO_BUF);
		addr_6845 = MONO_BASE;
	} else {
		*cp = was;
		addr_6845 = CGA_BASE;
	}

	/* Extract cursor location */
	outb(addr_6845, 14);
	pos = inb(addr_6845 + 1) << 8;
	outb(addr_6845, 15);
	pos |= inb(addr_6845 + 1);

	crt_buf = (uint16_t*) cp;
	crt_pos = pos;
}

// 将文字显示到屏幕上 还有随时移动的光标
static void
cga_putc(int c)
{
	// if no attribute given, then use black on white
	if (!(c & ~0xFF))
		c |= 0x0700;

	switch (c & 0xff) {
	case '\b':
		if (crt_pos > 0) {
			crt_pos--;
			crt_buf[crt_pos] = (c & ~0xff) | ' ';
		}
		break;
	case '\n':
		crt_pos += CRT_COLS;
		/* fallthru */
	case '\r':
		crt_pos -= (crt_pos % CRT_COLS);
		break;
	case '\t':
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		cons_putc(' ');
		break;
	default:
		crt_buf[crt_pos++] = c;		/* write the character */
		break;
	}

	// What is the purpose of this?
	// 一页写满，滚动一行。
	// crt_pos是命令行光标的位置，而CRT_SIZE是屏幕能够显示的双字节字符的数量
	if (crt_pos >= CRT_SIZE) {	// 如果光标到了屏幕结尾
		int i;
		// 把从第1~n行的内容复制到0~(n-1)行，第n行未变化
    	// 通过这一行代码完成了整个屏幕向上移动一行的操作。
		memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
		for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
		// 把最后一行清空
			crt_buf[i] = 0x0700 | ' ';
			// 光标位置来到n行开头
		crt_pos -= CRT_COLS;
	}

	/* move that little blinky thing */
	outb(addr_6845, 14);
	outb(addr_6845 + 1, crt_pos >> 8);
	outb(addr_6845, 15);
	outb(addr_6845 + 1, crt_pos);
}


```

2. 见上当显示页面溢出如何处理

3. 对于cprintf函数中的两个参数fmt 和 ap都有什么作用

​	<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240125103646065.png" alt="image-20240125103646065" style="zoom:50%;" />

例如：

```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

**这时cprintf中的 fmt 就是一个字符串 ap 就是一个链栈指向栈顶** (ap的数据类型是char *进行typedef后的va_list 其可以指向任一个地址)

在运行时会首先从高地址到低地址将各个参数地址压入栈中 也就是 z y x fmt(fmt现在是栈顶)

然后调用vprintfmt函数 先将fmt中%前面的字符通过putch函数输出

接着遇到%后判断%后具体是什么格式化输出(%c %d %s...)

找到ap的开始地址也就是x的地址然后通过调用不同case中的输出语句将每个ap对应的参数输出

**ap的作用实际上就是利用fmt里面的`%`依次把后面的类型提出来**

4. 以下代码输出什么 解释为什么

```
 unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```

%x是16进制 所以57616的16进制是0xe110

%s是字符串 将i转换为4个char类型的字符 即对应char str[4] = {0x72, 0x6c, 0x64, 0x00}; // = {'r', 'l', 'd', 0}

所以输出是He110 World

5. y会输出什么 为什么

```
  cprintf("x=%d y=%d", 3);
```

在这里只有一个整型数，但是输出的时候按照两个变量输出，**所以输出了地址空间上紧挨着栈，但不属于栈的这部分变量**，造成了不确定的打印结果。

6. 如果栈顺序反过来会怎么样(即第一个参数先入栈 最后一个参数最后入栈)

   若使得输出正常 那么就需要将va_list ap的顺序也翻转一下

### The Stack

### **Exercise 9.**

**确定内核初始化其堆栈的位置，以及其堆栈在内存中的确切位置。内核如何为其堆栈保留空间？堆栈指针初始化指向该保留区域的哪一个“末端”？**

首先在boot.S中初始化stack ：

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240125114009005.png" alt="image-20240125114009005" style="zoom: 67%;" />

然后进入kern/entry.S 这里是把ebp设置为0。后面在递归回溯的时候，遇到ebp ＝ 0的时候，应该停止了

bootstacktop就是栈顶 KSTKSIZE就是栈的大小 

**ESP**：栈指针寄存器(extended stack pointer)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的栈顶。

**EBP**：基址指针寄存器(extended base pointer)，其内存放着一个指针，该指针永远指向系统栈上面一个栈帧的底部。

即当esp到0时表示栈为空 

**在进入 C 函数时，函数的序言代码通常将前一个函数的基指针推到堆栈上，然后在函数执行期间将当前 esp 值复制到 ebp 中，从而保存前一个函数的基指针。其中也会开辟一个固定大小栈空间将函数中参数等变量信息存入这个空间中 这样就组成了一个栈帧**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240125115606639.png" alt="image-20240125115606639" style="zoom:67%;" />

内核内存分布图

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240125120147789.png" alt="image-20240125120147789" style="zoom:50%;" />

可以看出来。kernel就是把内核栈放到了kernel代码的后面。大小就是一个`KSTKSIZE`

### **Exercise 10.**

**为了能够更好的了解在x86上的C程序调用过程的细节，我们首先找到在obj/kern/kern.asm中test_backtrace子程序的地址， 设置断点，并且探讨一下在内核启动后，这个程序被调用时发生了什么。对于这个循环嵌套调用的程序test_backtrace，它一共压入了多少信息到堆栈之中。并且它们都代表什么含义？**

由于我下载的源码中竟然没有test_backtrace函数 所以只能借鉴一下网上的资源

\#结论 查看kern/kernel.asm的反汇编文件，如下：

```
    test_backtrace(5);
f0100138:   c7 04 24 05 00 00 00    movl   $0x5,(%esp)
f010013f:   e8 fc fe ff ff          call   f0100040 <test_backtrace>
```



test_backtrace的代码在0xf0100138地址处，在该处断点调试：

```
(gdb) b *0xf0100138
Breakpoint 1 at 0xf0100138: file kern/init.c, line 51.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100138 <i386_init+155>:  movl   $0x5,(%esp)

Breakpoint 1, i386_init () at kern/init.c:51
51              test_backtrace(5);
(gdb) info r
eax            0x0      0
ecx            0x3d4    980
edx            0x0      0
ebx            0x10094  65684
esp            0xf010ffd0       0xf010ffd0
```



在进入test_backtrace函数之前，栈顶位置为0xf010ffd0。

通过源代码分析可以得知，递归的基本条件是x=0，当x=0时会执行到mon_backtrace，调用mon_backtrace函数的代码地址是0xf0100082，在此处设置断点并查看栈的内容如下：

```
(gdb) x /52xw $esp
0xf010ff10:     0x00000000      0x00000000      0x00000000      0x00000000
0xf010ff20:     0xf010093b      0x00000001      0xf010ff48      0xf0100069
0xf010ff30:     0x00000000      0x00000001      0xf010ff68      0x00000000
0xf010ff40:     0xf010093b      0x00000002      0xf010ff68      0xf0100069
0xf010ff50:     0x00000001      0x00000002      0xf010ff88      0x00000000
0xf010ff60:     0xf010093b      0x00000003      0xf010ff88      0xf0100069
0xf010ff70:     0x00000002      0x00000003      0xf010ffa8      0x00000000
0xf010ff80:     0xf010093b      0x00000004      0xf010ffa8      0xf0100069
0xf010ff90:     0x00000003      0x00000004      0x00000000      0x00000000
0xf010ffa0:     0x00000000      0x00000005      0xf010ffc8      0xf0100069
0xf010ffb0:     0x00000004      0x00000005      0x00000000      0x00010094
0xf010ffc0:     0x00010094      0x00010094      0xf010fff8      0xf0100144
0xf010ffd0:     0x00000005      0x0000e110      0xf010ffec      0x00000004
```



因为最后一次test_backtrace返回前，所有的cprintf函数都已经返回，所以栈中看不到cprintf栈的痕迹。从下往上看是函数调用的顺序，可以看出来还是很有规律性的，先来猜测一下，稍后分析：

1. 第隔两行就出现的5,4,3,2,1,0肯定是传入到test_backtrace的参数，按从下到上的函数调用的顺序依次是5，4，3，2，1，0
2. 最后一列出现多次的0xf0100069则是函数递归调用的test_backtrace的返回地址，最右下的那个0xf0100144则是i386_init中test_backtrace函数调用的返回地址
3. 最下面一条的第一个数字是对应的内存地址是0xf010ffd0，内存内容是0x00000005，是第一次调用test_backtrace时传入的参数，是分析的起点

其它的好像看不出来什么有意思的东西了，现在在分析下，直接看反汇编的代码(为清晰起见，将源代码与反汇编后的代码分开)：

```
test_backtrace(int x)
{
        cprintf("entering test_backtrace %d\n", x);
        if (x > 0)
                test_backtrace(x-1);
        else
                mon_backtrace(0, 0, 0);
        cprintf("leaving test_backtrace %d\n", x);
}

f0100040:       55                      push   %ebp                             ;压入调用函数的%ebp
f0100041:       89 e5                   mov    %esp,%ebp                        ;将当前%esp存到%ebp中，作为栈帧
f0100043:       53                      push   %ebx                             ;保存%ebx当前值，防止寄存器状态被破坏
f0100044:       83 ec 14                sub    $0x14,%esp                       ;开辟20字节栈空间用于本函数内使用
f0100047:       8b 5d 08                mov    0x8(%ebp),%ebx                   ;取出调用函数传入的第一个参数
f010004a:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010004e:       c7 04 24 e0 19 10 f0    movl   $0xf01019e0,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"entering test_backtrace %d\n"
f0100055:       e8 27 09 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印entering test_backtrace (x)
f010005a:       85 db                   test   %ebx,%ebx                        ;测试是否小于0
f010005c:       7e 0d                   jle    f010006b <test_backtrace+0x2b>   ;如果小于0，则结束递归，跳转到0xf010006b处执行
f010005e:       8d 43 ff                lea    -0x1(%ebx),%eax                  ;如果不小于0，则将x的值减1，复制到栈上
f0100061:       89 04 24                mov    %eax,(%esp)                      ;接上一行
f0100064:       e8 d7 ff ff ff          call   f0100040 <test_backtrace>        ;递归调用test_backtrace
f0100069:       eb 1c                   jmp    f0100087 <test_backtrace+0x47>   ;跳转到f0100087执行
f010006b:       c7 44 24 08 00 00 00    movl   $0x0,0x8(%esp)                   ;如果x小于等于0，则跳到这里执行，压入mon_backtrace的最后一个参数
f0100072:       00 
f0100073:       c7 44 24 04 00 00 00    movl   $0x0,0x4(%esp)                   ;压入mon_backtrace的倒数第二个参数
f010007a:       00 
f010007b:       c7 04 24 00 00 00 00    movl   $0x0,(%esp)                      ;压入mon_backtrace的倒数第三个参数
f0100082:       e8 68 07 00 00          call   f01007ef <mon_backtrace>         ;调用mon_backtrace，这是这个练习需要实现的函数
f0100087:       89 5c 24 04             mov    %ebx,0x4(%esp)                   ;压入cprintf的最后一个参数，x的值
f010008b:       c7 04 24 fc 19 10 f0    movl   $0xf01019fc,(%esp)               ;压入cprintf的倒数第二个参数，指向格式化字符串"leaving test_backtrace %d\n"
f0100092:       e8 ea 08 00 00          call   f0100981 <cprintf>               ;调用cprintf函数，打印leaving test_backtrace (x)
f0100097:       83 c4 14                add    $0x14,%esp                       ;回收开辟的栈空间
f010009a:       5b                      pop    %ebx                             ;恢复寄存器%ebx的值
f010009b:       5d                      pop    %ebp                             ;恢复寄存器%ebp的值
f010009c:       c3                      ret                                     ;函数返回
```



一个栈帧(stack frame)的大小计算如下：

1. 在执行call test_backtrace时有一个副作用就是压入这条指令下一条指令的地址，**压入4字节返回地址**
2. push %ebp，将上一个栈帧的地址压入，**增加4字节**
3. push %ebx，保存ebx寄存器的值，**增加4字节**
4. sub $0x14, %esp，**开辟20字节的栈空间**，后面的函数调用传参直接操作这个栈空间中的数，而不是用pu sh的方式压入栈中

加起来一共是32字节，也就是8个int。因此上面打印出来的栈内容，每两行表示一个栈帧，看起来还算清晰。

**所以每次递归压入8个字(一个字word是4个字节)** 

\#第一次调用分析 以第一调用栈为例分析，32个字节代码的含义如下图所示：

```
0xf010ffb0:     0x00000004      0x00000005      0x00000000      0x00010094
0xf010ffc0:     0x00010094      0x00010094      0xf010fff8      0xf0100144
             +--------------------------------------------------------------+
             |    next x    |     this x     |  don't know   |  don't know  |
             +--------------+----------------+---------------+--------------+
             |  don't know  |    last ebx    |  last ebp     | return addr  |
             +------ -------------------------------------------------------+
```



中间的两字节不知道是干嘛用的(靠近this x的那一个在调用mon_backtrace时会用到)，按照理论分析，一个完整的调用栈最少需要的字节数等于4+4+4+4*3=24字节，即返回地址，上一个函数的ebp，保存的ebx，函数内没有分配局部变量，需要再加12个字节用来调用mon_backtrace时传参数。

有一个说法是，因为x86的栈大小必须是16的整数倍(内存对齐)，所以才分配了32个字节的栈大小。

### **Exercise.11**

**将mon_backtrace函数补全使得最后输出形式如下**

![image-20240127213959099](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127213959099.png)

通过函数调用时栈中的变化图，可以看出ebp就是每个栈帧的底 其指向上一个函数的edp

edp + 1 就是eip指针地址 也就是函数返回地址

edp + 2 / + 3 / + 4 / + 5 / + 6 就是各个参数地址

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127203524623.png" alt="image-20240127203524623" style="zoom:50%;" />

补全的mon_backtrace函数 将ebp eip 和各个参数args地址打印出来

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127203414172.png" alt="image-20240127203414172" style="zoom: 67%;" />

### **Exercise.12**

**修改`stack backtrace`函数来显示每个`eip, fun_name, source_file_name, line_number`等与`eip`相关联系的信息。**

![image-20240127214034087](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127214034087.png)

利用**`objdump -h obj/kern/kernel`**命令得到kernel中的可执行文件中所有部分的名称、大小和链接地址的完整列表

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231212180523682.png" alt="image-20231212180523682" style="zoom: 67%;" />

其中的.stab 和 .stabstr是调试符号表（debug symbol table）相关的两个节，它们在调试过程中起不同的作用。

1. `.stabstr` 节：
   - `.stabstr` 节存储了调试符号表中的字符串信息，如符号的名称、类型等。
   - 它包含了在程序中使用的字符串，这些字符串在调试过程中用于解析和显示符号信息。
   - 通常，`.stabstr` 节中的字符串是由编译器生成，并在调试过程中用于解释和展示调试信息。
2. `.stab` 节：
   - `.stab` 节存储了调试符号表的主要信息，包括符号的类型、位置、大小等。
   - 它是一个结构化的数据节，用于存储调试符号表的条目。
   - `.stab` 节中的条目描述了程序中的符号和相关的调试信息，如函数、变量、类型等。
   - 在调试过程中，调试器可以使用 `.stab` 节的信息来定位和跟踪程序中的符号。

在`kern/kdebug.c`中**debuginfo_eip**函数用于通过eip查找到具体的函数位置 对于debug很有用

在这个函数中利用二分查找的思想找到具体函数的位置：

​    利用stab和stabstr中的信息先找到包含eip的文件位置，在符号表区间[lfile, rfile]中

  <img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127215412843.png" alt="image-20240127215412843" style="zoom:80%;" />

​	然后在这个区间中定位函数的地址空间，为[lfun, rfun]

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127215514627.png" alt="image-20240127215514627" style="zoom:80%;" />

​	找到函数所处的地址空间后，所以stab[lfun]这样的数组恰能找到所需的符号地址空间

​	当没有找到对应的符号地址 那么就在整个文件中查找函数的行号

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127221553135.png" alt="image-20240127221553135" style="zoom:67%;" />

​	再在函数的地址空间[lfun, rfun]中找到行号的区间[lline, rline]。也就是需要添加的代码：

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127215657564.png" alt="image-20240127215657564" style="zoom:80%;" />

​	并且将mon_backtrace函数修改一下

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240127220334177.png" alt="image-20240127220334177" style="zoom: 67%;" />

**lab1大功告成！**
