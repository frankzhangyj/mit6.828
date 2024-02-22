# Lab 5: File system, Spawn and Shell

# File system preliminaries

jos操作系统是单用户的，不提供多用户的保护。因此，jos文件系统不支持UNIX文件所有权和权限的概念，不支持硬链接、符号链接、时间戳或特殊设备文件。

## On-Disk File System Structure

大多数unix文件系统将**磁盘空间**分为两种区域类型：**inode regions**和**data regions**。unix文件系统为每个文件分配一个inode，文件的inode保存了文件的关键元数据（**meta-data**）：如stat属性和指向data blocks的指针。data regions被分为8KB或更大的数据块data blocks，用于保存文件数据和目录元数据（**directory meta-data**）。**目录项directory entries包含文件名和指向inodes的指针**。假如有多个目录项指向了同一个文件的inodes，说明该文件被hard-linked。由于jos文件系统不支持hard links，所以不需要这种级别的“间接寻址”，**jos文件系统不会使用inodes，而是在描述文件的相关目录项中保存文件/子目录的元数据。**
文件和目录在逻辑上都会关联到一系列data blocks，这些**data blocks可能是分散在磁盘上**，就好比虚拟地址空间关联到一系列分散的物理页。文件系统隐藏了数据块分布的细节，提供在文件内任意偏移处读写字节序列的接口。对于创建和删除文件等操作，文件系统内部处理了所有相关的目录修改。jos文件系统允许用户进程直接读取目录元数据（e.g., read），用户进程可以自己实现目录浏览（e.g. 实现ls），而不必依赖于特定的文件系统调用接口。

meta-data: 描述数据的数据 (例如一张照片是数据 那么照片的参数就是元数据)

### Sectors and Blocks

**操作系统使用sector(扇区)作为基本的数据读取和写入单位**

**文件系统分配和使用磁盘存储的单元是block**。块通常是对逻辑数据的抽象概念，它是由一定数量的连续扇区组成的。**文件系统在管理文件和存储空间时使用块**
xv6将block size定义为512字节。由于存储空间越来越便宜，而且更大粒度有利于存储管理，所以大多数现代文件系统使用了更大的block size。**jos文件系统使用了4096字节大小的block size**，方便于匹配页大小。

### Superblocks

文件系统通常在磁盘某个容易寻找的位置保留特定的磁盘块，用于存储文件系统的元数据描述属性。比如：block size、disk size、查找root目录所需的元数据、文件系统上次挂载时间、上次检查磁盘错误的时间等。这些特殊的磁盘块称为superblocks。
**jos文件系统有一块superblock，位于磁盘的block 1**，它的定义是inc/fs.h的struct Super。**block 0用于保留启动引导程序和分区表**，所以文件系统没有使用它。许多现代的文件系统维护了多个superblocks，并在磁盘的多个位置进行复制，一旦某一个损坏了或其他原因，就可以使用下一个superblock。

​																						     			**disk布局**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240221112634018.png" alt="image-20240221112634018" style="zoom:50%;" />

### File Meta-data

inc/fs.h的struct File定义了描述文件的元数据布局。jos的文件元数据包括文件名、大小、类型（常规文件还是目录）、指向blocks的指针。**jos没有inodes，所以文件元数据直接存储在磁盘的目录项中。**不像其他文件系统，无论是在磁盘上还是在内存上，**jos使用File数据结构来代表文件元数据。**

​																										**file元数据结构**

<img src="C:\Users\zyj123\AppData\Roaming\Typora\typora-user-images\image-20240221112928580.png" alt="image-20240221112928580" style="zoom: 67%;" />

**struct File的`f_direct`数组的前十个成员存储了文件前10个blocks的block numbers，这10个blocks被称为文件的direct blocks。**对于小于10*4096 = 40KB的文件，意味着该文件所有blocks的block numbers将直接在File结构上直接匹配。**对于更大的文件，则另外分配一个磁盘块以保存4096/4 = 1024个blocks的block number(一个Block大小是4096B 用这个块存储1024个block地址 每个地址占4B)，该磁盘块称为文件的indirect block。**因此，文件大小的上限是1034 blocks(10 + 1024)，大于4M。为了支持更大的文件，其他文件系统通常支持double-indirect和triple-indirect blocks。

### Directories versus Regular Files

