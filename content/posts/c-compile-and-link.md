---
title: "从 C 源码到内存布局：编译、链接与启动全过程解析"
date: 2026-02-20T17:46:18+08:00
draft: true
tags: ["C语言", "编译器", "链接", "操作系统"]
categories: ["底层原理"]
---

[TOC]

## 第五部分：虚拟地址空间与内存布局

### 第11章：栈与堆的底层原理

#### 回顾一下C语言

很好，这一步确实有必要补齐。

如果不把 **栈（stack）** 和 **堆（heap）** 的本质讲清楚，启动代码为什么要先设栈、为什么链接脚本要预留 heap/stack 空间，就会显得“理所当然但没讲透”。

我们从体系结构 + ABI + 内存布局三个层面讲清楚。

------

#### 一、先纠正一个常见误解

> 栈和堆不是 C 语言发明的，而是运行时内存管理策略。

C 语言只是“使用”它们。

真正决定它们存在的是：

- CPU 调用约定（Calling Convention）
- ABI
- 运行时内存布局
- 链接脚本

------

#### 二、栈（Stack）

#### 1️⃣ 栈是什么？

栈是：

> 一块由 CPU 自动维护的连续内存区域，用于函数调用上下文。

特点：

- 后进先出（LIFO）
- 由 SP（Stack Pointer）寄存器指向
- 通常向低地址增长（ARM/x86 都是）

------

#### 2️⃣ 栈里存什么？

以 ARM 为例：

#### 函数调用时会发生：

```
int foo(int a) {
    int b = 5;
    return a + b;
}
```

汇编大致会做：

```
push {lr}       ; 保存返回地址
sub sp, sp, #4  ; 给 b 分配空间
...
add sp, sp, #4
pop {pc}
```

栈中存：

| 内容             | 为什么     |
| ---------------- | ---------- |
| 返回地址         | 函数返回用 |
| 保存寄存器       | 遵守 ABI   |
| 局部变量         | 自动变量   |
| 函数参数（部分） | 视 ABI     |

------

#### 3️⃣ 为什么启动代码必须先设栈？

因为：

- 函数调用一定会访问栈
- C 语言任何函数都依赖栈

如果 SP 不合法：

- push 会写到 0 地址
- 会触发 HardFault / SegFault

所以 Reset_Handler 的第一条指令通常是：

```
ldr sp, =_stack_top
```

------

#### 4️⃣ 栈是“自动内存”

它的生命周期：

- 函数进入 → 分配
- 函数退出 → 自动释放

特点：

- 分配快（只是移动 SP）
- 无碎片
- 空间有限
- 不能跨函数长期存在

------

#### 三、堆（Heap）

#### 1️⃣ 堆是什么？

堆是：

> 一块由程序员动态申请、运行时分配器管理的内存区域。

典型 API：

```
malloc()
free()
```

------

#### 2️⃣ 堆由谁管理？

- Linux：glibc malloc
- 裸机：newlib 或自己实现
- RTOS：自己的分配器

底层依赖：

```
sbrk()
```

或你自己实现的 `_sbrk`

------

#### 3️⃣ 堆在内存中的位置？

典型裸机布局：

```
RAM:
---------------------------------
| .data                        |
| .bss                         |
| heap  ↑ 向上增长             |
|                               |
|                               |
| stack ↓ 向下增长             |
---------------------------------
```

heap 从低地址向上增长
 stack 从高地址向下增长

中间如果相撞：

> heap overflow / stack overflow

------

### 四、栈 vs 堆 对比

| 特性             | 栈         | 堆                  |
| ---------------- | ---------- | ------------------- |
| 管理方式         | CPU 自动   | malloc/free         |
| 生命周期         | 函数作用域 | 手动控制            |
| 分配速度         | 极快       | 较慢                |
| 是否碎片化       | 不会       | 会                  |
| 大小             | 较小       | 取决于RAM           |
| 是否需要启动代码 | 必须       | 取决于是否用 malloc |

------

### 五、为什么启动代码一般不管堆？

因为：

- 堆不是 CPU 必需的
- 只有当你调用 malloc 才需要
- 裸机下很多程序根本不用 heap

但是：

链接脚本通常会：

```
_heap_start = .;
. += 0x1000;
_heap_end = .;
```

只是预留空间。

真正的堆初始化通常发生在：

```
__libc_init_array
或
_sbrk 初始化
```

------

#### 六、为什么栈是必须的，而堆不是？

因为：

函数调用是语言核心语义

而 malloc 不是。

没有堆：

程序还能运行

没有栈：

程序根本无法执行函数

## 议插入的内容：小结——C 语言的逻辑内存模型

通过前面的分析，我们可以把 C 语言程序运行时的内存布局总结为经典的**“五大分区”**。这个模型是跨平台的，无论是在 Linux 还是在裸机嵌入式系统中，其逻辑结构高度一致：

### 1. 内存模型全景图

