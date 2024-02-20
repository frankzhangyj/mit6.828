# Lab 4: Preemptive Multitasking

## Part A: Multiprocessor Support and Cooperative Multitasking

### Multiprocessor Support

JOS支持SMP(对称多处理机)，这是一种多处理器模型，其中所有CPU都具有对系统资源的等效访问权限。

根据引导过程可以将SMP分为：

- BSP(引导处理器)负责初始化系统和引导操作系统
- AP(应用程序处理器)只在操作系统启动并运行后才被BSP激活。

在SMP系统中，每个 CPU 都有一个伴随的本地 APIC (LAPIC)单元，LAPIC单元负责在整个系统中交付中断。LAPIC 还为其连接的 CPU 提供了一个唯一标识符。**到目前为止，所有现有的 JOS 代码都在 BSP 上运行。**

处理器使用内存映射 I/O (MMIO)访问其 LAPIC，JOS 虚拟内存映射在 MMIOBASE 上留下了4 MB 的空白，因此我们可以像这样映射设备

![image-20240213211935475](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240213211935475.png)

#### **Exercise 1.** 

**在 kern/pmap.c 中实现 mmio _ map _ region。**

将给定物理地址映射到虚拟地址从MMIOBASE开始的4MB空白空间中，并设置权限为PTE_PCD | PTE_PWT | PTE_W

	void *
	mmio_map_region(physaddr_t pa, size_t size)
	{
	// Where to start the next region.  Initially, this is the
	// beginning of the MMIO region.  Because this is static,its
	// value will be preserved between calls to mmio_map_region
	// (just like nextfree in boot_alloc).
	static uintptr_t base = MMIOBASE;
	// Reserve size bytes of virtual memory starting at base and
	// map physical pages [pa,pa+size) to virtual addresses
	// [base,base+size).  Since this is device memory and not
	// regular DRAM, you'll have to tell the CPU that it isn't
	// safe to cache access to this memory.  Luckily, the page
	// tables provide bits for this purpose; simply create the
	// mapping with PTE_PCD|PTE_PWT (cache-disable and
	// write-through) in addition to PTE_W.  (If you're interested
	// in more details on this, see section 10.5 of IA32 volume
	// 3A.)
	//
	// Be sure to round size up to a multiple of PGSIZE and to
	// handle if this reservation would overflow MMIOLIM (it's
	// okay to simply panic if this happens).
	//
	// Hint: The staff solution uses boot_map_region.
	//
	// Your code here:
	size_t start = ROUNDDOWN(pa, PGSIZE);
	size_t end = ROUNDUP(size + pa, PGSIZE);
	if (base + end - start >= MMIOLIM) {
		panic("mmio_map_region: overflow MMIOLIM!\n");
	}
	
	size = end - start;
	boot_map_region(kern_pgdir, base, size, start, PTE_PCD | PTE_PWT | PTE_W);
	base += size;
	return (void *) (base - size);
	}
#### Application Processor Bootstrap

Boot _ aps ()函数(在 kern/init.c 中)驱动 AP 引导进程。AP以实模式启动，然后转到32位保护模式，类似将bootloader在boot/boot.S中启动一样，

	// Start the non-boot (AP) processors.
	static void
	boot_aps(void)
	{
		extern unsigned char mpentry_start[], mpentry_end[];
		void *code;
		struct CpuInfo *c;
	// Write entry code to unused memory at MPENTRY_PADDR
	code = KADDR(MPENTRY_PADDR);
	memmove(code, mpentry_start, mpentry_end - mpentry_start);
	
	// Boot each AP one at a time
	for (c = cpus; c < cpus + ncpu; c++) {
		if (c == cpus + cpunum())  // We've started already.
			continue;
	
		// Tell mpentry.S what stack to use 
		mpentry_kstack = percpu_kstacks[c - cpus] + KSTKSIZE;
		// Start the CPU at mpentry_start
		lapic_startap(c->cpu_id, PADDR(code));
		// Wait for the CPU to finish some basic setup in mp_main()
		while(c->cpu_status != CPU_STARTED)
			;
	}
	}
#### **Exercise 2.** 

**在 kern/pmap.c 中修改 page _ init ()的实现，以避免将位于 MPENTRY _ PADDR 的页面添加到空闲列表中**

第一个物理页存储的是实模式下的IDT和BIOS结构 

[PGSIZE, npages_basemem * PGSIZE)是空页表 加入到page_free_list中

[IOPHYSMEM, EXTPHYSMEM)这部分IO漏洞不能被分配页

[EXTPHYSMEM, ...)额外部分一些正在被利用一些是空闲

==**注意：**==物理地址是MPENTRY_PADDR不能加入到page_free_list中 因为需要将这个页用于映射设备(在exercise1中实现)

	void
	page_init(void)
	{
		// LAB 4:
		// Change your code to mark the physical page at MPENTRY_PADDR
		// as in use
	// The example code here marks all physical pages as free.
	// However this is not truly the case.  What memory is free?
	//  1) Mark physical page 0 as in use.
	//     This way we preserve the real-mode IDT and BIOS structures
	//     in case we ever need them.  (Currently we don't, but...)
	//  2) The rest of base memory, [PGSIZE, npages_basemem * PGSIZE)
	//     is free.
	//  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must
	//     never be allocated.
	//  4) Then extended memory [EXTPHYSMEM, ...).
	//     Some of it is in use, some is free. Where is the kernel
	//     in physical memory?  Which pages are already in use for
	//     page tables and other data structures?
	//
	// Change the code to reflect this.
	// NB: DO NOT actually touch the physical memory corresponding to
	// free pages!
	pages[0].pp_ref = 1;
	pages[0].pp_link = NULL;
	size_t i;
	size_t kernel_end_page = PADDR(boot_alloc(0)) / PGSIZE;
	size_t mpentry = MPENTRY_PADDR / PGSIZE;
	for (i = 1; i < npages; i++) {
	    if (i >= npages_basemem && i < kernel_end_page) {
	        pages[i].pp_ref = 1;
	        pages[i].pp_link = NULL;
	    } else if (i == mpentry) {
			pages[i].pp_ref = 1;
	        pages[i].pp_link = NULL;
		} else {
	        pages[i].pp_ref = 0;
	        pages[i].pp_link = page_free_list;
	        page_free_list = &pages[i];
	    }
	}
	}
#### **Question**

**将 kern/mpentry.S 与 boot/boot.S 并排比较。请记住，kern/mpentry.S 是编译并链接到 KernBASE 上运行的，就像内核中的其他所有内容一样，宏 MPBOOTPHYS 的目的是什么？为什么在 kern/mpentry.S 中需要它，而在 boot/boot.S 中不需要它？**

boot/boot.S

```
 lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
  
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg
```

kern/mpentry.S

```
  lgdt    MPBOOTPHYS(gdtdesc)
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0

  ljmpl   $(PROT_MODE_CSEG), $(MPBOOTPHYS(start32))
```

两个代码跳转的位置不同。这是因为AP的保护模式还没有开启，所以无法寻址到3G以上的空间，所以通过MPBOOTPHYS来计算物理地址偏置，不直接指定物理地址是因为在运行mpentry.S时，CPU已经开启保护模式了。

#### Per-CPU State and Initialization

**每一个cpu都有一个与之对应的CpuInfo**

