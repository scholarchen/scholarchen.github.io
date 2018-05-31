---
layout: post
title: linux文件随机被修改前几个字节bug分析
category: debug
tags: ram，cache
---

## 问题现象：
mips开发板，正常使用，小概率出现系统启动后，只读rootfs文件系统中随机的文件头被修改，修改的值固定为0b 41开头，导致对应文件无法正常识别和使用。
## 重现步骤：
系统不断重启，问题就会出现，详细触发条件未找到。为了使问题必现，写脚本实现系统检测文件系统是否更改，如果没有更改则重启系统，如果更改了则打印并等待分析！
因好几次出现文件头都被更改三个字节，前两个字节为“0b41”,以此为判断依据写脚本，遍历rom文件夹。
```
#!/bin/sh

check_hex()
{

        files=`ls $1`

        for f in $files; do
                if [ -d $1/$f ];then
                        check_hex $1/$f
                else
                        elf=`hexdump -C -n 32 $1/$f | grep "0b 41"`
                        if [ "$elf" != "" ];then
                                echo [WARN] $1/$f Has parttens > /tmp/hexlogs
                                exit 1
                        fi
                fi
        done

}

while true
do
        runtime=`cat /proc/uptime | awk -F "." '{print $1}'`
        if [ $runtime -gt 90 ]
        then
                check_hex /rom
                echo -e "\033[;32m[DEBUG] All is well , reboot\033[0m"
                reboot 
        fi
        sleep 10
done
```

## 问题分析：
### 1. 分析系统
文件系统挂载信息如下

```
rootfs on / type rootfs (rw)
/dev/root on /rom type squashfs (ro,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,mode=0755)
proc on /proc type proc (rw,noatime)
sysfs on /sys type sysfs (rw,noatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noatime,size=57933824,mode=1777)
devpts on /dev/pts type devpts (rw,noatime,mode=600)
/dev/mtdblock5 on /overlay type jffs2 (rw,noatime)
overlayfs:/overlay on / type overlayfs (rw,noatime,lowerdir=/,upperdir=/overlay)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
```
发生错误时，被修改文件位于rom下的某一个文件，同步的rom外的文件也被修改，也就是overlay上的对应文件。因每次出问题后，重启就会恢复，所以首先排除文件修改后写入flash文件的可能。
### 2. 假设验证
#### 2.1 假设1  系统启动时，squashfs文件系统解压错误。
#### 2.2 假设2 在系统启动过程中因文件系统比较多，挂载过程中，某一时刻将squashfs文件系统变为了读写，然后被改变。
分析系统启动流程，发现squashfs挂载是在内核启动阶段挂载根文件系统时挂载的，其解压过程较复杂，先观察待定。继续分析假设2，挂载中主要有
挂载devtmpfs/proc/sysfs/tmpfs/devpts/jffs2/overlayfs/debugfs.这个过程中主要操作了jffs2和overlayfs。主要代码如下

```
 mount "$(find_mtd_part rootfs_data)" /tmp/overlay -t jffs2
 mount -o move /tmp/overlay /overlay 
fopivot /overlay /rom 
```
fopivot定义如下（代码已做删减）

```
pivot() { # <new_root> <old_root>
        mount -o move /proc $1/proc && \
        pivot_root $1 $1$2 && {
                mount -o move $2/dev /dev
                mount -o move $2/tmp /tmp
                mount -o move $2/sys /sys 2>&-
                mount -o move $2/overlay /overlay 2>&-
                return 0
        }
}

fopivot() { # <rw_root> <ro_root> <dupe?>
        root=$1
        {
                mount -t overlayfs -olowerdir=/rom,upperdir=$1 "overlayfs:$1" /mnt && root=/mnt
        } 
        pivot $root $2
}

```
怀疑mount -o move 或者 overlayfs有漏洞导致文件被变更。这些文件系统挂载，都是在init进程之前，经过同事提醒，曾经出现过lib库文件出错的情况，所以前面两个假设都不成立（lib库出错，但是对应的程序已经正常运行）。
#### 2.3 假设3
系统起来后，某个操作修改了文件，或者什么程序覆盖了文件，至于为什么更改只读，可能是某个文件系统有问题（squashfs、jffs2、overlayfs三者之一）。

