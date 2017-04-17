---
layout: post
title: xhci驱动分析
category: usb
tags: usb,xhci,usb3.0,hub
---

最开始每个模块都是module_init
module_init(xhci_hcd_init);
在xhci_hcd_init中只调用了xhci_register_pci
```
int xhci_register_pci(void)
{
	return pci_register_driver(&xhci_pci_driver);
}
```
xhci_pci_driver中定义了驱动probe函数，pci注册函数__pci_register_driver，然后把pci_bus_type赋给了xhci_pci_driver的driver.bus(在pci/pci-driver.c)

``` 

__pci_register_driver()
{//初始化
	drv->driver.bus = &pci_bus_type;
//一些初始化
	error = driver_register(&drv->driver);
//一部分无关紧要东东，proc文件出错判断什么的 
}

```

然后在pci的驱动中调用基本的driver_register（在base的driver.c）
```
int driver_register(struct device_driver * drv)
{
        klist_init(&drv->klist_devices, klist_devices_get, klist_devices_put);
        init_completion(&drv->unloaded);
        return bus_add_driver(drv);
}
```

klist_init与init_completion是一些设备模型的东东，留个坑，以后占用。然后bus_add_driver,将driver添加到总线bus，
``` c
int driver_register(struct device_driver *drv)
{    .....
       other = driver_find(drv->name, drv->bus);
//一些是否已经注册的判断
	ret = bus_add_driver(drv);
//添加到总线，关键函数
	ret = driver_add_groups(drv, drv->groups);
//添加到组	
return ret;
}
```
driver_find函数通过驱动名字在设备模型中查找是否存在，drv->name也就是"xhci_hcd",如果没有就bus_add_driver
在bus_add_driver中就将驱动结合设备模型联系起来kobject之类的，以备其他地方使用，关键函数driver_attach
``` c
int driver_attach(struct device_driver *drv)
{
	return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}
```
挨个遍历bus中的设备列表klist_devices，调用__driver_attach进行匹配，然后匹配中比较像的就是driver_probe_device(drv, dev);
进去函数也只调用了一个比较有用的函数really_probe(dev, drv);在里面完成驱动上probe调用
``` c
static int really_probe(struct device *dev, struct device_driver *drv)
{
...
//调用的驱动所属总线的probe函数：
if (dev->bus->probe) {
   ret = dev->bus->probe(dev);
   if (ret)
    goto probe_failed;
} else if (drv->probe) {
//再调用自己的驱动中的probe函数：
   ret = drv->probe(dev);
   if (ret)
    goto probe_failed;
}
...
}
```
然后进入正式的xhci hcd驱动
``` c
int usb_hcd_pci_probe(struct pci_dev *dev, const struct pci_device_id *id)
{
   ...
	if (pci_enable_device(dev) < 0)//设置开启I/O、内存空间,IRQ等标志
		return -ENODEV;
	dev->current_state = PCI_D0;
	if (!dev->irq) {//没有中断，直接跳出
		retval = -ENODEV;
		goto err1;
	}
	hcd = usb_create_hcd(driver, &dev->dev, pci_name(dev));//创建hcd，给hcd各项赋值
	if (!hcd) {
		retval = -ENOMEM;
		goto err1;
	}
//主机控制器分配地址空间资源。。。
	pci_set_master(dev);//s使能pcimaster能力
	retval = usb_add_hcd(hcd, dev->irq, IRQF_DISABLED | IRQF_SHARED);
//后面电源管理，出错处理等等
}
```
usb_hcd_pci_probe中主要初始化usb_create_hcd一个hcd，并且调用usb_add_hcd

usb_create_hcd
函数初始化hcd主要看有一个rh_timer，这个是root_hub轮询计时器
控制器以轮询的方式查找端口变化状态
init_timer(&hcd->rh_timer);
hcd->rh_timer.function = rh_timer_func;


下面进入rh_timer_func轮询函数
rh_timer_func调用usb_hcd_poll_rh_status
``` c
usb_hcd_poll_rh_status{  
   length = hcd->driver->hub_status_data(hcd, buffer);（xhci_hub_status_data读取状态寄存器）
   if（有变化）{
    urb = hcd->status_urb;//状态寄存器赋值新的urb，并且设置一些条件什么的
    memcpy(urb->transfer_buffer, buffer, length);//这里是对对应指针变量拷贝，也就可以传递值，一般提交urb时会有数据都通过这个指针赋给调用者
    usb_hcd_giveback_urb(hcd, urb, 0);
   }
}
```
usb_hcd_giveback_urb


