# Lab 3: User Environments

## Part A: User Environments and Exception Handling

**在kern/env.c中有三个重要的全局变量用来维护进程** 

> 系统启动后 **envs 指向一个固定大小的Env结构体数组** 为每个可用的进程提供一个Env实例
>
> **curenv 指向当前运行的进程**
>
> **env_free_list 用来存储所有空闲Env结构在链表上**

![image-20240205160059561](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240205160059561.png)

**在inc/env.h中有Env结构存储的具体信息**

> **env_tf**:当进程不运行时保存进程的寄存器值 以便以后在重新运行时在停止的地方恢复
>
> **env_link**: 这是用来从Env_free_list中插入或删除的链接
>
> **env_id**: 该值唯一标记一个Env结构的进程
>
> **env_parent_id**: 存储创建当前进程的进程env_id 便于管理父子进程之间的关系
>
> **env_type**: 用来区分特殊进程 大多数为ENV_TYPE_USER
>
> **env_status**: 共5种值来存储当前进程的状态
>
> - `ENV_FREE`: 当前Env结构是非活动的 即在Env_free_list上
> - `ENV_RUNNABLE`: 表示Env结构(进程)在等待处理器运行
> - `ENV_RUNNING`: 进程正在运行
> - `ENV_NOT_RUNNABLE`: 当前进程还没有准备好处理器运行(例如进程在和其他进程进行通信)
> - `ENV_DYING`: 僵尸进程 当僵尸进程下一次运行时会被释放
>
> **env_pgdir**: 进程的页目录虚拟地址

![image-20240205155544661](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240205155544661.png)

**线程主要由env_tf字段定义 地址空间由env_pgdir指向的页目录和页表定义**

系统在**进程之间切换**实际上就是**挂起当前运行的线程**，**恢复另一个进程的线程**。

==xv6和JOS的一些区别==:

**xv6** 使用结构体 `struct proc` 来维护一个进程的状态 **JOS**使用结构体`Env`来维护一个进程的状态 

**xv6**中每个进程都拥有一个内核栈和用户栈 一次可以运行多个进程 **JOS**系统一次只能在内核中有一个活动的进程 因此JOS只需要一个内核栈

### Allocating the Environments Array

#### **Exercise 1.**

**完善kern/pmap.c中的 `mem_init()` 函数 **

**和分配页一样分配一个大小为NENV的Env结构体数组 并且在UENVS上建立用户只读的映射** 

分配空间大小:

![image-20240205163808344](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240205163808344.png)

建立映射关系

![image-20240205163852785](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240205163852785.png)

### Creating and Running Environments

#### **Exercise 2.** 

**在env.c中完善env_init() env_setup_vm() region_alloc() load_icode() env_create() env_run()这几个函数**

==**env_init()**==

初始化大小为NENV的env数组 将每个env结构体中的id设为0 status设为ENV_FREE 头插法插到env_free_list中

最终env_free_list头节点就是envs[0]

```
void
env_init(void)
{
	// Set up envs array
	// LAB 3: Your code here.
	int i;
	env_free_list = NULL;
	for(i=NENV-1; i>=0; i--){
		envs[i].env_id = 0;
		envs[i].env_status = ENV_FREE;
		envs[i].env_link = env_free_list;
		env_free_list = &envs[i];
	}
	// Per-CPU part of the initialization
	env_init_percpu();
}
```

==**env_setup_vm()**==

**给进程分配页目录表** 

