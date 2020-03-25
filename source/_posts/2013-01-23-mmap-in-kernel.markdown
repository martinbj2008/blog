---
layout: post
title: "mmap in kernel"
date: 2013-01-23 00:00
comments: true
categories: [memory]
tags: [kernel, memory, mmap]
---

## call trace

```c
> vmalloc
> > __vmalloc_node_flags
> > > __vmalloc_node
> > > > __vmalloc_node_range
> > > > > get_vm_area_node
> > > > > vmalloc_area_node
> > > > > insert_vmalloc_vmlist
```
<!-- more -->

```c
1731 void *vmalloc(unsigned long size)
1732 {
1733         return __vmalloc_node_flags(size, -1, GFP_KERNEL | __GFP_HIGHMEM);
1734 }
```

```c
1715 static inline void *__vmalloc_node_flags(unsigned long size,
1716                                         int node, gfp_t flags)
1717 {
1718         return __vmalloc_node(size, 1, flags, PAGE_KERNEL,
1719                                         node, __builtin_return_address(0));
1720 }
```

```c
1700 static void *__vmalloc_node(unsigned long size, unsigned long align,
1701                             gfp_t gfp_mask, pgprot_t prot,
1702                             int node, void *caller)
1703 {
1704         return __vmalloc_node_range(size, align, VMALLOC_START, VMALLOC_END,
1705                                 gfp_mask, prot, node, caller);
1706 }
```

相当于调用
```c
__vmalloc_node_range(size, 1, VMALLOC_START, VMALLOC_END, GFP_KERNEL | __GFP_HIGHMEM,PAGE_KERNEL, -1, __builtin_return_address(0));
```

```c
1644 void *__vmalloc_node_range(unsigned long size, unsigned long align,
1645                         unsigned long start, unsigned long end, gfp_t gfp_mask,
1646                         pgprot_t prot, int node, void *caller)
1647 {
1648         struct vm_struct *area;
1649         void *addr;
1650         unsigned long real_size = size;
1651
1652         size = PAGE_ALIGN(size);
1653         if (!size || (size >> PAGE_SHIFT) > totalram_pages)
1654                 goto fail;
1655
1656         area = __get_vm_area_node(size, align, VM_ALLOC | VM_UNLIST,
1657                                   start, end, node, gfp_mask, caller);
1658         if (!area)
1659                 goto fail;
1660
1661         addr = __vmalloc_area_node(area, gfp_mask, prot, node, caller);
1662         if (!addr)
1663                 return NULL;
1664
1665         /*
1666          * In this function, newly allocated vm_struct is not added
1667          * to vmlist at __get_vm_area_node(). so, it is added here.
1668          */
1669         insert_vmalloc_vmlist(area);
1670
1671         /*
1672          * A ref_count = 3 is needed because the vm_struct and vmap_area
1673          * structures allocated in the __get_vm_area_node() function contain
1674          * references to the virtual address of the vmalloc'ed block.
1675          */
1676         kmemleak_alloc(addr, real_size, 3, gfp_mask);
1677
1678         return addr;
1679
1680 fail:
1681         warn_alloc_failed(gfp_mask, 0,
1682                           "vmalloc: allocation failure: %lu bytes\n",
1683                           real_size);
1684         return NULL;
1685 }
```

```c
1315 static struct vm_struct *__get_vm_area_node(unsigned long size,
1316                 unsigned long align, unsigned long flags, unsigned long start,
1317                 unsigned long end, int node, gfp_t gfp_mask, void *caller)
1318 {
1319         struct vmap_area *va;
1320         struct vm_struct *area;
1321
1322         BUG_ON(in_interrupt());
1323         if (flags & VM_IOREMAP) {
1324                 int bit = fls(size);
1325
1326                 if (bit > IOREMAP_MAX_ORDER)
1327                         bit = IOREMAP_MAX_ORDER;
1328                 else if (bit < PAGE_SHIFT)
1329                         bit = PAGE_SHIFT;
1330
1331                 align = 1ul << bit;
1332         }
1333
1334         size = PAGE_ALIGN(size);
1335         if (unlikely(!size))
1336                 return NULL;
1337
1338         area = kzalloc_node(sizeof(*area), gfp_mask & GFP_RECLAIM_MASK, node);
1339         if (unlikely(!area))
1340                 return NULL;
1341
1342         /*
1343          * We always allocate a guard page.
1344          */
1345         size += PAGE_SIZE;
1346
1347         va = alloc_vmap_area(size, align, start, end, node, gfp_mask);
1348         if (IS_ERR(va)) {
1349                 kfree(area);
1350                 return NULL;
1351         }
1352
1353         /*
1354          * When this function is called from __vmalloc_node_range,
1355          * we do not add vm_struct to vmlist here to avoid
1356          * accessing uninitialized members of vm_struct such as
1357          * pages and nr_pages fields. They will be set later.
1358          * To distinguish it from others, we use a VM_UNLIST flag.
1359          */
1360         if (flags & VM_UNLIST)
1361                 setup_vmalloc_vm(area, va, flags, caller);
1362         else
1363                 insert_vmalloc_vm(area, va, flags, caller);
1364
1365         return area;
1366 }
```
