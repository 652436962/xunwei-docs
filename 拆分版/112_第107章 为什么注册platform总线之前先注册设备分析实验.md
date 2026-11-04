# 第107章 为什么注册platform总线之前先注册设备分析实验

在之前的章节已经分析了设备注册到总线的过程、platform 设备注册到platform 总线的具体实现、以及总线注册的通用流程、platform 总线的注册逻辑。在此基础上，本章将深入探讨一个关键实现细节：在platform 总线初始化函数platform_bus_init()中，为何会先调用device_register()函数注册一个platform_bus 设备，而后才调用bus_register()完成总线注册？本章来探讨一下这一顺序背后的设计逻辑与内核设备模型的层级关系。
platform_bus_init 函数实现如代码清单107.1 所示。代码清单107.1
```c
int __init platform_bus_init(void)
{
    int error;
    early_platform_cleanup();
    error = device_register(&platform_bus); //步骤一先注册platform_bus 设备
    if (error)
    {
      put_device(&platform_bus);
      return error;
    }
    error = bus_register(&platform_bus_type); //步骤二再注册platform_bus_type 总线
    if (error)
      device_unregister(&platform_bus);
    of_platform_register_reconfig_notifier();
    return error;
}
```
在代码清单107.1 中，可以看到第5 行代码先调用device_register()函数注册paltform_bus这个设备,也就是platform 总线的根设备，然后第11 行代码调用bus_register()函数注册platform总线。二者的注册顺序是先设备后总线。Linux 设备模型要求所有设备必须有一个父设备，platform 总线管理的设备(platform_device)需要一个统一的父设备来组织，而platform_bus 设备就是根父设备。当调用platform_device_register()函数注册设备时，其函数调用了platform_device_add()函数，代码实现如代码清单107.2 所示。代码清单107.2
```c
struct device platform_bus = {
    .init_name= "platform",
};
```

```c
int platform_device_add(struct platform_device *pdev)
{
    ...
    pdev->dev.parent = &platform_bus; //父设备指向platform_bus
    pdev->dev.bus = &platform_bus_type; //总线指向platform_bus_type
    ...
```
}也就是说，当通过platform_device_register()注册设备时，其dev.parent 会默认设置为&platform_bus。platform_bus 设备注册后，会在sysfs 中创建/sys/devices/platform 目录，所有平台设备的sysfs 节点都会在/sys/devices/platform 下创建子目录，所以在platform_bus_init()中先注册设备后注册总线的顺序，目的是先构建platform 顶层目录。

