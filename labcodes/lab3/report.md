# lab3实验报告

## 练习1
```c
#if 0
    /*LAB3 EXERCISE 1: YOUR CODE*/
    ptep = ???              //(1) try to find a pte, if pte's PT(Page Table) isn't existed, then create a PT.
    if (*ptep == 0) {
                            //(2) if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr

    }
    else {
    /*LAB3 EXERCISE 2: YOUR CODE
    * Now we think this pte is a  swap entry, we should load data from disk to a page with phy addr,
    * and map the phy addr with logical addr, trigger swap manager to record the access situation of this page.
    *
    *  Some Useful MACROs and DEFINEs, you can use them in below implementation.
    *  MACROs or Functions:
    *    swap_in(mm, addr, &page) : alloc a memory page, then according to the swap entry in PTE for addr,
    *                               find the addr of disk page, read the content of disk page into this memroy page
    *    page_insert ï¼š build the map of phy addr of an Page with the linear addr la
    *    swap_map_swappable ï¼š set the page swappable
    */
        if(swap_init_ok) {
            struct Page *page=NULL;
                                    //(1ï¼‰According to the mm AND addr, try to load the content of right disk page
                                    //    into the memory which page managed.
                                    //(2) According to the mm, addr AND page, setup the map of phy addr <---> logical addr
                                    //(3) make the page swappable.
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }
#endif
    if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL)
    {
        goto failed;
    }
    if (*ptep == 0)
    {
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL)
        {
            goto failed;
        }
    }
    else
    {
        if (swap_init_ok){
            struct Page *page=NULL;
            if ((ret = swap_in(mm, addr, &page)) != 0) {
                cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }    
            page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
            page->pra_vaddr = addr;
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
    }
   ret = 0;
failed:
    return ret;
```
由上文的代码可以看出，先调用`get_pte()`函数得到页表地址，此时如果不存在，需要区分两种情况，一种是尚未分配，一种是已经分配了，对于前一种，直接分配新的物理内存就可以了
对于后者，说明相关物理内存被置换到了硬盘上，则应该在分配内存之后再将其从硬盘上读取过来
对于标志位的用处，举个例子，某些内存是不允许置换的，需要标记。

## 练习2
FIFO算法主要思想就是去置换出在内存中存放时间最长的内存，我们可以把程序调入内存的页按照先后顺序插入链表，这样从队列头很容易查找到需要淘汰的页。
这一部分需要关注几个数据结构和他们代表的含义
```c
struct vma_struct {
    // the set of vma using the same PDT
    struct mm_struct *vm_mm;
    uintptr_t vm_start; // start addr of vma
    uintptr_t vm_end; // end addr of vma
    uint32_t vm_flags; // flags of vma
    //linear list link which sorted by start addr of vma
    list_entry_t list_link;
};

struct mm_struct {
    // linear list link which sorted by start addr of vma
    list_entry_t mmap_list;
    // current accessed vma, used for speed purpose
    struct vma_struct *mmap_cache;
    pde_t *pgdir; // the PDT of these vma
    int map_count; // the count of these vma
    void *sm_priv; // the private data for swap manager
};
```
需要特别注意，`sm_priv`指向的是可交换页的头部
```c
static int
_fifo_init_mm(struct mm_struct *mm)
{     
     list_init(&pra_list_head);
     mm->sm_priv = &pra_list_head;
     //cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);
     return 0;
}
```
所以主要代码如下，需要注意的就是双向链表的特性，`head->prev == back`
```c
//record the page access situlation
    /*LAB3 EXERCISE 2: YOUR CODE*/
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add(head, entry);


/*LAB3 EXERCISE 2: YOUR CODE*/
    //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
    //(2)  assign the value of *ptr_page to the addr of this page
    list_entry_t *le = head->prev;
    assert(head != le);
    struct Page *p = le2page(le, pra_page_link);
    list_del(le);
    assert(p != NULL);
    *ptr_page = p;
    return 0;
```
