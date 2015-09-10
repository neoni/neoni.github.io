---
layout: post
title: ".got.plt Research"
tags: [exploit, .dynamic, link_map]
comments: true
date: 2015-09-09T10:50:17+08:00
---

.got.plt的前3项，分别是`.dynamic`的地址，`link_map` 的地址和 `_dl_runtime_resolve` 的地址

由此可以得到很多有用的信息

## .dynamic

.dynamic 保存了动态链接器所需要的基本信息，比如依赖哪些共享对象、动态链接符号表的位置、动态链接重定位表的位置、共享对象初始化代码的地址等。

使用 `readelf -d xxx` 可以查看 .dynamic

```python
Dynamic section at offset 0x1f04 contains 26 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libncurses.so.5]
 0x00000001 (NEEDED)                     Shared library: [libtinfo.so.5]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x80486bc
 0x0000000d (FINI)                       0x8049934
 0x00000019 (INIT_ARRAY)                 0x804aef8
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x0000001a (FINI_ARRAY)                 0x804aefc
 0x0000001c (FINI_ARRAYSZ)               4 (bytes)
 0x6ffffef5 (GNU_HASH)                   0x80481ac
 0x00000005 (STRTAB)                     0x8048400
 0x00000006 (SYMTAB)                     0x80481f0
 0x0000000a (STRSZ)                      393 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000015 (DEBUG)                      0x0
 0x00000003 (PLTGOT)                     0x804b000
 0x00000002 (PLTRELSZ)                   168 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x8048614
 0x00000011 (REL)                        0x80485fc
 0x00000012 (RELSZ)                      24 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffe (VERNEED)                    0x80485cc
 0x6fffffff (VERNEEDNUM)                 1
 0x6ffffff0 (VERSYM)                     0x804858a
 0x00000000 (NULL)                       0x0
 ```

#### .dynamic 的结构

```C
typedef struct {
        Elf32_Sword d_tag;
        union {
                Elf32_Word      d_val;
                Elf32_Addr      d_ptr;
                Elf32_Off       d_off;
        } d_un;
} Elf32_Dyn;

typedef struct {
        Elf64_Xword d_tag;
        union {
                Elf64_Xword     d_val;
                Elf64_Addr      d_ptr;
        } d_un;
} Elf64_Dyn;
```

对每一个有该类型的object，d\_tag控制着d\_un的解释。

* d\_val
  
  那些Elf32\_Word object描绘了具有不同解释的整形变量。

* d\_ptr
  
  那些Elf32\_Word object描绘了程序的虚拟地址。就象以前提到的，在执行时，
  文件的虚拟地址可能和内存虚拟地址不匹配。当解释包含在动态结构中的地址
  时是基于原始文件的值和内存的基地址。为了一致性，文件不包含在
  重定位入口来纠正在动态结构中的地址。

#### 在gdb中查看 .dynamic 的内容

可以看到 `0x0000000c (INIT) 0x80486bc` 在 0x804af1c 中相对应
 
```python
gdb-peda$ x/40xw 0x804af04
0x804af04:  0x00000001  0x00000001  0x00000001  0x000000e0
0x804af14:  0x00000001  0x000000ee  0x0000000c  0x080486bc
0x804af24:  0x0000000d  0x08049934  0x00000019  0x0804aef8
0x804af34:  0x0000001b  0x00000004  0x0000001a  0x0804aefc
0x804af44:  0x0000001c  0x00000004  0x6ffffef5  0x080481ac
0x804af54:  0x00000005  0x08048400  0x00000006  0x080481f0
0x804af64:  0x0000000a  0x00000189  0x0000000b  0x00000010
0x804af74:  0x00000015  0x55575918  0x00000003  0x0804b000
```
---

## link\_map

每个被加载入内存的so文件都会在内存中有一个link\_map结构，并且整个进程地址空间中所有的这些map结构由r\_debug为头组成一个链表，在知道了这个链表的起始地址之后，我们就可以遍历整个链表，得到可执行文件所有的加载so文件。

#### link\_map 的结构