```
// Per-CPU state
struct CpuInfo {
	uint8_t cpu_id;                 // Local APIC ID; index into cpus[] below
	volatile unsigned cpu_status;   // The status of the CPU
	struct Env *cpu_env;            // The currently-running environment.
	struct Taskstate cpu_ts;        // Used by x86 to find stack for interrupt
};
```

#### **Exercise 3.** 

**修改 mem _ init _ mp ()(在 kern/pmap.c 中)以映射从 KSTACKTOP 开始的每个 CPU 堆栈**

在lab2中的mem_init()函数中只将cpu0与物理地址建立映射 在这个练习中需要将所有cpu都映射到物理地址上

 * ```
    *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
    *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
    *    | - - - - - - - - - - - - - - -|                   |
    *    |      Invalid Memory (*)      | --/--  KSTKGAP    |
    *    +------------------------------+                   |
    *    |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
    *    | - - - - - - - - - - - - - - -|                 PTSIZE
    *    |      Invalid Memory (*)      | --/--  KSTKGAP    |
    *    +------------------------------+                   |
    *    :              .               :                   |
    *    :              .               :                   |
    *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
   ```

每个cpu之间都有一个KSTKGAP大小的空间来防止溢出，所以可以利用**KSTACKTOP - KSTKSIZE - i * (KSTKSIZE + KSTKGAP)**得到每个**cpu的起始虚拟地址** 

```
// Modify mappings in kern_pgdir to support SMP
//   - Map the per-CPU stacks in the region [KSTACKTOP-PTSIZE, KSTACKTOP)
//
static void
mem_init_mp(void)
{
	// Map per-CPU stacks starting at KSTACKTOP, for up to 'NCPU' CPUs.
	//
	// For CPU i, use the physical memory that 'percpu_kstacks[i]' refers
	// to as its kernel stack. CPU i's kernel stack grows down from virtual
	// address kstacktop_i = KSTACKTOP - i * (KSTKSIZE + KSTKGAP), and is
	// divided into two pieces, just like the single stack you set up in
	// mem_init:
	//     * [kstacktop_i - KSTKSIZE, kstacktop_i)
	//          -- backed by physical memory
	//     * [kstacktop_i - (KSTKSIZE + KSTKGAP), kstacktop_i - KSTKSIZE)
	//          -- not backed; so if the kernel overflows its stack,
	//             it will fault rather than overwrite another CPU's stack.
	//             Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	//
	// LAB 4: Your code here:
	for (int i = 0; i < NCPU; i++) {
		boot_map_region(kern_pgdir, KSTACKTOP - KSTKSIZE - i * (KSTKSIZE + KSTKGAP), KSTKSIZE, PADDR(percpu_kstacks[i]), PTE_W);
	}
}
```

#### **Exercise 4.** 

**Trap _ init _ perpu ()(kern/trap.c)中的代码为 BSP 初始化 TSS 和 TSS 描述符。**

**TSS: 任务状态段** 

**CPU i 的 TSS 存储在 cpus [ i ]中。Cpu _ ts，相应的 TSS 描述符在 GDT 条目 GDT [(GD _ TSS0 > > 3) + i ]中定义。**

	// Initialize and load the per-CPU TSS and IDT
	void
	trap_init_percpu(void)
	{
		// The example code here sets up the Task State Segment (TSS) and
		// the TSS descriptor for CPU 0. But it is incorrect if we are
		// running on other CPUs because each CPU has its own kernel stack.
		// Fix the code so that it works for all CPUs.
		//
		// Hints:
		//   - The macro "thiscpu" always refers to the current CPU's
		//     struct CpuInfo;
		//   - The ID of the current CPU is given by cpunum() or
		//     thiscpu->cpu_id;
		//   - Use "thiscpu->cpu_ts" as the TSS for the current CPU,
		//     rather than the global "ts" variable;
		//   - Use gdt[(GD_TSS0 >> 3) + i] for CPU i's TSS descriptor;
		//   - You mapped the per-CPU kernel stacks in mem_init_mp()
		//   - Initialize cpu_ts.ts_iomb to prevent unauthorized environments
		//     from doing IO (0 is not the correct value!)
		//
		// ltr sets a 'busy' flag in the TSS selector, so if you
		// accidentally load the same TSS on more than one CPU, you'll
		// get a triple fault.  If you set up an individual CPU's TSS
		// wrong, you may not get a fault until you try to return from
		// user space on that CPU.
		//
		// LAB 4: Your code here:
	// Setup a TSS so that we get the right stack
	// when we trap to the kernel.
	thiscpu->cpu_ts .ts_esp0 = (uintptr_t) percpu_kstacks[cpunum()];
	thiscpu->cpu_ts.ts_ss0 = GD_KD;
	thiscpu->cpu_ts.ts_iomb = sizeof(struct Taskstate);
	
	// Initialize the TSS slot of the gdt.
	gdt[(GD_TSS0 >> 3) + cpunum()] = SEG16(STS_T32A, (uint32_t) (&(thiscpu->cpu_ts)),
					sizeof(struct Taskstate) - 1, 0);
	gdt[(GD_TSS0 >> 3) + cpunum()].sd_s = 0;
	
	// Load the TSS selector (like other segment selectors, the
	// bottom three bits are special; we leave them 0)
	ltr(GD_TSS0 + (cpunum() << 3));
	
	// Load the IDT
	lidt(&idt_pd);
	}
**主要就是把ts换成thiscpu->cpu_ts**

#### Locking

锁是一种用于控制对共享资源访问的机制。它们是并发编程中常用的工具，用于确保多个线程或进程之间的互斥访问，以避免数据竞争和其他并发问题。

当我们要保护内核的数据结构时，使用一个内核锁还是值得的，当进入内核时必须持有该锁，而退出内核时就释放该锁，但使用这种方法就牺牲了并发性：**即一时间只有一个 CPU 可以运行在内核上。**

#### **Exercise 5.**

在以下四个状态时使用锁机制lock_kernel()和unlock_kernel()

- 在 i386 _ init ()中，在 BSP 唤醒其他 CPU 之前获取锁。
- 在 mp _ main ()中，在初始化 AP 之后获取锁，然后调用 sched _ production ()启动此 AP 上的运行环境。
- 在 trap ()中，在从用户模式捕获时获取锁。要确定陷阱是发生在用户模式还是内核模式下，请检查 tf _ cs 的低位。
- 在 env _ run ()中，在切换到用户模式之前释放锁。不要做得太早或太晚，否则你将经历竞争或死锁。

```
i386_init():
	// Acquire the big kernel lock before waking up APs
	// Your code here:
	lock_kernel();
	// Starting non-boot CPUs
	boot_aps(); 

mp_main():
	// Now that we have finished some basic setup, call sched_yield()
	// to start running processes on this CPU.  But make sure that
	// only one CPU can enter the scheduler at a time!
	//
	// Your code here: 
	lock_kernel();
	sched_yield(); 

trap():
        if ((tf->tf_cs & 3) == 3) {
		// Trapped from user mode.
		// Acquire the big kernel lock before doing any
		// serious kernel work.
		// LAB 4: Your code here.
		lock_kernel();
		assert(curenv);

env_run():
	lcr3(PADDR(e->env_pgdir)); 
	//lab4
	unlock_kernel(); 
	env_pop_tf(&(e->env_tf));//never return
```

#### **Question**

**似乎使用大内核锁保证了一次只有一个 CPU 可以运行内核代码。为什么我们仍然需要为每个 CPU 单独的内核堆栈？描述一个使用共享内核堆栈会出错的场景，即使有大内核锁的保护。**