usb_add_hcd
这个函数主要完成hcd初始化剩下的部分，申请buffters，注册bus申请中断号然后调用驱动reset或者start
``` c
int usb_add_hcd(struct usb_hcd *hcd,
		unsigned int irqnum, unsigned long irqflags)
{
	if ((retval = hcd_buffer_create(hcd)) != 0) {}  //创建dmz buffer
	if ((retval = usb_register_bus(&hcd->self)) < 0){}//放到usb_bus_list中去
	if ((rhdev = usb_alloc_dev(NULL, &hcd->self, 0)) == NULL) {	}//为roothub分配usb_device

	hcd->self.root_hub = rhdev;
	switch (hcd->driver->flags & HCD_MASK) {
	case HCD_USB11:
		rhdev->speed = USB_SPEED_FULL;
		break;
	case HCD_USB2:
		rhdev->speed = USB_SPEED_HIGH;
		break;
	case HCD_USB3:
		rhdev->speed = USB_SPEED_SUPER;
		break;
	default:
		goto err_set_rh_speed;
	}
	device_init_wakeup(&rhdev->dev, 1);
	if (hcd->driver->reset && (retval = hcd->driver->reset(hcd)) < 0) {   //xhci_pci_hc_driver中的xhci_pci_setup
	}
	hcd->rh_pollable = 1;
	/* NOTE: root hub and controller capabilities may not be the same */
	/* enable irqs just before we start the controller */
	if (hcd->driver->irq) {
		 request_irq(irqnum, &usb_hcd_irq, irqflags,	hcd->irq_descr, hcd);//创建了一个线程监听中断，中断处理函数usb_hcd_irq
		hcd->irq = irqnum;
	} else {
		hcd->irq = -1;
	}
	if ((retval = hcd->driver->start(hcd)) < 0) {  //xhci就是xhci_run
	}
	/* starting here, usbcore will pay attention to this root hub */
	rhdev->bus_mA = min(500u, hcd->power_budget);
	retval = register_root_hub(hcd) //
	if (hcd->uses_new_polling && HCD_POLL_RH(hcd))
		usb_hcd_poll_rh_status(hcd);
	return retval;
} 
```
这里跟踪得hcd->driver就是xhci_pci_hc_driver，主要有6个重点函数
1、usb_alloc_dev
2、xhci_pci_setup
``` c
xhci_pci_setup（）
{
  //寄存器赋值
    xhci_halt()
    xhci_reset()
    xhci_init()
}
```