```C
/* Structure describing a loaded shared object.  The `l_next' and `l_prev'
   members form a chain of all the shared objects loaded at startup.
   These data structures exist in space used by the run-time dynamic linker;
   modifying them may have disastrous results.  */
struct link_map
{
    /* These first few members are part of the protocol with the debugger.
       This is the same format used in SVR4.  */

    ElfW(Addr) l_addr;          /* Difference between the address in the ELF
                                   file and the addresses in memory.  */
    char *l_name;               /* Absolute file name object was found in.  */
    ElfW(Dyn) *l_ld;            /* Dynamic section of the shared object.  */
    struct link_map *l_next, *l_prev; /* Chain of loaded objects.  */
};
```

可以通过遍历link\_map寻找库，对比l\_name，找到目标之后，可以通过l\_addr获得库的基址。

#### DT\_DEBUG 寻找 link\_map

在linux下的动态加载器ld.so为一个可执行文件执行动态链接时，它会在动态链接完成之后，将可执行文件这个位置填充上整个进程的全局的一个debug信息的位置。

还可以通过 DT\_DEBUG 来寻找 link\_map

* 在 .dynamic 找到 DT\_DEBUG (0x00000015) 的地址值

* DT\_DEBUG 结构
        
    可以看到 link_map 是 DT\_DEBUG 的第二个元素 


```C
/* Rendezvous structure used by the run-time dynamic linker to communicate
   details of shared object loading to the debugger.  If the executable's
   dynamic section has a DT_DEBUG element, the run-time linker sets that
   element's value to the address where this structure can be found.  */

struct r_debug
  { 
    int r_version;              /* Version number for this protocol.  */

    struct link_map *r_map;     /* Head of the chain of loaded objects.  */

    /* This is the address of a function internal to the run-time linker,
       that will always be called when the linker begins to map in a
       library or unmap it, and again when the mapping change is complete.
       The debugger can set a breakpoint at this address if it wants to
       notice shared object mapping changes.  */
    ElfW(Addr) r_brk;
    enum
      { 
        /* This state value describes the mapping change taking place when
           the `r_brk' address is called.  */
        RT_CONSISTENT,          /* Mapping change is complete.  */
        RT_ADD,                 /* Beginning to add a new object.  */
        RT_DELETE               /* Beginning to remove an object mapping.  */
      } r_state;

    ElfW(Addr) r_ldbase;        /* Base address the linker is loaded at.  */
  };
```
  
* 可以通过遍历link\_map寻找库，对比l\_name，找到目标之后，可以通过l\_addr获得库的基址。

```python
gdb-peda$ x/10xw 0x55575918
0x55575918 <_r_debug>:  0x00000001  0x55575930  0x55564090  0x00000000
0x55575928 <_r_debug+16>:   0x55555000  0x00000000  0x00000000  0x55575c1c
0x55575938: 0x0804af04  0x55575c20
gdb-peda$ x/10xw 0x55575930
0x55575930: 0x00000000  0x55575c1c  0x0804af04  0x55575c20
0x55575940: 0x00000000  0x55575930  0x00000000  0x55575c10
0x55575950: 0x00000000  0x0804af14
gdb-peda$ x/s 0x55575c1c
0x55575c1c: ""
gdb-peda$ x/10xw 0x55575c20
0x55575c20: 0x55578000  0x55575e90  0x55578278  0x5557a8e8
0x55575c30: 0x55575930  0x55575c20  0x00000000  0x55575e80
0x55575c40: 0x00000000  0x00000000
gdb-peda$ x/s 0x55575e90
0x55575e90: "linux-gate.so.1"
```

## \_dl\_runtime\_resolver

在下一篇中介绍


## 参考

[Dynamic Section](http://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html#scrolltoc)

[elf文件类型六 Dynamic Section（动态section）](http://linux.chinaunix.net/techdoc/system/2008/01/28/977695.shtml)

[通过DT_DEBUG来获得各个库的基址](http://rk700.github.io/article/2015/04/09/dt_debug-read/)

[gdb对attach和core文件的处理方式](http://tsecer.blog.163.com/blog/static/15018172013924104112444/)