创建一个空PageInfo页表项

	static int
	env_setup_vm(struct Env *e)
	{
		int i;
		struct PageInfo *p = NULL;
	// Allocate a page for the page directory
	if (!(p = page_alloc(ALLOC_ZERO)))
		return -E_NO_MEM;

将进程的env_pgdir字段设置为上面新创建的页表项的页目录地址

pp_ref高于UTOP的部分是不维护的。因为只有内核在用。但是env_pgdir是个例外，因为可能其他进程的页表会引用到 所以需要p->pp_ref++;

再**将kern_pgdir作为模板 设置进程的页目录**

[0, UTOP) 为空 [UTOP, NPDENTRIES)与kern_pgdir相同

把`UVPT`这块空间映射到了页目录表这里。通过这样一个映射。进程在查看自己的
内存信息的时候，可以直接通过`UVPT`这个地址得到。

![image-20240205232314122](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240205232314122.png)

	// LAB 3: Your code here.
	e->env_pgdir = (pde_t *)page2kva(p);
	p->pp_ref++;
	
	//Map the directory below UTOP.
	for(i = 0; i < PDX(UTOP); i++) {
		e->env_pgdir[i] = 0;		
	}
	
	//Map the directory above UTOP
	for(i = PDX(UTOP); i < NPDENTRIES; i++) {
		e->env_pgdir[i] = kern_pgdir[i];
	}
		
	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
	
	return 0;
}

==**region_alloc()**==

将上面env_setup_vm()函数给一个进程开辟的页索引空间后 这里regin_alloc()为这个索引开辟[va, va+len]的虚拟地址空间并且建立映射、设置permission。分配len字节的物理地址给进程env，并且要映射到虚拟地址va. env_pgdir的虚拟空间就是[va, va+len]

首先设置虚拟地址起始和终止边界(起始：将变量`va`的值向下舍入到最接近的`PGSIZE`的倍数 和 终止：`va+len`后向上进位到与`PGSIZE`的倍数) 序号上是[va, va+len]/PGSIZE 要分配给e->env_pgdir   内存对齐 **一个页大小是PGSIZE**  

然后每次循环分配一个物理页page_info 通过page_insert()将这个物理页与虚拟地址建立映射 设置权限

	static void
	region_alloc(struct Env *e, void *va, size_t len)
	{
		// LAB 3: Your code here.
		// (But only if you need it for load_icode.)
		void* start = (void *)ROUNDDOWN((uint32_t)va, PGSIZE);
		void* end = (void *)ROUNDUP((uint32_t)va+len, PGSIZE);
		struct PageInfo *p = NULL;
		void* i;
		int r;
		for(i=start; i<end; i+=PGSIZE){
			p = page_alloc(0);
			if(p == NULL)
				panic(" region alloc, allocation failed.");
		r = page_insert(e->env_pgdir, p, i, PTE_W | PTE_U);
		if(r != 0) {
			panic("region alloc error");
		}
	}
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)
	}
==**load_icode()**==

**将程序通过ELF格式的二进制文件加载到虚拟空间中 类似boot loader在boot/main.c中作用**

设置entry point的方式和bootmain()不同，在bootmain()中直接调用了，在这里要设置好跳转，通过env_tf保存的eip寄存器值实现。即程序的入口地址。

header里面指明了第一个program section header的位置 将二进制文件从给定的地址加上偏移量ph 到结束地址eph 通过region_alloc()建立映射 并且利用memmove()函数将二进制文件中每个段整体移动到建立映射后的指针上 大小是p_filesz 然后调用memset()函数将剩下的地址清除p_filesz到p_memsz

最后初始化用户栈的映射，为它映射一个页

==**注意**== 需要设置CR3寄存器 否则在保护模式下程序就不能进行 

	// LAB 3: Your code here.
		struct Elf* header = (struct Elf*)binary;
		
	if(header->e_magic != ELF_MAGIC) {
		panic("load_icode failed: The binary we load is not elf.\n");
	}
	
	if(header->e_entry == 0){
		panic("load_icode failed: The elf file can't be excuterd.\n");
	}
	
	e->env_tf.tf_eip = header->e_entry;
	
	lcr3(PADDR(e->env_pgdir));   //?????
	
	struct Proghdr *ph, *eph;
	ph = (struct Proghdr* )((uint8_t *)header + header->e_phoff);
	eph = ph + header->e_phnum;
	for(; ph < eph; ph++) {
		if(ph->p_type == ELF_PROG_LOAD) {
			if(ph->p_memsz - ph->p_filesz < 0) {
				panic("load icode failed : p_memsz < p_filesz.\n");
			}
	
			region_alloc(e, (void *)ph->p_va, ph->p_memsz);
			memmove((void *)ph->p_va, binary + ph->p_offset, ph->p_filesz);
			memset((void *)(ph->p_va + ph->p_filesz), 0, ph->p_memsz - ph->p_filesz);
		}
	} 
	 
	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.
	region_alloc(e,(void *)(USTACKTOP-PGSIZE), PGSIZE);

