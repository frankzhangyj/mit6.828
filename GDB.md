# GCC & GDB

[GCC 是GNU 工具链](https://en.wikipedia.org/wiki/GNU_toolchain)的关键组件，也是大多数与[GNU](https://en.wikipedia.org/wiki/GNU)和[Linux 内核](https://en.wikipedia.org/wiki/Linux_kernel)相关的项目的标准编译器。

GDB（GNU Debugger）是UNIX及UNIX-like下的强大调试工具，可以调试ada, c, c++, asm, minimal, d, fortran, objective-c, go, java,pascal等语言。

一般在程序调试之前需要进行编译

`gcc -g debug_test1.c -o debug_test1`

使用gcc -g 选项，可以生成一个debug模式的可执行文件 使用gcc -o将`debug_test1.c`预处理、汇编、编译并链接形成可执行文件 `debug_test1.c`

编译完成后进行gdb调试

`gdb ./debug_test1`

使用readelf查看elf文件 并且使用grep查找.debug文件 若没有debug文件 则证明在编译时没有使用-g参数

`readelf -S helloWorld|grep debug`

## 在os开发过程中常用的命令

`Ctrl-c`

停止机器并在当前指令处中断到 GDB。如果 QEMU 有多个虚拟 CPU，这会停止所有虚拟 CPU。

`c (or continue)`

继续执行直到下一个断点或 Ctrl-c。

`si (or stepi)`

执行一条机器指令 单步调试

`b function or b file:line (or breakpoint)`

在相应指令设置断点

`b *addr (or breakpoint)`

在EIP地址处设置断点(即在指令地址下设置断点)。

`set print pretty`

启用后可以使p(print)命令显示的信息更加规范化 方便查看

`p variable(or print)`

将调试过程中变量、表达式、数组元素、结构体或其成员打印出来

`info registers`

显示当前CPU寄存器的内容和状态信息。它允许开发者查看程序执行过程中寄存器的值，以及了解寄存器在程序中的作用。(可以通过寄存器状态值分析出函数调用 程序进行状态等有用信息)

`x/Nx addr`

用于在指定的内存地址（addr）处查看存储的数据，"N"是一个整数，表示要显示的内存单元的数量。"x"表示以十六进制格式显示数据。

`watch *p` 

监视指针地址p所指向的一段内存中值是否变化，若该内存地址中的值发生变化，dgb就会终止调试。

被监视的内存地址会产生一个watch point(观察点) 配合`x/Nx addr` 命令查看该内存中值的变化情况

`symbol-file file`

用于指定要加载符号信息的可执行文件，符号文件包含了程序中符号（如函数名、变量名）与其在内存中的地址之间的映射关系，这对于调试器的符号解析和调试过程非常重要。

`thread n`

用于在多线程程序中切换到指定线程的上下文进行调试。该命令允许开发者选择要单独查看和操作的线程。

`info threads`

列出所有线程（即 CPU），包括它们的状态（活动或停止）以及它们所处的功能。