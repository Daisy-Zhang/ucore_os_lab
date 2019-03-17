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

* 
* 