| **分区名称**           | **对应 ELF Section**              | **存放内容**                           | **生命周期**   | **读写属性** |
| ---------------------- | --------------------------------- | -------------------------------------- | -------------- | ------------ |
| **代码段 (Code/Text)** | `.text`                           | 机器指令、只读常量                     | 永久（随进程） | 只读/可执行  |
| **数据段 (Data)**      | `.data`                           | 已初始化的全局/静态变量                | 永久（随进程） | 可读写       |
| **BSS 段**             | `.bss`                            | 未初始化（或初始化为0）的全局/静态变量 | 永久（随进程） | 可读写       |
| **堆 (Heap)**          | 无直接对应 (运行时由管理函数分配) | 动态申请的内存 (`malloc`)              | 手动控制       | 可读写       |
| **栈 (Stack)**         | 无直接对应 (由系统/启动代码预留)  | 局部变量、函数参数、返回地址           | 函数作用域     | 可读写       |

### 2. “动静结合”的视角

- **静态区（Text, Data, BSS）：** 它们的大小在**链接阶段**就已经完全确定了。链接脚本里的每一个地址分配，都是在为这三个区画地盘。
- **动态区（Stack, Heap）：** 它们的大小在运行时是**变动**的。栈向下长，堆向上长，它们共同瓜分剩下的 RAM 空间。

### 3. 为什么 C 程序员必须理解这个模型？

理解了这个模型，很多“神鬼莫测”的 Bug 就会变成常识：

- **为什么全局变量没初始化默认是 0？** 因为启动代码在跳转到 `main` 之前，把整个 `.bss` 段都清零了。
- **为什么局部变量不初始化是随机值？** 因为局部变量在栈上，栈只是移动了 `SP` 指针，并没有清理上一个函数留下的“垃圾”数据。
- **为什么不能返回局部变量的地址？** 因为函数退出后，栈帧被销毁，那个地址虽然还在，但里面的内容随时会被下一个函数调用覆盖。
- **什么是“栈溢出” (Stack Overflow)？** 就是函数调用太深（如死递归），导致栈指针一路向下，撞到了堆区或者 BSS 段的边界。



在下一张来到最硬核的实战篇。我们将撕开 C 语言的温情面纱，直击**启动代码（Startup Code）**与**链接脚本（Linker Script）**的战场。看程序如何在荒芜的硬件上，一砖一瓦地盖起那座名为“运行时环境”的大厦。



## 第四部分：程序真正的入口

### 链接脚本与 ENTRY 指令

gcc 默认ld

