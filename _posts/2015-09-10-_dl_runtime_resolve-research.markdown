---
layout: post
title: "_dl_runtime_resolve Research"
tags: [exploit, _dl_runtime_resolve, link_map]
comments: true
date: 2015-09-10T19:34:05+08:00
---

## 调试相关

安装libc debug symbol.

```sh
sudo apt-get install libc-dbg
```

安装libc6 source

```sh
sudo apt-get install build-essential
sudo apt-get source libc6
```

gdb 中添加符号文件，源码

```python
gdb-peda$ info sharedlibrary
From                To                  Syms Read   Shared Object Library
0x00007fe2880e1ae0  0x00007fe2880fa170  Yes         /lib64/ld-linux-x86-64.so.2
                                        No          linux-vdso.so.1
0x00007fe287d574a0  0x00007fe287e816d3  Yes         /lib/x86_64-linux-gnu/libc.so.6

gdb-peda$ add-symbol-file /usr/lib/debug/lib/x86_64-linux-gnu/ld-2.19.so 0x00007fe2880e1ae0
add symbol table from file "/usr/lib/debug/lib/x86_64-linux-gnu/ld-2.19.so" at
    .text_addr = 0x7fe2880e1ae0
Reading symbols from /usr/lib/debug/lib/x86_64-linux-gnu/ld-2.19.so...done.

gdb-peda$ directory ~/glibc-debug/glibc-2.19/elf/
Source directories searched: /home/neoni/glibc-debug/glibc-2.19/elf:$cdir:$cwd
gdb-peda$ set disassemble-next-line on
```

---

## PLT 到 \_dl\_runtime\_resolve

在第一次调用时，xxx@got.plt会跳回xxx@plt。接下来，会将参数push到栈上并跳至.got.plt+0x8。压入reloc_offset 和 link\_map。

```python
(gdb) disas some_func
Dump of assembler code for function some_func:
0x804xxx4 <some_func>:     jmp    *some_func_dyn_reloc_entry
0x804xxxa <some_func+6>:   push   $reloc_offset
0x804xxxf <some_func+11>:  jmp    beginning_of_.plt_section
````

然后调用\_dl\_runtime\_resolve

```c
_dl_runtime_resolve(link_map, rel_offset);
```

\_dl\_runtime\_resolve则会完成具体的符号解析，填充结果，和调用的工作。

其中\_dl\_runtime\_resolve调用了\_dl\_fixup， \_dl\_fixup返回函数地址，然后跳转到该地址上


```asm
=> 0x7fdd1d9032f0 <_dl_runtime_resolve>:    sub    rsp,0x38
   0x7fdd1d9032f4 <_dl_runtime_resolve+4>:  mov    QWORD PTR [rsp],rax
   0x7fdd1d9032f8 <_dl_runtime_resolve+8>:  mov    QWORD PTR [rsp+0x8],rcx
   0x7fdd1d9032fd <_dl_runtime_resolve+13>: mov    QWORD PTR [rsp+0x10],rdx
   0x7fdd1d903302 <_dl_runtime_resolve+18>: mov    QWORD PTR [rsp+0x18],rsi
   0x7fdd1d903307 <_dl_runtime_resolve+23>: mov    QWORD PTR [rsp+0x20],rdi
   0x7fdd1d90330c <_dl_runtime_resolve+28>: mov    QWORD PTR [rsp+0x28],r8
   0x7fdd1d903311 <_dl_runtime_resolve+33>: mov    QWORD PTR [rsp+0x30],r9
   0x7fdd1d903316 <_dl_runtime_resolve+38>: mov    rsi,QWORD PTR [rsp+0x40]
   0x7fdd1d90331b <_dl_runtime_resolve+43>: mov    rdi,QWORD PTR [rsp+0x38]
   0x7fdd1d903320 <_dl_runtime_resolve+48>: call   0x7fdd1d8fcd10 <_dl_fixup>
   0x7fdd1d903325 <_dl_runtime_resolve+53>: mov    r11,rax
   0x7fdd1d903328 <_dl_runtime_resolve+56>: mov    r9,QWORD PTR [rsp+0x30]
   0x7fdd1d90332d <_dl_runtime_resolve+61>: mov    r8,QWORD PTR [rsp+0x28]
   0x7fdd1d903332 <_dl_runtime_resolve+66>: mov    rdi,QWORD PTR [rsp+0x20]
   0x7fdd1d903337 <_dl_runtime_resolve+71>: mov    rsi,QWORD PTR [rsp+0x18]
   0x7fdd1d90333c <_dl_runtime_resolve+76>: mov    rdx,QWORD PTR [rsp+0x10]
   0x7fdd1d903341 <_dl_runtime_resolve+81>: mov    rcx,QWORD PTR [rsp+0x8]
   0x7fdd1d903346 <_dl_runtime_resolve+86>: mov    rax,QWORD PTR [rsp]
   0x7fdd1d90334a <_dl_runtime_resolve+90>: add    rsp,0x48
   0x7fdd1d90334e <_dl_runtime_resolve+94>: jmp    r11
