# ucore-lab1 启动操作系统（2019.11.5）
WA062 
## 练习一 
**[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中 每一条相关命令和命令参数的含义,以及说明命令导致的结果)**
    
使用命令 `make clean; make V= > make_output.log 2>&1` 生成make的过程，位于 make_output.log 中。大致可以分为如下几个步骤：

1. 编译内核代码，举例：

+
``` 
cc kern/init/init.c

gcc -Ikern/init/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```
参数含义：

`-I：编译时引用该文件目录；
-fno-builtin -nostdinc：关闭内建库
-fno-PIC：减小代码体积，否则无法生成bootloader（#22）
-Wall：开启警告
-ggdb -gstabs：开启gdb调试功能
-m32：生成32位代码
-fno-stack-protector：取消栈保护
-o：输出文件
`
**2.链接生成kernel**

```
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
参数含义：

```
`-m elf_i386：生成32位ELF可执行文件
-nostdlib：关闭内建库
-T tools/kernel.ld：使用脚本 toold/kernel.ld
-o bin/kernel *.o：使用一系列的 .o 文件生成 bin/kernel
'

**3.编译bootloader**
举例：

```
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
参数含义：同1，-Os 与 -fno-PIC 为对代码大小进行优化，否则无法塞入512字节

```

**4.生成bootloader**


```
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
参数含义：
`
-N：设置代码段和数据段均可读写，并且关闭动态链接
-e start：设置入口点符号为start
-Ttext 0x7c00：设置代码段起始地址为 0x7c00
其余同2
之后生成二进制代码，使用 objcopy 和 objdump：
`

```
@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
```
参数含义：