jos的**File结构体既可以代表常规文件，也可以代表目录。**文件系统以同样的方式管理常规文件和目录文件，除了：文件系统不解析常规文件相关联的数据块内容，但会**将目录文件的内容解析为一系列的File结构，用于描述目录中的文件和子目录。**
**superblock包含了一个File结构（struct Super的root区域），用于保存root目录的元数据。**该目录文件的内容是一系列File结构，用于描述root目录中的文件和子目录。

# The File System

主要实现以下关键功能：读取blocks到block cache中、刷新block cache到磁盘中、分配磁盘块、映射文件偏移到磁盘块、实现read/write/open的IPC接口。

## Disk Access

操作系统需要能够访问磁盘，传统的monolithic操作系统策略是在内核中添加IDE磁盘驱动并且提供文件系统访问磁盘所需的系统调用接口，而jos的做法是将IDE磁盘驱动作为用户级别文件系统进程的一部分。需要对内核作轻微修改，确保文件系统进程拥有实现磁盘访问所需的相关特权。

**依赖轮询和基于可编程输入输出**（”programmed I/O” - PIO）的磁盘访问，可以很方便在用户空间实现磁盘访问。

在jos中，我们**将文件系统实现为一个用户进程**，其他进程使用IPC与文件系统进程通信，并且**只允许文件系统进程访问I/O space**，但不允许其他进程访问I/O space。device I/O指令需要访问的**所有IDE磁盘寄存器都在x86的I/O space上而不是被内存映射**

#### **Exercise 1.**

**i386_init使用了ENV_TYPE_FS来标识文件系统进程，修改env.c的env_create函数，使其能根据ENV_TYPE_FS向文件系统进程给出相关的I/O权限。**

	void
	env_create(uint8_t *binary, enum EnvType type)
	{
		// LAB 3: Your code here.
		struct Env *env;
		if (env_alloc(&env, 0) != 0) {
			cprintf("env_create: error create new env.\n");
			return;
		}
	env->env_type = type;
	load_icode(env, binary);
	
	// If this is the file server (type == ENV_TYPE_FS) give it I/O privileges.
	// LAB 5: Your code here.
	if (type == ENV_TYPE_FS) {
		env->env_tf.tf_eflags |= FL_IOPL_MASK;
	}
	}
#### **Question**

**是否需要对内核作出什么修改，确保在进行进程切换时相关的I/O权限设置能够正常地保存和恢复？**

不需要。进程切换时硬件会自动保存elfags，而在执行新进程时，`env_pop_tf`的iret指令会恢复eflags。

## The Block Cache

jos文件系统能够处理的磁盘大小被限制在3GB以内。文件系统进程的地址空间[DISKMAP,DISKMAP+DISKMAX)，即[0x10000000,0xD0000000)被保留，用于磁盘的内存映射，如：block 0映射到0x10000000，block 1映射到0x10001000。fs/bc.c的diskaddr函数实现了磁盘块到虚拟地址的映射。
**由于文件系统进程的虚拟地址空间完全独立于其他进程**，且文件系统进程唯一要完成的工作是实现文件访问，所以通过上述方式保留大部分进程地址空间的做法是合理的；但是对于32位机子上真正实现的文件系统而言，上述方式并不合适，因为现代磁盘大于3G。不过，类似的buffer cache管理方法仍然适用于64位地址空间的机器。
**为了避免将整个磁盘读取内存，jos实现了一种请求分页demand paging的形式，当在某个磁盘映射区域发生页错误时，分配相关的页并且读取相关的block。**

#### **Exercise 2.**

**实现fs/bc.c的bc_pgfault和flush_block函数。**

==**bc_pgfault()**==

按照注释，要对齐addr，然后用lab 4的sys_page_alloc分配页空间，再调用fs/ide.c中的ide_read()将磁盘中的内容读入页空间即可。**ide_read操作的是sectors而不是blocks。**

	static void
	bc_pgfault(struct UTrapframe *utf)
	{
		void *addr = (void *) utf->utf_fault_va;
		uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
		int r;
	// Check that the fault was within the block cache region
	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("page fault in FS: eip %08x, va %08x, err %04x",
		      utf->utf_eip, addr, utf->utf_err);
	
	// Sanity check the block number.
	if (super && blockno >= super->s_nblocks)
		panic("reading non-existent block %08x\n", blockno);
	
	// Allocate a page in the disk map region, read the contents
	// of the block from the disk into that page.
	// Hint: first round addr to page boundary. fs/ide.c has code to read
	// the disk.
	//
	// LAB 5: you code here:
	addr = ROUNDDOWN(addr, PGSIZE);
	if ((r = sys_page_alloc(0, addr, PTE_P | PTE_U | PTE_W)) < 0)
		panic("in bc_pgfault, sys_page_alloc: %e", r);
	
	if ((r = ide_read(blockno * BLKSIZE / SECTSIZE, addr, BLKSIZE / SECTSIZE)) < 0)
		panic("in bc_pgfault, ide_read: %e", r);
	// Clear the dirty bit for the disk block page since we just read the
	// block from disk
	if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
		panic("in bc_pgfault, sys_page_map: %e", r);
	
	// Check that the block we read was allocated. (exercise for
	// the reader: why do we do this *after* reading the block
	// in?)
	if (bitmap && block_is_free(blockno))
		panic("reading free block %08x\n", blockno);
		}

