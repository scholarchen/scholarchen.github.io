---
layout: post
title: usb hub插入分析
category: usb
tags: usb,xhci,usb3.0,hub
---
在root_hub注册的时候，添加了一个 status = usb_submit_urb(hub->urb, GFP_NOIO);
然后一直调用 rh_timer_func 查询端口状态，当端口状态改变时，调用urb->irq，就是上面的hub_irq
hub_irq里面调用 kick_khubd(hub);通知khubd处理，如果出错就重新发送urb
kick_khubd中将list_add_tail(&hub->event_list, &hub_event_list);
khubd线程中一直有一个hub_thread循环检测hub_event_list，当有事物时，唤醒进程，进入hub_events
hub_events
    //循环遍历hub_event_list
      hub_port_status
    然后对获取的状态进行一系列复杂的判断，主要有端口物理变化，端口本身重新使能，还有复位设备描述符变化了，如果状态有改变调用处理函数
      hub_port_connect_change
    使能的清除标志什么也不做
    如果udev存在，说明是由有设备变无设备，直接断开然后再判断下状态就结束了
    状态如果从无到有，那么就需要去抖动然后在判断状态	
                status = hub_port_debounce(hub, port1);//这个函数就是100ms检测状态一直连接着的