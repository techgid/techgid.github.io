---
title: Kernel apanic（KE） 分析方法
date: 2017-04-06 22:13:39
catalog: flase
categories:
  - 学习笔记
tags:
  - Android kernel
---

### kernel apanic介绍

apanic属于内核级别的异常，log打印出的是堆栈地址。

<!-- more -->
> apanic一般是死机重启的问题，常见空指针（null pointer）一般都与cpu位转换相关，由于地址中的某一位跳变，导致中断地址异常。高通对此有patch但是任然具有一定的复现概率。  

apanic log例如：

```
Call Trace:
[ef12f700] [c00081e0] show_stack+0x3c/0x194 (unreliable)
[ef12f730] [c0019b2c] __schedule_bug+0x64/0x78
[ef12f750] [c0350f50] schedule+0x324/0x34c
[ef12f7a0] [c03515c0] schedule_timeout+0x68/0xe4
[ef12f7e0] [c027938c] fsl_elbc_run_command+0x138/0x1c0
[ef12f820] [c0275820] nand_do_read_ops+0x130/0x3dc
[ef12f880] [c0275ebc] nand_read+0xac/0xe0
[ef12f8b0] [c0262d98] part_read+0x5c/0xe4
[ef12f8c0] [c017bcac] jffs2_flash_read+0x68/0x254
[ef12f8f0] [c0170550] jffs2_read_dnode+0x60/0x304
[ef12f940] [c017088c] jffs2_read_inode_range+0x98/0x180
[ef12f970] [c016e610] jffs2_do_readpage_nolock+0x94/0x1ac
[ef12f990] [c016ee04] jffs2_write_begin+0x2b0/0x330
[ef12fa10] [c005144c] generic_file_buffered_write+0x11c/0x8d0
[ef12fab0] [c0051e48] __generic_file_aio_write_nolock+0x248/0x500
[ef12fb20] [c0052168] generic_file_aio_write+0x68/0x10c
[ef12fb50] [c007ca80] do_sync_write+0xc4/0x138
[ef12fc10] [f107c0dc] oops_log+0xdc/0x1e8 [oopslog]
[ef12fe70] [f3087058] oops_log_init+0x58/0xa0 [oopslog]
[ef12fe80] [c00477bc] sys_init_module+0x130/0x17dc
[ef12ff40] [c00104b0] ret_from_syscall+0x0/0x38
--- Exception: c01 at 0xff29658
    LR = 0x10031300
```

我们可以对vmlinux进行反编译，获取堆栈地址对应的函数信息。gdb是我们常用的反编译工具，其他反编译如IDA也可以尝试。

### 相关工具及基本认识
>SP（R13）堆栈指针， 指向异常模式所专用的堆栈

>LR（R14）连接寄存器
* 用来存放子程序返回地址
* 异常发生时保存异常返回地址（一般为PC减去0x04或0x02）

>PC（R15）存放当前指令地址

Android源码中自带gdb调试工具(不同的Android版本可能位置不同)：
```
prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin/arm-linux-androideabi-gdb
```
Android版本对应的vmlinux：
```
out/target/product/xxx/obj/KERNEL_OBJ/vmlinux
```

### gdb 反编译常用命令
>加载symbol文件

```
file vmlinux
```

>反编译函数形成汇编语言

```
disassemble (/m) do_sync_write (/m反编译成汇编与c混合)
```
>利用加假断点可以查看对应代码位置与行号

```
d *(do_sync_write+0xc4)
```
>查看函数信息

```
info line do_sync_write
info line *0xff29658
```

我们可以利用反编译可以很轻松的查到对应的代码调用信息。