```bash
lzk@lzk-Virtual-Machine:~$ gcc -Wl,--verbose hello.c -o hello
GNU ld (GNU Binutils for Ubuntu) 2.38
  支持的仿真：
   elf_x86_64
   elf32_x86_64
   elf_i386
   elf_iamcu
   elf_l1om
   elf_k1om
   i386pep
   i386pe
使用内部链接脚本：
==================================================
/* Script for -pie -z combreloc -z separate-code -z relro -z now */
/* Copyright (C) 2014-2022 Free Software Foundation, Inc.
   Copying and distribution of this script, with or without modification,
   are permitted in any medium without royalty provided the copyright
   notice and this notice are preserved.  */
OUTPUT_FORMAT("elf64-x86-64", "elf64-x86-64",
	      "elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)
SEARCH_DIR("=/usr/local/lib/x86_64-linux-gnu"); SEARCH_DIR("=/lib/x86_64-linux-gnu"); SEARCH_DIR("=/usr/lib/x86_64-linux-gnu"); SEARCH_DIR("=/usr/lib/x86_64-linux-gnu64"); SEARCH_DIR("=/usr/local/lib64"); SEARCH_DIR("=/lib64"); SEARCH_DIR("=/usr/lib64"); SEARCH_DIR("=/usr/local/lib"); SEARCH_DIR("=/lib"); SEARCH_DIR("=/usr/lib"); SEARCH_DIR("=/usr/x86_64-linux-gnu/lib64"); SEARCH_DIR("=/usr/x86_64-linux-gnu/lib");
SECTIONS
{
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0)); . = SEGMENT_START("text-segment", 0) + SIZEOF_HEADERS;
  .interp         : { *(.interp) }
  .note.gnu.build-id  : { *(.note.gnu.build-id) }
  .hash           : { *(.hash) }
  .gnu.hash       : { *(.gnu.hash) }
  .dynsym         : { *(.dynsym) }
  .dynstr         : { *(.dynstr) }
  .gnu.version    : { *(.gnu.version) }
  .gnu.version_d  : { *(.gnu.version_d) }
  .gnu.version_r  : { *(.gnu.version_r) }
  .rela.dyn       :
    {
      *(.rela.init)
      *(.rela.text .rela.text.* .rela.gnu.linkonce.t.*)
      *(.rela.fini)
      *(.rela.rodata .rela.rodata.* .rela.gnu.linkonce.r.*)
      *(.rela.data .rela.data.* .rela.gnu.linkonce.d.*)
      *(.rela.tdata .rela.tdata.* .rela.gnu.linkonce.td.*)
      *(.rela.tbss .rela.tbss.* .rela.gnu.linkonce.tb.*)
      *(.rela.ctors)
      *(.rela.dtors)
      *(.rela.got)
      *(.rela.bss .rela.bss.* .rela.gnu.linkonce.b.*)
      *(.rela.ldata .rela.ldata.* .rela.gnu.linkonce.l.*)
      *(.rela.lbss .rela.lbss.* .rela.gnu.linkonce.lb.*)
      *(.rela.lrodata .rela.lrodata.* .rela.gnu.linkonce.lr.*)
      *(.rela.ifunc)
    }
  .rela.plt       :
    {
      *(.rela.plt)
      *(.rela.iplt)
    }
  .relr.dyn : { *(.relr.dyn) }
  . = ALIGN(CONSTANT (MAXPAGESIZE));
  .init           :
  {
    KEEP (*(SORT_NONE(.init)))
  }
  .plt            : { *(.plt) *(.iplt) }
.plt.got        : { *(.plt.got) }
.plt.sec        : { *(.plt.sec) }
  .text           :
  {
    *(.text.unlikely .text.*_unlikely .text.unlikely.*)
    *(.text.exit .text.exit.*)
    *(.text.startup .text.startup.*)
    *(.text.hot .text.hot.*)
    *(SORT(.text.sorted.*))
    *(.text .stub .text.* .gnu.linkonce.t.*)
    /* .gnu.warning sections are handled specially by elf.em.  */
    *(.gnu.warning)
  }
  .fini           :
  {
    KEEP (*(SORT_NONE(.fini)))
  }
  PROVIDE (__etext = .);
  PROVIDE (_etext = .);
  PROVIDE (etext = .);
  . = ALIGN(CONSTANT (MAXPAGESIZE));
  /* Adjust the address for the rodata segment.  We want to adjust up to
     the same address within the page on the next page up.  */
  . = SEGMENT_START("rodata-segment", ALIGN(CONSTANT (MAXPAGESIZE)) + (. & (CONSTANT (MAXPAGESIZE) - 1)));
  .rodata         : { *(.rodata .rodata.* .gnu.linkonce.r.*) }
  .rodata1        : { *(.rodata1) }
  .eh_frame_hdr   : { *(.eh_frame_hdr) *(.eh_frame_entry .eh_frame_entry.*) }
  .eh_frame       : ONLY_IF_RO { KEEP (*(.eh_frame)) *(.eh_frame.*) }
  .gcc_except_table   : ONLY_IF_RO { *(.gcc_except_table .gcc_except_table.*) }
  .gnu_extab   : ONLY_IF_RO { *(.gnu_extab*) }
  /* These sections are generated by the Sun/Oracle C++ compiler.  */
  .exception_ranges   : ONLY_IF_RO { *(.exception_ranges*) }
  /* Adjust the address for the data segment.  We want to adjust up to
     the same address within the page on the next page up.  */
  . = DATA_SEGMENT_ALIGN (CONSTANT (MAXPAGESIZE), CONSTANT (COMMONPAGESIZE));
  /* Exception handling  */
  .eh_frame       : ONLY_IF_RW { KEEP (*(.eh_frame)) *(.eh_frame.*) }
  .gnu_extab      : ONLY_IF_RW { *(.gnu_extab) }
  .gcc_except_table   : ONLY_IF_RW { *(.gcc_except_table .gcc_except_table.*) }
  .exception_ranges   : ONLY_IF_RW { *(.exception_ranges*) }
  /* Thread Local Storage sections  */
  .tdata	  :
   {
     PROVIDE_HIDDEN (__tdata_start = .);
     *(.tdata .tdata.* .gnu.linkonce.td.*)
   }
  .tbss		  : { *(.tbss .tbss.* .gnu.linkonce.tb.*) *(.tcommon) }
  .preinit_array    :
  {
    PROVIDE_HIDDEN (__preinit_array_start = .);
    KEEP (*(.preinit_array))
    PROVIDE_HIDDEN (__preinit_array_end = .);
  }
  .init_array    :
  {
    PROVIDE_HIDDEN (__init_array_start = .);
    KEEP (*(SORT_BY_INIT_PRIORITY(.init_array.*) SORT_BY_INIT_PRIORITY(.ctors.*)))
    KEEP (*(.init_array EXCLUDE_FILE (*crtbegin.o *crtbegin?.o *crtend.o *crtend?.o ) .ctors))
    PROVIDE_HIDDEN (__init_array_end = .);
  }
  .fini_array    :
  {
    PROVIDE_HIDDEN (__fini_array_start = .);
    KEEP (*(SORT_BY_INIT_PRIORITY(.fini_array.*) SORT_BY_INIT_PRIORITY(.dtors.*)))
    KEEP (*(.fini_array EXCLUDE_FILE (*crtbegin.o *crtbegin?.o *crtend.o *crtend?.o ) .dtors))
    PROVIDE_HIDDEN (__fini_array_end = .);
  }
  .ctors          :
  {
    /* gcc uses crtbegin.o to find the start of
       the constructors, so we make sure it is
       first.  Because this is a wildcard, it
       doesn't matter if the user does not
       actually link against crtbegin.o; the
       linker won't look for a file to match a
       wildcard.  The wildcard also means that it
       doesn't matter which directory crtbegin.o
       is in.  */
    KEEP (*crtbegin.o(.ctors))
    KEEP (*crtbegin?.o(.ctors))
    /* We don't want to include the .ctor section from
       the crtend.o file until after the sorted ctors.
       The .ctor section from the crtend file contains the
       end of ctors marker and it must be last */
    KEEP (*(EXCLUDE_FILE (*crtend.o *crtend?.o ) .ctors))
    KEEP (*(SORT(.ctors.*)))
    KEEP (*(.ctors))
  }
  .dtors          :
  {
    KEEP (*crtbegin.o(.dtors))
    KEEP (*crtbegin?.o(.dtors))
    KEEP (*(EXCLUDE_FILE (*crtend.o *crtend?.o ) .dtors))
    KEEP (*(SORT(.dtors.*)))
    KEEP (*(.dtors))
  }
  .jcr            : { KEEP (*(.jcr)) }
  .data.rel.ro : { *(.data.rel.ro.local* .gnu.linkonce.d.rel.ro.local.*) *(.data.rel.ro .data.rel.ro.* .gnu.linkonce.d.rel.ro.*) }
  .dynamic        : { *(.dynamic) }
  .got            : { *(.got.plt) *(.igot.plt) *(.got) *(.igot) }
  . = DATA_SEGMENT_RELRO_END (0, .);
  .data           :
  {
    *(.data .data.* .gnu.linkonce.d.*)
    SORT(CONSTRUCTORS)
  }
  .data1          : { *(.data1) }
  _edata = .; PROVIDE (edata = .);
  . = .;
  __bss_start = .;
  .bss            :
  {
   *(.dynbss)
   *(.bss .bss.* .gnu.linkonce.b.*)
   *(COMMON)
   /* Align here to ensure that the .bss section occupies space up to
      _end.  Align after .bss to ensure correct alignment even if the
      .bss section disappears because there are no input sections.
      FIXME: Why do we need it? When there is no .bss section, we do not
      pad the .data section.  */
   . = ALIGN(. != 0 ? 64 / 8 : 1);
  }
  .lbss   :
  {
    *(.dynlbss)
    *(.lbss .lbss.* .gnu.linkonce.lb.*)
    *(LARGE_COMMON)
  }
  . = ALIGN(64 / 8);
  . = SEGMENT_START("ldata-segment", .);
  .lrodata   ALIGN(CONSTANT (MAXPAGESIZE)) + (. & (CONSTANT (MAXPAGESIZE) - 1)) :
  {
    *(.lrodata .lrodata.* .gnu.linkonce.lr.*)
  }
  .ldata   ALIGN(CONSTANT (MAXPAGESIZE)) + (. & (CONSTANT (MAXPAGESIZE) - 1)) :
  {
    *(.ldata .ldata.* .gnu.linkonce.l.*)
    . = ALIGN(. != 0 ? 64 / 8 : 1);
  }
  . = ALIGN(64 / 8);
  _end = .; PROVIDE (end = .);
  . = DATA_SEGMENT_END (.);
  /* Stabs debugging sections.  */
  .stab          0 : { *(.stab) }
  .stabstr       0 : { *(.stabstr) }
  .stab.excl     0 : { *(.stab.excl) }
  .stab.exclstr  0 : { *(.stab.exclstr) }
  .stab.index    0 : { *(.stab.index) }
  .stab.indexstr 0 : { *(.stab.indexstr) }
  .comment       0 : { *(.comment) }
  .gnu.build.attributes : { *(.gnu.build.attributes .gnu.build.attributes.*) }
  /* DWARF debug sections.
     Symbols in the DWARF debugging sections are relative to the beginning
     of the section so we begin them at 0.  */
  /* DWARF 1.  */
  .debug          0 : { *(.debug) }
  .line           0 : { *(.line) }
  /* GNU DWARF 1 extensions.  */
  .debug_srcinfo  0 : { *(.debug_srcinfo) }
  .debug_sfnames  0 : { *(.debug_sfnames) }
  /* DWARF 1.1 and DWARF 2.  */
  .debug_aranges  0 : { *(.debug_aranges) }
  .debug_pubnames 0 : { *(.debug_pubnames) }
  /* DWARF 2.  */
  .debug_info     0 : { *(.debug_info .gnu.linkonce.wi.*) }
  .debug_abbrev   0 : { *(.debug_abbrev) }
  .debug_line     0 : { *(.debug_line .debug_line.* .debug_line_end) }
  .debug_frame    0 : { *(.debug_frame) }
  .debug_str      0 : { *(.debug_str) }
  .debug_loc      0 : { *(.debug_loc) }
  .debug_macinfo  0 : { *(.debug_macinfo) }
  /* SGI/MIPS DWARF 2 extensions.  */
  .debug_weaknames 0 : { *(.debug_weaknames) }
  .debug_funcnames 0 : { *(.debug_funcnames) }
  .debug_typenames 0 : { *(.debug_typenames) }
  .debug_varnames  0 : { *(.debug_varnames) }
  /* DWARF 3.  */
  .debug_pubtypes 0 : { *(.debug_pubtypes) }
  .debug_ranges   0 : { *(.debug_ranges) }
  /* DWARF 5.  */
  .debug_addr     0 : { *(.debug_addr) }
  .debug_line_str 0 : { *(.debug_line_str) }
  .debug_loclists 0 : { *(.debug_loclists) }
  .debug_macro    0 : { *(.debug_macro) }
  .debug_names    0 : { *(.debug_names) }
  .debug_rnglists 0 : { *(.debug_rnglists) }
  .debug_str_offsets 0 : { *(.debug_str_offsets) }
  .debug_sup      0 : { *(.debug_sup) }
  .gnu.attributes 0 : { KEEP (*(.gnu.attributes)) }
  /DISCARD/ : { *(.note.GNU-stack) *(.gnu_debuglink) *(.gnu.lto_*) }
}


==================================================

```

