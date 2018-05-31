---
layout: post
title: df命令卡住问题分析
category: debug
tags: df,ufsd
---


### 路由器存储挂机，df磁盘卡死，导致文件共享和DLNA无法使用
    这个bug重现需要硬盘挂载，多个smb拷贝文件连接，重现时间不定。
#### 现象：
应该是拷贝时因路由器异常重启导致硬盘文件系统出错，然后拷贝文件过程中遇到错误而无法访问磁盘，ls，df磁盘目录等都无法执行

补充：后面验证是多个拷贝过程中，statfs不返回，导致卡死
#### 异常重启有两个情况：
1、多个拷贝和dlna使用时导致内存不足，从而触发oom重启

可以通过cgroups限制smb和dlna的内存使用来优化

补充：这个优化无法达到效果，因为dlna扫描时占用很大内存，测试了40M也无法满足需求，当到达内存后，不会再扫描，导致资源媒体扫描不了。评估是否需要限制dlna和samba内存，防止其使用过多导致oom。

2、还有一个未知原因重启，使用过程中没有任何打印。

#### 猜想
1) 系统调用卡住 可能为磁盘拷贝过程中重启导致磁盘错误，下次拷贝遍历到磁盘错误位置后重启。
2) 单核多核smp以及交换分区等影响 导致进程D状态卡住
3) D状态资源死锁

4) 重启 可能为硬盘拷贝时需要电流太大，导致供电不足重启（供电问题也是直接重启没有任何打印）
5) 重启 也可能为看门狗重启
#### 调试：

**断电重启验证**

验证电源重启，同样的软件同样的磁盘，烧录另一个软件，未重启，证明与电源无关

验证看门狗，系统调用磁盘卡住，导致看门狗重启
关闭看门狗，挂机导致机器卡死，所有功能不能使用，也不重启，说明异常重启很有可能就是看门狗导致。进一步确定需要在内核看门狗代码处添加打印

#### 结果
经过验证 用华硕的ufsd驱动挂载磁盘，拷贝6个小时左右，还未出现重启




**df卡住验证**

df 卡住网上说一般是等待nfs资源，对端被down掉，同理本地磁盘就是无法访问磁盘。因问题出现时间不确定，机会少，所以先前期排除大致可能情况，问题出现后可以尽快定位。
查看df源码，就是简单的访问磁盘然后获取信息。strace查看系统调用，卡在statfs。写了一个statfs系统调用小程序。出现后马上运行小程序，确实卡死在statfs系统调用上。

这里调试系统调用栈可以通过内核CONFIG_STACKTRACE,可以在proc生成栈信息，方便查看，不过我的单核用了smp，导致没有信息。

在内核添加动态打印DYNAMIC_DEBUG，获取卡住流程，在statfs调用ufsd的具体statfs函数时没有返回

查看驱动，测试的都为ntfs格式，并且ntfs驱动用的另一个内核编译出得ko文件，所以判断 可能为ufsd驱动和内核不匹配，使用另一个华硕ufsd驱动，然后再换ntfs驱动，最后换ntfs-3g

后续验证：使用华硕ufsd验证后，没有重启，但是也会出现卡住，卡住时栈信息

```
cat /proc/3334/stack 
[<bf12520c>] ufsd_get_sb+0x20c/0x330 [ufsd]
[<bf12d5f0>] ufsd_file_open+0xa0/0x25c [ufsd]
[<c00f96f8>] __dentry_open.isra.14+0x234/0x340
[<c00fa5ec>] nameidata_to_filp+0x88/0x9c
[<c010869c>] do_last.isra.41+0x4a0/0x5e8
[<c010895c>] do_filp_open+0x178/0x4d4
[<c00fa66c>] do_sys_open+0x6c/0x108
[<c00fa738>] sys_open+0x30/0x34
[<c0040880>] ret_fast_syscall+0x0/0x30
[<ffffffff>] 0xffffffff
```
因没有源代码，无法查看ufsd_get_sb信息
通过sysrq+w查看阻塞进程


