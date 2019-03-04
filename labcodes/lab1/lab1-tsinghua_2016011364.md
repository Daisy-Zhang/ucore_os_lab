# Lab1 实验报告

2016011364 张岱墀

## 练习1：理解通过make生成执行文件的过程

### 1. 操作系统镜像文件ucore.img是如何一步步生成的？

答：

#### 生成`ucore.img`的代码为：

``` 
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

生成镜像后，该镜像有count=10000个block，其中第一个块用于存下bootblock，剩余的块用于存下kernel。

同时可以看出生成` ucore.img`时用到了`kernel`和`bootblock`这两个文件，以下分别看生成这两个文件的Makefile代码。

#### 生成`kernel` 的代码如下：

```
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

生成`kernel`目标文件之前所执行的相关编译代码如下：

```
INCLUDE	+= libs/

CFLAGS	+= $(addprefix -I,$(INCLUDE))

LIBDIR	+= libs

$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)	

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)
```

这段代码解释如下：

​	首先通过`KINCLUDE`，`KSRCDIR`以及`LIBDIR`把需要编译的文件include进来，然后通过`$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))`和`$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)   `make语句将目录下的.c文件编译为.o文件。在命令行中执行`make "V="`后可以看到相关命令格式为：（下表以第一个init.c举例，环境为mac OS）

```
+ cc kern/init/init.c
i386-elf-gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```

其中参数解释如下：

* `-fno-builtin`: 忽略运行库中不以`_builtin`开头的函数。
* -Wall：能够提示绝大多数warning，便于调试。
* -ggdb：使GCC为GDB生成调试信息。
* -nostadinc：使编译器不在系统缺省的库里寻找头文件。
* -fno-stack-protector：禁用栈保护措施。
* -m32：在64位机器上跑32位代码时需要的额外库支持。

在生成目标.o文件之后，通过kernel.ld的link脚本将这些.o文件链接起来，具体命令行命令如下：

```
+ ld bin/kernel
i386-elf-ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
```

其中参数解释如下：

* -nostdlib：不链接系统标准库文件。
* -T：指定使用的链接脚本为tools/kernel.ld
* 后方为指定的目标文件目录bin/kernel，和所需.o文件

之后成功在bin/kernel目录下生成kernel目标文件。

#### 生成`bootblock`代码如下：

``` 
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

首先会执行两条生成目标.o文件的语句：

```
+ cc boot/bootasm.S
i386-elf-gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
i386-elf-gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
```

两句命令用`bootasm.S`和`bootmain.c`生成了目标文件`bootasm.o`和`bootmain.o`，然后由于编译参数和上述解释过的有部分冲突，这里只举出不同的参数：

* -gstabs：此选项以stabs格式声称调试信息，但是不包括gdb调试信息。

然后会生成所需要的文件sign，生成命令如下：

```
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

最后由上述.o文件可生成最终的bootblock目标文件：

```
+ ld bin/bootblock
i386-elf-ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

其中指定了程序开始地址0x7C00。

### 2. 一个系统被认为是符合规范的系统主导扇区的特征是什么？

答：

由sign.c文件中可有：

* block大小为512
* 第511个字节为  0x55;
* 第523个字节为  0xAA;



## 练习2：使用qemu执行并调试lab1中软件

### 1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

答：在`lab1/tools`目录下找到文件`gdbinit`,然后对该gdb配置文件进行更改如下：

```
define hook-stepi
x/2i $pc
end

set architecture i386
target remote :1234
```

其中：

* 前三行：使用stepi指令查看执行的下一行代码地址前进行强制反汇编。
* set architecture i386：使用 i386 CPU去执行指令。
* target remote ：1234：qemu与gdb用网络端口1234进行通讯。

然后终端执行：`make debug`进入debug模式，在debug模式下使用命令：`stepi`，即可一步步跟踪当前执行到的代码并可以看到其反汇编后的汇编码。

### 2. 在初始化位置0x7c00设置实地址断点，测试断点正常。

答：在`lab1/tools`目录下找到文件`gdbinit`,然后在上一问的基础上继续添加配置代码：

```
break *0x7c00
continue
x/5i $pc
```

其中：

* break *0x7c00：在地址0x7c00处设置断点。
* continue：继续执行。
* x/5i $pc：打印从pc寄存器开始的5条汇编指令。

使用`make debug`指令执行后结果如下：

![result](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab1/report_image/exercise2.2.png)

### 3. 从0x7c00开始跟踪代码执行，单步反汇编代码与bootasm.S和bootblock.asm进行比较。

答：先找到`lab1/boot/bootasm.S`以及`lab1/obj/bootblock.asm`文件，两个文件中均为汇编码，不同点是后者将汇编码进行了一些靠近底层的优化变动以及其带有执行地址。

然后执行`make debug`，通过`stepi`命令将逐步得到的结果与上述两个文件进行比较发现整个运行过程如下：

* 把每条指令与上述两个文件中的指令对比可以得到最终的执行顺序(以bootblock.asm中的函数为标志)：start -> 

seta 20.1 -> seta 20.2 -> protcseg -> bootmain -> readsect ->(static) inline void -> 0x7ccc

* 在运行到`bootmain`之前，所有指令执行顺序与bootblock.asm中一致，在其之后由于涉及到一些跳转等指令导致执行顺序发生了变化。

### 4. 自己找一个bootloader或内核中的代码位置设置断点并进行测试。

答：通过查看`lab1/obj/bootblock.asm`文件，选择在bootmain函数段的比较靠前的一个地址`0x7cd4`设置断点进行调试。设置断点方法同上述第二个小练习，设置断点之后直接`make qemu`依旧无法成功运行程序，通过`make debug`可以正确看到程序在0x7cd4处中断，运行截图如下：

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab1/report_image/exercise2.4.png)



## 练习3：分析bootloader进入保护模式的过程

答：分析目录`lab1/boot/bootasm.S`中的`bootasm.S`文件：
实模式下运行`bootloader`的程序起始指令之前，将数据段，代码段段选择子置为0x8和0x10，以及置保护模式标志位：

```
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
.set CR0_PE_ON,             0x1                     # protected mode enable flag
```

开始执行bootloader，对DS，ES，SS寄存器赋初值，此时处于实模式：

```
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss           
```

为了能够从实模式转为保护模式，需要将A20使能，具体使能方法是：与键盘控制器的一个输出做`与`操作（键盘控制器8024），从而使访问内存时不出现`回卷`现象，达到能够访问4GB内存的目的，这段代码如下：

```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```

由上不难发现除了使能A20之外，在从实模式跳转到保护模式的过程中还用到了GDT表，GDT表初始化的过程为分别为代码段和数据段设置描述符：

```
# Bootstrap GDT
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

