---
layout: post
title: openwrt procd 分析
category: openwrt
tags: procd
---
# 介绍
procd 是openwrt进程管理守护进程，它可以通过ubus跟踪从init脚本启动的进程，并且通过配置改变来让服务启动或重启。procd已经替换hotplug2，init，watchdog
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

# watch  机制
## 代码分析
watch_ubus  主要创建watch_event和watch_subscribe，event用于监听object创建事件，subscribe用于procd订阅创建的object。调用流程如下  
创建event监听(procd):  
```
watch_ubus->ubus_register_event_handler(ctx, &watch_event, "ubus.object.add")->watch_subscribe_cb
```
创建object:
```
ubus_add_object(client)->(ubusd)ubusd_proto_receive_message(UBUS_MSG_ADD_OBJECT)->ubusd_handle_add_object->ubusd_create_object->ubusd_send_obj_event(触发刚刚注册函数watch_subscribe_cb)->ubus_subscribe(procd订阅新创建的object).
```
如果对应的object发送notify，procd收到notify消息执行trigger_event

## 示例
在network启动脚本中添加watch network.interface
```
start_service() {           
        init_switch 
                   
        procd_open_instance
        procd_set_param command /sbin/netifd
        procd_set_param respawn             
        procd_set_param watch network.interface
        [ -e /proc/sys/kernel/core_pattern ] && {
                procd_set_param limits core="unlimited"
                echo '/tmp/%e.%p.%s.%t.core' > /proc/sys/kernel/core_pattern
        }                                                                   
        procd_close_instance                                                
}
```
首先在系统启动时，procd中有一个watch_objects的链表，启动network服务时，会将network.interface 调用watch_add添加到链表watch_objects中。

```
static struct ubus_object iface_object = {
	.name = "network.interface",
	.type = &iface_object_type,
	.n_methods = ARRAY_SIZE(iface_object_methods),
};
netifd_add_object(&iface_object);  //netifd的源码ubus.c文件中
```
创建network.interface的ubus object时会通过上面的创建object流程，ubusd_send_obj_event发送“ubus.object.add”事件，触发watch_subscribe_cb，然后watch_subscribe_cb中遍历watch_objects，发现具有该对象，然后就订阅该对象ubus_subscribe，并且设置消息处理函数watch_notify_cb。
```
list_for_each_entry(o, &watch_objects, list) {
		unsigned int id;

		if (strcmp(o->name, path))
			continue;
		if (ubus_lookup_id(ctx, path, &id))
			continue;
		if (!ubus_subscribe(ctx, &watch_subscribe, id))
			return;
		ERROR("failed to subscribe %d\n", id);
	}
```
# trigger 事件触发机制
## 代码分析
trigger主要是通过libjson_script.so实现的事件机制。在procd全局list  triggers保存事件。在各个子进程的service或者instance中添加triggers，调用trigger_add，将事件类型type和需要执行的动作（一个array开头的json复合结构，其书写格式由libjson_script.so定义，并且json_script_ctx结构体在procd中填充）通过trigger_add添加到triggers链表。当事件触发时，调用trigger_event遍历trigger，匹配到type后调用 json_script_run 执行相应动作
## json_script执行
json_script_init 注册处理函数
```

static struct trigger* _trigger_add(char *type, struct blob_attr *rule, int timeout, void *id)
{
	char *_t;
	struct blob_attr *_r;
	struct trigger *t = calloc_a(sizeof(*t), &_t, strlen(type) + 1, &_r, blob_pad_len(rule));//一次分配，多次指针赋值

	t->type = _t;
	t->rule = _r;
	t->delay.cb = trigger_delay_cb;
	t->timeout = timeout;
	t->pending = 0;
	t->remove = 0;
	t->id = id;
	t->jctx.handle_var = rule_handle_var,
	t->jctx.handle_error = rule_handle_error,
	t->jctx.handle_command = rule_handle_command,
	t->jctx.handle_file = rule_load_script,  //

	strcpy(t->type, type);
	memcpy(t->rule, rule, blob_pad_len(rule));

	list_add(&t->list, &triggers);
	json_script_init(&t->jctx); //初始化和赋值

	return t;
}

```
trigger_event 中调用json_script_run(&t->jctx, "foo", data);然后挨个解析data中的数据，执行预定义脚本

json_script格式与/etc/hotplug.json的格式一致，主要有如下几种方法

