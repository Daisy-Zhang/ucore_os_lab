# Lab3 实验报告

##### 2016011364 张岱墀

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

完成后相关截图：

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab3/report_image/exercise1.png)

回答问题：

* 在分页机制管理中，`CR3`寄存器存下`page directory table`的基地址，通过线性地址的高十位索引该一级`table`，`entry`里存下对应二级`table`的基地址，线性地址中紧跟着的十位用来索引二级`table`中的`entry`，根据该中的二十位加上线性地址的最低十二位得到最终的物理地址。除了在页替换算法中直接索引得到物理页之外，对于某些页替换算法中对物理页的记录(如记录访问位，访问次数等)可以转化为对页表项的位操作，这也体现了其潜在用处。
* `ucore`访问内存出现页访问异常时，可能的情况有：相关映射被撤除或未建立，相应的物理页不在内存中或者不存在。硬件需要做的事可能包括：保存异常现场，加载外存，然后调用`ucore`的相关异常处理程序，把剩余的例如申请新物理页，建立映射关系等工作交给操作系统。



## 练习2：补充完成基于FIFO的页面替换算法

答：首先继续完善`kern/mm/vmm.c`中的`do_pgfault()`函数：

```
if(swap_init_ok) {
            struct Page *page = NULL;
            int si = swap_in(mm, addr, &page);   
            int pi = page_insert(mm -> pgdir, page, addr, perm);  
            int sms = swap_map_swappable(mm, addr, page, 1);    
            page -> pra_vaddr = addr;
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
```

依照`lab3相关代码注释`，当我们认为当前页表项是一个可以交换的页时，我们需要做的事包括：

* 首先从外部存储设备，如`disk`中将其内存读到指定的内存页，即上述中的`swap_in()`函数
* 然后建立新的物理地址和逻辑地址的映射关系，即上述中的`page_insert()函数`
* 最后，标记该页可被交换并将该`page`中的`pra_vaddr`(在页面置换算法中所用到的虚拟地址)，置为当前页地址

接下来在`kern/mm/swap_fifo.c`中实现`first in first out`页置换算法，补全`_fifo_map_swappable()`和`_fifo_swap_out_victim()`函数，由于试验的该部分是基于双向链表，我首先研读了`list.h`中双向链表数据结构的实现，比较强调的一点是：该置换算法中所用到的双向链表是首尾成环的双向链表，且存下的为该链表中不变的一个表项`head`，实现上述两个函数时由于没有注意这一点导致调试了一段时间。

* 对于`_fifo_map_swappable()`函数：该函数的功能是：依据`FIFO`算法，最近被访问的页最后被换出，故将被访问到的页面加入到双向链表的`队尾`，由于是成环的双向链表，故只需要在指定的`head`处进行`add_after`操作即可：

  ```
  static int _fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
  {
      list_entry_t *head=(list_entry_t*) mm->sm_priv;
      list_entry_t *entry=&(page->pra_page_link);
      list_init(entry);
      assert(entry != NULL && head != NULL);
      //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
      list_add_after(head, entry);
      return 0;
  }
  ```


* 对于`_fifo_swap_out_victim()`函数：该函数的功能是：将最先被访问且还在双向链表中的对应页置换出，同时将`ptr_page`的值置为被换出的页的地址，由于双向链表是首尾成环，且每次定位的`head`不变，向链表中添加时是加在`队尾`，即`head`之后，故链表中代表最先访问的项为`head`的前驱节点对应的页，找到该页后，利用`struct Page`的`le2page()`函数由链表项找到页面即可：

  ```
  static int _fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
  {
      list_entry_t *head=(list_entry_t*) mm->sm_priv;
      assert(head != NULL);
      assert(in_tick==0);
      //(1)  unlink the earliest arrival page in front of pra_list_head qeueue
      head = head -> prev;
      struct Page *p = le2page(head, pra_page_link);
      list_del(head);
      //(2)  assign the value of *ptr_page to the addr of this page
      * ptr_page = p;
      return 0;
  }
  ```

回答问题：

* 被置换出的页的特征：按照换出优先级由高到低：`未被引用且未被修改 > 未被引用被修改 > 被引用未被修改 > 被引用被修改`

* 在`ucore`中如何判断具有该特征的页：在页表项中预留两位：`修改位`和`引用位`。用此两位标志位的值，在每次需要置换时判断页的特征
* 进行换入和换出的时机：当发生缺页时，依据页的`修改位`和`引用位`确定页置换

觉得当前的`swap_manager`框架可以实现`extended clock`算法：在`map_swappable()`中实现对当前传入`page`的`修改位`和`引用位`的修改，在`swap_out_victim()`中遍历所有页表项，按以上优先级选出最高且最先搜索到的页表项进行替换



## 实验完成后相关截图

![](/Users/macbookair/Desktop/大三下/操作系统/ucore_os_lab/labcodes/lab3/report_image/lab3_res.png)



## 与ucore_lab中的参考答案进行对比

#### 练习一：给未被映射的地址映射上物理页

> 实现的功能和调用的相关函数，宏基本一致，不过感觉答案实现的比较优美的一点是：其有效利用函数的返回值进行了一些逻辑性判断，输出了一些打印信息并且跳转到相应处理区域，或者在出现问题时直接中断当前进程。这一点是我在实现中没有太过于考虑到的

#### 练习二：补充完成基于FIFO的页替换算法

> 实现的功能以及调用的相关函数和宏基本一致，不过答案的实现相对更加鲁棒一些：对于一些函数的返回值加上了`assert()`函数，有效地进行了一些逻辑检查并且也方便了相关调试



## 列出本实验个人觉得重要的知识点以及与课程对应内容的对应

- 虚拟内存的页管理机制，即本实验的练习一所涉及到的内容，与课程中的`缺页以及虚拟内存页管理机制`相对应
- 页置换算法，即本实验的练习二所涉及到的内容，与课程中的`局部页置换算法`和`全局页置换算法`这一块内容相对应



## 列出觉得OS原理中很重要的但是该实验中未被涉及的知识点

- 某些置换算法的`Belady`现象。个人感觉`Belady`现象对于操作系统实际影响还是挺大的，但是本次实验中并没有涉及该方面的相关探究内容
- 全局置换算法。本次实验中涉及到的置换算法只涉及了课上所讲到的`局部页置换算法`，但是课上所讲还有一块`全局页置换算法`，这一块页置换算法个人感觉也是很重要的，不过客观上本次实验还未涉及到并发进程，所以也无法探究