这样就完成了从实模式到保护模式的切换，之后再初始化栈指针和段寄存器，然后再调用bootmain：

```
    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebpb
    movl $start, %esp
    call bootmain
```



## 练习4：分析bootloader加载ELF和OS的过程。

答：在`lab1/boot/bootmain.c`文件下进行相关分析：

1. 由于操作系统是存在硬盘(Disk)上的，所以第一步是访问`disk`，从其中读取出操作系统，由于硬盘是由扇区存放，故按照一个个扇区进行读取，首先有等待disk准备读取数据：

```
/* waitdisk - wait for disk ready */
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* do nothing */;
}
```

2. 然后由于disk上是扇区组成，故应按照扇区读取，将读取的数据存放在dst地址：

```
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

3. 利用已经写好的扇区读取的函数写出加载任意大小的disk数据到内存的函数：

```
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

4. bootmain启动时，由于ELF格式的文件结构为elf header加上各种program segment，elf header中存下了该elf文件的一些基本信息以及各个program segment的数量，偏移量等信息，因此先通过读取elf header，再读取每个segment，然后调用elf的入口：

```
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
}
```



## 练习5：实现函数调用堆栈跟踪函数

答：代码实现如下：

```
void
print_stackframe(void) {
    uint32_t ebp = read_ebp();		// 得到当前ebp寄存器值
    uint32_t eip = read_eip();		// 得到当前eip寄存器值

    int i;
    for(i = 0; i < STACKFRAME_DEPTH; i ++) {
	cprintf("ebp: 0x%08x ", ebp);
	cprintf("eip: 0x%08x ", eip);
	
	int j;       
	cprintf("args: ");
	uint32_t *args = (uint32_t *)ebp + 2;	// 由ebp偏移两个单位得到最后入栈的参数地址
	for(j = 0; j < 4; j ++) {
	    cprintf("0x%08x ", args[j]);        
	}
        cprintf("\n");

        print_debuginfo(eip - 1);
	
	// 分清地址和值
	eip = ((uint32_t *)ebp + 1)[0];
	ebp = ((uint32_t *)ebp)[0];
     }
}
```

输出结果截图如下：

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab1/report_image/exercise5.png)

输出结果与题干所给相似，以下解释最后一行的各个参数：

最后一行应该为最外层的调用函数，可以从打印语句看到其下一层为函数`kern_init`，故该层函数应该为`bootmain`函数，第一个参数为`ebp`寄存器中的值，此时由于起始地址为0x7c00，故调用`bootmain`后会push入栈一次，导致ebp值为0x7bf8，第二个值为`eip`寄存器里的值，其为将要执行的指令的地址（也就是该层函数执行完后的返回地址），打印信息之后依次为此时传入的参数。



## 练习6：完善中断初始化和处理。

1. 答：`IDT`中每一个表项占`8`个字节，对于`Interrupt-gate descriptor`和`trap-gate descriptor`也就是中断和异常而言，其第1-2字节和第7-8字节代表偏移量`offset`，其第3-4字节代表选择`selector`，共同构成入口地址。

2. 实现过程：按照提示`trap.c`文件中，`idt[256]`数组为待初始化的IDT，初始化使用定义的宏`SETGATE`，同时入口的偏移地址在`__vectors[]`中，段选择器选择定义的常量：内核代码段，`exception/interrupt`依照实验指导书后面所讲，`0-31`为异常，`32-255`为中断。最后依照要求调用`lidt()`传入参数`idt_pd`告知CPU，IDT的初始化已经完成。

   代码如下：

```
void idt_init(void) {
     // use SETGATE and __vectors[] set idt[256], kernel code segment, 0-31 for exception, 32-255 for interrupt
    extern uintptr_t __vectors[];
    int i;
    for(i = 0; i <= 255; i ++) {
        if(i <= 31) {
            SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_KERNEL);    
        }
        else {
            SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
        }
    }
    // switch for DPL_USER
    SETGATE(idt[T_SWITCH_TOU], 0, GD_KTEXT, __vectors[T_SWITCH_TOU], DPL_USER);
    // tell CPU
    lidt(&idt_pd);
}
```

3. 实现过程：使用全局变量`ticks`，由于该变量初始化为0，故每次进入该`trap`处理时，值加一，到`99`时即达到了100次，然后调用`print_ticks()`输出指定字符串并将全局变量归零。

   代码如下：

```
 case IRQ_OFFSET + IRQ_TIMER:
        ticks ++;
        if(ticks == 99) {
            print_ticks("100 ticks\n");
            ticks = 0;
        }
        break;
```