```
static const struct json_handler cmd[] = {
	{ "if", handle_if },
	{ "case", handle_case },
	{ "return", handle_return },
	{ "include", handle_include },
};

```
```
static const struct json_handler expr[] = {
	{ "eq", handle_expr_eq },
	{ "regex", handle_expr_regex },
	{ "has", handle_expr_has },
	{ "and", handle_expr_and },
	{ "or", handle_expr_or },
	{ "not", handle_expr_not },
};

```
上面的方法可以层层包含，可以一直叠加，执行时递归调用，如果没有上面的方法则调用注册的rule_handle_command方法。
trigger只是rule_handle_command中只定义了run_script一种方法。其格式为
```
["if",["eq","package","network"],["run_script","\\/etc\\/init.d\\/network","reload"]]
```

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
  #procd_set_param nice -10  # -19~19 
  #procd_set_param triggers array  #
  #procd_setparam error  array #?
  #procd_set_param watch ubusobject  # procd service will subscribe ubusobject which  is created by this instance
  #procd_set_param limits core="unlimited"  # If you need to set ulimit for your process
  #procd_set_param file /var/etc/your_service.conf # /etc/init.d/your_service reload will restart the daemon if these files have changed
  #procd_set_param netdev dev # likewise, except if dev's ifindex changes.
  #procd_set_param data name=value ... # likewise, except if this data changes.
  #procd_set_param stdout 1 # forward stdout of the command to logd
  #procd_set_param stderr 1 # same for stderr
  #procd_set_param user nobody # run service as user nobody  procd_close_instance
}
```
3. stop_service 如果在procd结束进程后不需要清理自定义的配置信息等，可以不用写

4. service_triggers 配置改变时重启服务，

```
service_triggers()
{
        procd_add_reload_trigger "uci-file-name"
}
```
就是相当于添加config.change 事件，等同于命令
```
ubus call service set '{ "name": "service", "triggers": [ [ "config.change", [ "if", [ "eq", "package", "uci-file-name" ], [ "run_script", "\/etc\/init.d\/service", "reload" ] ] ] ]}'
``` 
当修改uci-file-name后，其md5值变化，然后调用/sbin/reload_config 发现变化，触发事件config.change
```
[ -f $MD5FILE ] && {
        for c in `md5sum -c $MD5FILE 2>/dev/null| grep FAILED | cut -d: -f1`; do
                ubus call service event "{ \"type\": \"config.change\", \"data\": { \"package\": \"$(basename $c)\" }}"
        done
}
```

如果需要不依赖配置重启，可以定义reload_service
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
# 数据合法性校验
## validate_data
数据校验主要通过validate_data实现，其有三种处理
1. 参数只有三个 validate_data <datatype> <value> 返回数据是否符合true or false
```
validate_data   uinteger   "aa"
uinteger - aa = false
validate_data   uinteger   "11"
uinteger - 11 = true
```
2. validate_data <package> <section_type> <section_name> 'option:datatype:default' 'option:datatype:default' ...  
当section_name为空"\0"时,将package 等通过调用json，添加进json对象
```
validate_data  firewall rule "" 'proto:or(uinteger, string)' 'src:string' 'dest:string' 'src_port:or(port, portrange)'  'dest_port:or(port, portrange)'  'target:string'