`-S：不拷贝relocation和debug信息
-O：拷贝二进制代码
**4.编译sign.c**

判断 obj/bootblock.out 大小是否超过510字节

**5.生成ucore.img**


```
dd if=/dev/zero of=bin/ucore.img count=10000
记录了10000+0 的读入
记录了10000+0 的写出
5120000 bytes (5.1 MB, 4.9 MiB) copied, 0.0138703 s, 369 MB/s
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
记录了1+0 的读入
记录了1+0 的写出
512 bytes copied, 6.4636e-05 s, 7.9 MB/s
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
记录了146+1 的读入
记录了146+1 的写出
74820 bytes (75 kB, 73 KiB) copied, 0.000219237 s, 341 MB/s
```
第一个dd是写0到 ucore.img 中，大小为5120000字节；第二个dd是把 `bin/bootblock `写入 `ucore.img`，`conv=notrunc` 表示不覆盖之前的数据；第三个dd 将 `bin/kernel` 写入 `ucore.img`，`seek=1` 表示从第一扇区开始写，同样不覆盖之前的数据。
##练习2 
**[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。**

将 `tools/gdbinit` 中的最后一行continue删掉，随后使用 `make debug-nox` 开始运行（由于我gdb是gef拓展，tui参数会挂）。

首先停在了 `0xfff0` 位置，`cs=0xf000`，`eip=0xfff0`，如下图所示：

[![MKpLZT.png](https://s2.ax1x.com/2019/11/10/MKpLZT.png)](https://imgchr.com/i/MKpLZT)

**[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。**

然而真正的指令并不是位于 `0xfff0` 的这个，根据 https://chyyuu.gitbooks.io/ucore_os_docs/lab1/lab1_3_1_bios_booting.html ，指令其实位于 `0xfffffff0` 处。直接查看：输入 `x/i 0xfffffff0` 会得到 `jmp 0x3630:0xf000e05b`，看起来是gdb默认32位，可是我输入命令 `set arch i8086` 之后再次执行，还是同一输出。`si` 之后下一步确实跳到了 `0xe05b`。

使用命令 `b *0x7c00` 设置断点，结果如下：

[![MKClcR.png](https://s2.ax1x.com/2019/11/10/MKClcR.png)](https://imgchr.com/i/MKClcR)

可以看到 `0x7c00` 之后一串的汇编代码，与 `boot/bootasm.S` 的16行到23行一致，也与 `obj/bootblock.asm`  中 `code16` 代码段一致。

从 `obj/bootblock.asm` 中可以看出 `seta20` 代码段起始地址位于 `0x7c0a`，设置断点之后进行测试，验证与 `obj/bootblock.asm` 和 `boot/bootasm.S` 中的汇编代码均一致。
	   
**[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。 将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。**

在tools/gdbinit结尾加上

	b *0x7c00
	c
	x /10i $pc
	
便可以在q.log中读到"call bootmain"前执行的命令
	----------------
	IN: 
	0x00007c00:  cli    
	
	----------------
	IN: 
	0x00007c01:  cld    
	0x00007c02:  xor    %ax,%ax
	0x00007c04:  mov    %ax,%ds
	0x00007c06:  mov    %ax,%es
	0x00007c08:  mov    %ax,%ss
	
	----------------
	IN: 
	0x00007c0a:  in     $0x64,%al
	
	----------------
	IN: 
	0x00007c0c:  test   $0x2,%al
	0x00007c0e:  jne    0x7c0a
	
	----------------
	IN: 
	0x00007c10:  mov    $0xd1,%al
	0x00007c12:  out    %al,$0x64
	0x00007c14:  in     $0x64,%al
	0x00007c16:  test   $0x2,%al
	0x00007c18:  jne    0x7c14
	
	----------------
	IN: 
	0x00007c1a:  mov    $0xdf,%al
	0x00007c1c:  out    %al,$0x60
	0x00007c1e:  lgdtw  0x7c6c
	0x00007c23:  mov    %cr0,%eax
	0x00007c26:  or     $0x1,%eax
	0x00007c2a:  mov    %eax,%cr0
	
	----------------
	IN: 
	0x00007c2d:  ljmp   $0x8,$0x7c32
	
	----------------
	IN: 
	0x00007c32:  mov    $0x10,%ax
	0x00007c36:  mov    %eax,%ds
	
	----------------
	IN: 
	0x00007c38:  mov    %eax,%es
	
	----------------
	IN: 
	0x00007c3a:  mov    %eax,%fs
	0x00007c3c:  mov    %eax,%gs
	0x00007c3e:  mov    %eax,%ss
	
	----------------
	IN: 
	0x00007c40:  mov    $0x0,%ebp
	
	----------------
	IN: 
	0x00007c45:  mov    $0x7c00,%esp
	0x00007c4a:  call   0x7d0d
	
	----------------
	IN: 
	0x00007d0d:  push   %ebp


其与bootasm.S和bootblock.asm中的代码相同。

-------

## [练习3] 分析bootloader 进入保护模式的过程。


-------
根据 `boot/bootasm.S` 中的代码，具体分为如下几个步骤：

1. 开启A20地址线：（line 29-43）根据 https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_appendix_a20.html ，分为如下小步骤：
   1. 等待直到8042不忙，之后写入0xd1到0x64端口，意为写P2
   2. 等待直到8042不忙，之后写入0xdf到0x60端口，意为置P2的A20位为1
2. 配置GDT并开启保护模式：（line 49-52，77-87）
   1. GDT强制4字节对齐，生成三个描述符：空描述符、代码段描述符和数据段描述符；之后使用 `lgdt` 命令加载
   2. 将 `%cr0` 的最低位（表示PE）置为1，开启保护模式
3. 使用 `ljmp` 命令跳转至32位代码段起始地址（line 56-71）：将CS寄存器设置为代码段选择子，进入32位保护模式，依次初始化 ds、es、fs、gs、ss，并使 ebp 为 0，esp 为 start（0x7c00），调用 `bootmain` 函数

-------

##[练习4]分析bootloader加载ELF格式的OS的过程。

-------

首先看readsect函数， readsect从设备的第secno扇区读取数据到dst位置

	static void
	readsect(void *dst, uint32_t secno) {
	    waitdisk();
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	        // 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 幻数4因为这里以DW为单位
	}
readseg简单包装了readsect，可以从设备读取任意长度的内容。

	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;
	
	    va -= offset % SECTSIZE;
	
	    uint32_t secno = (offset / SECTSIZE) + 1; 
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	}
在bootmain函数中，

	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
	
	
-------

##[练习5] 实现函数调用堆栈跟踪函数
	
-------
练习五主要是针对调用栈进行理解及相关的简单实践；但是对于我们理解堆栈的调用还是很有帮助的，后续我还是会在我的blog中贴出一个简单的堆栈调用的讲解；

看了一下实验答案，发现答案还针对最初始的状态对ebp进行了解释，那么我也利用这个思路进行一下解释；不过没有按照实验答案的思路来走，感兴趣的可以看一下；

我们指导在执行被调用函数体时，会执行如下的汇编指令：


```
push ebp
mov esp ebp
```
这样新的ebp的指向的堆栈内存中保存的时原来的ebp的值(该内存位置向栈顶方向则是函数体的执行，该内存位置向栈底方向则是该被调用函数的return address及各个实参值（也可能不含有参数）),当函数体执行完毕时，又会执行一次如下汇编指令：


```
pop ebp
```
又将原ebp的值进行了恢复；这样通过更新ebp的值并存储原来ebp的值，就会将调用的函数形成一个调用堆栈链，从而正确地执行函数调用；

我们可以通过该实验的一个调用输出来阐述一下：


```
moocos-> cat tmp
 make qemu
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x001032f9 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd20 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b08 eip:0x001009ad args: 0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:311: print_stackframe+28
ebp:0x00007b18 eip:0x00100ccb 
ebp:0x00007b18 eip:0x00100ccb args: 0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 
ebp:0x00007b38 eip:0x00100092 args: 0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb 
ebp:0x00007b58 eip:0x001000bb args: 0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 
ebp:0x00007b78 eip:0x001000d9 args: 0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe 
ebp:0x00007b98 eip:0x001000fe args: 0x0010331c 0x00103300 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 
ebp:0x00007bc8 eip:0x00100055 args: 0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 
ebp:0x00007bf8 eip:0x00007d68 args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
ebp:0x00000000 eip:0x00007c4f 
ebp:0x00000000 eip:0x00007c4f args: 0xf000e2c3 0xf000ff53 0xf000ff53 0xf000ff53