==**flush_block()**==

**flush_block函数根据需要将block写入到磁盘。如果block没有在block cache中（该页没被映射）或者block不是dirty的，flush_block函数什么都不做。**

	void
	flush_block(void *addr)
	{
		uint32_t blockno = ((uint32_t)addr - DISKMAP) / BLKSIZE;
	if (addr < (void*)DISKMAP || addr >= (void*)(DISKMAP + DISKSIZE))
		panic("flush_block of bad va %08x", addr);
	
	// LAB 5: Your code here.
	int r;
	addr = ROUNDDOWN(addr, PGSIZE);
	if (!va_is_mapped(addr) || !va_is_dirty(addr)) return;
	if ((r = ide_write(blockno * BLKSECTS, addr, BLKSECTS)) < 0)
		panic("in flush_block, ide_write: %e", r);
	if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
		panic("in flush_block, sys_page_map: %e", r);
		}
==**bc_init()**==

**设置页错误中断函数来将super block数据读入到block cache** 

	void
	bc_init(void)
	{
		struct Super super;
		set_pgfault_handler(bc_pgfault);
		check_bc();
	// cache the super block by reading it once
	memmove(&super, diskaddr(1), sizeof super);
	}
==**fs.c/fs_init()**==

**初始化文件系统 `fs_init`函数将全局结构体指针变量super指向了磁盘disk 1的内存映射区域，之后就可以读取super结构体。**

	// Initialize the file system
	void
	fs_init(void)
	{
		static_assert(sizeof(struct File) == 256);
	// Find a JOS disk.  Use the second IDE disk (number 1) if available
	if (ide_probe_disk1())
		ide_set_disk(1);
	else
		ide_set_disk(0);
	bc_init();
	
	// Set "super" to point to the super block.
	super = diskaddr(1);
	check_super();
	
	// Set "bitmap" to the beginning of the first bitmap block.
	bitmap = diskaddr(2);
	check_bitmap();
	}

## The Block Bitmap

`fs_init`设置了bitmap指针指向block 2的起始位置（但不一定就是说只有block 2被用于block map），bitmap是一个`uint32_t *`类型的指针，用于判断磁盘块是否已被分配使用，**每一个bit位代表一个block，置0代表对应的block已被用**。**在fs被制成镜像时已经默认将block 0、block 1以及被用于bitmap的块对应的bit位置0**。

#### **Exercise 3.** 

**实现alloc_block，在bitmap中查找可用的block，一旦找到，更新对应的bit位，并将该bit位所在的bitmap block更新到磁盘中（注意不一定是block 2），返回对应的block number。*****（block0和block1已经被用 所以bitmap的第1位和第2位为0）**

	int
	alloc_block(void)
	{
		// The bitmap consists of one or more blocks.  A single bitmap block
		// contains the in-use bits for BLKBITSIZE blocks.  There are
		// super->s_nblocks blocks in the disk altogether.
	// LAB 5: Your code here.
	for (int i = 3; i < super->s_nblocks; i++) {
		if (block_is_free(i)) {
			bitmap[i/32] &= ~(1<<(i%32));
			return i;
		}
	}
	
	return -E_NO_DISK;
	}
## File Operations

fs/fs.c提供了一系列函数用于解析和管理File结构、浏览和管理目录文件项、由根目录开始解析一个绝对路径等。

#### **Exercise 4.** 

**完成file_block_walk和file_get_block函数。**

==**file_block_walk()**==

**通过filebno找到指定文件f对应的block 将该block项地址存储到ppdiskbno**

如果filebno在f_direct[NDIRECT]中 那么直接将ppdiskbno指向该项地址

