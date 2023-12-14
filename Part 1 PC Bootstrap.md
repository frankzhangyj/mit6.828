lab 1 

## Part 1: PC Bootstrap

### The PC's Physical Address Space

 PC 的物理地址空间通过硬连线具有以下总体布局：

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20230913215342024.png" alt="image-20230913215342024" style="zoom: 80%;" />

第一批 PC 基于 16 位 Intel 8088 处理器，只能寻址 1MB 物理内存。PC 的物理地址空间将从 0x00000000 开始，但以 0x000FFFFF 结束，而不是 0xFFFFFFFF。

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

### exercise 2: 利用单步调试命令 si(step instruction) 进行调试操作

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

通过对比将**boot/boot.S** 和**boot/main.C**与 **obj/boot.asm**(这个文件记录了从0x7c00开始执行的所有命令的地址及其汇编指令 先从boot.S开始转换模式 然后进入bootmain函数 将加载引导扇区内容加载到内存中 然后运行内核)对比找到每个指令执行的地址

- **处理器什么时候开始执行 32 位代码？究竟是什么原因导致从 16 位模式切换到 32 位模式？**

  `movl    %eax, %cr0`  cr0寄存器用于控制处理器的各种特性和操作模式。这个指令将 `%eax` 寄存器的值复制到 CR0 寄存器中，以便对处理器的控制和配置进行相应的更改。

  `# Jump to next instruction, but in 32-bit code segment.`

  `# Switches processor into 32-bit mode.`  

  `ljmp    $PROT_MODE_CSEG, $protcseg`  用一个长跳转来进入到32位的代码并能够执行32位的代码 通过 `ljmp` 指令的执行，程序将跳转到给定选择子和偏移量所指定的代码段，并在跳转时改变特权级别，以便在新的特权级别下执行代码。

- **引导加载程序执行的*最后一条*指令 是什么，刚刚加载的内核的*第一条指令是什么？***

  引导加载程序执行*最后一条*指令：`((void (*)(void)) (ELFHDR->e_entry))();` 即boot/main.c 中bootmain函数的最后一个代码

  加载的内核的第一条指令：`movw    $0x1234,0x472                   # warm boot` 即kern/entry.S的第一条指令

- **引导加载程序如何决定必须读取多少个扇区才能从磁盘获取整个内核？它在哪里找到这些信息？**

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

简单而言，通过**ELF文件的头**(**program header**)中的信息得到每个程序段的开始和结束位置(**目标物理地址**)，通过一个循环 调用readseg函数 根据函数中的参数`uint32_t offset`确定读取的扇区位置 循环结束后 执行内核程序(**BIOS将引导扇区加载到 0x7c00内存中 然后将控制器转交给引导加载程序 然后引导加载程序运行内核**)

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

  如果更改boot/Makerag文件中的-Ttext里面的起始地址。后面程序加载地址就会发生变化，运行报错。

  <img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20231213170549869.png" alt="image-20231213170549869" style="zoom:67%;" />

  ### exercise 6: 

  理解从boot loader到kernel的链接过程:

  通过`objdump -h obj/kern/kernel`可以看到.text的链接地址（虚拟内存地址）VMA是f0100000，被映射到加载地址LMA100000。

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