```

我们可以看到ebp的值是如下变化的： 0x00000000，'0x00007bf8'，'0x00007bc8'，0x00007b98，0x00007b98，0x00007b78，0x00007b58，0x00007b38，0x00007b18，0x00007b08；

对于上述的变化有两点要说一下：

1. 为什么我要逆序来写呢？因为我从初始调用状态开始的，首先在操作系统的bootloader会初始化相关的寄存器/A20/全局描述符表及堆栈信息（即让ebp为0x00000000，esp为0x00007c00）；所以初始状态的ebp就是0x00000000；
2. 为什么之后ebp的值是一直在变小的，因为堆栈的增长是向虚拟地址空间小的方向进行的；

-------

## [练习6] 完善中断初始化和处理

-------

**[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**

通过实验指导书中lab1的【中断与异常】的图9中对中断门的格式描述可以得出以下结论：

* 中断描述符长为8个字节，其中0-1字节和6-7字节为偏移地址；第2-3个字节为段选择子；一旦CPU获取了中断向量（可以理解为中断向量表的index）就会根据向量值从IDT中获取到该位置上的中断描述符；该位置的中断描述符有通过段选择子从GDT表中找到对应的段描述符，由于段描述符中存有相应的基址地址而中断描述符中又存有偏移地址，二者就可以确定中断例程的入口地址；

* 根据/kern/mm/mmu.h中对中断描述符（interrupt and trap）的定义可以进行更好的理解；


```
//以上是我没有使用过的这种限定符号，但是通过成员后面的冒号方法来指定元素bit数的方法，新奇~
/* Gate descriptors for interrupts and traps */
struct gatedesc {
  unsigned gd_off_15_0 : 16; // low 16 bits of offset in segment
  unsigned gd_ss : 16; // segment selector
  unsigned gd_args : 5; // # args, 0 for interrupt/trap gates
  unsigned gd_rsv1 : 3; // reserved(should be zero I guess)
  unsigned gd_type : 4; // type(STS_{TG,IG32,TG32})
  unsigned gd_s : 1; // must be 0 (system)
  unsigned gd_dpl : 2; // descriptor(meaning new) privilege level
  unsigned gd_p : 1; // Present
  unsigned gd_off_31_16 : 16; // high bits of offset in segment
};
```

* 使用uintptr_t一般是与机器的指针长度相同，主要的作用有两个：
    1. 将地址转换成usigned int 类型，从而可以对指针进行int类型才可以进行的相关操作；
    2. 是用来当做句柄来使用的，例如某一个资源的资源描述信息；


-------
**[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。**


```
//idt的初始化
extern uintptr_t __vectors[];//构建保护模式下的trap/exception vector，里面用于存储中断服务例程的入口地址（注，是offset地址），并且[0,31]是定好的留给exception使用的，[32,255]可以留给用户用来设置interrupt，exception或system call来使用；

for (int i = 0; i < 256; i++)
{
     //初始化全局描述符表，即初始化所有表项的的段描述符；
     //GD_KTEXT为内核的代码段的段描述符
       //DPL_KERNEL为特权级标识，用来控制中断处理的方式
     SETGATE(idt[i], 0, GD_KTEXT, __vector[i], DPL_KERNEL)
}
     //这里idt_pd之所以叫伪描述符是因为其存了相关中断描述符信息，这个信息与IDTR寄存器相关（即伪描述符的信息是存储在IDTR中的）
     //lidt和sidt是操作6字节的操作数，用于设定和存储idt的位置信息
     lidt(&idt_pd)
```

-------
**[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数**


```
//处理时钟中断
ticks++;
if (0 == ticks % TICK_NUM)
{
	print_ticks();
}
```

## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. Makefile：生成镜像、bootloader的过程
2. A20
3. GDT
4. 中断描述符表
5. 中断处理过程
6. ELF格式
7. 磁盘读取
8. 函数堆栈
9. 特权转换、中断堆栈切换