因为big kernel lock并不能真正保证每次只有一个CPU运行内核代码。比如说如果发生中断，寄存器被push进CPU Stack是在进行锁检查之前发生的，所以不同的CPU之间就会被搞混。

### Round-Robin Scheduling

**时间片轮转（Time Slice Round-Robin）**是一种用于调度多个进程（或线程）的调度算法。在时间片轮转调度中，CPU 将时间分成固定长度的时间片（time slice），然后按照轮流的方式分配给各个进程执行。当一个进程的时间片用完后，调度器会暂停该进程的执行，并将CPU 分配给下一个就绪的进程，直到所有进程都执行完一轮。

在这个模块中只是使用了简单的循环调度。

此部分内容将修改jos内核使之可以以循环机制（round-robin）交替调度多个进程。jos的循环机制如下：
1、kern/sched.c的`sched_yield()`函数负责选择一个新的进程来执行。它从上一个运行进程之后开始，以循环方式顺序搜索envs[]数组，选择第一个状态为`ENV_RUNNABLE`的进程，并调用`env_run()`函数跳转到该进程。
2、`sched_yield()`不能同时在两个cpu上执行调度同一个进程。区分某个进程是否正在某个cpu上运行的方法是：该进程的状态是`ENV_RUNNING`。
3、用户进程可以调用`sys_yield()`系统调用（该函数调用`sched_yield()`函数），自愿放弃cpu资源给其他进程。

#### **Exercise 6.**

**在sched_yield()中实现循环调度。在syscall()中分发sys_yield()。确保在mp_main()中调用sched_yield()。在kern/init.c中创建三个或三个以上进程，执行user/yield.c。在yield程序退出之后，系统将不存在可执行进程，并陷入到monitor中。**



	void
	sched_yield(void)
	{
		// Implement simple round-robin scheduling.
		//
		// Search through 'envs' for an ENV_RUNNABLE environment in
		// circular fashion starting just after the env this CPU was
		// last running.  Switch to the first such environment found.
		//
		// If no envs are runnable, but the environment previously
		// running on this CPU is still ENV_RUNNING, it's okay to
		// choose that environment. Make sure curenv is not null before
		// dereferencing it.
		//
		// Never choose an environment that's currently running on
		// another CPU (env_status == ENV_RUNNING). If there are
		// no runnable environments, simply drop through to the code
		// below to halt the cpu.
	// LAB 4: Your code here.
	size_t start = 0;
	if (curenv) {
		start = ENVX(curenv->env_id) + 1;
	}
	
	for (size_t i = 0; i < NENV; i++) {
		size_t index = (start + i) % NENV;
		if (envs[index].env_status == ENV_RUNNABLE) {
			env_run(&envs[index]);
		}
	}
	// 此时curenv已经切换
	if(curenv && curenv->env_status == ENV_RUNNING) {
		env_run(curenv);
	}
	// sched_halt never returns
	sched_halt();
	}
#### **Question**

1. **env_run()修改了参数e的成员状态之后就会调用lcr3切换到进程的页目录，这时候内存管理单元MMU所使用的寻址上下文立即发生了变化。在地址切换前后，为什么参数e仍能够被引用？**

因为对于所有env来说，内核空间的虚拟地址是相同的，这部分地址也被映射进env的页目录。参数e本身也是内核空间的一部分。

2. **每次内核从一个进程切换到另一个进程的时候，需要保存旧进程的寄存器以便后面恢复，怎么做？什么时候这样做？**

kern/trap.c:curenv->env_tf = *tf保存了当前的trap帧。

### System Calls for Environment Creation

#### **Exercise 7.** 

==**sys_exofork：**==创建一个进程，创建页目录（映射内核空间，UTOP以下部分在页目录中的映射为空）、填充trapframe（包括`env_alloc()`中用户栈指针指向`e->env_tf.tf_esp = USTACKTOP;`，请注意，子进程只有将esp指向USTACKTOP，并没有再次为栈分配空间，而根父进程的栈空间是在`env_create()`的时候通过调用`env_alloc()`之后调用`load_icode()`实现的）、复制父进程的trapframe…，没有用户程序地址映射到地址空间、状态为不可执行。新进程的寄存器状态与调用`sys_exofork`的进程一样。调用`sys_exofork`的父进程返回子进程id，子进程返回0（具体原因可参考x86的fork实现）。

	// Allocate a new environment.
	// Returns envid of new environment, or < 0 on error.  Errors are:
	//	-E_NO_FREE_ENV if no free environment is available.
	//	-E_NO_MEM on memory exhaustion.
	static envid_t
	sys_exofork(void)
	{
		// Create the new environment with env_alloc(), from kern/env.c.
		// It should be left as env_alloc created it, except that
		// status is set to ENV_NOT_RUNNABLE, and the register set is copied
		// from the current environment -- but tweaked so sys_exofork
		// will appear to return 0.
	// LAB 4: Your code here.
	struct Env *child;
	int status = env_alloc(&child, curenv->env_id);
	if (status < 0) return status;
	child->env_tf = curenv->env_tf;
	child->env_status = ENV_NOT_RUNNABLE;
	child->env_tf.tf_regs.reg_eax = 0;		//新的用户环境从sys_exofork()的返回值应该为0
	return child->env_id;
	}
==**sys_env_set_status：**==设置进程状态为`ENV_RUNNABLE`或`ENV_NOT_RUNNABLE`，通常是进程的地址映射和寄存器状态设置完毕之后设置进程可被执行。

	// Set envid's env_status to status, which must be ENV_RUNNABLE
	// or ENV_NOT_RUNNABLE.
	//
	// Returns 0 on success, < 0 on error.  Errors are:
	//	-E_BAD_ENV if environment envid doesn't currently exist,
	//		or the caller doesn't have permission to change envid.
	//	-E_INVAL if status is not a valid status for an environment.
	static int
	sys_env_set_status(envid_t envid, int status)
	{
		// Hint: Use the 'envid2env' function from kern/env.c to translate an
		// envid to a struct Env.
		// You should set envid2env's third argument to 1, which will
		// check whether the current environment has permission to set
		// envid's status.
	// LAB 4: Your code here.
	struct Env *env;
	int state = envid2env(envid, &env, 1);
	if (state < 0) return state;
	if (status != ENV_NOT_RUNNABLE && status != ENV_RUNNABLE) {
		return -E_INVAL;
	}
	
	env->env_status = status;
	return 0;
	}