1. 验证一、将正常系统的所有分区导出，然后将系统出问题的所有分区内容导出，主要查看squashfs分区，经过对比正常和出错都是一样的。那就只能是同样的内容，文件系统表现出不一样。

2. 验证二、
将文件系统squashfs卸载，并在不重启情况下在相同和不同文件夹挂载，发现错误文件和内容一样
```
umount /rom
mount -t squashfs /dev/root /rom/
```
加大对overlayfs怀疑，但是因其已经被变更为根文件系统，无法卸载。所以调整根文件系统。通过前面fopivot的分析，知道通过pivot_root调整根文件系统。原来由/mnt变成了根文件系统，/mnt/rom为根文件系统备份

```
pivot_root /rom/ /rom/mnt/
mount -o /mnt/proc  /proc
```
其中需要通过"mount -o move "修改proc/sys/dev等文件系统的挂载目录 ，让系统显示正常。
经过验证后发现系统全部将文件系统卸载后，文件错误任然存在，并且重新挂载的文件任然有错。

#### 2.4 假设4
既然和文件系统无关，并且重启就恢复，灵机一动，就考虑到是内存影响。
通过free查看，发现buffer确实很多内容，通过命令手动释放缓存
```
echo 1 > /proc/sys/vm/drop_caches
```
释放后，文件修复。可以确认是缓存文件被修改，导致每次查询该文件时，inode没有改变，每次命中缓存中的文件，所以不管怎么挂载，出错后都是一个错误文件
### 3. 分析缓存修改问题
#### 3.1 linux文件缓存分析
    文件-》inode-》dentry-》address_map-》物理内存
#### 3.2 内存缓存分析工具及源码
    vmtouch ：管理和控制文件系统缓存的工具（主要通过mmap、posix_fadvise、mincore系统调用实现），主要判断文件或目录哪些内容在内存中，将文件内容存入或调出内存，将文件锁定在内存
    pagemap ：通过/proc/pid/pagemap查找指定进程虚拟地址对应的物理地址
    hexdump ：读取/dev/mem内容，查看实际物理内存的内容
#### 3.3 错误猜想
    怀疑是由某一个进程修改或者映射的内存超过了其范围，修改了下一个页的数据。通过出错文件定位出其对应的物理内存地址，然后将对应地址前后内容输出。发现前面内容已被清空，需要调节减少内存回收，找到错误文件前面页的数据来源。
    
    通过vmtouch获取错误文件对应进程的虚拟地址（打印或者通过proc/pid/maps查看），然后通过pagemap工具查找虚拟地址对应的物理地址，然后用hexdump将对应物理地址附近的内存信息全部dump出来，如下
    
    发现其前面已经被初始化为0了，所以无法查看是否前面写入页超出范围导致被覆盖。可能前面的页已经被释放并重新使用了。
#### 3.4 通过内存检测工具测试
    SLUB debug调试，尝试获取信息
    