```

---

## \_dl\_fixup

```c
_dl_fixup(link_map, rel_offset);
```

#### .dynamic中的一些重要元素

* DT_STRTAB
  
    该元素保存着字符串表地址，包括了符号名，库名，和一些其他的在该表中的字符串。
  
* DT_SYMTAB
  
    该元素保存着符号表的地址。
  
* DT\_JMPREL
  
    假如存在，它的入口d\_ptr 成员保存着重定位入口（该入口单独关联着PLT）的地址。假如lazy方式打开，那么分离它们的重定位入口让动态连接器在进程初始化时忽略它们。假如该入口存在，相关联的类型入口DT\_PLTRELSZ和DT\_PLTREL一定要存在。
  
* VERSYM
    
    该元素保存着版本表的地址。     
    
下面以 `puts@plt` 为例，探寻其 \_dl\_runtime\_resolve 过程

首先得到其 .dynamic 的 信息

```python
$ readelf -d xxx

Dynamic section at offset 0xe50 contains 20 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x400400
 0x000000000000000d (FINI)               0x400668
 0x000000006ffffef5 (GNU_HASH)           0x400298
 0x0000000000000005 (STRTAB)             0x400330
 0x0000000000000006 (SYMTAB)             0x4002b8
 0x000000000000000a (STRSZ)              66 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x600fe8
 0x0000000000000002 (PLTRELSZ)           72 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x4003b8
 0x0000000000000007 (RELA)               0x4003a0
 0x0000000000000008 (RELASZ)             24 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x400380
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x400372
 0x0000000000000000 (NULL)               0x0
```

#### \_dl\_fixup 的工作

下面关于 \_dl\_fixup 的介绍省略了一些细节，介绍了一些与 .dynamic 相关的主要工作

1) 计算 reloc entry

```c  
const PLTREL *const reloc = (const void *) (D_PTR (l, l_info[DT_JMPREL]) + reloc_offset);
```

根据 DT\_JMPREL 找到偏移值， JMPREL segment 对应于.rel.plt section，是用来保存运行时重定位表的。它与.rel.dyn类似，只不过.rel.plt是用于函数重定位，.rel.dyn是用于变量重定位。具体地，其内容如下:

```python
readelf -r xxx

Relocation section '.rela.dyn' at offset 0x3a0 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000600fe0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x3b8 contains 3 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000601000  000100000007 R_X86_64_JUMP_SLO 0000000000000000 puts + 0
000000601008  000200000007 R_X86_64_JUMP_SLO 0000000000000000 read + 0
000000601010  000300000007 R_X86_64_JUMP_SLO 0000000000000000 __libc_start_main + 0
```

这些条目的类型是

```c 
typedef struct
{
  Elf32_Addr    r_offset;       /* Address */
  Elf32_Word    r_info;         /* Relocation type and symbol index */
} Elf32_Rel;

