---
layout:post
title:openwrt procd系统启动分析
category:openwrt
tags:procd
---
# 介绍
procd 是openwrt进程管理守护进程，它可以通过ubus跟踪从init脚本启动的进程，并且通过配置改变来让服务启动或重启。procd已经替换hotplug2，init，klogd，syslogd，watchdog
# linux启动流程分析
[内核启动流程](https://blog.csdn.net/viewsky11/article/details/53040564)主要启动入口start_kernel()(init/main.c),函数最后调用rest_init，创建两个内核线程init和kthreadd，kthreadd用于创建内核进程使用，一般处于idle状态，kthreadd进程创建好之后，init进程开始执行kernel_init，通过判断系统启动参数行中是否有init来执行init用户空间进程。

```
console=ttyS0,115200 root=/dev/mtdblock4 init=/etc/preinit init earlyprintk debug
```
参数有则作为init进程，没有则按照下面顺序执行

```
	if (!run_init_process("/sbin/init") ||
	    !run_init_process("/etc/init") ||
	    !run_init_process("/bin/init") ||
	    !run_init_process("/bin/sh"))
		return 0;
```
所以以/etc/preinit作为init进程。
# init启动流程
preinit为shell脚本
主要有五个链表，如下
```
boot_hook_init preinit_essential
boot_hook_init preinit_main
boot_hook_init failsafe
boot_hook_init initramfs
boot_hook_init preinit_mount_root
```
然后在/lib/preinit/目录下依次执行每一个文件，通过boot_hook_add将函数加入到相应链表，然后通过boot_run_hook执行指定链表上的函数，从而实现文件系统挂载，系统初始化等，主要有下面函数

```
preinit_essential_hook=do_print_mount2 do_mount_procfs do_mount_sysfs do_mount_tmpfs choose_device_fs init_device_fs init_shm init_devpts do_mount_devpts choose_console init_console

failsafe_hook=indicate_failsafe failsafe_netlogin failsafe_shell

preinit_mount_root_hook=check_for_mtd check_for_jffs2 do_mount_jffs2 merge_overlay_hooks rootfs_pivot change_passwd do_mount_no_jffs2 check_web_passwd check_ate_mode

preinit_main_hook=define_default_set_state preinit_ip pi_indicate_preinit failsafe_wait run_failsafe_hook indicate_regular_preinit init_hotplug initramfs_test do_mount_root restore_config check_nvram_mac run_init

```

最后执行run_init，调用/sbin/init（原来为busybox中init）init读取配置inittab，并执行sysinit也就是/etc/init.d/rcS,rcS遍历执行/etc/rc.d/*下面的文件，启动所有程序
# procd启动流程
procd启动流程与init流程类似。  
内核启动/etc/preinit,在文件开头通过PREINIT环境变量调用init进程（procd生成）
```
[ -z "$PREINIT" ] && exec /sbin/init
```
### init
- 调用early挂载proc，sysfs，dev，dev/pts文件系统，
- watchdog_init初始化watchdog，
- fork子进程执行/sbin/kmodloader /etc/modules-boot.d/下内核模块，
- 父进程再创建子进程“/sbin/procd -h /etc/hotplug-preinit.json”监听hostplug
- 设置$PREINIT为1，并创建子进程执行/etc/preinit
- preinit执行完后会调spawn_procd函数，函数创建进程执行/sbin/procd,为最后ps显示的pid=1的init进程。

### procd
procd有如下状态，通过调用procd_state_next按照顺序变化。
```
enum {
	STATE_NONE = 0,
	STATE_EARLY,
	STATE_UBUS,
	STATE_INIT,
	STATE_RUNNING,
	STATE_SHUTDOWN,
	STATE_HALT,
	__STATE_MAX,
};
```
STATE_EARLY状态：
- watchdog_init
- 根据"/etc/hotplug.json"规则监听hotplug
- procd_coldplug()函数处理，把/dev挂载到tmpfs中，fork udevtrigger进程产生冷插拔事件，以便让hotplug监听进行处理
- udevtrigger调用完成后调用procd_state_next将状态从STATE_EARLY变为STATE_UBUS

STATE_UBUS状态：
- set_stdio("console");/dev/console重定向标准输入输出和错误输出
- procd_connect_ubus 连接ubus，此时ubusd不存在，通过定时器重连，
- service_init初始化两个avl_tree
- service_start_early启动ubusd进程。调用过程：service_alloc中分配services服务并注册回调函数service_instance_update，service_update->service_instance_add,instance_init初始化各种字段（/sbin/ubusd）,tree->update(tree, node, old_node);调用刚刚服务注册的service_instance_update->instance_start->instance_run启动服务
- 启动ubusd后procd_connect_ubus 函数就可以连接上ubusd，然后注册main_object对象，system_object对象、watch_event对象。main_object主要管理通过procd启动的进程，system_object主要获取系统信息如内存、开发板信息等
- 最后调用procd_state_ubus_connect进入下一个状态STATE_INIT

STATE_INIT状态
- procd_inittab将/etc/inittab内容把cmd、handler对应关系加入全局链表actions中
- 顺序执行respawn、askconsole、askfirst、sysinit命令对应的函数（handlers）
- sysinit对应执行/etc/init.d/rcS脚本，rcS执行/etc/rc.d/*下面的shell脚本文件，从而启动所有进程
- 完成后调用rcdone将状态变更为STATE_RUNNING。

STATE_RUNNING状态:
- 进入该状态后，退出state_enter，然后返回主程序执行uloop_run，系统启动完成

# procd 初始化脚本
1. procd启动服务运行在前台，为了兼容需要明确指定USE_PROCD=1
2. 开启一个服务需要start_service

```
start_service() {
  procd_open_instance [instance_name]
  procd_set_param command /sbin/your_service_daemon -b -a --foo
  procd_append_param command -bar 42 # append command parameters

  # respawn automatically if something died, be careful if you have an alternative process supervisor
  # if process dies sooner than respawn_threshold, it is considered crashed and after 5 retries the service is stopped
  procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}

  procd_set_param env SOME_VARIABLE=funtimes  # pass environment variables to your process
  procd_set_param limits core="unlimited"  # If you need to set ulimit for your process
  procd_set_param file /var/etc/your_service.conf # /etc/init.d/your_service reload will restart the daemon if these files have changed
  procd_set_param netdev dev # likewise, except if dev's ifindex changes.
  procd_set_param data name=value ... # likewise, except if this data changes.
  procd_set_param stdout 1 # forward stdout of the command to logd
  procd_set_param stderr 1 # same for stderr
  procd_set_param user nobody # run service as user nobody
  procd_set_param pidfile /var/run/somefile.pid # write a pid file on instance start and remote it on stop
  procd_close_instance
}
```
3. stop_service 如果在procd结束进程后不需要清理自定义的配置信息等，可以不用写
4. service_triggers 配置改变时重启服务，如果需要不依赖配置重启，可以定义reload_service

```
service_triggers()
{
        procd_add_reload_trigger "uci-file-name"
}
```
```
reload_service()
{
        echo "Explicitly restarting service, are you sure you need this?"
        stop
        start
}
```
具体参见
[https://openwrt.org/docs/guide-developer/procd-init-scripts](https://openwrt.org/docs/guide-developer/procd-init-scripts)

[http://wiki.prplfoundation.org/wiki/Procd_reference](http://wiki.prplfoundation.org/wiki/Procd_reference)

# procd hotplug功能
procd通过/etc/hotplug.d/下的脚本文件来实现热插拔事件。作用等同于以前的hotplug2进程

[https://openwrt.org/docs/guide-user/base-system/hotplug_lede?s[]=procd&s[]=hotplug](https://openwrt.org/docs/guide-user/base-system/hotplug_lede?s[]=procd&s[]=hotplug)