==**sys_page_alloc：**==分配一页物理页，并在指定进程的地址空间上映射到给定的虚拟地址。

	// Allocate a page of memory and map it at 'va' with permission
	// 'perm' in the address space of 'envid'.
	// The page's contents are set to 0.
	// If a page is already mapped at 'va', that page is unmapped as a
	// side effect.
	//
	// perm -- PTE_U | PTE_P must be set, PTE_AVAIL | PTE_W may or may not be set,
	//         but no other bits may be set.  See PTE_SYSCALL in inc/mmu.h.
	//
	// Return 0 on success, < 0 on error.  Errors are:
	//	-E_BAD_ENV if environment envid doesn't currently exist,
	//		or the caller doesn't have permission to change envid.
	//	-E_INVAL if va >= UTOP, or va is not page-aligned.
	//	-E_INVAL if perm is inappropriate (see above).
	//	-E_NO_MEM if there's no memory to allocate the new page,
	//		or to allocate any necessary page tables.
	static int
	sys_page_alloc(envid_t envid, void *va, int perm)
	{
		// Hint: This function is a wrapper around page_alloc() and
		//   page_insert() from kern/pmap.c.
		//   Most of the new code you write should be to check the
		//   parameters for correctness.
		//   If page_insert() fails, remember to free the page you
		//   allocated!
	// LAB 4: Your code here.
	struct Env *env;
	int state = envid2env(envid, &env, 1);
	if (state < 0) return state;
	
	if ((size_t) va >= UTOP || ((size_t) va % PGSIZE) != 0) return -E_INVAL;
	if ((perm & PTE_U) != PTE_U || (perm & PTE_P) != PTE_P) return -E_INVAL;
	
	struct PageInfo *pp = page_alloc(1);
	if (pp == NULL) return -E_NO_MEM;
	if (page_insert(env->env_pgdir, pp, va, perm) < 0) {
		page_free(pp);
		return -E_NO_MEM;
	}
	
	return 0;
	}
==**sys_page_map：**==将A进程中映射到虚拟地址`va_a`的物理页映射到B进程的虚拟地址`va_b`，设置给出的权限。从而两个进程可以以不同的权限访问同一个物理页。

	// Map the page of memory at 'srcva' in srcenvid's address space
	// at 'dstva' in dstenvid's address space with permission 'perm'.
	// Perm has the same restrictions as in sys_page_alloc, except
	// that it also must not grant write access to a read-only
	// page.
	//
	// Return 0 on success, < 0 on error.  Errors are:
	//	-E_BAD_ENV if srcenvid and/or dstenvid doesn't currently exist,
	//		or the caller doesn't have permission to change one of them.
	//	-E_INVAL if srcva >= UTOP or srcva is not page-aligned,
	//		or dstva >= UTOP or dstva is not page-aligned.
	//	-E_INVAL is srcva is not mapped in srcenvid's address space.
	//	-E_INVAL if perm is inappropriate (see sys_page_alloc).
	//	-E_INVAL if (perm & PTE_W), but srcva is read-only in srcenvid's
	//		address space.
	//	-E_NO_MEM if there's no memory to allocate any necessary page tables.
	static int
	sys_page_map(envid_t srcenvid, void *srcva,
		     envid_t dstenvid, void *dstva, int perm)
	{
		// Hint: This function is a wrapper around page_lookup() and
		//   page_insert() from kern/pmap.c.
		//   Again, most of the new code you write should be to check the
		//   parameters for correctness.
		//   Use the third argument to page_lookup() to
		//   check the current permissions on the page.
	// LAB 4: Your code here.
	if ((size_t) srcva >= UTOP || ((size_t) srcva % PGSIZE) != 0 || (size_t) dstva >= UTOP || ((size_t) dstva % PGSIZE) != 0) {
		return -E_INVAL;
	}
	
	struct Env *senv, *denv;
	int state1, state2;
	state1 = envid2env(srcenvid, &senv, 1);
	state2 = envid2env(dstenvid, &denv, 1);
	if (state1 < 0) return state1;
	if (state2 < 0) return state2;
	
	pte_t *pte;
	struct PageInfo *pp = page_lookup(senv->env_pgdir, srcva, &pte);
	if (pp == NULL) return -E_INVAL;
	
	if (((*pte & PTE_W) == 0) && (perm & PTE_W)) return -E_INVAL;
	
	return page_insert(denv->env_pgdir, pp, dstva, perm);
	}

==**sys_page_unmap：**==在指定进程的地址空间上解除给定的虚拟地址上的映射。

	// Unmap the page of memory at 'va' in the address space of 'envid'.
	// If no page is mapped, the function silently succeeds.
	//
	// Return 0 on success, < 0 on error.  Errors are:
	//	-E_BAD_ENV if environment envid doesn't currently exist,
	//		or the caller doesn't have permission to change envid.
	//	-E_INVAL if va >= UTOP, or va is not page-aligned.
	static int
	sys_page_unmap(envid_t envid, void *va)
	{
		// Hint: This function is a wrapper around page_remove().
	// LAB 4: Your code here.
	struct Env *env;
	if (envid2env(envid, &env, 1) < 0) return -E_BAD_ENV;
	if ((size_t) va >= UTOP || ((size_t) va % PGSIZE) != 0) {
		return -E_INVAL;
	}
	
	page_remove(env->env_pgdir, va);
	return 0;
	}
## Part B: Copy-on-Write Fork

(copy-on-write): 父进程将地址空间映射复制到子进程，允许父进程和子进程共享映射到各自地址空间的内存，直到其中一个进程实际修改它。

当两个进程中的一个尝试写入这些共享页面中的一个时，该进程将出现页面错误。此时，Unix 内核意识到该页面实际上是一个“虚拟”或“即写即复制”的副本，因此它为错误进程创建一个新的、私有的、可写的页面副本。这样，各个页面的内容实际上不会被复制，直到它们被实际写入。

### User-level page fault handling

用户级的 write-on-write fork ()需要了解写保护页面上的页面错误，所以这是首先要实现的。在写时复制只是用户级页面错误处理的许多可能用途之一。

常见的做法是设置一个地址空间，以便页面错误指示何时需要执行某些操作。

#### Setting the Page Fault Handler

#### **Exercise 8.** 

为了处理自己的页面错误，用户环境需要向 JOS 内核注册一个页面错误处理程序入口点。用户环境通过新的 sys _ env _ set _ pgfault _ upcall 系统调用注册其页面错误入口点。我们在 Env 结构中添加了一个新成员 Env _ pgfault _ upcall 来记录这些信息。

```
// Set the page fault upcall for 'envid' by modifying the corresponding struct
// Env's 'env_pgfault_upcall' field.  When 'envid' causes a page fault, the
// kernel will push a fault record onto the exception stack, then branch to
// 'func'.
//
// Returns 0 on success, < 0 on error.  Errors are:
//	-E_BAD_ENV if environment envid doesn't currently exist,
//		or the caller doesn't have permission to change envid.
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
	// LAB 4: Your code here.
	struct Env *env;
	int state = envid2env(envid, &env, 1);
	if (state < 0) return state;
	env->env_pgfault_upcall = func;
	return 0;
}
```

#### Normal and Exception Stacks in User Environments

**用户正常堆栈**数据驻留在 **USTACKTOP-pgSIZE** 和 **USTACKTOP-1**之间的页面上。

当在用户模式下出现页面错误时，内核将重新启动用户环境，在不同的堆栈(即用户异常堆栈)上运行指定的用户级页面错误处理程序。

**用户异常堆栈**的有效字节来自 **UXSTACKTOP-PGSIZE** 到 **UXSTACKTOP-1**(包括 UXSTACKTOP-1)。

![image-20240219102706384](C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240219102706384.png)

#### Invoking the User Page Fault Handler

如果没有注册页面错误处理程序，那么 JOS 内核将像前面一样使用消息销毁用户环境。否则，内核将在异常堆栈上设置一个trap框架，该框架看起来类似于

`inc/trap.h`:

```
                    <-- UXSTACKTOP
trap-time esp
trap-time eflags
trap-time eip
trap-time eax       start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi       end of struct PushRegs
tf_err (error code)
fault_va            <-- %esp when handler is run
```

内核在异常栈上设置struct UTrapframe（见inc/trap.h），UTrapframe的值主要来源于内核栈的trap frame。之后，内核安排用户级页错误处理程序在异常栈上执行。