这个就是向对应的寄存器写入命令，然后等待芯片处理，处理完成后返回结果
3、request_irq
4、xhci_run
``` c
{	ret = xhci_setup_msix(xhci);
	if (ret)
		/* fall back to msi*/
		ret = xhci_setup_msi(xhci);
	if (ret) {
		/* fall back to legacy interrupt*/
		ret = request_irq(pdev->irq, &usb_hcd_irq, IRQF_SHARED,
					hcd->irq_descr, hcd);
		hcd->irq = pdev->irq;
	}
       //设置相关寄存器，开启中断等，然后调用start
       xhci_start(xhci)；//Set the run bit and wait for the host to be running.
}
``` c
xhci_setup_msix函数中设置中断函数xhci_msi_irq，或者usb_hcd_irq（hcd->driver->irq）中调用驱动中断函数xhci_irq。
先尝试msi的中断方式，如果不支持再回滚为原来的中断方式就是上一个函数request_irq申请的。
5、register_root_hub
``` c
register_root_hub{      
      usb_set_device_state(usb_dev, USB_STATE_ADDRESS);//root hub地址为1
      usb_get_device_descriptor(usb_dev, USB_DT_DEVICE_SIZE);//获取设备描述符
      usb_new_device (usb_dev);//将root hub 添加到usb系统
      hcd->rh_registered = 1;//设置标志
}
```
usb_new_device是通用添加设备函数，只有在hub驱动和root——hub注册的时候调用。我们这里是添加root_hub
``` c
usb_new_device{
    err = usb_enumerate_device(udev);	/* Read descriptors ，调用usb_get_configuration等等*/
    announce_device(udev);//打印开始获取的相关参数就是产商等
    err = device_add(&udev->dev);添加设备
//中间有创建sysfs文件等操作
	(void) usb_create_ep_devs(&udev->dev, &udev->ep0, udev);//每个设备都有控制端点，用于配置设备等，需要使用它就要添加进内核，调用该函数添加设备
}
``` 
device_add中调用bus_probe_device然后调用device_attach ,与driver_attach差不多，
``` c
int device_attach(struct device *dev)
{
	if (dev->driver) {
		ret = device_bind_driver(dev);
	} else {
		ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
	}
}
```
__device_attach然后调用usb_bus_type.mach()找到设备驱动然后调用really_probe
``` c
	if (dev->bus->probe) {
		ret = dev->bus->probe(dev);
		if (ret)
			goto probe_failed;
	} else if (drv->probe) {
		ret = drv->probe(dev);
		if (ret)
			goto probe_failed;
	}
```
这里匹配usb_generic_driver中的generic_probe
``` c
static int generic_probe(struct usb_device *udev)
{
	c = usb_choose_configuration(udev);
	if (c >= 0) {
	     err = usb_set_configuration(udev, c);
	}
}
``` 
usb_set_configuration里面配置接口并且调用device_add和create_intf_ep_devs(intf);这里device_add添加hub接口，又会走一遍驱动和设备匹配过程，因不同的值匹配不一样的驱动，设置接口这里匹配
后，调用probe函数为hub_probe，hub_probe里面主要调用了hub_configure
hub_configure 中 usb_fill_int_urb初始化urb结构，写入hub的中断服务程序hub_irq等。
hub_activate调用usb_submit_urb，然后kick_khubd(hub);触发hub_event_list（在usb_hub_init中注册时启动了一个线程khubd用于监听事件），hub_events处理相应操作，然后注册root_hub结束
``` c
hub_configure{
     //获取hub状态get_hub_descriptor， 然后初始化tt，定义每个端口电流，然后更新hcd中状态
     hub->urb = usb_alloc_urb(0, GFP_KERNEL);
	if (!hub->urb) {
		ret = -ENOMEM;
		goto fail;
	}
	usb_fill_int_urb(hub->urb, hdev, pipe, *hub->buffer, maxp, hub_irq,
		hub, endpoint->bInterval);
      hub_activate(hub, HUB_INIT);
}
```
get_hub_descriptor()
```
//调用usb_control_msg，然后usb_internal_control_msg       //循环了3次，向0号端点发送命令，用data接收数据（已在外面分配空间）
   usb_control_msg(hdev, usb_rcvctrlpipe(hdev, 0),
			USB_REQ_GET_DESCRIPTOR, USB_DIR_IN | USB_RT_HUB,
			USB_DT_HUB << 8, 0, data, size,
			USB_CTRL_GET_TIMEOUT)

	    usb_internal_control_msg	 //  填充 控制urb ，并且设置回调函数usb_api_blocking_completion,然后调用usb_start_wait_urb，等待urb返回，把结果返回给调用者
			
            usb_start_wait_urb
                  usb_submit_urb
                  wait_for_completion_timeout
```
usb_submit_urb
```
usb_hcd_submit_urb->rh_urb_enqueue->rh_call_control
  rh_call_control
    list_add_tail(&urb->urb_list, &urb->ep->urb_list);//添加到链表
    hcd->driver->hub_control() //这里就是xhci_hub_control，就可以获取到相应状态了
  //然后会回到wait_for_completion_timeout
```
hub_activate 该函数就是hub配置时，激活hub，检测端口变化并且告诉khubd，让其处理，并且如果有设备就处理设备，这里的流程和插入一个设备流程一样。如果接口上没有设备，那么root_hub 的配置就已经完成
 先调用HUB_INIT，然后工作队列调用hub_init_func2，进入HUB_INIT2,遍历hub每一个端口，获取端口状态
 status = hub_port_status(hub, port1, &portstatus, &portchange);
 然后根据端口状态清除相关feature，设置相应的change_bits[1]等
 status = usb_submit_urb(hub->urb, GFP_NOIO);
 kick_khubd(hub);
 



6、usb_hcd_poll_rh_status调用hcd->driver->hub_status_data,xhci里就是调用xhci_hub_status_data，该函数很简单就是读取端口改变寄存器
返回结果然后将返回的状态（如果有状态也就是新的设备等在roothub上）urb通过函数usb_hcd_giveback_urb(hcd, urb, 0)返回给设备驱动，然后进入设备驱动的回调完成函数
usb_hcd_giveback_urb该函数返回的状态回调到hub_irq中断函数中。
具体调用过程：赋值状态urb->status = status;设置urb完成标志urb->complete (urb);这个完成标志函数在hub注册的时候有一个usb_fill_int_urb函数，该函数将urb->complete 赋值为hub_irq。

到此处，hcd已经添加到usb系统中，并且roothub已经处于监听状态

在定时函数rh_timer_func中，每隔一段时间就去检查端口状态，当hub上有新的设备接入时检查usb_hcd_poll_rh_status，端口状态就会变化，然后就会调用usb_hcd_giveback_urb然后进入hub_irq函数