#### 关键结构（以 x86_64 Linux 为例）

默认脚本大概结构是：

```
OUTPUT_FORMAT("elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
ENTRY(_start)

SECTIONS
{
  . = 0x400000;

  .text :
  {
    *(.text*)
  }

  .rodata :
  {
    *(.rodata*)
  }

  .data :
  {
    *(.data*)
  }

  .bss :
  {
    *(.bss*)
    *(COMMON)
  }
}
```

当然真实脚本复杂得多，但核心结构就是这样。

------

#### 3️⃣ 先理解三个顶层指令

------

##### 🔹 OUTPUT_FORMAT

```
OUTPUT_FORMAT("elf64-x86-64")
```

告诉链接器：

> 生成什么格式的 ELF

这和目标架构绑定。

------

##### 🔹 OUTPUT_ARCH

```
OUTPUT_ARCH(i386:x86-64)
```

指定 CPU 架构。

------

##### 🔹 ENTRY(_start)

这里和你写的一样。

它只是：

> 设置 ELF Header 中的 e_entry 字段。

⚠️ 重点：

这不会影响 CPU reset 地址。

它只是告诉 loader：

> 程序入口在哪里。

------

#### 现在我们可以回到你的 ld 脚本

```
.text :
{
  *(entry)
  *(.text*)
} > MROM
```

