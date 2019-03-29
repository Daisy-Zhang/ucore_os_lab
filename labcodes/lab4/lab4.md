# Lab4 实验报告

##### 2016011364 张岱墀

## 练习1：分配并初始化一个进程控制块

答：补全`kern/process/proc.c`中的函数`alloc_proc()`:

```
static struct proc_struct *alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    // 初始化清零操作
    proc -> state = PROC_UNINIT;
    proc -> pid = -1;
    proc -> runs = 0;
    proc -> kstack = 0;
    proc -> need_resched = 0;
    proc -> parent = NULL;
    proc -> mm = NULL;
    proc -> tf = NULL;
    proc -> cr3 = boot_cr3;
    memset(proc->name, 0, sizeof(proc->name));
    memset(&(proc -> context), 0, sizeof(proc -> context));
    proc -> flags = 0;
    }
    return proc;
}
```

查看相应结构体`proc_struct`以及相应实验提示后发现待初始化的成员变量包括：

* `proc_state state`：进程状态，进程新创建初始化时置为常量`PROC_UNINIT`，即还未被赋予相应状态
* `int pid`：进程的独有的id，新创建时置为`-1`，之后可以调用`get_pid()`函数来得到独有的`pid`
* `int runs`：进程运行总时间，新创建的进程运行时间自然设为`0`
* `uintptr_t kstack`：进程的内核栈，堆栈地址由于未设置故为`0`
* `bool need_resched`：进程是否需要被重分配来释放CPU，是否需要释放未知，初始化时置`0`
* `proc_struct *parent`：指向父进程的指针，创建时并未指定父进程，置`NULL`
* `mm_struct mm`：进程的内存管理类，由于新创建时并未分配，置`NULL`
* `context context`：进程的上下文(执行现场)，使用`memset()`将整个该块内存置`0`
* `trapframe tf`：进程的中断帧，指针成员同理未分配置`NULL`
* `uintptr_t cr3`：CR3寄存器的值，即PDT的基地址，由于整个操作系统的`CR3`寄存器的值存在`boot_cr3`这个变量中，故可直接设为该变量
* `uint32_t flags`：进程的标志位，未指定该标志位，全部初始化为`0`
* `char name[]`：进程的名称，由`set_proc_name()`函数可知，之后是传入指定参数设置进程名称，故初始化时使用`memset`将该块内存全置`0`



回答问题：

* 由后续的`copy_thread()`，`do_fork()`，`switch_to()`这些函数可以大致分析出：
  * `struct context context`：其作用类似于存下当前线程的执行现场信息，`context`结构体内部存下了诸多寄存器的值，通常用于新建线程的`copy`以及线程的`switch`等操作
  * `struct trapframe *tf`：其作用是作为线程的中断帧，`trapframe`这个结构体中存下的是有关该线程的相关中断信息，这些信息在后续`do_fork()`，`copy_thread()`时会被用作复制给新建的线程



## 练习2：为新创建的内核线程分配资源

答：按照相应实验要求和代码中注释，补全`kern/process/proc.c`中的函数`do_fork()`:

