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

apanic属于内核级别的异常，log打印出的是堆栈地址。apanic一般是死机重启的问题，常见空指针（null pointer）或地址加载异常，一般都与cpu稳定性相关。

<!-- more -->

### 常见的错误类型：

```
Unable to handle kernel NULL pointer dereference at virtual address 00000012
Unable to handle kernel paging request at virtual address 4042a00c
```

apanic log例如：

```
[   20.896935] c3 554 (netd) Unable to handle kernel NULL pointer dereference at virtual address 00000012
[   20.906200] c3 554 (netd) pgd = ffffffc02f746000
[   20.910793] c3 554 (netd) SEH:seh_api_ioctl_handler 7
[   20.910793]
[   20.917357] c3 554 (netd) [00000012] *pgd=0000000000000000
[   20.922798] c3 554 (netd) Internal error: Oops: 96000005 [#1] PREEMPT SMP
[   20.929523] Modules linked in: audiostub cidatattydev gs_modem ccinetdev cci_datastub citty iml_module seh cploaddev msocketk tzdd galcore(O)
[   20.942183] c3 554 (netd) CPU: 3 PID: 554 Comm: netd Tainted: G           O 3.10.33 #1
[   20.950030] c3 554 (netd) task: ffffffc02e0a0000 ti: ffffffc029b0c000 task.ti: ffffffc029b0c000
[   20.958659] c3 554 (netd) PC is at __kill_pgrp_info+0x1c/0x84
[   20.964360] c3 554 (netd) LR is at kill_orphaned_pgrp+0xa4/0xb8
[   20.970232] c3 554 (netd) pc : [<ffffffc0000b39f0>] lr : [<ffffffc0000a3d10>] pstate: 80000145
[   20.978769] c3 554 (netd) sp : ffffffc029b0fc40
[   20.983257] R29: ffffffc029b0fc40 R28: ffffffc02e0a0200
[   20.988537] R27: 0000000000000001 R26: 00000000fffffff6
[   20.993819] R25: ffffffc00072f000 R24: ffffffc029b0fda8
[   20.999100] R23: ffffffc029b0c000 R22: 0000000000000001
[   21.004381] R21: 0000000000000001 R20: ffffffc029b0fd80
[   21.009663] R19: ffffffc02356eb40 R18: 0000000000000000
[   21.014935] R17: 0000000000000000 R16: ffffffc000102988
[   21.020216] R15: 0000000000000000 R14: 00000000f755fdb7
[   21.025497] R13: 00000000f6f0bb78 R12: 0000000000000000
[   21.030778] R11: 00000000f6f0bb9c R10: 0000000000000fff
[   21.036060] R9 : 00000001ffffffff R8 : 0000000033ffff7f
[   21.041342] R7 : 00000000000185ad R6 : ffffffc02f14614c
[   21.046623] R5 : 0000000000000000 R4 : 0000000000000000
[   21.051903] R3 : ffffffc02f146140 R2 : 0000000000000002
[   21.057185] R1 : 0000000000000001 R0 : 0000000000000001
...

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
上面的log我们可以看到进程栈，以及栈帧（stack frame）调用关系。还可以看到此时各寄存器对应的值。（关于 stack和 stack frame以后会介绍）
我们可以对vmlinux进行反编译，获取堆栈地址对应的函数信息。gdb是我们常用的反编译工具，其他反编译如IDA也可以尝试。

### 相关工具及基本认识
>SP（R13）堆栈指针， 指向异常模式所专用的堆栈

>LR（R14）连接寄存器
* 用来存放子程序返回地址
* 异常发生时保存异常返回地址（一般为PC减去0x04或0x02）

>PC（R15）存放当前指令地址
(此处后面会补充常用汇编命令和R1-R15寄存器意义)

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

我们可以利用反编译可以很轻松的查到对应的代码调用信息。根据反汇编函数可以看到对应寄存操作，在panic问题出现时一般会打印此时所有寄存器的值，我们可以在反汇编的基础上来推算运算值是否准确，从计算的结果可以方便的判断出计算值是否发生bitflip,也可以知道cpu的ALU计算单元错误，这和cpu的设计和稳定性相关。