```
 BUG bio-0 (Tainted: G           O): Poison overwritten
[  215.520000] -----------------------------------------------------------------------------
[  215.520000] 
[  215.520000] Disabling lock debugging due to kernel taint
[  215.520000] INFO: 0x81300000-0x81300002. First byte 0xb instead of 0x6b
[  215.520000] INFO: Allocated in mempool_alloc+0x54/0x164 age=15 cpu=0 pid=2990
[  215.520000]  __slab_alloc.constprop.59+0x46c/0x4e8
[  215.520000]  kmem_cache_alloc+0x12c/0x134
[  215.520000]  mempool_alloc+0x54/0x164
[  215.520000]  bio_alloc_bioset+0x6c/0x1d4
[  215.520000]  _submit_bh+0xa0/0x2e4
[  215.520000]  ll_rw_block+0xa8/0x134
[  215.520000]  squashfs_read_data+0x1f4/0x710
[  215.520000]  squashfs_cache_get+0x1f8/0x3a4
[  215.520000]  squashfs_readpage+0x7d4/0x8f8
[  215.520000]  __do_page_cache_readahead+0x27c/0x290
[  215.520000]  ra_submit+0x28/0x34
[  215.520000]  filemap_fault+0x380/0x470
[  215.520000]  __do_fault+0x88/0x5ac
[  215.520000]  handle_pte_fault+0xc4/0x1008
[  215.520000]  handle_mm_fault+0x9c/0xe0
[  215.520000]  do_page_fault+0x124/0x430
[  215.520000] INFO: Freed in blk_update_request+0x108/0x44c age=4 cpu=0 pid=790
[  215.520000]  __slab_free+0x40/0x2d8
[  215.520000]  kmem_cache_free+0x18c/0x1ac
[  215.520000]  blk_update_request+0x108/0x44c
[  215.520000]  blk_update_bidi_request+0x24/0xac
[  215.520000]  __blk_end_bidi_request+0x1c/0x4c
[  215.520000]  mtd_blktrans_thread+0x290/0x528
[  215.520000]  kthread+0xb8/0xc0
[  215.520000]  ret_from_kernel_thread+0x10/0x18
[  215.520000] INFO: Slab 0x8115d800 objects=51 used=51 fp=0x  (null) flags=0x0080
[  215.520000] INFO: Object 0x81300000 @offset=0 fp=0x81300280
[  215.520000] 
[  215.520000] Object 81300000: 0b 41 2e 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  .A.kkkkkkkkkkkkk
[  215.520000] Object 81300010: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300020: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300030: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300040: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300050: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300060: 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b 6b  kkkkkkkkkkkkkkkk
[  215.520000] Object 81300070: 6b 6b 6b 6b 6b 6b 6b a5                          kkkkkkk.
[  215.520000] Redzone 81300078: bb bb bb bb                                      ....
[  215.520000] Padding 81300120: 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a  ZZZZZZZZZZZZZZZZ
[  215.520000] Padding 81300130: 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a 5a  ZZZZZZZZZZZZZZZZ
[  215.520000] CPU: 0 PID: 3427 Comm: rtlwlconf Tainted: G    B      O 3.10.90 #3
[  215.520000] Stack : 00000000 00000000 806a9b6a 00000042 871b8300 81300000 804d0978 806a8ae4
          869f41b0 8053b367 871b8300 81300000 8115d800 00101b46 00100100 8045fff8
          00000001 8001fe04 81300130 00000000 804d2194 86c678cc 86c678cc 804d0978
          00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
          00000000 00000000 00000000 00000000 00000000 00000000 00000000 86c67860
          ...
[  215.520000] Call Trace:
[  215.520000] [<80007664>] show_stack+0x64/0x7c
[  215.520000] [<80090b58>] check_bytes_and_report+0x104/0x13c
[  215.520000] [<80090dd8>] check_object+0x248/0x294
[  215.520000] [<80460dc8>] alloc_debug_processing+0xa4/0x174
[  215.520000] [<804618e4>] __slab_alloc.constprop.59+0x46c/0x4e8
[  215.520000] [<80092ab0>] kmem_cache_alloc+0x12c/0x134
[  215.520000] [<80069f84>] mempool_alloc+0x54/0x164
[  215.520000] [<800c9390>] bio_alloc_bioset+0x6c/0x1d4
[  215.520000] [<800c25e4>] _submit_bh+0xa0/0x2e4
[  215.520000] [<800c3370>] ll_rw_block+0xa8/0x134
[  215.520000] [<800f6124>] squashfs_read_data+0x1f4/0x710
[  215.520000] [<800f6838>] squashfs_cache_get+0x1f8/0x3a4
[  215.520000] [<800f7a78>] squashfs_readpage+0xd0/0x8f8
[  215.520000] [<80071e0c>] __do_page_cache_readahead+0x27c/0x290
[  215.520000] [<8007215c>] ra_submit+0x28/0x34
[  215.520000] [<80069b00>] filemap_fault+0x380/0x470
[  215.520000] [<8007db5c>] __do_fault+0x88/0x5ac
[  215.520000] [<80080b3c>] handle_pte_fault+0xc4/0x1008
[  215.520000] [<80081b1c>] handle_mm_fault+0x9c/0xe0
[  215.520000] [<8000cf04>] do_page_fault+0x124/0x430
[  215.520000] [<800030c0>] ret_from_exception+0x0/0xc
[  215.520000] 
[  215.520000] FIX bio-0: Restoring 0x81300000-0x81300002=0x6b
[  215.520000] 
[  215.520000] FIX bio-0: Marking all objects used
[  225.480000] mm/memory.c:397: bad pgd 862e410b.
[  225.490000] ------------[ cut here ]------------
[  225.490000] WARNING: at mm/mmap.c:2760 exit_mmap+0x178/0x180()
[  225.500000] Modules linked in: fuse usb_storage xt_set(O) ip_set_list_set(O) ip_set_hash_netport(O) ip_set_hash_netiface(O) ip_set_hash_net(O) ip_set_hash_ipportnet(O) ip_set_hash_ipportip(O) ip_set_hash_ipport(O) ip_set_hash_ip(O) ip_set_bitmap_port(O) ip_set_bitmap_ipmac(O) ip_set_bitmap_ip(O) ip_set(O) l2tp_ppp sd_mod xt_multiport ppp_mppe hfsplus ext4 jbd2 mbcache exfat(O) nls_utf8 scsi_mod crc16 pre_dns(O) qos(O) ai(O) ecb mac_filter(O) portal_filter(O) app_filter(O) url_filter(O) bm(O)
[  225.540000] CPU: 0 PID: 6987 Comm: Plugin Tainted: G    B      O 3.10.90 #3
[  225.550000] Stack : 00000000 00000000 806a9b6a 0000003f 804d89b4 80530000 804d0978 806a8ae4
          86d60b80 8053b367 804d89b4 80530000 86eb7e68 80660000 804d46ac 8045fff8
          8053ff74 8001fe04 00000003 00000000 804d2194 86eb7c7c 86eb7c7c 804d0978
          00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
          00000000 00000000 00000000 00000000 00000000 00000000 00000000 86eb7c10
          ...
[  225.580000] Call Trace:
[  225.590000] [<80007664>] show_stack+0x64/0x7c
[  225.590000] [<8001ffc8>] warn_slowpath_common+0x78/0xa8
[  225.590000] [<80020010>] warn_slowpath_null+0x18/0x24
[  225.600000] [<80086fe0>] exit_mmap+0x178/0x180
[  225.600000] [<8001dc6c>] mmput+0x48/0xd4
[  225.610000] [<80025858>] do_exit+0x1f0/0x7d8
[  225.610000] [<80025fb0>] do_group_exit+0x48/0xa8
[  225.620000] [<800316c0>] get_signal_to_deliver+0x1cc/0x500
[  225.620000] [<80005dc0>] do_signal+0x28/0x248
[  225.630000] [<800069c8>] do_notify_resume+0x60/0x78
[  225.630000] [<80003240>] work_notifysig+0xc/0x14
[  225.640000] 
[  225.640000] ---[ end trace 41740236f90d62bb ]---
[  225.640000] BUG: Bad rss-counter state mm:86496860 idx:1 val:5

[  321.370000] SQUASHFS error: xz_dec_run error, data probably corrupt
[  321.380000] SQUASHFS error: squashfs_read_data failed to read block 0x7c612
[  321.380000] SQUASHFS error: Unable to read data cache entry [7c612]
[  321.390000] SQUASHFS error: Unable to read page, block 7c612, size be50
[  321.400000] SQUASHFS error: Unable to read data cache entry [7c612]
[  321.400000] SQUASHFS error: Unable to read page, block 7c612, size be50
[  321.410000] SQUASHFS error: Unable to read data cache entry [7c612]
[  321.420000] SQUASHFS error: Unable to read page, block 7c612, size be50
```

    