这一句做了两件事：

1. 决定顺序（entry 先放）
2. 决定地址（从 0x20000000 开始）

这就是“分配地址”阶段。

```ld
MEMORY
{
  MROM (rx)  : ORIGIN = 0x20000000, LENGTH = 128K
  SRAM (rwx) : ORIGIN = 0x0f000000, LENGTH = 8K
}

ENTRY(_start)

SECTIONS
{
  . = ORIGIN(MROM);

  .text :
  {
    *(entry)
    *(.text*)
  } > MROM

  .rodata :
  {
    *(.rodata*)
  } > MROM

  . = ORIGIN(SRAM);

  .data : AT(LOADADDR(.rodata) + SIZEOF(.rodata))
  {
    _data_start = .;
    *(.data*)
    _data_end = .;
  } > SRAM

  _data_load = LOADADDR(.data);

  .bss :
  {
    _bss_start = .;
    *(.bss*)
    *(COMMON)
    _bss_end = .;
  } > SRAM

  . = ALIGN(8);
  _heap_start = .;

  _stack_top = ORIGIN(SRAM) + LENGTH(SRAM);
}

```

#### ENTRY(_start) 到底在干什么？

在链接脚本里：

```
ENTRY(_start)
```

它的作用只有一句话：

> 把 ELF Header 里的 `e_entry` 字段设置为符号 `_start` 的地址。

也就是说：

- 它不生成代码
- 它不改变 section 布局
- 它不影响 reset 向量
- 它只是指定“程序入口符号”

你可以验证：

```
readelf -h hello
```

会看到：

```
Entry point address: 0xXXXX
```

这个地址就是 `_start` 的 VMA。

------

#### 那 `_start` 是什么？

`_start` 只是一个普通符号。

它通常是：

> 程序真正的第一条指令

⚠ 注意：不是 `main`。

#### 在 Linux 中 `_start` 从哪来？

当你写：

```
gcc hello.c -o hello
```

即使你没有写 `_start`，程序仍然可以运行。

因为 GCC 自动链接了：

- `crt1.o`
- `crti.o`
- `crtn.o`

其中在 `crt1.o` 里定义了：

```
_start:
    ...
    call __libc_start_main
```

然后：

```
__libc_start_main → 调用 main
```

所以执行流程是：

```
_start → libc 初始化 → main → exit
```

------

#### 在裸机中 `_start` 是谁？

在裸机环境里：

- 没有 libc
- 没有 crt1.o
- 没有 loader

所以：

> 你必须自己写 `_start`

通常是：

```
.section entry
.global _start

_start:
    la sp, _stack_top
    call init_data
    call zero_bss
    call main
loop:
    j loop
```

------

#### ENTRY() 还能写别的吗？

当然可以。

```
ENTRY(reset_handler)
```

或者：

```
ENTRY(main)
```