==**env_create()**==

**这里就是申请一个进程描述符，然后把相应的代码加载上去。**

根据提示创建一个新env 将elf二进制文件通过load_icde()函数加载 然后将这个env的env_type字段设置为参数type

```

// Allocates a new env with env_alloc, loads the named elf
// binary into it with load_icode, and sets its env_type.
// This function is ONLY called during kernel initialization,
// before running the first user-mode environment.
// The new env's parent ID is set to 0.
//
void
env_create(uint8_t *binary, enum EnvType type)
{
	// LAB 3: Your code here.
	struct Env *e;
	int rc;
	if((rc = env_alloc(&e, 0)) != 0) {
		panic("env_create failed: env_alloc failed.\n");
	}

	load_icode(e, binary);
	e->env_type = type;

}
```

==**env_run()**==

**调度到用户进程上执行。**

将指向当前运行的进程指针curenv指向e 并将上一个运行的进程状态设置为ENV_RUNNABLE

将CR3寄存器中的进程地址改变为新进程地址

调用env_pop_tf()函数 来存储新进程状态 切换上下文

	// Context switch from curenv to env e.
	// Note: if this is the first call to env_run, curenv is NULL.
	//
	// This function does not return.
	//
	void
	env_run(struct Env *e)
	{
		// Step 1: If this is a context switch (a new environment is running):
		//	   1. Set the current environment (if any) back to
		//	      ENV_RUNNABLE if it is ENV_RUNNING (think about
		//	      what other states it can be in),
		//	   2. Set 'curenv' to the new environment,
		//	   3. Set its status to ENV_RUNNING,
		//	   4. Update its 'env_runs' counter,
		//	   5. Use lcr3() to switch to its address space.
		// Step 2: Use env_pop_tf() to restore the environment's
		//	   registers and drop into user mode in the
		//	   environment.
		if(curenv != NULL && curenv->env_status == ENV_RUNNING) {
			curenv->env_status = ENV_RUNNABLE;
		}
	curenv = e;
	curenv->env_status = ENV_RUNNING;
	curenv->env_runs++;
	lcr3(PADDR(curenv->env_pgdir));
	// Hint: This function loads the new environment's state from
	//	e->env_tf.  Go back through the code you wrote above
	//	and make sure you have set the relevant parts of
	//	e->env_tf to sensible values.
	env_pop_tf(&curenv->env_tf);
	// LAB 3: Your code here.
	
	panic("env_run not yet implemented");
	}

**利用gdb调试验证进入用户模式**

下图是运行用户代码之前函数调用图：

![image-20240207141422521](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240207141422521.png)

在env_pop_tf()处设置一个断点，之后单步执行，直到完成iret指令，可以看到之后第一条用户指令。

env_pop_tf()应该是在实际进入用户模式之前命中的最后一个函数。使用 si 单步完成此函数; 处理器应该在 iret 指令之后进入用户模式。

然后再在obj/user/hello.asm中找到sys_cputs()函数，我的机器上，它的最后一条指令是：  `800a4d:	cd 30     int   $0x30   //系统调用中断` 