如果fileno在f_indirect中 并且如果f_indirect中没有项 那么就新建一个alloc_block() 然后将这个block原有的数据写入到磁盘flush_block()

	static int
	file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)
	{
	    // LAB 5: Your code here.
		if (filebno > NDIRECT + NINDIRECT) return -E_INVAL;
	    if (filebno < NDIRECT) {
			*ppdiskbno = &(f->f_direct[filebno]);
		} else {
			uint32_t blockno, *indirects;
		if (f->f_indirect) {
			indirects = diskaddr(f->f_indirect);
			*ppdiskbno = &(indirects[filebno - NDIRECT]);
		} else {
			if (!alloc)
				return -E_NOT_FOUND;
			if ((blockno = alloc_block()) < 0)
				return blockno;
			f->f_indirect = blockno;
			flush_block(diskaddr(blockno));
			indirects = diskaddr(blockno);
			*ppdiskbno = &(indirects[filebno - NDIRECT]);
		}
	}
	
	return 0;
	}
==**file_get_block()**==

**利用file_block_walk()找到指定文件f对应的block 并将blk设置为ppdiskbno的地址**

在`f_direct[NDIRECT]`或`f_indirect`对应的间接块中找到逻辑block number对应的磁盘block存储位置pdiskbno，如果该位置数据为空，则分配一个新的磁盘block并将磁盘block number存到pdiskbno上。将新分配的磁盘block对应的虚拟地址存储到blk中。

	int
	file_get_block(struct File *f, uint32_t filebno, char **blk)
	{
	    // LAB 5: Your code here.
		int r;
		uint32_t *ppdiskbno;
		if ((r = file_block_walk(f, filebno, &ppdiskbno, 1)) < 0) {
			return r;
		}
	int blockno;
	if (*ppdiskbno == 0) {
		if ((blockno = alloc_block()) < 0) {
			return blockno;
		}
	
		*ppdiskbno = blockno;
		flush_block(diskaddr(blockno));
	}
	
	*blk = diskaddr(*ppdiskbno);
	return 0;
	}
## The file system interface

客户端进程不能直接调用文件系统进程的函数，而是通过远程过程调用（RPC - remote procedure call）的形式，jos的RPC基于IPC机制。以读取文件为例，RPC过程如下：