只要这个符号存在即可。

链接器只做一件事：

> 找到这个符号的地址 → 写入 ELF header

------

#### 如果不写 ENTRY() 会怎样？

ld 有默认规则：

优先级是：

1. 命令行 `-e symbol`
2. 链接脚本 `ENTRY(symbol)`
3. 查找 `_start`
4. 查找 `start`
5. 取 `.text` 开头

所以即使你不写 ENTRY：

如果存在 `_start`，通常仍然会选它。

------

#### 重要区别：ENTRY ≠ 硬件复位地址

这点非常关键（尤其对你现在的裸机环境）。

在 Linux：

- 内核读取 ELF header
- 跳转到 `e_entry`

在裸机：

CPU 上电后：

- PC 从固定物理地址开始（例如 0x20000000）
- 这与 ELF header 无关

所以在裸机中：

如果你把代码烧到 0x20000000
 那 `_start` 必须刚好位于这个地址

否则 ENTRY 没意义。





### Linux 用户态的启动流程

```bash
:~$ readelf -h hello
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (Position-Independent Executable file)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：               0x1060
  程序头起点：          64 (bytes into file)
  Start of section headers:          13976 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30

```

```
lzk@lzk-Virtual-Machine:~$ gcc hello.c -no-pie -o hello
lzk@lzk-Virtual-Machine:~$ readelf -h hello
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              EXEC (可执行文件)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：               0x401050
  程序头起点：          64 (bytes into file)
  Start of section headers:          13904 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30

```

很好，这个问题非常关键。你已经开始真正“看懂 ELF 了”。

你现在看到：

```
类型: DYN (Position-Independent Executable file)
入口点地址：0x1060
```

你没看到：

```
. = 0x400000
```

这不是你看漏了，而是因为：

> 你现在编译出来的是 PIE（Position Independent Executable）。

------

#### 为什么是 DYN 而不是 EXEC？

注意这一行：

```
类型: DYN (Position-Independent Executable file)
```

在现代 Linux（Ubuntu 默认 GCC）中：

```
gcc hello.c -o hello
```

默认开启：

```
-fPIE
-pie
```

所以生成的不是传统的：

```
EXEC
```

而是：

```
DYN
```

它和共享库使用同一种 ELF 类型。

------

#### PIE 意味着什么？

PIE 的含义是：

> 程序可以被加载到任意虚拟地址。

因此：

- 链接时不能固定 `. = 0x400000`
- 所有代码是相对地址
- 真正加载地址由内核随机决定（ASLR）

所以你看到：

```
入口点地址：0x1060
```

这其实是：

```
相对于加载基址的偏移
```

而不是绝对虚拟地址。

------

#### 那 0x400000 到底去哪了？

在老式（非 PIE）可执行文件中：

```
gcc hello.c -no-pie -o hello
```

然后再：

```
readelf -h hello
```

你会看到：

```
类型: EXEC
入口点地址: 0x401060
```

这时候：

- `.text` 从 0x400000 开始
- `_start` 在 0x401060

也就是说：

```
0x400000 + 0x1060 = 0x401060
```

而你现在的：

```
0x1060
```

是 PIE 模式下的“相对入口”。



所以在非PIE版本中所有程序的入口都在0x400000？那为什么两个不一样的程序不会进入到同一段代码呢

即使两个非-PIE 程序的入口都是 0x400000，它们也不会进入同一段代码，因为每个进程拥有独立的虚拟地址空间。



#### 虚拟地址空间是“每个进程独立的”

在 Linux 上：

- 每个进程都有自己的页表
- 每个进程的虚拟地址空间互相隔离
- 0x400000 在不同进程里是不同的映射

举个例子：

假设你有两个程序：

- hello
- world

它们都是 non-PIE，入口都是 0x401060。

当你同时运行：

```
./hello &
./world &
```

它们的 `/proc/<pid>/maps` 可能看起来像：

```
00400000-00401000 r-xp hello
00400000-00401000 r-xp world
```

看起来地址一样。

但：

- hello 的 0x400000 映射到物理页 A
- world 的 0x400000 映射到物理页 B

因为它们的页表不同。



#### crt1.o / crti.o / crtn.o 分工

很好，现在我们把 Linux 下那三个“神秘文件”彻底讲清楚。

你每次执行：

```
gcc hello.c -o hello
```

实际上 **自动链接了三类启动文件**：

- `crt1.o`
- `crti.o`
- `crtn.o`

它们统称为：

> C runtime startup objects

------

#### 一句话总结它们的职责

| 文件   | 作用                        |
| ------ | --------------------------- |
| crt1.o | 提供 `_start`               |
| crti.o | 提供 `.init` / `.fini` 开头 |
| crtn.o | 提供 `.init` / `.fini` 结尾 |

它们一起构成：

> 程序从进入 `_start` 到调用 `main` 的桥梁

------

#### crt1.o —— 程序真正的入口

这是最核心的。

里面定义了：

```
_start:
    ...
    call __libc_start_main
```

执行流程大概是：

```
_start
  ↓
__libc_start_main
  ↓
main
  ↓
exit
```