json_add_object; json_add_string "package" "firewall"; json_add_string "type" "rule"; json_add_object "data"; json_add_string "proto" "or(uinteger, string)"; json_add_string "src" "string"; json_add_string "dest" "string"; json_add_string "src_port" "or(port, portrange)"; json_add_string "dest_port" "or(port, portrange)"; json_add_string "target" "string"; json_close_object; json_close_object; 
```

3. validate_data <package> <section_type> <section_name> 'option:datatype:default' 'option:datatype:default' ...  
当section_name为package对应的配置项时，由后面的option字段挨个匹配对应字段，校验其合法性

4. 校验类型
```
static struct dt_fun dt_types[] = {
	{ "or",			DT_INVALID,	dt_type_or		},
	{ "and",		DT_INVALID,	dt_type_and		},
	{ "not",		DT_INVALID,	dt_type_not		},
	{ "neg",		DT_INVALID,	dt_type_neg		},
	{ "list",		DT_INVALID,	dt_type_list		},
	{ "min",		DT_NUMBER,	dt_type_min		},
	{ "max",		DT_NUMBER,	dt_type_max		},
	{ "range",		DT_NUMBER,	dt_type_range		},
	{ "minlength",		DT_STRING,	dt_type_minlen		},
	{ "maxlength",		DT_STRING,	dt_type_maxlen		},
	{ "rangelength",	DT_STRING,	dt_type_rangelen	},
	{ "integer",		DT_NUMBER,	dt_type_int		},
	{ "uinteger",		DT_NUMBER,	dt_type_uint		},
	{ "float",		DT_NUMBER,	dt_type_float		},
	{ "ufloat",		DT_NUMBER,	dt_type_ufloat		},
	{ "bool",		DT_BOOL,	dt_type_bool		},
	{ "string",		DT_STRING,	dt_type_string		},
	{ "hexstring",		DT_STRING,	dt_type_hexstring	},
	{ "ip4addr",		DT_STRING,	dt_type_ip4addr		},
	{ "ip6addr",		DT_STRING,	dt_type_ip6addr		},
	{ "ipaddr",		DT_STRING,	dt_type_ipaddr		},
	{ "cidr4",		DT_STRING,	dt_type_cidr4		},
	{ "cidr6",		DT_STRING,	dt_type_cidr6		},
	{ "cidr",		DT_STRING,	dt_type_cidr		},
	{ "netmask4",		DT_STRING,	dt_type_netmask4	},
	{ "netmask6",		DT_STRING,	dt_type_netmask6	},
	{ "ipmask4",		DT_STRING,	dt_type_ipmask4		},
	{ "ipmask6",		DT_STRING,	dt_type_ipmask6		},
	{ "ipmask",		DT_STRING,	dt_type_ipmask		},
	{ "port",		DT_NUMBER,	dt_type_port		},
	{ "portrange",		DT_STRING,	dt_type_portrange	},
	{ "macaddr",		DT_STRING,	dt_type_macaddr		},
	{ "uciname",		DT_STRING,	dt_type_uciname		},
	{ "wpakey",		DT_STRING,	dt_type_wpakey		},
	{ "wepkey",		DT_STRING,	dt_type_wepkey		},
	{ "hostname",		DT_STRING,	dt_type_hostname	},
	{ "host",		DT_STRING,	dt_type_host		},
	{ "network",		DT_STRING,	dt_type_network		},
	{ "phonedigit",		DT_STRING,	dt_type_phonedigit	},
	{ "directory",		DT_STRING,	dt_type_directory	},
	{ "device",		DT_STRING,	dt_type_device		},
	{ "file",		DT_STRING,	dt_type_file		},
	{ "regex",		DT_STRING,	dt_type_regex		},
	{ "uci",		DT_STRING,	dt_type_uci		},

	{ }
};

```
# procd hotplug功能
procd通过/etc/hotplug.d/下的脚本文件来实现热插拔事件。作用等同于以前的hotplug2进程
## 代码分析
procd启动时，执行hotplug("/etc/hotplug.json"),监听内核NETLINK_KOBJECT_UEVENT事件，初始化jctx,并注册netlink处理函数hotplug_handler,
```
static struct json_script_ctx jctx = {
	.handle_var = rule_handle_var,
	.handle_error = rule_handle_error,
	.handle_command = rule_handle_command,
	.handle_file = rule_handle_file,
};
```
当收到NETLINK_KOBJECT_UEVENT消息时，解析信息填入blobmsg，并执行json_script_run(&jctx, rule_file, blob_data(b.head));
rule_handle_command中注册了makedev，rm，exec，button，load-fireware方法。具体json解析格式如/etc/hotplug.json

## 事件消息通过hotplug实现
1. 在procd中添加ubus接口，用于接收用户层应用发送event的hotplug事件。
2. hotplug json格式中添加事件的特殊处理（在/etc/hotplug.json添加如下json_script解析格式）
```
        [ "if",
                [ "has", "event" ],
                [ "exec", "/sbin/hotplug-call", "%event%" ]
        ],
```
3. 添加shell脚本事件创建和事件发送函数，用于其他应用调用。
4. 事件使用
注册 event_subscribe event  scriptfullpath params
注销 event_unsubscribe event  script
事件发送 event_post event  key1=value1  key2=value2 ... 
也可以直接调用ubus接口 发送事件
```
 ubus call service  hotplug '{ "data": "Table" }'
```
 data 中为Table格式,如下
```
 ubus -t 3 call service hotplug "{'data':{\"event\":\"$event\",\"IP\":\"$ipaddr\", \"MASK\":\"$mask\", \"LINK\":\"up\"}}" 
 
```
[https://openwrt.org/docs/guide-user/base-system/hotplug_lede?s[]=procd&s[]=hotplug](https://openwrt.org/docs/guide-user/base-system/hotplug_lede?s[]=procd&s[]=hotplug)

## hotplug button按键处理
需要内核模块支持，后续分析

# procd system info
主要通过ubus接口提供系统信息获取接口

```c
    UBUS_METHOD_NOARG("board", system_board),
	UBUS_METHOD_NOARG("info",  system_info),
	UBUS_METHOD_NOARG("upgrade", system_upgrade),
	UBUS_METHOD_NOARG("reboot", system_reboot),
	UBUS_METHOD_NOARG("reset", system_reset),
	UBUS_METHOD("watchdog", watchdog_set, watchdog_policy),
	UBUS_METHOD("signal", proc_signal, signal_policy),
```