如果发生页错误的时候用户进程已经在异常栈上执行（`tf->tf_esp`在**[UXSTACKTOP-PGSIZE, UXSTACKTOP-1]**内），说明页错误处理程序本身触发了页错误。这种时候应该在`tf->tf_esp`而不是`UXSTACKTOP`的以下部分存储UTrapframe，而且需要先留空4个字节，再存储UTrapframe(预留4个字节将指向utf的esp压入栈内，这样就可以作为参数传递给后来的函数。)

#### **Exercise 9.** 

**在 kern/trap.c 中实现 page _ fault _ handler 中的代码，这是将页面错误分派给用户模式处理程序所必需的。**

将出现页错误的进程使用参数提供的trap框架 并将trap框架压入用户异常堆栈中 当前进程esp指针指向用户异常堆栈 当前进程的eip指针指向系统调用sys_env_set_pgfault_upcall()中的参数fuc函数来运行处理函数

	void
	page_fault_handler(struct Trapframe *tf)
	{
		uint32_t fault_va;
	// Read processor's CR2 register to find the faulting address
	fault_va = rcr2();
	
	// Handle kernel-mode page faults.
	
	// LAB 3: Your code here.
	if ((tf->tf_cs & 3) == 0) {
		panic("page_fault_handler: system page fault.\n");
	}
	
	// LAB 4: Your code here.
	if (curenv->env_pgfault_upcall) {
		size_t stacktop = UXSTACKTOP;
		if (tf->tf_esp >= UXSTACKTOP - PGSIZE && tf->tf_esp < UXSTACKTOP) {
			stacktop = tf->tf_esp - sizeof(size_t);
		}
		
		struct UTrapframe *utf = (struct UTrapframe *) (stacktop - sizeof(struct UTrapframe));
		user_mem_assert(curenv, (void *) utf, sizeof(struct UTrapframe), PTE_W | PTE_U | PTE_P);
		utf->utf_eflags = tf->tf_eflags;
		utf->utf_eip = tf->tf_eip;
		utf->utf_err = tf->tf_err;
		utf->utf_esp = tf->tf_esp;
		utf->utf_fault_va = fault_va;
		utf->utf_regs = tf->tf_regs;
	
		curenv->env_tf.tf_esp = (uintptr_t) utf;
		curenv->env_tf.tf_eip = (uintptr_t) curenv->env_pgfault_upcall;
		env_run(curenv);
	}
	
	// Destroy the environment that caused the fault.
	cprintf("[%08x] user fault va %08x ip %08x\n",
		curenv->env_id, fault_va, tf->tf_eip);
	print_trapframe(tf);
	env_destroy(curenv);
	}

#### User-mode Page Fault Entrypoint

#### **Exercise 10.** 

**在 lib/pfentry.S 中实现 _ pgfault _ upcall 。触发页错误时内核实际上是跳转执行该文件声明的入口_pgfault_upcall。pfentry.S主要是在用户态的page_fault_handler结束后如何恢复现场并跳回原程序执行。**

​		`addl $4, %esp			# pop function argument  `  这里将esp指针加上4个字节 原因是在page_fault_handler函数中预留4个字节将指向utf的esp压入栈内，这样就可以作为参数传递给后来的函数。

	.text
	.globl _pgfault_upcall
	_pgfault_upcall:
	
	# Call the C page fault handler.
	
	​	pushl %esp			# function argument: pointer to UTF
	​	movl _pgfault_handler, %eax
	​	call *%eax
	​	addl $4, %esp			# pop function argument
	# Now the C page fault handler has returned and you must return
	# to the trap time state.
	# Push trap-time %eip onto the trap-time stack.
	#
	# Explanation:
	#   We must prepare the trap-time stack for our eventual return to
	#   re-execute the instruction that faulted.
	#   Unfortunately, we can't return directly from the exception stack:
	#   We can't call 'jmp', since that requires that we load the address
	#   into a register, and all registers must have their trap-time
	#   values after the return.
	#   We can't call 'ret' from the exception stack either, since if we
	#   did, %esp would have the wrong value.
	#   So instead, we push the trap-time %eip onto the *trap-time* stack!
	#   Below we'll switch to that stack and call 'ret', which will
	#   restore %eip to its pre-fault value.
	#
	#   In the case of a recursive fault on the exception stack,
	#   note that the word we're pushing now will fit in the
	#   blank word that the kernel reserved for us.
	#
	# Throughout the remaining code, think carefully about what
	# registers are available for intermediate calculations.  You
	# may find that you have to rearrange your code in non-obvious
	# ways as registers become unavailable as scratch space.
	#
	# LAB 4: Your code here.
	movl 40(%esp), %eax # eip
	movl 48(%esp), %ecx # esp
	movl %eax, -4(%ecx) # 将原来的 eip 放入到原来的用户栈
	# Restore the trap-time registers.  After you do this, you
	# can no longer modify any general-purpose registers.
	# LAB 4: Your code here.
	popl %eax
	popl %eax # 跳过 fault_va 和 err
	popal
	# Restore eflags from the stack.  After you do this, you can
	# no longer use arithmetic operations or anything else that
	# modifies eflags.
	# LAB 4: Your code here.
	addl $4, %esp #跳过eip
	popfl
	# Switch back to the adjusted trap-time stack.
	# LAB 4: Your code here.
	popl %esp # 恢复到用户栈 USTACKTOP 在此之前都是UXSTACKTOP
	lea -4(%esp), %esp # 刚才没减去栈的 4 个字节 现在补上
	# Return to re-execute the instruction that faulted.
	# LAB 4: Your code here.
	ret

#### **Exercise 11.** 

**在 lib/pgfault.c 中完成 set _ pgfault _ handler ()。**

	void
	set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
	{
		int r;
	if (_pgfault_handler == 0) {
		// First time through!
		// LAB 4: Your code here.
		if ((r = sys_page_alloc(thisenv->env_id, (void *) (UXSTACKTOP - PGSIZE), PTE_W | PTE_U | PTE_P)) < 0) {
			panic("set_pgfault_handler: %e\n", r);
		}
	
		if ((r = sys_env_set_pgfault_upcall(thisenv->env_id, _pgfault_upcall)) < 0) {
			panic("set_pgfault_handler: %e\n", r);
		}
	}
	
	// Save handler pointer for assembly to call.
	_pgfault_handler = handler;
	}
**用户级别的页错误处理流程如下：**

1、用户程序通过`lib/pgfault.c`的`set_pgfault_handler()`函数注册页错误处理函数入口`_pgfault_upcall`（见lib/pfentry.S），并指定页错误处理函数`_pgfault_handler`，该函数指针将在`_pgfault_upcall`中被使用。第一次注册`_pgfault_upcall`时将申请分配并映射异常栈。
2、用户程序触发页错误，切换到内核模式。
3、内核检查页错误处理程序是否已设置、异常栈指针是否越界、是否递归触发页错误、是否已为异常栈分配物理页，然后在异常栈上存储UTrapframe，将栈指针指向异常栈，并跳转执行当前进程的`env_pgfault_upcall`（被设为用户级页错误处理程序入口`_pgfault_upcall`）。
4、用户模式下`_pgfault_upcall`调用`_pgfault_upcall`，执行用户级别页错误处理函数，在用户态的`page_fault_handler`结束后恢复现场并跳回原程序执行。

### Implementing Copy-on-Write Fork