也就是说：

> 你写的 main 从来不是 CPU 的第一条指令。

第一条指令永远是 `_start`。

------

#### 那 __libc_start_main 干嘛？

它负责：

- 解析 argc / argv
- 初始化 libc
- 初始化全局构造函数
- 调用 main
- 调用 exit

它是 glibc 里的核心入口函数。

------

#### crti.o 和 crtn.o 是干嘛的？

这两个文件很多人完全不知道。

它们是配合 `.init` / `.fini` section 用的。

在 ELF 里有两个特殊 section：

- `.init`
- `.fini`

用于：

- 构造函数初始化
- 程序结束清理

------

##### crti.o 的作用

提供：

```
.section .init
_init:
    push ...
    ...
```

也就是说：

> 提供 .init section 的“开头”

------

##### crtn.o 的作用

提供：

```
.section .init
    pop ...
    ret
```

它补上“结尾”。

------

#### 为什么要拆成两个文件？

因为：

链接顺序是：

```
crti.o
你的 .o 文件
crtn.o
```

所以 `.init` section 会变成：

```
[crti 的代码]
[你的构造函数]
[crtn 的收尾代码]
```

这是一种“拼接结构”。

------

#### 完整启动流程图（Linux）

```
内核
  ↓
加载 ELF
  ↓
跳转到 e_entry
  ↓
_start (crt1.o)
  ↓
__libc_start_main
  ↓
调用全局构造函数
  ↓
main
  ↓
exit
  ↓
调用析构函数
```

------

#### 如果你用 -nostdlib 会发生什么？

试试：

```
gcc hello.c -nostdlib -o hello
```

你会得到：

```
undefined reference to `_start`
```

因为：

- crt1.o 没有被链接
- 没有人提供 `_start`

这就和你裸机环境一模一样了。

现在我们正式进入**链接脚本的语义层**，而不是停留在“它可以控制地址”这种表层说法。

这一节我们要解决四个问题：

1. 链接脚本本质在做什么？
2. VMA 和 LMA 是如何被表达的？
3. `AT()` 到底改变了什么？
4. 为什么嵌入式必须关心 LMA，而 Linux 通常不用？

### 裸机环境的启动流程

现在你可以看到完整逻辑：

1. CPU 复位
2. SP 未知
3. 必须先设置栈
4. 才能安全调用函数
5. 再初始化 .data / .bss
6. 再进入 C 语言世界

堆是“附加能力”
 栈是“基础设施”

#### 一、启动代码（startup code）本质是什么？

一句话定义：

> 启动代码 = 在进入 C 语言世界之前，建立运行环境的那段最低级代码。

CPU 上电后不会自动帮你：

- 设置栈
- 初始化全局变量
- 清零未初始化变量
- 调用构造函数
- 跳转到 main()

这些都必须由你（或者 libc）完成。

------

#### 启动代码必须做的几件事（按顺序）

你目前讲的是偏裸机 / bootloader 场景，我重点讲这个。

------

#### 1️⃣ 必须先设置栈（Stack Pointer）

因为 C 语言依赖栈。

例如：

```
int main() {
    int a = 5;
}
```

变量 `a` 在栈上。

函数调用：

```
foo();
```

会：

- 把返回地址压栈
- 保存寄存器
- 分配局部变量空间

如果 SP 没有指向合法内存：

- push 会写到未知地址
- 函数调用直接崩溃
- 甚至访问到非法地址导致异常

所以启动代码第一步通常是：

```
ldr sp, =_stack_top
```

或

```
mov sp, #0x20010000
```

这一步建立运行时环境的“地基”。

------

#### 2️⃣ 为什么必须复制 .data 段？

这涉及 LMA / VMA。

典型裸机内存布局：

Flash：

```
.text
.rodata
.data  (初始化数据的存储副本)
```

RAM：

```
.data  (真正运行时的)
.bss
.stack
.heap
```

关键点：

.data 段：

- 运行地址在 RAM
- 初始值存储在 Flash

例如：

```
int g = 5;
```

编译器会把 5 放到 Flash。

但运行时：

```
g 必须在 RAM 中
```

因为：

- 全局变量可能被修改
- Flash 通常不可写

所以启动代码必须：

```
把 Flash 中的 .data 复制到 RAM 中的 .data
```

否则：

- g 读取值是错的
- 或者访问未初始化内存

典型代码：

```
uint32_t *src = &_sidata;   // LMA
uint32_t *dst = &_sdata;    // VMA
while (dst < &_edata) {
    *dst++ = *src++;
}
```

#### 为什么必须清零 .bss？

.bss 段：

存放未初始化的全局变量

```
int x;
```

按照 C 标准：

> 未初始化的全局变量默认必须为 0

但：

- Flash 里不会给它存 0
- RAM 上电后是随机值

所以必须清零。

启动代码必须做：

```
uint32_t *p = &_sbss;
while (p < &_ebss) {
    *p++ = 0;
}
```

否则：

程序逻辑会随机错误。

------

#### 最后才能调用 main()

完整流程：

