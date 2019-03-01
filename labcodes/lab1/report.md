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