```
   Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

**将文件系统env看做一个server，常规env看做一个client**，通过ipc_send, ipc_recv完成

#### **Exercise 5.**

**实现fs/serv.c的serve_read。serve_read调用file_read函数读取磁盘文件，serve_read本身只需要提供文件读取的RPC接口。**

	int
	serve_read(envid_t envid, union Fsipc *ipc)
	{
		struct Fsreq_read *req = &ipc->read;
		struct Fsret_read *ret = &ipc->readRet;
	if (debug)
		cprintf("serve_read %08x %08x %08x\n", envid, req->req_fileid, req->req_n);
	
	// Lab 5: Your code here:
	struct OpenFile *o;
	int r;
	if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
		return r;
	if ((r = file_read(o->o_file, ret->ret_buf, req->req_n, o->o_fd->fd_offset)) < 0)
		return r;
	o->o_fd->fd_offset += r;
	return r;
	}

#### **Exercise 6.** 

**实现fs/serv.c的serve_write、lib/file.c的devfile_write**

==**serve_write()**==

**将客户端env进程中的数据通过文件系统服务器利用file_write写入到磁盘中**

	int
	serve_write(envid_t envid, struct Fsreq_write *req)
	{
		if (debug)
			cprintf("serve_write %08x %08x %08x\n", envid, req->req_fileid, req->req_n);
	// LAB 5: Your code here.
	struct OpenFile *o;
	int r;
	if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
		return r;
	if ((r = file_write(o->o_file, req->req_buf, req->req_n, o->o_fd->fd_offset)) < 0)
		return r;
	o->o_fd->fd_offset += req->req_n;
	return r;
	}
# Spawning Processes

lib/spawn.c用于创建新进程（设置进程内核空间部分的页目录、复制父进程的trapframe等，但没有映射用户地址空间），设置子进程的trapframe（如修改eip入口为程序入口），初始化子进程用户栈（分配空间、存储传递的参数），从文件系统加载程序文件到新进程中（分配新物理页存储），然后让子进程开始执行该程序。spawn类似于UNIX的fork，在子进程中直接跟进exec。

#### **Exercise 7.**

**spawn函数需要设置子进程的trapframe。实现sys_env_set_trapframe系统调用，需要检查给出的trapframe地址是否可写，并且需要确保子进程执行期间能够触发中断。**
**注意到：系统调用过程中传递struct Trapframe *地址的方式是调用前先转换为uint32_t类型，调用后再转换为struct Trapframe *。**

```
static int
sys_env_set_trapframe(envid_t envid, struct Trapframe *tf)
{
	// LAB 5: Your code here.
	// Remember to check whether the user has supplied us with a good
	// address!
	struct Env *env;
	int r;
	if ((r = envid2env(envid, &env, 1)) < 0) return r;
	tf->tf_eflags |= FL_IF;
	tf->tf_eflags &= ~FL_IOPL_MASK;
	tf->tf_cs = GD_UT | 3;
	env->env_tf = *tf;
	return 0;
}
```

- **xv6创建子线程并执行新程序的流程大致是：**1、用户态调用fork创建子进程，包括复制物理页。2、用户态执行用户子进程，调用exec尝试执行新程序。3、内核态读取新程序内容，映射到新的地址空间中。4、内核态更换子进程的地址空间，释放旧地址空间。
- **jos使用spawn创建子线程并执行新程序的流程大致是：**1、用户态spawn创建子线程，只设置进程内核空间部分的页目录、复制父进程的trapframe等，但没有映射用户地址空间。2、用户态spawn读取程序文件到子进程地址空间中（调用系统调用函数分配、映射物理页）。3、用户态spawn设置子进程用户栈和trapframe（包括执行入口），调用系统调用函数更新子进程trapframe。4、用户态spawn设置子进程为可执行。

## Sharing library state across fork and spawn

unix文件描述符是一个综合概念，包括文件、管道、console I/O等设备类型。

我们希望通过fork和spawn来共享文件描述符状态，但是文件描述符状态是保存在用户空间中的。目前，fork将内存映射为写时复制，文件描述符状态会被复制而不是共享（意味着进程不能seek这些不是由它们本身打开的文件，也不能使用管道）；spawn则完全不处理这些相关内存，因此创建的新进程没有打开任何文件描述符。
因此，我们将标记特定部分的内存区域是共享的。采取的方法是使用页表项中的一个未使用位（如`PTE_COW`的做法一样）。具体使用的位是定义于inc/lib.h的`PTE_SHARE`位，Intel和AMD保留了三个软件可用的PTE位，这是其中之一。一旦页表项设置了`PTE_SHARE`位，则fork和spawn需要直接复制物理页到子进程中。

#### **Exercise 8.**

**修改lib/fork.c的duppage和lib/spawn.c的copy_shared_pages，实现PTE_SHARE。**

==**duppage()**==

**在fork时调用的duppage将父进程和自进程设置为写时复制 并且文件描述符状态设置为PTE_SHARE**

	static int
	duppage(envid_t envid, unsigned pn)
	{
		int r;
		// LAB 4: Your code here.
		int perm = PTE_U | PTE_P;
		void * addr = (void *) (pn * PGSIZE);
		if ((uvpt[pn] & PTE_SHARE)) {
			sys_page_map(0, addr, envid, addr, PTE_SYSCALL);
		} else if ((uvpt[pn] & PTE_W) || (uvpt[pn] & PTE_COW)) {
			perm |= PTE_COW;
			if ((r = sys_page_map(0, addr, envid, addr, perm)) < 0) {
				panic("duppage: %e\n", r);
			}
		if ((r = sys_page_map(0, addr, 0, addr, perm)) < 0) {
			panic("duppage: %e\n", r);
		}
	} else if ((r = sys_page_map(0, addr, envid, addr, perm)) < 0) {
		panic("duppage: %e\n", r);
	}
	
	return 0;
	}

==**copy_shared_pages()**==

**在swpan中调用copy_shared_pages将子进程文件描述符状态设置为PTE_SHARE**

```
// Copy the mappings for shared pages into the child address space.
static int
copy_shared_pages(envid_t child)
{
	// LAB 5: Your code here.
	uint32_t addr;
	for (addr = 0; addr < USTACKTOP; addr += PGSIZE) {
		// uvpd是有1024个pde的一维数组，而uvpt是有2^20个pte的一维数组,与物理页号刚好一一对应
		if ((uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_U) && (uvpt[PGNUM(addr)] & PTE_SHARE)) {
            sys_page_map(0, (void*) addr, child, (void*) addr, (uvpt[PGNUM(addr)] & PTE_SYSCALL));
        }
	}

	return 0;

}
```

# The keyboard interface

QEMU显示输出到CGA和串行口，但目前为止我们只在嵌入内核监控的时候进行输入。在QEMU图形窗口中的键入表现为键盘键入，在控制台的键入表现为串行口的特性。kern/console.c包含了内核监控所使用的键盘和串行驱动。

#### **Exercise 9.**

**在kern/trap.c中，调用kbd_intr处理IRQ_OFFSET+IRQ_KBD中断，调用serial_intr来处理IRQ_OFFSET+IRQ_SERIAL中断。**

# The Shell

#### **Exercise 10.**

**在user/sh.c中添加相关内容，使之支持重定向**

从参数列表中获取文件名，并将该文件以只读模式打开。如果文件打开成功，它会检查文件描述符是否为0（标准输入）。如果文件描述符不为0，它会使用dup()函数将文件描述符复制到文件描述符0，然后关闭原始的文件描述符。

这样，程序就可以从该文件中读取输入，而不是从标准输入中读取。这种方式可以用于从文件中读取输入数据，而不是从键盘或其他标准输入源读取。

		case '<':	// Input redirection
			// Grab the filename from the argument list
			if (gettoken(0, &t) != 'w') {
				cprintf("syntax error: < not followed by word\n");
				exit();
			}
			// Open 't' for reading as file descriptor 0
			// (which environments use as standard input).
			// We can't open a file onto a particular descriptor,
			// so open the file as 'fd',
			// then check whether 'fd' is 0.
			// If not, dup 'fd' onto file descriptor 0,
			// then close the original 'fd'.
	
			// LAB 5: Your code here.
			if ((fd = open(t, O_RDONLY)) < 0) {
				cprintf("open %s for read: %e", t, fd);
				exit();
			}
			if (fd != 0) {
				dup(fd, 0);
				close(fd);
			}
			break;

# fd、dev、file

**jos的文件系统进程负责维护File数据结构、fd与file之间的关系，并以fd与其他进程进行交互。其他进程并不操作File数据结构，而是以fd与文件系统进程进行交互，具体是共享填充fd数据结构的页面。**
jos的文件描述符包括文件、管道、console I/O三种设备dev类型，每种类型都有相应的文件读写接口，定义一致。**用户进程在使用时，具体指定一种dev类型。每个类型的dev都有对应的id，并存在fd中，然后调用库函数与文件系统交互。**在上述三种设备类型中，只有涉及文件类型时才会与文件系统进程进程交互。

File的定义如下：

```
struct File {
	char f_name[MAXNAMELEN];	// filename
	off_t f_size;			// file size in bytes
	uint32_t f_type;		// file type