```

[51293.320000] SysRq : Show Blocked State
[51293.320000]   task                PC stack   pid father
[51293.320000] sync_supers   D c038f9c0     0    75      2 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c03901a4>] (schedule_timeout+0x28/0x31c)
[51293.320000] [<c03901a4>] (schedule_timeout+0x28/0x31c) from [<c038fce4>] (wait_for_common+0xf8/0x1b8)
[51293.320000] [<c038fce4>] (wait_for_common+0xf8/0x1b8) from [<c038fdc4>] (wait_for_completion+0x20/0x24)
[51293.320000] [<c038fdc4>] (wait_for_completion+0x20/0x24) from [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc)
[51293.320000] [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc) from [<bf125cb8>] (ufsd_write_super+0xb4/0x11c [ufsd])
[51293.320000] [<bf125cb8>] (ufsd_write_super+0xb4/0x11c [ufsd]) from [<c00fdd24>] (sync_supers+0xc4/0x140)
[51293.320000] [<c00fdd24>] (sync_supers+0xc4/0x140) from [<c00d6b80>] (bdi_sync_supers+0x44/0x58)
[51293.320000] [<c00d6b80>] (bdi_sync_supers+0x44/0x58) from [<c0083df0>] (kthread+0x98/0xa0)
[51293.320000] [<c0083df0>] (kthread+0x98/0xa0) from [<c00419ac>] (kernel_thread_exit+0x0/0x8)
[51293.320000] khubd         D c038f9c0     0   666      2 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c03901a4>] (schedule_timeout+0x28/0x31c)
[51293.320000] [<c03901a4>] (schedule_timeout+0x28/0x31c) from [<c038fce4>] (wait_for_common+0xf8/0x1b8)
[51293.320000] [<c038fce4>] (wait_for_common+0xf8/0x1b8) from [<c038fdc4>] (wait_for_completion+0x20/0x24)
[51293.320000] [<c038fdc4>] (wait_for_completion+0x20/0x24) from [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc)
[51293.320000] [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc) from [<c0121ac0>] (__sync_filesystem+0x68/0xa0)
[51293.320000] [<c0121ac0>] (__sync_filesystem+0x68/0xa0) from [<c0121b6c>] (sync_filesystem+0x48/0x6c)
[51293.320000] [<c0121b6c>] (sync_filesystem+0x48/0x6c) from [<c012d148>] (fsync_bdev+0x28/0x4c)
[51293.320000] [<c012d148>] (fsync_bdev+0x28/0x4c) from [<c01c4768>] (invalidate_partition+0x28/0x44)
[51293.320000] [<c01c4768>] (invalidate_partition+0x28/0x44) from [<c014fe38>] (del_gendisk+0x40/0xcc)
[51293.320000] [<c014fe38>] (del_gendisk+0x40/0xcc) from [<bf60aea4>] (sd_remove+0x5c/0x98 [sd_mod])
[51293.320000] [<bf60aea4>] (sd_remove+0x5c/0x98 [sd_mod]) from [<c020ead4>] (__device_release_driver+0x74/0xb4)
[51293.320000] [<c020ead4>] (__device_release_driver+0x74/0xb4) from [<c020eb40>] (device_release_driver+0x2c/0x38)
[51293.320000] [<c020eb40>] (device_release_driver+0x2c/0x38) from [<c020e5f0>] (bus_remove_device+0x94/0xa8)
[51293.320000] [<c020e5f0>] (bus_remove_device+0x94/0xa8) from [<c020b638>] (device_del+0x118/0x168)
[51293.320000] [<c020b638>] (device_del+0x118/0x168) from [<c0223ae4>] (__scsi_remove_device+0x50/0x98)
[51293.320000] [<c0223ae4>] (__scsi_remove_device+0x50/0x98) from [<c02227c0>] (scsi_forget_host+0x7c/0xc4)
[51293.320000] [<c02227c0>] (scsi_forget_host+0x7c/0xc4) from [<c0219a74>] (scsi_remove_host+0xd8/0x194)
[51293.320000] [<c0219a74>] (scsi_remove_host+0xd8/0x194) from [<bf707fc4>] (usb_stor_invoke_transport+0x430/0x528 [usb_storage])
[51293.320000] [<bf707fc4>] (usb_stor_invoke_transport+0x430/0x528 [usb_storage]) from [<bf7080e0>] (usb_stor_disconnect+0x24/0x30 [usb_storage])
[51293.320000] [<bf7080e0>] (usb_stor_disconnect+0x24/0x30 [usb_storage]) from [<bf0f4d54>] (usb_unbind_interface+0x5c/0xf8 [usbcore])
[51293.320000] [<bf0f4d54>] (usb_unbind_interface+0x5c/0xf8 [usbcore]) from [<c020ead4>] (__device_release_driver+0x74/0xb4)
[51293.320000] [<c020ead4>] (__device_release_driver+0x74/0xb4) from [<c020eb40>] (device_release_driver+0x2c/0x38)
[51293.320000] [<c020eb40>] (device_release_driver+0x2c/0x38) from [<c020e5f0>] (bus_remove_device+0x94/0xa8)
[51293.320000] [<c020e5f0>] (bus_remove_device+0x94/0xa8) from [<c020b638>] (device_del+0x118/0x168)
[51293.320000] [<c020b638>] (device_del+0x118/0x168) from [<bf0f3aac>] (usb_disable_device+0x60/0x100 [usbcore])
[51293.320000] [<bf0f3aac>] (usb_disable_device+0x60/0x100 [usbcore]) from [<bf0ecfc8>] (usb_disconnect+0x8c/0x1c0 [usbcore])
[51293.320000] [<bf0ecfc8>] (usb_disconnect+0x8c/0x1c0 [usbcore]) from [<bf0edfb8>] (hub_thread+0x3d8/0xe10 [usbcore])
[51293.320000] [<bf0edfb8>] (hub_thread+0x3d8/0xe10 [usbcore]) from [<c0083df0>] (kthread+0x98/0xa0)
[51293.320000] [<c0083df0>] (kthread+0x98/0xa0) from [<c00419ac>] (kernel_thread_exit+0x0/0x8)
[51293.320000] smbd          D c038f9c0     0  1909   1887 0x00000001
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] flush-8:0     D c038f9c0     0  1910      2 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c03901a4>] (schedule_timeout+0x28/0x31c)
[51293.320000] [<c03901a4>] (schedule_timeout+0x28/0x31c) from [<c038fce4>] (wait_for_common+0xf8/0x1b8)
[51293.320000] [<c038fce4>] (wait_for_common+0xf8/0x1b8) from [<c038fdc4>] (wait_for_completion+0x20/0x24)
[51293.320000] [<c038fdc4>] (wait_for_completion+0x20/0x24) from [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc)
[51293.320000] [<c011cc2c>] (writeback_inodes_sb+0xa8/0xbc) from [<bf1250ac>] (ufsd_get_sb+0xac/0x330 [ufsd])
[51293.320000] [<bf1250ac>] (ufsd_get_sb+0xac/0x330 [ufsd]) from [<bf1252c8>] (ufsd_get_sb+0x2c8/0x330 [ufsd])
[51293.320000] [<bf1252c8>] (ufsd_get_sb+0x2c8/0x330 [ufsd]) from [<bf12678c>] (ufsd_write_inode+0x210/0x288 [ufsd])
[51293.320000] [<bf12678c>] (ufsd_write_inode+0x210/0x288 [ufsd]) from [<c011c7cc>] (writeback_single_inode+0x184/0x258)
[51293.320000] [<c011c7cc>] (writeback_single_inode+0x184/0x258) from [<c011cebc>] (writeback_sb_inodes+0xe0/0x184)
[51293.320000] [<c011cebc>] (writeback_sb_inodes+0xe0/0x184) from [<c011d998>] (writeback_inodes_wb+0x1b4/0x1d0)
[51293.320000] [<c011d998>] (writeback_inodes_wb+0x1b4/0x1d0) from [<c011dc58>] (wb_writeback+0x2a4/0x470)
[51293.320000] [<c011dc58>] (wb_writeback+0x2a4/0x470) from [<c011df28>] (wb_do_writeback+0x104/0x254)
[51293.320000] [<c011df28>] (wb_do_writeback+0x104/0x254) from [<c011e168>] (bdi_writeback_thread+0xf0/0x304)
[51293.320000] [<c011e168>] (bdi_writeback_thread+0xf0/0x304) from [<c0083df0>] (kthread+0x98/0xa0)
[51293.320000] [<c0083df0>] (kthread+0x98/0xa0) from [<c00419ac>] (kernel_thread_exit+0x0/0x8)
[51293.320000] smbd          D c038f9c0     0  3308   1887 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] smbd          D c038f9c0     0  3323   1887 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] smbd          D c038f9c0     0  3334   1887 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] smbd          D c038f9c0     0  3349   1887 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] ls            D c038f9c0     0 10409   1899 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd])
[51293.320000] [<bf12d5f0>] (ufsd_file_open+0xa0/0x25c [ufsd]) from [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340)
[51293.320000] [<c00f96f8>] (__dentry_open.isra.14+0x234/0x340) from [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c)
[51293.320000] [<c00fa5ec>] (nameidata_to_filp+0x88/0x9c) from [<c010869c>] (do_last.isra.41+0x4a0/0x5e8)
[51293.320000] [<c010869c>] (do_last.isra.41+0x4a0/0x5e8) from [<c010895c>] (do_filp_open+0x178/0x4d4)
[51293.320000] [<c010895c>] (do_filp_open+0x178/0x4d4) from [<c00fa66c>] (do_sys_open+0x6c/0x108)
[51293.320000] [<c00fa66c>] (do_sys_open+0x6c/0x108) from [<c00fa738>] (sys_open+0x30/0x34)
[51293.320000] [<c00fa738>] (sys_open+0x30/0x34) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
[51293.320000] ls            D c038f9c0     0 10413   9752 0x00000000
[51293.320000] [<c038f9c0>] (schedule+0x490/0x524) from [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158)
[51293.320000] [<c0390d90>] (__mutex_lock_slowpath+0xd0/0x158) from [<c0390e44>] (mutex_lock+0x2c/0x30)
[51293.320000] [<c0390e44>] (mutex_lock+0x2c/0x30) from [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd])
[51293.320000] [<bf12520c>] (ufsd_get_sb+0x20c/0x330 [ufsd]) from [<bf12de38>] (ufsd_xattr_acl_access_set+0x1d4/0x648 [ufsd])
[51293.320000] [<bf12de38>] (ufsd_xattr_acl_access_set+0x1d4/0x648 [ufsd]) from [<bf12e748>] (ufsd_lookup+0x2c/0x110 [ufsd])
[51293.320000] [<bf12e748>] (ufsd_lookup+0x2c/0x110 [ufsd]) from [<c01054f4>] (d_alloc_and_lookup+0x54/0x70)
[51293.320000] [<c01054f4>] (d_alloc_and_lookup+0x54/0x70) from [<c01055d4>] (do_lookup+0xc4/0x134)
[51293.320000] [<c01055d4>] (do_lookup+0xc4/0x134) from [<c01063e8>] (link_path_walk+0x6ac/0xb08)
[51293.320000] [<c01063e8>] (link_path_walk+0x6ac/0xb08) from [<c0106964>] (path_walk+0x58/0xac)
[51293.320000] [<c0106964>] (path_walk+0x58/0xac) from [<c0107184>] (do_path_lookup+0x34/0x5c)
[51293.320000] [<c0107184>] (do_path_lookup+0x34/0x5c) from [<c0107fb4>] (user_path_at+0x60/0x90)
[51293.320000] [<c0107fb4>] (user_path_at+0x60/0x90) from [<c00ff838>] (vfs_fstatat+0x3c/0x6c)
[51293.320000] [<c00ff838>] (vfs_fstatat+0x3c/0x6c) from [<c00ff8c4>] (vfs_stat+0x2c/0x30)
[51293.320000] [<c00ff8c4>] (vfs_stat+0x2c/0x30) from [<c00ffabc>] (sys_stat64+0x24/0x40)
[51293.320000] [<c00ffabc>] (sys_stat64+0x24/0x40) from [<c0040880>] (ret_fast_syscall+0x0/0x30)
```

发现是卡在mutex_lock处，进一步发现是由于writeback_inodes_sb写入不动，具体栈信息如上。


调试：
    内核函数中添加函数，确定函数执行流程，writeback_inodes_sb后面执行了一些列函数，最总调用卡住在具体fs的write_inode函数，也就是ufsd_write_inode，然后跟踪ufsd驱动定位为Mutex_trylock( &sb->ApiMutex )卡住，也就是mutex死锁，继续跟踪后是在锁住apiMutex后，又调用write_inode再次进入就发生自己调用自己，从而死锁




解决:
去掉  Writeback_inodes_sb( sb );

规避可以使用ntfs-3g