jos使用Copy-on-Write Fork时，将会扫描父进程的整个地址空间，并设置子进程的相关页映射，但不会复制页内容（dumpfork()复制页内容）。当父/子进程尝试写入某一页时，将申请新的一页并拷贝页内容到该页上。
**fork()的基本流程如下：**
1、父进程调用`set_pgfault_handler()`注册页错误处理函数`pgfault()`。
2、父进程调用`sys_exofork()`创建子进程。
3、对于[UTEXT,UTOP)每一个可写或写时复制的页，父进程调用duppage分别在子进程和自身的地址空间内映射该页为写时复制`PTE_COW`。

**注意：**不能将user exception stack映射为写时复制，而是应该为父子进程分别映射对应的物理页。这是因为页错误处理程序将在user exception stack上执行，当将user exception stack设为写时复制时，一旦发生页错误，内核将尝试往user exception stack写数据，由于user exception stack不可写而导致失败。
4、**父进程为子进程注册页错误处理函数。**
注意：**我们不在子进程中注册页错误处理函数。**首先，我们不在子进程中调用`set_pgfault_handler()`（该函数注册页错误处理函数，并在页错误处理函数未注册的情况下申请user exception stack），因为子进程是由父进程fork而来的，而父进程已经注册过页错误处理函数，所以在子进程中注册页错误处理函数的话子进程不会再申请user exception stack（`if (_pgfault_handler == 0)`条件判断失败)。其次，我们不在子进程中调用`sys_page_alloc()`和`sys_env_set_pgfault_upcall()`申请user exception stack，这是因为这涉及了函数调用，而父进程将子进程的用户栈映射为写时复制了，所以会触发页错误，但是此时还没有分配user exception stack，因此出错。
5、设置子进程状态为runnable。

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240220162721308.png" alt="image-20240220162721308" style="zoom:67%;" />

**页错误处理函数的处理流程如下：**
1、内核跳转执行`_pgfault_upcall`，`_pgfault_upcall`将调用页错误处理函数pgfault()。
2、pgfault()确认fault是可写的（错误码的FEC_WR），并且引起页错误的虚拟地址对应的页表项权限为`PTE_COW`。
3、pgfault()为申请新的一页，映射到一个临时地址，将faulting page的内容复制到该新页，调用`sys_page_map`将映射到临时地址的物理页映射到引起页错误的虚拟地址，设置权限为可写。

#### **Exercise 12.**

**实现lib/fork.c的fork、duppage、pgfault。**

==**pgfault():**==

这个函数判断当前的缺页类型是不是FEC_WR，当前的页是不是PTE_COW。然后将要写的页的内容复制到PFTEMMP，再把地址映射到PFTEMP。其中PFTEMP用作储存物理页的一个临时虚拟地址。

	static void
	pgfault(struct UTrapframe *utf)
	{
		void *addr = (void *) utf->utf_fault_va;
		uint32_t err = utf->utf_err;
		int r;
	
	// LAB 4: Your code here.
	if (!((err & FEC_WR) && (uvpt[PGNUM(addr)] & PTE_COW))) {
	    panic("pgfault: not copy-on-write\n");
	}
	
	// LAB 4: Your code here.
	addr = ROUNDDOWN(addr, PGSIZE);
	if ((r = sys_page_map(0, addr, 0, (void *) PFTEMP, PTE_U | PTE_P)) < 0) {
		panic("pgfault: %e\n", r);
	}
	
	if ((r = sys_page_alloc(0, addr, PTE_U | PTE_P | PTE_W)) < 0) {
		panic("pgfault: %e\n", r);
	}
	
	memmove(addr, PFTEMP, PGSIZE);								//将PFTEMP指向的物理页拷贝到addr指向的物理页
	
	if ((r = sys_page_unmap(0, (void *) PFTEMP)) < 0) {
		panic("pgfault: %e\n", r);
	}
	}
==**dumppage()**==

这里需要检查PTE_W, PTE_COW。如果当前页不是用户可写且不是COW，那就不改属性直接映射，如果是用户可写的就将属性修改为PTE_COW映射两次。父子进程之间是完全隔离的，所以两个进程都要做这个事情。

```
static int
duppage(envid_t envid, unsigned pn)
{
  int r;

  void *addr = (void*) (pn*PGSIZE);
  if ((uvpt[pn] & PTE_W) || (uvpt[pn] & PTE_COW)) {
      r = sys_page_map(0, addr, envid, addr, PTE_COW|PTE_U|PTE_P);
      if(r < 0)
          return r;
      // 在子进程写addr之前，如果父进程改变p的内容
      // 就会发生copy on write，这样就不会改变原来那一页的内容
      // 于是子进程的内容也就不会改变。
      r = sys_page_map(0, addr, 0, addr, PTE_COW|PTE_U|PTE_P);
      if(r < 0)
          return r;
  }
  else sys_page_map(0, addr, envid, addr, PTE_U|PTE_P);
  return 0;
  panic("duppage not implemented");
}
```

==**fork()**==

- **uvpd数组相当于包含1KB个页目录项的数组**
- **uvpt数组相当于包含1MB个页表项的数组**

	envid_t
	fork(void)
	{
		// LAB 4: Your code here.
		
		// 1、父进程调用`set_pgfault_handler()`注册页错误处理函数`pgfault()`。
		// 2、父进程调用`sys_exofork()`创建子进程。
		
		extern void _pgfault_upcall(void);
		set_pgfault_handler(pgfault);
		envid_t envid = sys_exofork();
		if (envid < 0)
			panic("sys_exofork: %e", envid);
		if (envid == 0) {
			// We're the child.
			// The copied value of the global variable 'thisenv'
			// is no longer valid (it refers to the parent!).
			// Fix it and return 0.
			thisenv = &envs[ENVX(sys_getenvid())];
			return 0;
		}
		
		// 3、对于[UTEXT,UTOP)每一个可写或写时复制的页，父进程调用duppage分别在子进程和自身的地址空间内映射该页为写时复制`PTE_COW`。
		
	uint32_t addr;
	for (addr = 0; addr < USTACKTOP; addr += PGSIZE) {
	
		// uvpd是有1024个pde的一维数组，而uvpt是有2^20个pte的一维数组,与物理页号刚好一一对应
		
		if ((uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_U)) {
	        duppage(envid, PGNUM(addr)); 
	    }
	}
	
	int r;
	if ((r = sys_page_alloc(envid, (void *) (UXSTACKTOP - PGSIZE), PTE_U | PTE_W | PTE_P)) < 0)
		panic("sys_page_alloc: %e", r);
		
	// 4、**父进程为子进程注册页错误处理函数。**
	
	sys_env_set_pgfault_upcall(envid, _pgfault_upcall);
	// Start the child environment running
	if ((r = sys_env_set_status(envid, ENV_RUNNABLE)) < 0)
		panic("sys_env_set_status: %e", r);
	
	return envid;
	}
## Part C: Preemptive Multitasking and Inter-Process communication (IPC)

### Clock Interrupts and Preemption

user/spin程序中子进程一旦获取CPU将永远嵌入循环。为了保证内核能够抢占正在运行的进程，需要扩展内核以支持时钟硬件的外部硬件中断。

#### Interrupt discipline

