# Lab3 实验报告

2016011364 张岱墀

## 练习1：给未被映射的地址映射上物理页

答：修改`kern/mm/vmm.c`目录下代码：

```
 /*LAB3 EXERCISE 1: 2016011364*/
    ptep = get_pte(mm -> pgdir, addr, 1);                       //(1) try to find a pte, if pte's PT(Page Table) isn't existed, then create a PT.
    if (*ptep == 0) {					
		struct Page *page = NULL;
        page = pgdir_alloc_page(mm -> pgdir, addr, perm);      //(2) if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr
    }
```

设计思路：按照指导，首先由`lab2`中实现了的`get_pte()`函数，传入参数`memory manager`的成员变量`pg`基地址：`pgdir`，待映射的虚拟(线性)地址：`addr`，以及在`PT`不存在时创建`PT`的标志位`create = 1`，得到返回的`pte`的指针，假如该指针的值即实际物理地址不存在，则调用函数`pgdir_alloc_page()`函数，传入`pg`基地址，待映射虚拟地址`addr`和相应的权限`perm`(该权限已经在前面代码中计算给出)。值得注意的是该函数是一个有返回值`page*`的函数，故应当用一个相应`page`类的指针来接受返回值。

回答问题：

* 在分页机制管理中，`CR3`寄存器存下`page directory table`的基地址，通过线性地址的高十位索引该一级`table`，`entry`里存下对应二级`table`的基地址，线性地址中紧跟着的十位用来索引二级`table`中的`entry`，根据该中的二十位加上线性地址的最低十二位得到最终的物理地址。除了在页替换算法中直接索引得到物理页之外，对于某些页替换算法中对物理页的记录(如记录访问位，访问次数等)可以转化为对页表项的位操作，这也体现了其潜在用处。
* `ucore`访问内存出现页访问异常时，可能的情况有：相关映射被撤除或未建立，相应的物理页不在内存中或者不存在。硬件需要做的事可能包括：保存异常现场，加载外存，然后调用`ucore`的相关异常处理程序，把剩余的例如申请新物理页，建立映射关系等工作交给操作系统。