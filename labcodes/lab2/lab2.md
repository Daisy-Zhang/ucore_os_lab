# Lab2 实验报告

2016011364 张岱墀

## 练习1：实现 first-fit 连续物理内存分配算法

答：在`lab2/kern/mm/default_pmm.c`目录下，选择性重写了四个函数： `default_init()`, `default_init_memmap()`,

 `default_alloc_pages()`, `default_free_pages()`，以下分别说明四个函数的设计流程：

* `default_init()`函数：

```
static void default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

该函数主要功能为，使用`list.h`中提供的初始化方法初始化用于存下整个`free block`的`list`，然后将用于记录总共`free`的`pages`的变量设为0。

* `default_init_memmap()`函数：

```
static void default_init_memmap(struct Page *base, size_t n) {  // 初始化一块block，相当于在list的每一项下面拉链，第一页存下size
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
		p -> flags = 0;
        p -> property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add_before(&free_list, &(base->page_link));
}
```

初始化由`base`为起始地址，`n`为页数目的一块`free block`，主要操作为：将每一页`page`的相关`bit`位设置为指定值，然后该块的第一页的`property`成员设置为该块的总页数，然后更新`nr_free`全局变量，将该块通过`list.h`中的相关接口加入到`list`中。

* `default_alloc_pages()`函数：

```
static struct Page *default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        list_del(&(page->page_link));       // 先把找到的该block从list里面删掉
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;   // 修改该block的free pages数量
            SetPageProperty(p);
            list_add(&free_list, &(p->page_link));  // 将新的block加入到list中
        }
        nr_free -= n;                       // 更新总的free pages数
        ClearPageProperty(page);            // 更新该block对应相关bit位
    }
    return page;
}
```

该函数主要功能为：在整个`free list`中寻找第一块满足`size`大于等于传入参数`n`的`block`，找到后从`list`中删除该块，将其前`n`页做上指定标记，然后取其剩下的页数重新作为一个新`block`加入到`list`中，更新相应的`property`和`nr_free`等参数。

* `default_free_pages()`函数：

```
static void default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    list_entry_t *le = list_next(&free_list);
    while (le != &free_list) {
        p = le2page(le, page_link);
        le = list_next(le);
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        else if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            list_del(&(p->page_link));
        }
    }
    nr_free += n;
    le = list_next(&free_list);
    while (le != &free_list) {
        p = le2page(le, page_link);
        if (base + base->property <= p) {
            assert(base + base->property != p);
            break;
        }
        le = list_next(le);
    }
    list_add_before(le, &(base->page_link));
}
```

该函数的主要功能为：对于指定的新的且为`free`页，首先按照地址高低遍历寻找到该页该被插入的地方，插入该页，更新全局的`nr_free`，然后和初始化类似地进行相关位的重置，最后通过再一次的遍历试图去`merge`该块周围的块。

（附：以上函数在`lab2`自身代码里已经实现得差不多了，感觉实际上没有太多要加的）

关于以上的`first fit`算法的进一步改进空间：

* 若比较看重时间复杂度时，可以考虑在遍历整个`free list`时将简单的遍历替换为二分查找以节约时间。
* 可以考虑在输入某一个`size n`时，若找到了第一个符合条件的`block`，则另开内存记录下来这个已经搜索得到的情况，再不插入新页以及合并的前提下，若下次输入`size >= n`，则一定不会在之前找到的块之前找到目标块，通过记录已得信息来减少相应搜索范围。



## 练习2：实现寻找虚拟地址对应的页表项

答：该部分主要实现的代码在目录`lab2/kern/mm/pmm.c`下，待修改函数为`get_pte()`，该函数接受输入`page directory table`的基址，待映射的线性地址，不存在二级页表时是否创建新页时的标志位`create`，返回对应的二级页表的`entry`地址。以下为具体实现代码：

```
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    pde_t *pdep = &pgdir[PDX(la)];   	// (1) find page directory entry
    struct Page *p = NULL;

    if ((*pdep & PTE_P) == 0x0) {       // (2) entry is not present, 0 is not the same as 0x0
        if(create) {
            if((p = alloc_page()) == NULL){return NULL;}       // (3) check if creating is needed, then alloc page for page table 
            	// CAUTION: this page is used for page table, not for common data page 
        }
        else {
            return NULL;
        }
        set_page_ref(p, 1);              		// (4) set page reference
        uintptr_t pa = page2pa(p);       		// (5) get linear address of page, first get physical address then get virtual address
        memset(KADDR(pa), 0, PGSIZE);           // (6) clear page content using memset
        *pdep = pa | PTE_P | PTE_W | PTE_U;    	// (7) set page directory entry's permission     }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];  // (8) return page table entry
}
```

首先通过由线性地址得到一级`PDT`的`index`，然后寻址得到`page directory entry`的地址，之后检查其`PTE_P`标志位，检查二级页表是否存在，若存在，则直接寻址后得到`page table entry`；若不存在，则由传入的`create`标志来决定是否新创建一页用作该一级页表项的二级页表项，若不创建新的二级页表，则直接返回`NULL`；若需要创建新的二级页表，则调用`alloc_page()`分配新的物理页。之后分别为该新页：`reference`成员变量加一；将该页的内容用`memset()`全置为零；最后将相应`page directory entry`的三个`PTE_P,PTE_W,PTE_U`标志位设为`1`。然后返回由`PTX()`得到的索引值偏移得到的`page table entry`地址。

### 回答相关问题：

- 在分页机制管理中，`CR3`寄存器存下`page directory table`的基地址，通过线性地址的高十位索引该一级`table`，`entry`里存下对应二级`table`的基地址，线性地址中紧跟着的十位用来索引二级`table`中的`entry`，根据该中的二十位加上线性地址的最低十二位得到最终的物理地址。
- `ucore`访问内存出现页访问异常时，可能的情况有：相关映射被撤除或未建立，相应的物理页不在内存中或者不存在。硬件需要做的事可能包括：保存异常现场，加载外存，然后调用`ucore`的相关异常处理程序，把剩余的例如申请新物理页，建立映射关系等工作交给操作系统。



## 练习3：释放某虚拟地址所在页并取消对应二级页表项的映射

答：该部分主要待实现目录`lab2/kern/mm/pmm.c`中的`page_remove_pte()`函数，该函数的主要功能为：接受输入的虚拟地址，`page directory table`的基地址，与该虚拟地址相关联的`page table entry`的指针。然后对该页进行相应的释放操作并取消二级表项和该页之间的映射。具体实现如下：

```
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {      // ptep: page table entry pointer
    if (*ptep & PTE_P) {                   		 //(1) check if this page table entry is present
        struct Page *page = NULL;                 
        page = pte2page(*ptep);                  //(2) find corresponding page to pte
        int ref = page_ref_dec(page);            //(3) decrease page reference
        if(ref == 0) {
            free_page(page);
        }                                      	 //(4) and free this page when page reference reachs 0
        ptep = NULL;                           	 //(5) clear second page table entry
        tlb_invalidate(pgdir, la);             	 //(6) flush tlb
    }
}
```

首先通过检查相关的`page table entry`的相应位(此处为`PTE_P`位)，在该`entry`实际存在的情况下，进行如下操作：得到该`entry`关联的页地址，然后将该页的成员变量`reference`减去一，若此后该页的`reference`成员变量为0，则代表该页中并不存在与之关联的虚拟地址，该页失去作用，调用`free_page()`将该页释放，之后将指向该页的`ptep`指针置空，最后调用`tlb_invalidate()`函数更新相关`tlb`。

### 回答相关问题

* 以下回答均基于假设：题目所指的全局变量`Page`是`pmm_manager`中的输入参数为`n`的`Page`数组成员变量：该成员变量为`pmm_manager`所管理的物理页数组，其中的所有物理地址均由线性地址再到虚拟地址与每一个虚拟地址之间形成了映射，这之中，每一页`physical page`的基地址的高二十位可以映射为特定页表项的内容，同时该页表项的基地址，也可以映射为某一特定页目录项的内容。
* 由实验指导中`系统执行地址映射的四个阶段`中的说明可以得知，此时实验中最终映射关系为：`virt address = linear address = phy address + 0xC0000000`，在`stage 4`之后，只需要再将虚拟地址对应的线性地址"向上平移4MB"即可得到，即认为将紧跟着`stage 4`中清零的页目录表项下一个表项清零。



## 实验完成相关截图：

当完成上述步骤后，在`lab2`目录下运行`make qemu`时的截图：(在未注释掉`lab1`中相关`print_stackframe()`相关打印信息时，整个`lab2`根本无法运行，以下运行截图是在注释掉上述部分后得到)

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab2/lab2_qemu.png)



## 与ucore_lab中的参考答案进行对比

#### 练习一：First-Fit连续物理内存分配算法的实现：

由于按照注释的提示实现的`First-Fit`算法，所以和参考答案中的实现基本一致，不过还是有少许不同，比如`free`时`merge`小块的方法和将物理页添加进`free_list`的`add`操作。

#### 练习二：实现寻找虚拟地址的页表项：

由于也是按照原本注释完成，所以使用到的一些函数接口大致一致，但是自己是直接注释掉了条件编译指令然后加在其原本位置，其余少许不同的地方是对于页表项的`present`的判定方式，以及参考答案的返回形式更简洁。

#### 练习三：释放某虚拟地址所在页并取消对应二级页表项的映射

同上，按照已有注释完成，使用的函数接口大致一致，同样注释掉了条件编译指令直接加在其原本位置，实现过程基本一样，不同之处在于清除二级表项时参考答案将指针值设为全0，而我是将原本指针置为`NULL`，思考之后觉得答案更在理，自己的设置并没有达到将页表项清除的目的，然后将其更改。



## 列出本实验个人觉得重要的知识点以及与课程对应内容的对应

* 第一个实验中的`first-fit`物理内存分配方法，对应OS课程上的连续内存管理与非连续内存管理算法。本次实验中实现的`first-fit`算法即为OS课程上讲过的诸多物理内存分配算法之一，同样也属于比较容易实现的算法，但是缺点在于随着内存占用的时间效率以及碎片内存的管理。
* 第二个实验和第三个实验中的对页表项的释放和建立映射等，对应OS课程上的内存访问的段页式机制。该两个实验任务所干的事就是实现了上课所讲的段页式内存管理机制中的分配和释放功能，涉及了段表和两级页表的相关理解和操作。



## 列出觉得OS原理中很重要的但是该实验中未被涉及的

* 个人觉得计算机内部的特权级机制(如上课所讲的`x86`特权级)在计算机内部十分重要，但在此次试验中并未涉及到太多。