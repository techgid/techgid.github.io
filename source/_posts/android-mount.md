---
title: android mount 简单分析
date: 2017-04-23 22:47:52
tags:
---
### 概述
在某些情况下挂着分区时需要单独做挂载的特性处理，比如小内存优化时，mount时不需要discard特性。因此我们在mount时做判断，符合条件加上相关特性，不符合则去除。这需要确定两点，首先我们要清楚mount流程，用来确定在什么地方做判断最合适。其次明确判断条件，mount data时是该版本去除discard属性，不是该版本保留discard属性。
> 参考源码： [AndroidXref Nougat 7.1.1](http://androidxref.com/7.1.1_r6/)

<!-- more -->

---

### mount相关命令介绍
> mount - 挂载文件系统

此命令的标准格式是：
```
mount -t type device dir
```
-t 文件系统类型 常用的有ext4 ufs vfat
device 设备节点
dir 挂载点

- 原 dir 里面的 内容/属主/权限 将被屏蔽，直到此设备被卸载。
- 如果只给出了 dir 或者只给出了 device ，那么将根据 /etc/fstab 的设置进行挂载。
- 如果想避免 dir 与 device 之间的混淆，可以使用 --target(表示dir) 或 --source(表示device) 进行明确标明：

```
mount --target /mountpoint
```
常用flag
noatime
常用option
discard
此处以后慢慢补充相关介绍。

在android系统终我们可以使用下面命令查看基本分区信息。
```
adb shell mount
adb shell df -h
```

### mount流程
Android系统中有很多分区，如"system" "data" "cache"分区等，通常在kernel起来后执行的第一个文件init进程中去挂载这些分区，init进程会根据init.rc的规则进行启动进程或相关服务。
>   /device/moto/shamu/init.shamu.rc

```
38on fs
39    mount_all fstab.shamu
40
41    # Keeping following partitions outside fstab file. As user may not have
42    # these partition flashed on the device. Failure to mount any partition in fstab file
43    # results in failure to launch late-start class.
44    wait /dev/block/platform/msm_sdcc.1/by-name/oem
45    mount ext4 /dev/block/platform/msm_sdcc.1/by-name/oem /oem ro nosuid nodev context=u:object_r:oemfs:s0
46
47    mkdir /fsg 0755 root root
48    mount ext4 /dev/block/platform/msm_sdcc.1/by-name/mdm1m9kefs3 /fsg ro nosuid nodev barrier=0 context=u:object_r:fsg_file:s0
```
在init.rc规则文件中我们看到挂载相关主要使用两个命令**mount**和**mount_all**，下面我们将分别分析这两个命令。
> system/core/init/builtins.c

```
1017        {"mount_all",               {1,     kMax, do_mount_all}},
1018        {"mount",                   {3,     kMax, do_mount}},
```
我们可以看到mount对应的函数为do_mount，因此我们继续追踪do_mount函数。

```
377/* mount <type> <device> <path> <flags ...> <options> */
378static int do_mount(const std::vector<std::string>& args) {
379    char tmp[64];
380    const char *source, *target, *system;
381    const char *options = NULL;
382    unsigned flags = 0;
383    std::size_t na = 0;
384    int n, i;
385    int wait = 0;
386
387    for (na = 4; na < args.size(); na++) {
388        for (i = 0; mount_flags[i].name; i++) {
389            if (!args[na].compare(mount_flags[i].name)) {
390                flags |= mount_flags[i].flag;
391                break;
392            }
393        }
394
395        if (!mount_flags[i].name) {
396            if (!args[na].compare("wait"))
397                wait = 1;
398            /* if our last argument isn't a flag, wolf it up as an option string */
399            else if (na + 1 == args.size())
400                options = args[na].c_str();
401        }
402    }
403
404    system = args[1].c_str();
405    source = args[2].c_str();
406    target = args[3].c_str();
...
418        if (mount(tmp, target, system, flags, options) < 0) {
```
do_mount函数比较简单，最后使用系统调用mount函数来挂载分区。至此mount命令结束，如果有特殊需求我们也可以在此函数上做简单修改。例如判断为对应版本的flash时追加discard属性：
```
diff --git a/init/builtins.c b/init/builtins.c
index 026115a..5246f32 100755
--- a/init/builtins.c
+++ b/init/builtins.c
@@ -887,6 +887,15 @@ char  emerg_flag = 0;
     type = args[1];
     source = args[2];
     target = args[3];
+
+    if (! strcmp(target, "/data")) {
+        if (! fs_mgr_is_hynic_emmc50()) {
+            options = "discard";
+	       }
+    }
+
 #if 0
    if (!strcmp(target, DATA_MNT_POINT)&&!strcmp(source,DATA_PATH) )
```
而mount_all命令对应do_mount_all函数。
```
596/* mount_all <fstab> [ <path> ]* [--<options>]*
597 *
598 * This function might request a reboot, in which case it will
599 * not return.
600 */
601static int do_mount_all(const std::vector<std::string>& args) {
602    std::size_t na = 0;
603    bool import_rc = true;
604    bool queue_event = true;
605    int mount_mode = MOUNT_MODE_DEFAULT;
606    const char* fstabfile = args[1].c_str();
607    std::size_t path_arg_end = args.size();
608
609    for (na = args.size() - 1; na > 1; --na) {
610        if (args[na] == "--early") {
611             path_arg_end = na;
612             queue_event = false;
613             mount_mode = MOUNT_MODE_EARLY;
614        } else if (args[na] == "--late") {
615            path_arg_end = na;
616            import_rc = false;
617            mount_mode = MOUNT_MODE_LATE;
618        }
619    }
620
621    int ret =  mount_fstab(fstabfile, mount_mode);
622
623    if (import_rc) {
624        /* Paths of .rc files are specified at the 2nd argument and beyond */
625        import_late(args, 2, path_arg_end);
626    }
627
628    if (queue_event) {
629        /* queue_fs_event will queue event based on mount_fstab return code
630         * and return processed return code*/
631        ret = queue_fs_event(ret);
632    }
633
634    return ret;
635}
```
do_mount_all函数传入的参数为fstab.shamu文件系统挂载表路径。该函数首先对参数进行了预处理。然后调用mount_fastab函数，进行mount操作。首先我们看下fstab。
> /device/moto/shamu/fstab.shamu

```
1# Android fstab file.
2# The filesystem that contains the filesystem checker binary (typically /system) cannot
3# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK
4#
5#<src>                                                <mnt_point>  <type>  <mnt_flags and options>                     <fs_mgr_flags>
6/dev/block/platform/msm_sdcc.1/by-name/system         /system      ext4    ro,barrier=1                                wait,verify=/dev/block/platform/msm_sdcc.1/by-name/metadata
7/dev/block/platform/msm_sdcc.1/by-name/userdata    /data        ext4    rw,nosuid,nodev,noatime,nodiratime,noauto_da_alloc,nobarrier    wait,check,formattable,forceencrypt=/dev/block/platform/msm_sdcc.1/by-name/metadata
8/dev/block/platform/msm_sdcc.1/by-name/cache       /cache       ext4    rw,noatime,nosuid,nodev,barrier=1,data=ordered   wait,check,formattable
9/dev/block/platform/msm_sdcc.1/by-name/modem       /firmware    ext4    ro,barrier=1,context=u:object_r:firmware_file:s0    wait
10/dev/block/platform/msm_sdcc.1/by-name/persist     /persist     ext4    rw,nosuid,nodev,barrier=1                      wait,check,notrim
11/dev/block/platform/msm_sdcc.1/by-name/boot         /boot           emmc    defaults                                                        defaults
12/dev/block/platform/msm_sdcc.1/by-name/recovery     /recovery       emmc    defaults                                                        defaults
13/dev/block/platform/msm_sdcc.1/by-name/misc         /misc           emmc    defaults                                                        defaults
14/dev/block/platform/msm_sdcc.1/by-name/modem        /modem          emmc    defaults                                                        defaults
15/dev/block/platform/msm_sdcc.1/by-name/mdm1m9kefs1  /mdm1m9kefs1    emmc    defaults                                                        defaults
16/dev/block/platform/msm_sdcc.1/by-name/mdm1m9kefs2  /mdm1m9kefs2    emmc    defaults                                                        defaults
17/dev/block/platform/msm_sdcc.1/by-name/mdm1m9kefs3  /mdm1m9kefs3    emmc    defaults                                                        defaults
18/dev/block/platform/msm_sdcc.1/by-name/sbl1         /sbl1           emmc    defaults                                                        defaults
19/dev/block/platform/msm_sdcc.1/by-name/tz           /tz             emmc    defaults                                                        defaults
20/dev/block/platform/msm_sdcc.1/by-name/rpm          /rpm            emmc    defaults                                                        defaults
21/dev/block/platform/msm_sdcc.1/by-name/sdi          /sdi            emmc    defaults                                                        defaults
22/dev/block/platform/msm_sdcc.1/by-name/aboot        /aboot          emmc    defaults                                                        defaults
23/dev/block/platform/msm_sdcc.1/by-name/versions     /versions       emmc    defaults                                                        defaults
24/dev/block/platform/msm_sdcc.1/by-name/logo         /logo           emmc    defaults                                                        defaults
25/dev/block/platform/msm_sdcc.1/by-name/frp          /persistent     emmc    defaults                                                        defaults
26/devices/*/xhci-hcd.0.auto/usb*                     auto            auto    defaults                                                        voldmanaged=usb:auto
```
每行对应的是设备节点，挂载点，flag以及options，我们继续追踪mount_fstab函数。
```
502/* mount_fstab
503 *
504 *  Call fs_mgr_mount_all() to mount the given fstab
505 */
506static int mount_fstab(const char* fstabfile, int mount_mode) {
507    pid_t pid;
508    int ret = -1;
509    int child_ret = -1;
510    int status;
511    struct fstab *fstab;
512
513    /*
514     * Call fs_mgr_mount_all() to mount all filesystems.  We fork(2) and
515     * do the call in the child to provide protection to the main init
516     * process if anything goes wrong (crash or memory leak), and wait for
517     * the child to finish in the parent.
518     */
519    pid = fork();
520    if (pid > 0) {
521        /* Parent.  Wait for the child to return */
522        int wp_ret = TEMP_FAILURE_RETRY(waitpid(pid, &status, 0));
523        if (wp_ret < 0) {
524            /* Unexpected error code. We will continue anyway. */
525            NOTICE("waitpid failed rc=%d: %s\n", wp_ret, strerror(errno));
526        }
527
528        if (WIFEXITED(status)) {
529            ret = WEXITSTATUS(status);
530        } else {
531            ret = -1;
532        }
533    } else if (pid == 0) {
534        /* child, call fs_mgr_mount_all() */
535        klog_set_level(6);  /* So we can see what fs_mgr_mount_all() does */
536        fstab = fs_mgr_read_fstab(fstabfile);
537        child_ret = fs_mgr_mount_all(fstab, mount_mode);
538        fs_mgr_free_fstab(fstab);
539        if (child_ret == -1) {
540            ERROR("fs_mgr_mount_all returned an error\n");
541        }
542        _exit(child_ret);
543    } else {
544        /* fork failed, return an error */
545        return -1;
546    }
547    return ret;
548}
```
该函数中主要调用两个函数，首先读取fstab挂载表，然后挂载。
```
536        fstab = fs_mgr_read_fstab(fstabfile);
537        child_ret = fs_mgr_mount_all(fstab, mount_mode);
```
我们继续看这两个函数
> /system/core/fs_mgr/fs_mgr_fstab.c

```
245struct fstab *fs_mgr_read_fstab(const char *fstab_path)
246{
247    FILE *fstab_file;
248    int cnt, entries;
249    ssize_t len;
250    size_t alloc_len = 0;
251    char *line = NULL;
252    const char *delim = " \t";
253    char *save_ptr, *p;
254    struct fstab *fstab = NULL;
255    struct fs_mgr_flag_values flag_vals;
256#define FS_OPTIONS_LEN 1024
257    char tmp_fs_options[FS_OPTIONS_LEN];
258
259    fstab_file = fopen(fstab_path, "r");
260    if (!fstab_file) {
261        ERROR("Cannot open file %s\n", fstab_path);
262        return 0;
263    }
264
265    entries = 0;
266    while ((len = getline(&line, &alloc_len, fstab_file)) != -1) {
267        /* if the last character is a newline, shorten the string by 1 byte */
268        if (line[len - 1] == '\n') {
269            line[len - 1] = '\0';
270        }
271        /* Skip any leading whitespace */
272        p = line;
273        while (isspace(*p)) {
274            p++;
275        }
276        /* ignore comments or empty lines */
277        if (*p == '#' || *p == '\0')
278            continue;
279        entries++;
280    }
281
282    if (!entries) {
283        ERROR("No entries found in fstab\n");
284        goto err;
285    }
286
287    /* Allocate and init the fstab structure */
288    fstab = calloc(1, sizeof(struct fstab));
289    fstab->num_entries = entries;
290    fstab->fstab_filename = strdup(fstab_path);
291    fstab->recs = calloc(fstab->num_entries, sizeof(struct fstab_rec));
292
293    fseek(fstab_file, 0, SEEK_SET);
294
295    cnt = 0;
296    while ((len = getline(&line, &alloc_len, fstab_file)) != -1) {
297        /* if the last character is a newline, shorten the string by 1 byte */
298        if (line[len - 1] == '\n') {
299            line[len - 1] = '\0';
300        }
301
302        /* Skip any leading whitespace */
303        p = line;
304        while (isspace(*p)) {
305            p++;
306        }
307        /* ignore comments or empty lines */
308        if (*p == '#' || *p == '\0')
309            continue;
...
```
可以看函数到按行读取fstab，返回fstab结构体实例.因此如果有特殊需求我们可以在读取文件系统表时做文章。下面就是fs_mgr_mount_all函数。
```
487/* When multiple fstab records share the same mount_point, it will
488 * try to mount each one in turn, and ignore any duplicates after a
489 * first successful mount.
490 * Returns -1 on error, and  FS_MGR_MNTALL_* otherwise.
491 */
492int fs_mgr_mount_all(struct fstab *fstab, int mount_mode)
493{
494    int i = 0;
495    int encryptable = FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE;
496    int error_count = 0;
497    int mret = -1;
498    int mount_errno = 0;
499    int attempted_idx = -1;
500
501    if (!fstab) {
502        return -1;
503    }
504
505    for (i = 0; i < fstab->num_entries; i++) {
506        /* Don't mount entries that are managed by vold or not for the mount mode*/
507        if ((fstab->recs[i].fs_mgr_flags & (MF_VOLDMANAGED | MF_RECOVERYONLY)) ||
508             ((mount_mode == MOUNT_MODE_LATE) && !fs_mgr_is_latemount(&fstab->recs[i])) ||
509             ((mount_mode == MOUNT_MODE_EARLY) && fs_mgr_is_latemount(&fstab->recs[i]))) {
510            continue;
511        }
512
...
603            if (fs_mgr_do_format(&fstab->recs[top_idx]) == 0) {
604                /* Let's replay the mount actions. */
605                i = top_idx - 1;
606                continue;
607            } else {
608                ERROR("%s(): Format failed. Suggest recovery...\n", __func__);
609                encryptable = FS_MGR_MNTALL_DEV_NEEDS_RECOVERY;
610                continue;
611            }
612        }
613        if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
...
634        } else {
635            if (fs_mgr_is_nofail(&fstab->recs[attempted_idx])) {
636                ERROR("Ignoring failure to mount an un-encryptable or wiped partition on"
637                       "%s at %s options: %s error: %s\n",
638                       fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
639                       fstab->recs[attempted_idx].fs_options, strerror(mount_errno));
640            } else {
641                ERROR("Failed to mount an un-encryptable or wiped partition on"
642                       "%s at %s options: %s error: %s\n",
643                       fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
644                       fstab->recs[attempted_idx].fs_options, strerror(mount_errno));
645                ++error_count;
646            }
647            continue;
648        }
649    }
...
656}
```
进一步调用fs_mgr_do_mount函数。然后走到__mount函数，最终任然使用系统调用mount函数：
```
ret = mount(source, target, rec->fs_type, mountflags, rec->fs_options);
```