外部中断被称为IRQs，共有16个，编号为0-15。IRQs到IDT表项的映射不是固定的，`picirq.c`的`pic_init`通过`IRQ_OFFSET`-`IRQ_OFFSET+15`将IRQs 0-15映射到IDT项中。在inc/trap.h，`IRQ_OFFSET`被定义为32，所以IDT的32-47项映射到IRQs 0-15。
在jos中，一个关键的简化是一旦处于内核模式就禁用了外部设备中断。外部中断由eflags的`FL_IF`位控制。虽然有多种方式可以修改该位，但由于jos的简化处理，我们仅需要在进入和退出用户模式的时候通过保存和恢复eflags寄存器来处理它即可。

#### **Exercise 13.**

**修改kern/trapentry.S和kern/trap.c，初始化相关的IDT项，并提供IRQs 0-15的处理函数。然后修改kern/env.c的env_alloc()，在允许外部中断的情况下执行用户进程。当调用硬件中断处理函数的时候，处理器不会将错误码进栈，也不会检查IDT项的DPL。**

**trapentry.S**

```
TRAPHANDLER_NOEC(irq_0_handler,  IRQ_OFFSET + 0);
TRAPHANDLER_NOEC(irq_1_handler,  IRQ_OFFSET + 1);
TRAPHANDLER_NOEC(irq_2_handler,  IRQ_OFFSET + 2);
TRAPHANDLER_NOEC(irq_3_handler,  IRQ_OFFSET + 3);
TRAPHANDLER_NOEC(irq_4_handler,  IRQ_OFFSET + 4);
TRAPHANDLER_NOEC(irq_5_handler,  IRQ_OFFSET + 5);
TRAPHANDLER_NOEC(irq_6_handler,  IRQ_OFFSET + 6);
TRAPHANDLER_NOEC(irq_7_handler,  IRQ_OFFSET + 7);
TRAPHANDLER_NOEC(irq_8_handler,  IRQ_OFFSET + 8);
TRAPHANDLER_NOEC(irq_9_handler,  IRQ_OFFSET + 9);
TRAPHANDLER_NOEC(irq_10_handler, IRQ_OFFSET + 10);
TRAPHANDLER_NOEC(irq_11_handler, IRQ_OFFSET + 11);
TRAPHANDLER_NOEC(irq_12_handler, IRQ_OFFSET + 12);
TRAPHANDLER_NOEC(irq_13_handler, IRQ_OFFSET + 13);
TRAPHANDLER_NOEC(irq_14_handler, IRQ_OFFSET + 14);
TRAPHANDLER_NOEC(irq_15_handler, IRQ_OFFSET + 15);
```

trap.c

	void
	trap_init(void)
	{
		extern struct Segdesc gdt[];
	// LAB 3: Your code here.
	void divide_error_handler();
	void debug_exception_handler();
	void non_maskable_interrupt_handler();
	void breakpoint_handler();
	void overflow_handler();
	void bounds_check_handler();
	void invalid_opcode_handler();
	void device_not_available_handler();
	void double_fault_handler();
	void invalid_tss_handler();
	void segment_not_present_handler();
	void stack_exception_handler();
	void general_protection_fault_handler();
	void pagefault_handler();
	void floating_point_error_handler();
	void alignment_check_handler();
	void machine_check_handler();
	void simd_floating_point_error_handler();
	void system_call();
	
	void irq_0_handler();
	void irq_1_handler();
	void irq_2_handler();
	void irq_3_handler();
	void irq_4_handler();
	void irq_5_handler();
	void irq_6_handler();
	void irq_7_handler();
	void irq_8_handler();
	void irq_9_handler();
	void irq_10_handler();
	void irq_11_handler();
	void irq_12_handler();
	void irq_13_handler();
	void irq_14_handler();
	void irq_15_handler();
	
	// set up trap gate descriptor
	SETGATE(idt[T_DIVIDE],	 0, GD_KT, divide_error_handler,           	   0);
	SETGATE(idt[T_DEBUG],    0, GD_KT, debug_exception_handler,            0);
	SETGATE(idt[T_NMI],      0, GD_KT, non_maskable_interrupt_handler,     0);
	SETGATE(idt[T_BRKPT],    0, GD_KT, breakpoint_handler,                 0);
	SETGATE(idt[T_OFLOW],    0, GD_KT, overflow_handler,                   0);
	SETGATE(idt[T_BOUND],    0, GD_KT, bounds_check_handler,               0);
	SETGATE(idt[T_ILLOP],    0, GD_KT, invalid_opcode_handler,             0);
	SETGATE(idt[T_DEVICE],   0, GD_KT, device_not_available_handler,       0);
	SETGATE(idt[T_DBLFLT],   0, GD_KT, double_fault_handler,               0);
	SETGATE(idt[T_TSS],      0, GD_KT, invalid_tss_handler,                0);
	SETGATE(idt[T_SEGNP],    0, GD_KT, segment_not_present_handler,        0);
	SETGATE(idt[T_STACK],    0, GD_KT, stack_exception_handler,            0);
	SETGATE(idt[T_GPFLT],    0, GD_KT, general_protection_fault_handler,   0);
	SETGATE(idt[T_PGFLT],    0, GD_KT, pagefault_handler,                  0);
	SETGATE(idt[T_FPERR],    0, GD_KT, floating_point_error_handler,       0);
	SETGATE(idt[T_ALIGN],    0, GD_KT, alignment_check_handler,            0);
	SETGATE(idt[T_MCHK],     0, GD_KT, machine_check_handler,              0);
	SETGATE(idt[T_SIMDERR],  0, GD_KT, simd_floating_point_error_handler,  0);
	SETGATE(idt[T_SYSCALL],  0, GD_KT, system_call,  3);
	
	SETGATE(idt[IRQ_OFFSET + 0],  0, GD_KT, irq_0_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 1],  0, GD_KT, irq_1_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 2],  0, GD_KT, irq_2_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 3],  0, GD_KT, irq_3_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 4],  0, GD_KT, irq_4_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 5],  0, GD_KT, irq_5_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 6],  0, GD_KT, irq_6_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 7],  0, GD_KT, irq_7_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 8],  0, GD_KT, irq_8_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 9],  0, GD_KT, irq_9_handler,  0);
	SETGATE(idt[IRQ_OFFSET + 10], 0, GD_KT, irq_10_handler, 0);
	SETGATE(idt[IRQ_OFFSET + 11], 0, GD_KT, irq_11_handler, 0);
	SETGATE(idt[IRQ_OFFSET + 12], 0, GD_KT, irq_12_handler, 0);
	SETGATE(idt[IRQ_OFFSET + 13], 0, GD_KT, irq_13_handler, 0);
	SETGATE(idt[IRQ_OFFSET + 14], 0, GD_KT, irq_14_handler, 0);
	SETGATE(idt[IRQ_OFFSET + 15], 0, GD_KT, irq_15_handler, 0);
	
	// Per-CPU setup 
	trap_init_percpu();
	}
**env_alloc()**

```
// Enable interrupts while in user mode.
	// LAB 4: Your code here.
	e->env_tf.tf_eflags |= FL_IF;
```

#### Handling Clock Interrupts

`lapic_init`和`pic_init`进行了时钟和中断的相关设置，以生成中断。需要对中断进行相关处理。

#### **Exercise 14.** 

**修改trap_dispatch()函数，在每次触发时间中断的时候调用sched_yield()函数切换执行其他进程。测试案例为user/spin。**