typedef struct
{
  Elf64_Addr    r_offset;       /* Address */
  Elf64_Xword   r_info;         /* Relocation type and symbol index */
} Elf64_Rel;
```

在 gdb 中查看jmprel中的内容，可以看到与 readelf -r 的内容一样

```python
gdb-peda$ x/10xg 0x4003b8
0x4003b8:   0x0000000000601000  0x0000000100000007
0x4003c8:   0x0000000000000000  0x0000000000601008
0x4003d8:   0x0000000200000007  0x0000000000000000
0x4003e8:   0x0000000000601010  0x0000000300000007
```

加上对应的 reloc\_offset，就可以找到函数的重定位入口 reloc

其中 puts@plt 的 reloc\_offset 为0，所以 reloc =  0x4003b8 + 0 = 0x4003b8, 其中 `r_offset` 为 0x0000000000601000， 即 `puts@got.plt` 的地址，`r_info` 为0x0000000100000007， `r_info` 保存的是其类型和符号序号。根据宏的定义，对于此条目，其类型为`ELF64_R_TYPE(r_info)=7`，对应于 `R_X86_64_JUMP_SLOT`；其symbol index则为 `RLF64_R_SYM(r_info)=1`。


2) 计算函数的 symtab entry

每一条symbol信息的大小在SYMENT中体现，可以在 readelf -d 中看到为 24 bytes。

```c        
const ElfW(Sym) *sym = &symtab[ELFW(R_SYM) (reloc->r_info)];
```

所以 sym = SYMTAB + 1*24 

```python
gdb-peda$ x/10xw 0x4002b8+24
0x4002d0:   0x0000001a  0x00000012  0x00000000  0x00000000
0x4002e0:   0x00000000  0x00000000  0x0000001f  0x00000012
0x4002f0:   0x00000000  0x00000000
```

其中 sym 的 结构为：

```c
typedef struct
{
  Elf32_Word    st_name;        /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;       /* Symbol value */
  Elf32_Word    st_size;        /* Symbol size */
  unsigned char st_info;        /* Symbol type and binding */
  unsigned char st_other;       /* Symbol visibility */
  Elf32_Section st_shndx;       /* Section index */
} Elf32_Sym;
 
typedef struct
{
  Elf64_Word    st_name;        /* Symbol name (string tbl index) */
  unsigned char st_info;        /* Symbol type and binding */
  unsigned char st_other;       /* Symbol visibility */
  Elf64_Section st_shndx;       /* Section index */
  Elf64_Addr    st_value;       /* Symbol value */
  Elf64_Xword   st_size;        /* Symbol size */
} Elf64_Sym;
```

3) sanity check 

```c
assert (ELFW(R_TYPE)(reloc->r_info) == ELF_MACHINE_JMP_SLOT);

#define ELF_MACHINE_JMP_SLOT    R_X86_64_JUMP_SLOT
```

4) 如果 `sym->st_other & 3 != 0`, 已经被 resolve 过了, 跳到 7，否则继续

5) 如果版本信息被激活，那么需要找到对应的版本信息 

```c
uint16_t ndx = VERSYM[ ELF64_R_SYM (reloc->r_info) ]; 

const struct r_found_version *version =&l->l_versions[ndx];
```

6) 找函数库:

```c
result = _dl_lookup_symbol_x (strtab + sym->st_name, l, &sym, l->l_scope,
                    version, ELF_RTYPE_CLASS_PLT, flags, NULL);
```

`strtab + sym->st_name` 为函数库名称

```python
gdb-peda$ x/10s 0x400330+0x0000001a
0x40034a:   "puts"
0x40034f:   "read"
0x400354:   "__libc_start_ma"...
0x400363:   "in"
```
`_dl_lookup_symbol_x` 将返回 以libc为当前节点的link\_map结构，并查找填充了sym 的信息


7) 添加函数偏移，找到了函数地址

```c
value = DL_FIXUP_MAKE_VALUE (result
                   sym ? (LOOKUP_VALUE_ADDRESS (result)
                      + sym->st_value) : 0);
```

8) 最后找到地址后，填充至.got.plt对应位置，调整栈，调用这一解析得到的函数。

---

## 总结


* `_dl_runtime_resolve` 调用 `_dl_fixup` 来完成符号解析。

* 首先通过 `.dynamic` 中的 `DT_JMPREL` 与参数 `reloc_offset` 的值找到重定位结构 `Elf64_Rel`，然后根据`Elf64_Rel->r_offset` 在 `DT_SYMTAB` 找到符号结构 Elf64_Sym。

* 并根据`Elf64_Rel->r_info` 与 `Elf64_Sym->st_other` 校验。

* 如果 `DT_VERSYM`存在，根据其查找版本信息。在 `DT_STRTAB` 根据 `Elf64_Sym->st_name` 找到函数库名称。

* 调用 `_dl_lookup_symbol_x`根据函数库名，符号结构，版本信息等搜索函数库以及符号信息，得到函数库基地址与完整的符号信息结构（包含函数在函数库中的偏移）。

* 基地址与函数偏移相加顺利得到函数地址，填充至.got.plt对应位置，调整栈，调用这一解析得到的函数。


---

## 参考

程序员的自我修养

[http://phrack.org/issues/58/4.html#article](http://phrack.org/issues/58/4.html#article)

[Dynamic Section](http://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html#scrolltoc)

[ROP之return to dl-resolve](http://rk700.github.io/article/2015/08/09/return-to-dl-resolve/)