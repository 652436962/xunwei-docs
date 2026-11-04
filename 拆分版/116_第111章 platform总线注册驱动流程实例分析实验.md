# 第111章 platform总线注册驱动流程实例分析实验

本章节分析平台驱动是如何注册到platform 平台总线上的。在linux 内核中，使用platform_driver_reigister 宏来将自定义驱动注册到内核中，而platform_driver_reigister 实际上是__platform_driver_reigister 的封装。调用platform_driver_reigister(drv)宏展开后相当于调__platform_driver_reigister(drv,THIS_MODULE)，如代码清单111.1 所示。代码清单111.1
```c
#define platform_driver_register(drv)__platform_driver_register(drv, THIS_MODULE)
```
_platform_driver_reigister 函数实现如代码清单111.2 所示。代码清单111.2
```c
int __platform_driver_register(struct platform_driver *drv,struct module *owner)
{
  drv->driver.owner = owner;
  drv->driver.bus = &platform_bus_type;//绑定驱动到platform 总线
  drv->driver.probe = platform_drv_probe;//设置驱动的probe 回调函数为platform_drv_probe
  drv->driver.remove = platform_drv_remove; //设置驱动的remove 回调函数为platform_drv_remove
 drv->driver.shutdown = platform_drv_shutdown; //设置驱动的shutdown 回调函数为platform_drv_shutdown
 return driver_register(&drv->driver);//调用driver_register（内核通用驱动注册函数）完成最终注册
}
```
driver_register()函数在前面章节中已经详细分析完毕，接下来我们重点看platform 总线的probe 函数是如何执行的。platform_drv_probe()函数如代码清单111.3 所示。代码清单111.3
```c
static int platform_drv_probe(struct device *_dev)
{
    // 将传递给驱动程序的设备指针转换为platform_driver 结构体指针
    struct platform_driver *drv = to_platform_driver(_dev->driver);
    // 将传递给驱动程序的设备指针转换为platform_device 结构体指针
    struct platform_device *dev = to_platform_device(_dev);
    int ret;
    // 设置时钟默认设置
    ret = of_clk_set_defaults(_dev->of_node, false);
    if (ret < 0)
        return ret;
    // 将设备附加到电源域
    ret = dev_pm_domain_attach(_dev, true);
    if (ret)
```

