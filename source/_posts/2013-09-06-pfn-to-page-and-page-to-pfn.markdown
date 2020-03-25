---
layout: post
title: "pfn_to_page and page_to_pfn"
date: 2013-09-06 17:11
comments: true
categories: [memory]
tags: [memory, pfn, page]
---

###summary
`page_to_pfn` and `pfn_to_page` are often used in kernel.

for PP81, kernel uses CONFIG_DISCONTIGMEM + NUMA.
Every node has `node_mem_map` and `node_start_pfn`to help the page/pfn covertion.
`node_start_pfn` is the first page's pfn of this node.
`node_mem_map` stores all the `struct page ` of this node.

so thete is a map between them:

    `node_start_pfn + i` <===> `node_mem_map[i]`

The key is how to get **node id** by pfn or page.
the corresponding function is `pfn_to_nid` and `page_to_nid`

`page_to_nid` is simpile. 
    the `node id` is store in `struct page ->flags`
while `pfn_to_nid`(for pp81), it convert to pageaddress and then turn to `pa_to_nid`.

感觉这个实现有点罗嗦！
直接在include/asm-generic/memory_model.h 把`pa_to_nid` 定义一个 `arch_pfn_to_nid`宏。
省得pfn转为pa以后，又转为pfn。


<!-- more -->

### `pfn_to_page`
funtion defination:
include/asm-generic/memory_model.h
```c
 73 #define pfn_to_page __pfn_to_page
```

```c
 33 #elif defined(CONFIG_DISCONTIGMEM)
 34    
 35 #define __pfn_to_page(pfn)                      \
 36 ({      unsigned long __pfn = (pfn);            \
 37         unsigned long __nid = arch_pfn_to_nid(__pfn);  \
 38         NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\
 39 }) 
```

#### ` arch_local_page_offset`
```c
 18 #ifndef arch_local_page_offset
 19 #define arch_local_page_offset(pfn, nid)        \
 20         ((pfn) - NODE_DATA(nid)->node_start_pfn)
 21 #endif
```

#### `arch_pfn_to_nid`

call trace:

```c
arch_pfn_to_nid 
  --> pfn_to_nid
    -->pa_to_nid((pfn) << PAGE_SHIFT)
```

```c
 14 #ifndef arch_pfn_to_nid
 15 #define arch_pfn_to_nid(pfn)    pfn_to_nid(pfn)
 16 #endif
```

arch/mips/include/asm/mmzone.h
```c
 11 #ifdef CONFIG_DISCONTIGMEM
 12 
 13 #define pfn_to_nid(pfn)         pa_to_nid((pfn) << PAGE_SHIFT)
```

arch/mips/include/asm/mach-netlogic/mmzone.h
```c
 52 static inline unsigned int pa_to_nid(unsigned long addr)
 53 {
 54         unsigned int  i;
 55         unsigned long pfn = addr >> PAGE_SHIFT;
 56
 57         /* TODO: Implement this using NODE_DATA */
 58         for (i = 0; i < NLM_MAX_CPU_NODE; i++) {
 59
 60                 if ((!node_online(i)) || ((NODE_MEM_DATA(i)->low_pfn == 0) && (NODE_MEM_DATA(i)->high_pfn == 0)))
 61                         continue;
 62
 63                 if (pfn >= NODE_MEM_DATA(i)->low_pfn && pfn <= NODE_MEM_DATA(i)->high_pfn)
 64                         return i;
 65         }
....
```

### `page_to_pfn`
include/asm-generic/memory_model.h
```c
 72 #define page_to_pfn __page_to_pfn
```

```c
 41 #define __page_to_pfn(pg)                                               \
 42 ({      struct page *__pg = (pg);                                       \
 43         struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg));     \
 44         (unsigned long)(__pg - __pgdat->node_mem_map) +                 \
 45          __pgdat->node_start_pfn;                                       \
 46 })
```

### `page_to_nid`
```c
 538 #ifdef NODE_NOT_IN_PAGE_FLAGS
 539 extern int page_to_nid(struct page *page);
 540 #else
 541 static inline int page_to_nid(struct page *page)
 542 {
 543         return (page->flags >> NODES_PGSHIFT) & NODES_MASK;
 544 }
 545 #endif
```

