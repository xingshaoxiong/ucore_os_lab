# lab2实验报告

## 练习1

first-fit算法主要是链表的遍历，其中`free_list`是一个双向链表，单个元素是连续的内存页组成的块。

程序的改动主要有一个地方，就是在分配和清空之后，都保持相对位置不变，而不是先删除整个元素，再在链表末尾更新。依据主要是程序的局部性原理。

具体的代码实现逻辑比较简单，就不赘述了。

## 练习2

ucore中线性地址的含义如下

```c
// A linear address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
//
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).
```

根据`mmu.h`可知，PDE和PTE有一些标志位，如下所示

```c
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
```

这些标志位正好占据了10位，因为地址都是对齐的，这些值在寻址时会被屏蔽。

可以看出这些表项有助于后续的内存管理，比如权限和页面置换，而且应该注意到，二级页表不是一开始分配好的，而是使用过程中分配的，这就是为什么

如果出现了页访问异常，硬件会主动进入中断，并进行处理，这一部分在Lab3中有比较详细的解释，这里就不再赘述。

```c
pde_t *pdep = &pgdir[PDX(la)];
    if (!(*pdep & PTE_P))
    {
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
```

此时我们已经由练习1做好了物理内存的检测，在这一部要完成映射，一级页表在`pmm.h`已经声明，起始地址存在CR3中，如果二级页表不存在或者指向的物理页不存在，我们则需要为其重新分配物理内存，在这里每次只分配一个物理页，得到分配到的物理地址后，进行标志位设定，然后返回虚拟地址。

## 练习3

```c
    if (0) {                      //(1) check if this page table entry is present
        struct Page *page = NULL; //(2) find corresponding page to pte
                                  //(3) decrease page reference
                                  //(4) and free this page when page reference reachs 0
                                  //(5) clear second page table entry
                                  //(6) flush tlb
    }
    if (*ptep & PTE_P)
    {
        struct Page *page = pte2page(*ptep);
        if (page_ref_dec(page) == 0)
        {
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```
上面的代码很容易理解，主要就是在释放之前查看是否引用已经为0，如果是，则释放，并清空相关物理内存，并且刷新tlb
Page的全局变量和页目录项和页表有关系，其中pages的顺序是严格按照物理内存的顺序的, 所以可以根据page在链表中的位置得到物理地址
```c
static inline ppn_t
page2ppn(struct Page *page) {
    return page - pages;
}
‵‵‵
如果希望虚拟地址和物理地址相同，需要修改KERNBASE为0x00000000，然后虚拟地址也要减去0xC0000000