```
CPU reset
  ↓
跳到 reset handler
  ↓
设置 SP
  ↓
复制 .data
  ↓
清零 .bss
  ↓
调用 C++ 构造函数（如果有）
  ↓
调用 main()
```

------

#### 如果不做这些会发生什么？

| 忽略步骤       | 后果                 |
| -------------- | -------------------- |
| 不设栈         | 函数调用直接崩溃     |
| 不复制 .data   | 初始化全局变量值错误 |
| 不清零 .bss    | 未初始化变量随机值   |
| 不调用构造函数 | C++ 静态对象出错     |

------

#### Linux 用户态程序是谁做这些？

在 Linux 下：

这些由 glibc 的启动文件完成：

- crt1.o
- crti.o
- crtn.o

入口函数是：

```
_start
```

由 glibc 提供。

它做的事情本质一样：

- 建立栈
- 解析 argc / argv
- 初始化 libc
- 调用 main
- 调用 exit















### 





## 第六部分：嵌入式视角 —— VMA 与 LMA

### 第13章：什么是 VMA 与 LMA？

#### VMA 和 LMA 的本质定义

在 ELF 中，每个输出 section 都有两个地址：

| 名称                         | 含义                   |
| ---------------------------- | ---------------------- |
| VMA (Virtual Memory Address) | 程序运行时访问的地址   |
| LMA (Load Memory Address)    | 程序被加载时存放的地址 |

默认情况下：

```
VMA == LMA
```

只有在嵌入式场景下才会分离。

------

#### 看你当前脚本的前半部分

```
MEMORY
{
  MROM (rx)  : ORIGIN = 0x20000000, LENGTH = 128K
  SRAM (rwx) : ORIGIN = 0x0f000000, LENGTH = 8K
}
```

这一步做的事情是：

- 定义两个可分配内存区域
- 后续 `> MROM` 或 `> SRAM` 会引用它

⚠ 注意：

`MEMORY` 不会自动放置 section
 它只是提供可选区域。

------

#### location counter（`.`）的真正含义

```
. = ORIGIN(MROM);
```

`.` 叫 location counter。

它代表：

> 当前 section 的 VMA 写入位置

此时：

```
. = 0x20000000
```

------

#### `.text` 的 VMA 如何确定？

```
.text :
{
  *(entry)
  *(.text*)
} > MROM
```

`> MROM` 的意思是：

> 把这个 section 的 VMA 放入 MROM 区域

因为当前 `.` 已经在：

```
0x20000000
```

所以：

```
.text VMA = 0x20000000
```

而因为没有写 `AT()`：

```
.text LMA = VMA
```

------

#### `.rodata` 同理

```
.rodata :
{
  *(.rodata*)
} > MROM
```

- 紧接在 `.text` 之后
- VMA 在 MROM
- LMA = VMA

到这里：

ROM 内部是连续布局的。

------

#### 现在进入关键部分：`.data`

你写了：

```
. = ORIGIN(SRAM);

.data : AT(LOADADDR(.rodata) + SIZEOF(.rodata))
{
  _data_start = .;
  *(.data*)
  _data_end = .;
} > SRAM
```

我们逐行拆。

------

##### 第一步

```
. = ORIGIN(SRAM);
```

把 VMA 指针改成：

```
0x0f000000
```

------

##### 第二步

```
> SRAM
```

表示：

```
.data 的 VMA 在 SRAM
```

### 第14章：链接脚本中的 AT 语法

## 第三步：AT()

这是核心。

```
AT(LOADADDR(.rodata) + SIZEOF(.rodata))
```

它改变的是：

> LMA

这句话的意思是：

```
.data 的 LMA = rodata 在 ROM 中的末尾
```

因此最终结果是：

| 属性      | 值          |
| --------- | ----------- |
| .data VMA | 0x0f000000  |
| .data LMA | 紧接 rodata |

------

# 八、这在 ELF 里如何体现？

在 ELF 的 Program Header 里：

你会看到：

```
LOAD segment:
  p_vaddr != p_paddr
```

VMA 是 p_vaddr
 LMA 是 p_paddr

在嵌入式中：

- p_paddr = 烧录地址
- p_vaddr = 运行地址

------

# 九、为什么必须这样做？

因为：

- ROM 不可写
- 全局变量必须可写
- 所以 data 必须运行在 SRAM

但：

- 初始值存储在 ROM
- 启动时必须 memcpy

这就是 startup code 的责任。

------

# 十、`.bss` 为什么没有 AT()？

```
.bss :
{
  *(.bss*)
} > SRAM
```

`.bss`：

- 没有初始值
- 不需要存储在 ROM
- LMA 不存在
- 只需要运行时清零

------

# 十一、总结一句话

`>` 决定 VMA
 `AT()` 决定 LMA

如果不写 AT：

```
LMA = VMA
```

------

# 十二、你这个脚本是一个典型的“ROM to RAM copy model”

内存布局是：

ROM:

```
.text
.rodata
.data (initial image)
```

SRAM:

```
.data (runtime)
.bss
.heap
.stack
```

## 第七部分：完整运行流程闭环