```
(gdb) break kern/env.c : env_pop_tf
Breakpoint 1 at 0xf0102eaa: file kern/env.c, line 490.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0102eaa <env_pop_tf>:	push   %ebp

Breakpoint 1, env_pop_tf (tf=0xf01a0000) at kern/env.c:490
490	{
(gdb) si
=> 0xf0102eab <env_pop_tf+1>:	mov    %esp,%ebp
0xf0102eab	490	{
(gdb) si
=> 0xf0102ead <env_pop_tf+3>:	sub    $0xc,%esp
0xf0102ead	490	{
(gdb) si
=> 0xf0102eb0 <env_pop_tf+6>:	mov    0x8(%ebp),%esp
491		__asm __volatile("movl %0,%%esp\n"
(gdb) si
=> 0xf0102eb3 <env_pop_tf+9>:	popa   
0xf0102eb3	491		__asm __volatile("movl %0,%%esp\n"
(gdb) si
=> 0xf0102eb4 <env_pop_tf+10>:	pop    %es
0xf0102eb4 in env_pop_tf (
    tf=<error reading variable: Unknown argument list address for `tf'.>)
    at kern/env.c:491
491		__asm __volatile("movl %0,%%esp\n"
(gdb) si
=> 0xf0102eb5 <env_pop_tf+11>:	pop    %ds
0xf0102eb5	491		__asm __volatile("movl %0,%%esp\n"
(gdb) si
=> 0xf0102eb6 <env_pop_tf+12>:	add    $0x8,%esp
0xf0102eb6	491		__asm __volatile("movl %0,%%esp\n"
(gdb) si
=> 0xf0102eb9 <env_pop_tf+15>:	iret   
0xf0102eb9	491		__asm __volatile("movl %0,%%esp\n" ////////////////////////////////////////////
(gdb) si
=> 0x800020:	cmp    $0xeebfe000,%esp
0x00800020 in ?? ()
(gdb) b* 0x800a4d
Breakpoint 2 at 0x800a4d
(gdb) c
Continuing.
=> 0xf0102eaa <env_pop_tf>:	push   %ebp

Breakpoint 1, env_pop_tf (tf=0xf01a0000) at kern/env.c:490
490	{
(gdb) c
Continuing.
=> 0x800a4d:	int    $0x30
```

### Handling Interrupts and Exceptions

中断和异常是计算机系统中处理外部事件和错误情况的机制。它们允许计算机在执行过程中暂停当前任务，转而处理优先级更高或更紧急的事件或错误。

1. **中断（Interrupt）**：
   中断是由外部设备或其他部分发出的信号，**用于通知计算机系统发生了一个事件，需要处理**。中断可以分为硬件中断和软件中断两种类型。

   - 硬件中断：由硬件设备产生，如定时器中断、输入/输出设备的数据准备好、硬件故障等。硬件中断通常通过硬件的中断控制器（如 PIC、APIC）发送给 CPU，引发 CPU 执行相应的中断处理程序。
   - 软件中断：由正在执行的程序主动发出的中断请求，通常用于特定的系统调用或异常处理。软件中断通过软件指令（如 INT 指令）触发，引发 CPU 执行相应的中断处理程序。

   **当中断发生时，CPU 会立即停止当前的任务，保存当前的上下文，然后跳转到中断处理程序执行相应的操作。处理完成后，CPU 恢复之前的上下文，继续执行被中断的任务。**

2. **异常（Exception）**：
   异常是指**在程序执行过程中发生的错误情况或非正常事件**。与中断不同，**异常是由 CPU 内部的执行过程引发的。异常通常是程序错误、非法指令、内存访问错误、除零错误等导致的。**

   当异常发生时，CPU 会立即停止当前的任务，保存当前的上下文，并跳转到异常处理程序执行相应的操作。异常处理程序通常用于处理错误情况、采取恢复措施或终止程序的执行。

   异常可以分为同步异常和异步异常：
   - 同步异常：在指令执行期间发生的异常，如除零错误、非法指令等。它们通常由当前正在执行的指令触发，并在指令执行完后立即处理。
   - 异步异常：在指令执行期间以外的时间发生的异常，如硬件错误、内存错误等。它们无法与特定的指令关联，并且随时可能发生。

**当发生中断或者异常时 这会导致处理器从用户模式切换到内核模式(CPL = 0) ，而不会给用户模式代码任何机会干扰内核或其他环境的功能。**

### Basics of Protected Control Transfer

异常和中断都是“受保护的控制传输”，这会导致处理器从用户模式切换到内核模式(CPL = 0) ，而不会给用户模式代码任何机会干扰内核或其他环境的功能。	

发生中断或异常后当前运行代码不能随意选择内核输入的位置或方式 所以x86提供了两个机制来保护

- **The Interrupt Descriptor Table** IDT 中断描述符表 用于将存储和管理中断和异常处理程序的信息。IDT 中的每个描述符与一个唯一的**中断向量**关联。**中断向量是一个索引值**，用于标识特定的中断或异常类型。当中断或异常发生时，CPU 使用中断向量来查找相应的描述符和处理程序。
- **The Task State Segment** 用于存储和管理任务（Task）的状态信息。它是操作系统和硬件之间的接口，用于实现任务切换和管理多任务环境。

### Setting Up the IDT

#### **Exercise 4.**

这里要求我们在`trap.c`和`trapentry.S`实现IDT表的初始化，由于执行中断处理程序要从用户模式切换到内核模式，因此在用户模式中当前进程的信息必须要以`trapframe`的结构存储在栈上，`trapframe`是一个数据结构，用于保存当前进程在被中断之前的状态信息，包括寄存器的值、程序计数器的值等。当中断发生时，操作系统会保存当前进程的`trapframe`到栈上。

首先初始化IDT 在操作系统的中断描述符表中设置一个页面错误中断门，用于处理页面错误中断，但实际上没有为页面错误中断提供有效的处理程序（因为偏移量设置为0）。

```
	void
	trap_init(void)
	{
			extern struct Segdesc gdt[];
	// LAB 3: Your code here.
	SETGATE(idt[T_DIVIDE], 0, GD_KT, t_divide, 0);
	SETGATE(idt[T_DEBUG], 0, GD_KT, t_debug, 0);
	SETGATE(idt[T_NMI], 0, GD_KT, t_nmi, 0);
	SETGATE(idt[T_BRKPT], 0, GD_KT, t_brkpt, 3);
	SETGATE(idt[T_OFLOW], 0, GD_KT, t_oflow, 0);
	SETGATE(idt[T_BOUND], 0, GD_KT, t_bound, 0);
	SETGATE(idt[T_ILLOP], 0, GD_KT, t_illop, 0);
	SETGATE(idt[T_DEVICE], 0, GD_KT, t_device, 0);
	SETGATE(idt[T_DBLFLT], 0, GD_KT, t_dblflt, 0);
	SETGATE(idt[T_TSS], 0, GD_KT, t_tss, 0);
	SETGATE(idt[T_SEGNP], 0, GD_KT, t_segnp, 0);
	SETGATE(idt[T_STACK], 0, GD_KT, t_stack, 0);
	SETGATE(idt[T_GPFLT], 0, GD_KT, t_gpflt, 0);
	SETGATE(idt[T_PGFLT], 0, GD_KT, t_pgflt, 0);
	SETGATE(idt[T_FPERR], 0, GD_KT, t_fperr, 0);
	SETGATE(idt[T_ALIGN], 0, GD_KT, t_align, 0);
	SETGATE(idt[T_MCHK], 0, GD_KT, t_mchk, 0);
	SETGATE(idt[T_SIMDERR], 0, GD_KT, t_simderr, 0);
	SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);
	// Per-CPU setup 
	trap_init_percpu();
	}
```

然后在trapentry.S中实现对于不同trap的entry point 

这里用到的两个宏TRAPHANDLER` and `TRAPHANDLER_NOEC唯一的区别是前者cpu会自动把error code入栈。而对于后者则要手动入栈一个0当作错误码.

```

TRAPHANDLER_NOEC(t_divide, T_DIVIDE)
TRAPHANDLER_NOEC(t_debug, T_DEBUG)
TRAPHANDLER_NOEC(t_nmi, T_NMI)
TRAPHANDLER_NOEC(t_brkpt, T_BRKPT)
TRAPHANDLER_NOEC(t_oflow, T_OFLOW)
TRAPHANDLER_NOEC(t_bound, T_BOUND)
TRAPHANDLER_NOEC(t_illop, T_ILLOP)
TRAPHANDLER_NOEC(t_device, T_DEVICE)
TRAPHANDLER(t_dblflt, T_DBLFLT)
TRAPHANDLER(t_tss, T_TSS)
TRAPHANDLER(t_segnp, T_SEGNP)
TRAPHANDLER(t_stack, T_STACK)
TRAPHANDLER(t_gpflt, T_GPFLT)
TRAPHANDLER(t_pgflt, T_PGFLT)
TRAPHANDLER_NOEC(t_fperr, T_FPERR)
TRAPHANDLER(t_align, T_ALIGN)
TRAPHANDLER_NOEC(t_mchk, T_MCHK)
TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)

TRAPHANDLER_NOEC(t_syscall, T_SYSCALL)
```

在trapentry.S中实现alltraps 之前的env_pop_tf()函数已经记录了当前新进程的有关信息，这样只需要压入err及以后的trapno, ds, es, regs中的寄存器：ds, es（按顺序）就可以了。

  _alltraps是所有trap handler共同执行的代码 目的是保存当前的寄存器状态和栈指针，并将数据段和附加段设置为内核数据段，然后调用 `trap` 函数进行中断或异常的处理。

```
_alltraps:
	pushl %ds
	pushl %es
	pushal 
movl $GD_KD, %eax
movw %ax, %ds
movw %ax, %es

push %esp
call trap	
```

![image-20240207171801430](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240207171801430.png)

**Questions**

1. 为每个异常/中断设置单独的处理程序函数的目的是什么？

为每个异常或中断拥有单独的处理函数可以实现特定的处理、错误报告、优先级排序、上下文管理和系统可扩展性。它使系统能够对不同的事件提供细粒度的控制和定制响应，提升系统的功能性、可靠性和可维护性。

2. 中断向量13对应通用保护错误，而中断向量14对应页故障 不同中断原因采用相对应的解决方法

##### ==**对于发生中断或者异常系统处理过程：**==

1. 发生中断或者trap，从ldtr里面找到ldt。

2. 根据中断号找到这一项，即ldt[中断号]

3. 根据ldt[中断号] == SETGATE(idt[T_MCHK], 0, GD_KT, T_MCHK_handler, 0);

   取出当时设置的中断处理函数

4. 跳转到中断函数

5. 中断处理函数再跳转到trap函数。

6. trap函数再根据tf->trap_no中断号来决定分发给哪个函数。

## Part B: Page Faults, Breakpoints Exceptions, and System Calls

### Handling Page Faults

#### **Exercise 5.**

**完善kern/trap.c中的trap_dispatch()函数来处理页中错误**

```
switch(tf->tf_trapno) {
		case (T_PGFLT):
			page_fault_handler(tf);
			break; 
```

### The Breakpoint Exception

#### **Exercise 6.**

**继续完善trap_dispatch()函数来处理断点错误T_BRKPT 并调用kernel monitor**

```
case (T_BRKPT):
			print_trapframe(tf);
			monitor(tf);		
			break;
```

#### Question 3

1. **断点测试用例将根据您在 IDT 中初始化断点条目的方式(例如，从 trap _ init 调用 SETGATE)生成断点异常或一般保护错误。为什么？您需要如何设置它以使断点异常按照上面指定的方式工作，以及什么不正确的设置会导致它触发一般的保护故障？**

 我们在设置中断门时将断点错误权限设置为3表示在用户态可以访问

```
SETGATE(idt[T_BRKPT], 0, GD_KT, t_brkpt, 3);
```

若设置为0表示只有内核态才能访问

```
SETGATE(idt[T_BRKPT], 0, GD_KT, int_brkpt, 0); 
```

2. **您认为这些机制的意义是什么，特别是考虑到用户/软件测试程序的功能？**

机制的目的都应该是保护内核代码或者给程序员提供方便

### System calls

应用程序将在寄存器中传递系统调用号和系统调用参数。这样，内核就不需要在用户环境的堆栈或指令流中进行挖掘。

#### **Exercise 7.** 

**继续完善trap_dispatch()函数来处理系统中断T_SYSCALL 根据inc/syscall.h 补全kern/syscall.c中的syscall()函数**

**inc/syscall.h**

主要列出了kern/syscall.c中不同系统调用函数需要根据不同系统调用号

```
/* system call numbers */
enum {
	SYS_cputs = 0,
	SYS_cgetc,
	SYS_getenvid,
	SYS_env_destroy,
	NSYSCALLS
};
```

**trap_dispatch()**

根据中断向量 调用系统调用

```
case (T_SYSCALL):
	//		print_trapframe(tf);
			ret_code = syscall(
					tf->tf_regs.reg_eax,
					tf->tf_regs.reg_edx,
					tf->tf_regs.reg_ecx,
					tf->tf_regs.reg_ebx,
					tf->tf_regs.reg_edi,
					tf->tf_regs.reg_esi);
			tf->tf_regs.reg_eax = ret_code;
			break;
```

**lib/syscall.c**

系统调用号将放在% eax 中，参数(最多五个)将分别放在% edx、% ecx、% ebx、% edi 和% esi 中。内核将返回值传递回% eax。

	static inline int32_t
	syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
	{
		int32_t ret;
	// Generic system call: pass system call number in AX,
	// up to five parameters in DX, CX, BX, DI, SI.
	// Interrupt kernel with T_SYSCALL.
	//
	// The "volatile" tells the assembler not to optimize
	// this instruction away just because we don't use the
	// return value.
	//
	// The last clause tells the assembler that this can
	// potentially change the condition codes and arbitrary
	// memory locations.
	
	asm volatile("int %1\n" // 用于触发一个T_SYSCALL中断
		: "=a" (ret) // ret是一个输出操作数，绑定到%eax寄存器，用于存储系统调用的返回值
		: "i" (T_SYSCALL), // T_SYSCALL是一个输入操作数，绑定到立即数，用于指定触发的中断号。
		  "a" (num), // 这些都是输入操作数，分别绑定到%eax、%edx、%ecx、%ebx、%edi、%esi寄存器
		  "d" (a1),
		  "c" (a2),
		  "b" (a3),
		  "D" (a4),
		  "S" (a5)
		: "cc", "memory"); // 输入输出约束（constraint），用于告诉编译器这段代码可能会改变标志位（condition codes）和内存内容，因此需要进行适当的处理。
	
	if(check && ret > 0)
		panic("syscall %d returned %d (> 0)", num, ret);
	
	return ret;
	}
**kern/syscall.c**

// 根据系统调用号(在inc/syscall.h中定义)来调用不同的系统函数

	// Dispatches to the correct kernel function, passing the arguments.
	int32_t
	syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
	{
		// Call the function corresponding to the 'syscallno' parameter.
		// Return any appropriate return value.
		// LAB 3: Your code here.
		//	panic("syscall not implemented");
	
	switch (syscallno) {
		case (SYS_cputs):
			sys_cputs((const char *)a1, a2);
			return 0;
		case (SYS_cgetc):
			return sys_cgetc();
		case (SYS_getenvid):
			return sys_getenvid();
		case (SYS_env_destroy):
			return sys_env_destroy(a1);
		default:
			return -E_INVAL;
	}

### User-mode startup

#### **Exercise 8.** 

用户程序开始在 lib/entry. S 的顶部运行。经过一些设置之后，这段代码调用 lib/libmain.c 中的 libmain ()。您应该修改 libmain ()来初始化全局指针 thisenv，使其指向 envs []数组中这个环境的 struct Env。

	void
	libmain(int argc, char **argv)
	{
		// set thisenv to point at our Env structure in envs[].
		// LAB 3: Your code here.
		thisenv = &envs[ENVX(sys_getenvid())]; // sys_getenvid()：找到当前进程的进程号 用ENVX处理就可以作为全局数组envs[]的偏置加上去了
	// save the name of the program so that panic() can use it
	if (argc > 0)
		binaryname = argv[0];
	
	// call user main routine
	umain(argc, argv); // hello.c
	
	// exit gracefully
	exit();
	}
![image-20240211223417694](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240211223417694.png)

### Page faults and memory protection

解决两个问题：

- 内核中的页面错误可能比用户程序中的页面错误严重得多。如果内核页面在操作自己的数据结构时发生故障，那就是内核错误，故障处理程序应该使内核(从而使整个系统)陷入恐慌。但是当内核解引用用户程序给它的指针时，它需要一种方法来记住这些解引用导致的任何页面错误实际上是代表用户程序的。
- 内核通常比用户程序拥有更多的内存权限。用户程序可以传递一个指向系统调用的指针，该系统调用指向内核可以读写但程序不能读写的内存。内核必须小心，不要被骗去解引用这样的指针，因为这可能会暴露私有信息或破坏内核的完整性。

#### **Exercise 9.**

1. **如果在内核模式下发生页面错误，则将 kern/trap.c 更改为 panic。**

page_fault_handler在内核模式下发生页面错误进行panic

```
if(tf->tf_cs && 0x01 == 0) {
		panic("page_fault in kernel mode, fault address %d\n", fault_va);
```

2. **读取 kern/pmap.c 中的 user _ mem _ asserat，并在同一文件中实现 user _ mem _ check。**

   只要判断两个条件：指针越界、权限错误，两者之一发生，就要返回错误。

   遍历[va/PGSIZE, (va+len)/PGSIZE] 为了确定用户指针的权限，必须要通过虚拟地址找到页表entry：

	```
	int
	user_mem_check(struct Env *env, const void *va, size_t len, int perm)
	{
		// LAB 3: Your code here.
		char * end = NULL;
		char * start = NULL;
		start = ROUNDDOWN((char *)va, PGSIZE); 
		end = ROUNDUP((char *)(va + len), PGSIZE);
		pte_t *cur = NULL;
	for(; start < end; start += PGSIZE) {
		cur = pgdir_walk(env->env_pgdir, (void *)start, 0); // 通过虚拟地址找到页表entry
		if((int)start > ULIM || cur == NULL || ((uint32_t)(*cur) & perm) != perm) { // 判断指针越界、权限错误
			  if(start == ROUNDDOWN((char *)va, PGSIZE)) {
					user_mem_check_addr = (uintptr_t)va;
			  }
			  else {
			  		user_mem_check_addr = (uintptr_t)start;
			  }
			  return -E_FAULT;
		}
		
	}
		
	return 0;
	}
	
	void
	user_mem_assert(struct Env *env, const void *va, size_t len, int perm)
	{
		if (user_mem_check(env, va, len, perm | PTE_U) < 0) {
			cprintf("[%08x] user_mem_check assertion failure for "
				"va %08x\n", env->env_id, user_mem_check_addr);
			env_destroy(env);	// may not return
		}
	}
	```
	
	3. **将 kern/syscall.c 更改为系统调用的健全性检查参数。**
	
	   调用user_mem_assert来检查进程是否正确使用地址空间
	
	   	static void
	   	sys_cputs(const char *s, size_t len)
	   	{
	   		// Check that the user has permission to read memory [s, s+len).
	   		// Destroy the environment if not:.
	   	// LAB 3: Your code here.
	   	user_mem_assert(curenv, s, len, 0);
	   	// Print the string supplied by the user.
	   	cprintf("%.*s", len, s);
	   	}

		4. **最后，修改 kern/kdebug. c 中的 debuginfo _ eip，在 usd、 stabs 和 stabstr 上调用 user _ mem _ check。**

#### **Exercise 10.**

启动内核，运行 user/evilhello。应该销毁环境，内核不应该恐慌。您应该看到:

```
	[00000000] new env 00001000
	...
	[00001000] user_mem_check assertion failure for va f010000c
	[00001000] free env 00001000
```
