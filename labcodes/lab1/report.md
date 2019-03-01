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