```
// Handle clock interrupts. Don't forget to acknowledge the
	// interrupt using lapic_eoi() before calling the scheduler!
	// LAB 4: Your code here.
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
		if (cpunum() == 0) time_tick();
        lapic_eoi();
        // 调用调度器的sched_yield()函数，将当前执行的进程放弃CPU时间片，以便其他进程可以运行。
        sched_yield();
    }
```

### Inter-Process communication (IPC)

进程隔离和进程通信是两个重要课题，Unix的管道模型是典型的进程通信例子。进程间有许多通信模型，jos将实现一种简单的IPC机制。

#### IPC in JOS

jos实现的IPC机制是扩展系统调用接口`sys_ipc_recv`和`sys_ipc_try_send`，并实现库函数`ipc_recv`和`ipc_send`。jos的进程间通信信息包括两个部分：一个32位数和一个可选的页面映射。允许传递页面映射的做法一方面可以传递更多数据，一方面进程可以更方便地设置和安排内存共享。

#### Sending and Receiving Messages

进程希望接收消息时，调用`sys_ipc_recv`。该系统调用取消进程的执行，直到接收到消息。如果一个进行在等待接收消息，任何进程都可以发消息给它，而不局限于特定进程或有父子关系的进程。因此，Part A的权限检查不适用于IPC，因为IPC的设计确保进程A发送消息给进程B不能导致进程B发生故障。
进程希望发送消息时，调用`sys_ipc_try_send`，并传递目标进程号和参数。如果目标进程调用了`sys_ipc_recv`并且还没收到消息，系统调用将递交消息并返回0，否则返回`-E_IPC_NOT_RECV`。

#### Transferring Pages

当进程携带`dstva`参数调用`sys_ipc_recv`时，表明它希望收到一个页映射。这样当发送者发送一个页时，接收者需要将`dstva`映射到自己的地址空间（如果原来已经有了页映射，则解除原来的映射）。
当进程携带`srcva`参数调用`sys_ipc_try_send`时，表明它希望将当前映射到`srcva`的页发送出去，权限是`perm`。发送成功后，发送者保持`srcva`的原有映射，接收者则有了新的页映射。从而达到共享页的目的。

#### Implementing IPC

#### **Exercise 15.**

**实现kern/syscall.c的sys_ipc_recv和sys_ipc_try_send。调用envid2env的时候将checkperm设为0，表示任何进程都可以发送IPC消息。实现lib/ipc.c的ipc_recv和ipc_send。测试案例为user/pingpong和user/primes。**

**进程在等待接受消息时为阻塞状态 若有另一个进程发送消息到该消息 这时该进程才会进入等待状态**

**sys_ipc_recv()**

`sys_ipc_recv`负责将给定的虚拟地址设置到env结构体的`env_ipc_dstva`上，表示希望发送者将页映射到其上，然后更改接收信息标识为等待接收，并更改进程的运行状态为不可执行。该系统调用将不会返回到用户模式，直至有发送者发送消息给它并解除其阻塞状态。

```
// Block until a value is ready.  Record that you want to receive
// using the env_ipc_recving and env_ipc_dstva fields of struct Env,
// mark yourself not runnable, and then give up the CPU.
//
// If 'dstva' is < UTOP, then you are willing to receive a page of data.
// 'dstva' is the virtual address at which the sent page should be mapped.
//
// This function only returns on error, but the system call will eventually
// return 0 on success.
// Return < 0 on error.  Errors are:
//	-E_INVAL if dstva < UTOP but dstva is not page-aligned.
static int
sys_ipc_recv(void *dstva)
{
	// LAB 4: Your code here.
	if ((size_t) dstva < UTOP && ((size_t) dstva % PGSIZE) != 0) return -E_INVAL;
	curenv->env_ipc_recving = 1;
	curenv->env_ipc_dstva = dstva;
	curenv->env_status = ENV_NOT_RUNNABLE;
	sys_yield();
	return 0;
}
```

**sys_ipc_try_send**

sys_ipc_try_send负责检查目标进程是否存在、目标进程是否准备好接收信息、目标进程是否要求接收页映射。如果目标进程要求接收页映射，检查发送者提供的虚拟地址是否有效并页对齐、发送者提供的映射权限是否合理、发送者提供的虚拟地址是否有映射页、是否尝试将只读页映射为可写、是否尚有物理页可供接收者分配和映射，检查通过后将物理页映射到目标进程指定的虚拟地址上。最后设置目标进程env结构体相关参数，以说明由谁发送、接收的值是什么，更改目标进程接收信息标识为已接收，将目标进程设为可执行，解除其阻塞状态。

	static int
	sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
	{
		// LAB 4: Your code here.
		struct Env *env;
		if (envid2env(envid, &env, 0) < 0) return -E_BAD_ENV;
		if (!env->env_ipc_recving) return -E_IPC_NOT_RECV;
	if ((size_t) srcva < UTOP) {
		if (((size_t) srcva % PGSIZE) != 0) {
			return -E_INVAL;
		}
	
		if ((perm & PTE_U) != PTE_U || (perm & PTE_P) != PTE_P) return -E_INVAL;
	
		pte_t *pte;
		struct PageInfo *pp = page_lookup(curenv->env_pgdir, srcva, &pte);
		if (!pp) return -E_INVAL;
	
		if ((perm & PTE_W) && ((size_t) *pte & PTE_W) != PTE_W) return -E_INVAL;
		if ((size_t) env->env_ipc_dstva < UTOP) {
			if (page_insert(env->env_pgdir, pp, env->env_ipc_dstva, perm) < 0) return -E_NO_MEM;
			env->env_ipc_perm = perm;
		}
	} else {
		env->env_ipc_perm = 0;
	}
	
	env->env_ipc_from = curenv->env_id;
	env->env_ipc_recving = 0;
	env->env_ipc_value = value;
	env->env_status = ENV_RUNNABLE;
	env->env_tf.tf_regs.reg_eax = 0;
	return 0;
	}
**ipc_recv()**

`ipc_recv`负责调用`sys_ipc_recv`，如果pg为空，代表接收者不希望接收页映射，否则表示希望发送者将页映射到pg上。调用`sys_ipc_recv`之后进程状态被更改为不可执行、阻塞，直至有进程发送消息给它并将其状态更改为可执行。接收到消息之后，存储发送者进程id以及映射权限，然后返回接收到的32位的ipc值。

	int32_t
	ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
	{
		// LAB 4: Your code here.
		if (!pg) pg = (void *)-1;
		int state = sys_ipc_recv(pg);
		if (state < 0) {
			if (from_env_store) *from_env_store = 0;
			if (perm_store) *perm_store = 0;
			return state;
		}
	if (from_env_store) *from_env_store = thisenv->env_ipc_from;
	if (perm_store) *perm_store = thisenv->env_ipc_perm;
	return thisenv->env_ipc_value;
	}
**ipc_send()**

`ipc_send`负责调用`sys_ipc_try_send`，如果调用失败（如目标进程尚未准备好接收）则循环尝试，但为了不霸占资源，在循环体内调用`sys_yield`由内核选择进程的执行。目标进程尚未准备好接收的情况下尝试循环send，对其他返回的错误直接panic。

	ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
	{
		// LAB 4: Your code here.
		if (!pg) {
			pg = (void *)-1;
			perm = 0;
		}
	int state;
	while (1) {
		state = sys_ipc_try_send(to_env, val, pg, perm);
		if (state == 0) return;
		if (state != -E_IPC_NOT_RECV) {
			panic("ipc_send: %e", state);
		}
	
		sys_yield();
	}
	}