	// Block pointers.
	// A block is allocated iff its value is != 0.
	uint32_t f_direct[NDIRECT];	// direct blocks
	uint32_t f_indirect;		// indirect block

	// Pad out to 256 bytes; must do arithmetic in case we're compiling
	// fsformat on a 64-bit machine.
	uint8_t f_pad[256 - MAXNAMELEN - 8 - 4*NDIRECT - 4];
} __attribute__((packed));	// required only on some 64-bit machines
```

fd的定义如下：

```
struct FdFile {
	int id;
};

struct Fd {
	int fd_dev_id;
	off_t fd_offset;
	int fd_omode;
	union {
		// File server files
		struct FdFile fd_file;
	};
};
```

dev的定义如下：

```
struct Dev {
	int dev_id;
	const char *dev_name;
	ssize_t (*dev_read)(struct Fd *fd, void *buf, size_t len);
	ssize_t (*dev_write)(struct Fd *fd, const void *buf, size_t len);
	int (*dev_close)(struct Fd *fd);
	int (*dev_stat)(struct Fd *fd, struct Stat *stat);
	int (*dev_trunc)(struct Fd *fd, off_t length);
};
 
extern struct Dev devfile;
extern struct Dev devcons;
extern struct Dev devpipe;
```

# **文件类型的交互**

文件系统进程使用**OpenFile关联file和fd**，OpenFile定义如下：

```
struct OpenFile {
	uint32_t o_fileid;	// file id
	struct File *o_file;	// mapped descriptor for open file
	int o_mode;		// open mode
	struct Fd *o_fd;	// Fd page
};
```

# 管道的交互

pipe的定义如下：

```
struct Pipe {
	off_t p_rpos;		// read position
	off_t p_wpos;		// write position
	uint8_t p_buf[PIPEBUFSIZ];	// data buffer
};
```
