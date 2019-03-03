# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？

  > 需要设置一些只有核心态才能使用的寄存器，并提供只能在核心态使用的修改这些寄存器的指令。为了实现这个功能，还是要实现特权级的机制。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？

  > 实模式就是对可用的物理内存空间不加区别地进行分段访问，早期作为操作系统的内存管理模式，后期只起到在系统启动时为了兼容性等原因进行某些初始化工作的功能。
  >
  > 保护模式能确保用户程序不会对操作系统产生破坏，提供更大的寻址空间，以及对虚拟内存、分页、多任务、多特权级的良好支持。
  >
  > 逻辑地址就是用户程序方位的地址，也称虚拟地址。
  >
  > 在分段机制启动的情况下，转化为线性地址。
  >
  > 如果没有分页机制，线性地址就是物理地址，否则要通过分页机制转化为物理地址。

- 你理解的risc-v的特权模式有什么区别？不同 模式在地址访问方面有何特征？

- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？

> 冒号的意思是这个结构体是一个位域结构，冒号后面的数字代表结构体每个域的位数。
>
> 代码中的结构体虽然使用了9个unsigned，但总共也只占用8字节空间。
>
> intr的值为3*65536+2=196610。

### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

   ```c
   static inline uint8_t
   inb(uint16_t port) {
       uint8_t data;
       asm volatile ("inb %1, %0" : "=a" (data) : "d" (port));
       return data;
   }
   ```

   这段代码的功能是利用``inb``指令，从16位的端口``port``读一个字节并返回给``data``。

2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。

> 1. 常量定义，如
>
>    ```c
>    #define SECTSIZE        512
>    ```
>
> 2. 避免头文件被重复引用
>
>    ```c
>    #ifndef XXX
>    #define XXX
>    ...
>    #endif
>    ```
>
> 3. 当做函数来使用，替换成一段代码
>
>    如在设置全局描述符表项时，使用到了如下代码
>
>    ```c
>    #define SEG_ASM(type,base,lim)                                  \
>        .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
>        .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
>            (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
>    ```

#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