```
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;

    if((proc = alloc_proc()) == NULL) {
        cprintf("alloc proc NULL\n");
    }
    proc -> pid = get_pid();
    proc -> parent = current;
    if(setup_kstack(proc) != 0) {
        cprintf("set up kstack error\n");
    }
    if(copy_mm(clone_flags, proc) != 0) {
        cprintf("copy mm error\n");
    }
    copy_thread(proc, stack, tf);
    hash_proc(proc);
    list_add_after(&proc_list, &(proc->list_link));
    nr_process ++;
    wakeup_proc(proc);
    ret = proc -> pid;

fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

完成相应的资源分配大致分为以下步骤：

1. 调用在上一练习中补全的函数`alloc_proc()`创建并初始化一个`proc_struct`
2. 成功初始化一个新的子线程后，调用`get_pid()`函数为该内核线程分配一个独有的`pid`，同时将其所属的父内核线程设置为全局变量中的`current`内核线程块
3. 调用`setup_kstack()`函数为该新建的线程分配内核栈
4. 调用`copy_mm`根据相应标志位设置新线程的内存管理类成员
5. 调用`copy_thread`为新线程设置中断帧和执行上下文(执行时相应寄存器值)
6. 调用`hash_proc()`和lab3中的`list_add_after()`函数，将该新线程插入到`hash_list`和`proc_list`中，并将储存全部线程数的`nr_process`加一
7. 由于该内核线程还未被唤醒，调用`wakeup_proc()`将该内核线程唤醒
8. 将返回值`ret`设置为该新建子内核线程的`pid`



回答问题：

* 分配`pid`的函数为`get_pid()`函数，阅读该函数后，可以发现，该函数是遍历了整个`list_link`即线程列表，保证了最后返回的值不在整个列表中出现过，但是这样的前提是`MAX_PID = MAX_PROCESS * 2`，最大分配的`pid`的值为最多线程数的两倍，如果使用`do_fork()`新创建的线程数大于这个值，分配的`pid`可能会出现问题，这通过相应线程数限制即可解决



## 练习3：阅读代码，理解`proc_run`函数和它调用的函数如何完成进程切换的

答：阅读完`proc_run()`函数之后可以发现其功能是：接收一个传入的进程指针，然后将当前执行的进程切换到该指定进程。其中步骤大致如下：

1. 判断传入的指定进程是否是当前正在执行的进程，若不是，则将`prev`设置为正在执行的进程，将`next`设置为传入的即要切换到的目的进程，准备进行进程间的切换
2. 调用`local_intr_save()`函数，其作用是调用`intr_disable()`使能关闭中断
3. 之后将全局的`current`进程设置为传入的进程，调用`load_esp0()`加载内核栈，调用`lcr3()`设置CR3寄存器的值，然后调用`switch_to()`函数，该函数的实现在`switch.S`中，是完全由汇编实现的，功能为存下之前进程的执行时`context`即执行时各个寄存器的值，然后将其赋给切换的目标进程的各个寄存器，完成执行现场的移交
4. 调用`local_intr_restore()`，其进一步调用了`intr_enble()`函数，恢复了中断



回答问题：

* 总共创建并执行了两个内核线程：
  * `idleproc内核线程`：该线程并未干实际工作，主要是用于检测线程切换，该线程创建后不久由于CPU释放资源而被`reschedule`
  * `initproc内核线程`：该线程创建之后主要的功能是输出指定字符串：`Hello,world`和`en...Bye,Bye`
* 如上述过程中所叙述，`local_intr_save()`函数的作用为关中断，`local_intr_restore()`函数为恢复中断



## 实验完成后相关截图

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab4/lab4_res.png)



## 与ucore_lab中的参考答案进行对比

#### 练习一：分配并初始化一个进程控制块

> 与`ucore_lab/labcodes_answer`中给出的答案基本一致，都只涉及变量初始化为0，指针初始化为NULL，连续内存空间置0的操作

#### 练习二：为新创建的内核线程分配资源

> 与`ucore_lab/labcodes_answer`中给出的答案完成的工作大致一致，不过答案中在进行全局进程数加一，分配`pid`，`hash_list`，`proc_list`的插入过程时先关闭了中断，之后再恢复了中断(类似第三个练习中`proc_run`的行为)
>
> 个人理解这样做的目的是，防止在切换进程时CPU被其他任务中断导致切换发生`error`，对于程序的鲁棒性可能有一定的增强

#### 练习三：阅读代码，理解`proc_run`函数和它调用的函数如何完成进程切换的

> 并未给出与该练习相关的答案



## 列出本实验个人觉得重要的知识点以及与课程对应内容的对应

- 内核线程(共享内存空间的轻量级进程)的管理，对应本实验中的练习一与练习二
- 内核线程(共享内存空间的轻量级进程)的切换，对应本实验中的练习三



## 列出觉得OS原理中很重要的但是该实验中未被涉及的知识点

- 关于进程概念中很重要的一块：`Process Control Block`，本次实验中只涉及到了其初始化和创建子进程(内核线程)的分配资源过程，对于其生命周期中的终止，多个`PCB`的管理等涉及较少
- 对于进程的`三状态模型`(或者`五状态模型`)，本次实验中只涉及到了进程切换，创建(子)进程，但是个人感觉其他的状态也有进一步在实验中认知的必要