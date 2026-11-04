# 第173章 i2c_client结构体转换流程

i2c client 部分会被转换成i2c_client 结构体。本章分析i2c client 转换成i2c_client 结构体的转换流程。
## 173.1 i2c_client 结构体介绍
i2c_client 结构体用于表示挂载在i2c 总线下的i2c 外设信息，如寄存器物理地址、设备信息、设备名字等信息，该结构体定义在include/linux/i2c.h 文件中，具体内容如代码清单173.1所示。代码清单173.1
```c
struct i2c_client {
    unsigned short flags;/* 标志位, 用于各种配置和标识*/
    unsigned short addr;/* 芯片地址- 注意: 地址是7 位的，存储在低7 位中*/
    char name[i2c_NAME_SIZE];/* 设备名称*/
    struct i2c_adapter *adapter;/* 所在的i2c 控制器*/
    struct device dev;/* 设备结构体*/
    int init_irq;/* 初始化时设置的中断号*/
    int irq;/* 设备产生的中断号*/
    struct list_head detected;/* 检测到的设备列表*/
    #if IS_ENABLED(CONFIG_i2c_SLAVE)
    i2c_slave_cb_t slave_cb; /* 从模式下的回调函数*/
    #endif
};
```
在编写i2c client 代码时提到了两种方法，第一种是使用设备树通过设备节点编写i2c client 代码，第二种方法是使用C 文件编写i2c client 代码，但是在C 文件中并没有创建的i2c_client 结构体，而是创建i2c_board_info 结构体，实际上i2c_board_info 结构体在调用i2c_new_device()函数时候会被转换成i2c_client 结构体并添加所属的i2c 控制器等信息。
## 173.2 使用C 文件实现i2c client 时的转换过程
分析使用C 文件创建i2c client 流程需要从平台文件入手，平台文件是描述硬件资源的文件，由于iTOP-RK3568 开发板的内核源码已经使用设备树代替了平台文件，所以本节使用iTOP-4412 开发板3.0 版本的内核源码分析使用C 文件创建i2c client 时的转换过程，该源码存放在网盘iTOP-RK3568 开发板\07_学习教程配套资料\03_Linux 驱动配套资料\04_Linux 驱动程序\10

7_i2c_client 路径下。首先找到itop-4412 开发板的平台文件，具体路径为arch/arm/mach-exynos/mach-itop4412.c，在该平台文件中使用了i2c_register_board_info()函数注册i2c_board_info 结构体，具体内容如代码清单173.2 所示。代码清单173.2
```c
s3c_i2c0_set_platdata(NULL);
i2c_register_board_info(0, i2c_devs0, ARRAY_SIZE(i2c_devs0));
s3c_i2c1_set_platdata(NULL);
i2c_register_board_info(1, i2c_devs1, ARRAY_SIZE(i2c_devs1));
s3c_i2c2_set_platdata(NULL);
i2c_register_board_info(2, i2c_devs2, ARRAY_SIZE(i2c_devs2));
init_lcd_type();
get_lcd_type();
setup_ft5x_width_height();
s3c_i2c3_set_platdata(NULL);
i2c_register_board_info(3, i2c_devs3, ARRAY_SIZE(i2c_devs3));
s3c_i2c4_set_platdata(NULL);
i2c_register_board_info(4, i2c_devs4, ARRAY_SIZE(i2c_devs4));
#ifdef CONFIG_MPU_SENSORS_MPU6050B1
sensor_hw_init();
#endif
s3c_i2c5_set_platdata(NULL);
i2c_register_board_info(5, i2c_devs5, ARRAY_SIZE(i2c_devs5));
//For S5K4EC (using i2c6)
//#ifndef CONFIG_CAN_MCP251X
#if !defined(CONFIG_CAN_MCP251X) && !defined(CONFIG_SPI_RC522)
s3c_i2c6_set_platdata(NULL);
i2c_register_board_info(6, i2c_devs6, ARRAY_SIZE(i2c_devs6));
#endif
s3c_i2c7_set_platdata(NULL);
i2c_register_board_info(7, i2c_devs7, ARRAY_SIZE(i2c_devs7));
```

i2c_register_board_info()函数定义在drivers/i2c/i2c-boardinfo.c 文件中，具体内容如代码清单173.3 所示。代码清单173.3
```c
/*
     * 注册i2c 设备到内核的i2c 子系统中
     *
     * @busnum: i2c 总线号
     * @info: i2c 设备信息数组
     * @len: 设备信息数组的长度
     *
     * 返回值:
     * 成功时返回0，失败时返回负值错误码
*/
int __init
i2c_register_board_info(int busnum,
    struct i2c_board_info const *info, unsigned len)
{
    int status;
    //获取写锁以保护全局i2c 设备列表
    down_write(&__i2c_board_lock);
    //如果总线号大于等于第一个动态分配的总线号，更新第一个动态分配的总线号
    if (busnum >= __i2c_first_dynamic_bus_num)
        __i2c_first_dynamic_bus_num = busnum + 1;
    //遍历设备信息数组
    for (status = 0; len; len--, info++) {
        //为每个设备分配i2c_devinfo 结构体
        struct i2c_devinfo*devinfo;
        devinfo = kzalloc(sizeof(*devinfo), GFP_KERNEL);
        if (!devinfo) {
             //内存分配失败，打印调试信息并退出
             pr_debug("i2c-core: can't register boardinfo!\n");
             status = -ENOMEM;
             break;
        }
        //填充i2c_devinfo 结构体的成员，包括总线号和设备信息，将新分配的i2c_devinfo 添加到全局i2c
设备列表的末尾
        devinfo->busnum = busnum;
        devinfo->board_info = *info;
```

```c
list_add_tail(&devinfo->list, &__i2c_board_list);
}
//释放写锁
up_write(&__i2c_board_lock);
return status;
}
```
i2c_register_board_info()函数有三个参数,其中函数参数busnum 表示i2c 的总线编号，例如i2c0、i2c1 中的0 和1 就属于总线编号，第二个函数参数info 表示i2c_board_info 结构体数组，第三个函数参数len 表示i2c_board_info 结构体数组的长度。该函数核心操作是代码清单178.3 中的第25 到41 行代码，第25 到41 行代码给i2c_devinfo 结构体分配了内容并设置该结构体中的busnum 和board_info 信息，然后将i2c_devinfo 添加到全局i2c 设备列表的末尾。接着打开iTOP-4412 开发板内核源码中的drivers/i2c/busses/i2c-s3c2410.c 文件，在该文件中的probe()函数中按照调用i2c_add_numbered_adapter()-> i2c_register_adapter() ->i2c_scan_static_board_info()函数调用关系调用i2c_scan_static_board_info()函数。i2c_scan_static_board_info()函数如代码清单178.6 所示，该函数遍历__i2c_board_list 链表并调用第13 行的i2c_new_device()函数创建一个新的i2c client。i2c_new_device()函数需要struct i2c_adapter *adap,和struct i2c_board_info const *info 俩个函数参数，struct i2c_board_info const *info 函数参数通过&devinfo->board_info 确定，即在i2c_register_board_info()函数中赋值的i2c_board_info，struct i2c_adapter *adap 函数参数通过代码devinfo->busnum==adapter->nr 确定当前设备是挂载在了哪个i2c 控制器下。代码清单173.4
```c
static void i2c_scan_static_board_info(struct i2c_adapter *adapter)
{
    //定义一个i2c_devinfo 结构体指针
    struct i2c_devinfo *devinfo;
    //获取__i2c_board_lock 读锁
    down_read(&__i2c_board_lock);
    //遍历__i2c_board_list 链表
    list_for_each_entry(devinfo, &__i2c_board_list, list) {
        //检查当前设备是否在当前适配器上
        if (devinfo->busnum == adapter->nr
```

```c
&& !i2c_new_device(adapter,
        &devinfo->board_info))
//如果添加设备失败,打印错误信息
dev_err(&adapter->dev,
"Can't create device at 0x%02x\n",
devinfo->board_info.addr);
}
//释放__i2c_board_lock 读锁
up_read(&__i2c_board_lock);
}
```
i2c-s3c2410.c 文件中的probe()函数按照i2c_add_numbered_adapter()-> i2c_register_adapter() ->i2c_scan_static_board_info()函数调用关系创建i2c client 而不是直接调用i2c_new_device()函数创建的原因是需要保证先让i2c 控制器工作起来再添加具体的i2c 外设信息这个顺序。因此关键代码为devinfo->busnum==adapter->nr。
## 173.3 使用设备树实现i2c client 时的转换过程
ft5x06 触摸芯片设备节点如代码清单173.5 所示。代码清单173.5
```c
i2c1: i2c@fe5a0000 {
    compatible = "rockchip,rk3399-i2c";
    reg = <0x0 0xfe5a0000 0x0 0x1000>;
    clocks = <&cru CLK_i2c1>, <&cru PCLK_i2c1>;
    clock-names = "i2c", "pclk";
    interrupts = <GIC_SPI 47 IRQ_TYPE_LEVEL_HIGH>;
    pinctrl-names = "default";
    pinctrl-0 = <&i2c1_xfer>;
    #address-cells = <1>;
    #size-cells = <0>;
    status = "disabled";
    myft5x06: my-ft5x06@38 {
        compatible = "my-ft5x06";
        reg = <0x38>;
        reset-gpios = <&gpio0 RK_PB6 GPIO_ACTIVE_LOW>;
        interrupt-parent = <&gpio3>;
        interrupts-gpio = <&gpio3 RK_PA5 GPIO_ACTIVE_LOW>;
        interrupts = <RK_PA5 IRQ_TYPE_LEVEL_LOW>;
        pinctrl-names = "default";
```

```c
pinctrl-0 = <&myft5x06_pins>;
};
};
```
根据设备树篇讲解的相关知识，i2c1 控制器节点会转换成为platform_device，但由于在该节点中并没有特殊的属性值，所以它的子节点myft5x06 并不会被转换成platform_device。这里老师直接讲结论，子节点myft5x06 会被转换成i2c client 结构体，接下来我们分析转化流程。打开内核源码drivers/i2c/busses/i2c-rk3x.c 路径中i2c 控制器驱动，在i2c 控制器驱动中的probe()函数中会调用i2c_add_adapter()函数注册i2c 控制器，具体内容如代码清单173.6 所示。代码清单173.6
```c
static int rk3x_i2c_probe(struct platform_device *pdev)
{
    ......
    //注册i2c 控制器
    ret = i2c_add_adapter(&i2c->adap);
    ......
}
```
i2c_add_adapter()用于注册i2c 控制器，该函数定义在drivers/i2c/i2c-core-base.c 文件中，具体内容如代码清单173.7 所示。代码清单173.7
```c
//添加i2c 控制器的函数
int i2c_add_adapter(struct i2c_adapter *adapter)
{
    //获取适配器的设备
    struct device *dev = &adapter->dev;
    //声明一个ID 变量
    int id;
    //如果设备节点存在
    if (dev->of_node) {
        //从设备树别名中获取i2c 总线号
        id = of_alias_get_id(dev->of_node, "i2c");
        //如果总线号有效
        if (id >= 0) {
            //设置适配器编号为获取的总线号
            adapter->nr = id;
            //添加编号的i2c 控制器
            return __i2c_add_numbered_adapter(adapter);
        }
    }
```

```c
//加锁，防止并发访问
mutex_lock(&core_lock);
//在i2c_adapter_idr 动态ID 池中分配一个ID
id = idr_alloc(&i2c_adapter_idr, adapter,
               __i2c_first_dynamic_bus_num, 0, GFP_KERNEL);
//解锁
mutex_unlock(&core_lock);
//如果分配ID 失败
if (WARN(id < 0, "couldn't get idr"))
    //返回错误码
    return id;
//设置适配器编号为分配的ID
adapter->nr = id;
//注册i2c 控制器
return i2c_register_adapter(adapter);
}
```
代码清单173.9 中第10-20 行首先尝试从设备树别名中获取i2c 总线号，i2c 总线号就是i2c0、i2c1 的0 和1 。如果获取总线号成功，则直接使用这个总线号并使用__i2c_add_numbered_adapter()函数注册i2c 控制器。如果获取总线号失败则通过第25-26 行代码从动态ID 池中分配一个ID 作为i2c 控制器编号。然后通过第38 行代码调用i2c_register_adapter() 函数注册i2c 控制器，在i2c_register_adapter() 函数中本质是通过of_i2c_register_devices()函数注册i2c 控制器。如代码清单173.8 所示。
代码清单173.8
```c
// i2c 控制器注册函数
static int i2c_register_adapter(struct i2c_adapter *adap)
{
    ......
    of_i2c_register_devices(adap);
    ......
}
```
of_i2c_register_devices()函数具体内容如代码清单173.9 所示。代码清单173.9
```c
void of_i2c_register_devices(struct i2c_adapter *adap)
{
    //声明两个设备树节点指针变量，分别用于表示i2c 总线节点和i2c 从设备节点
```

```c
struct device_node *bus, *node;
//声明一个i2c 从设备客户端指针变量
struct i2c_client *client;
//只有当i2c 控制器有关联的设备树节点时，才继续注册从设备
if (!adap->dev.of_node)
    return;
//打印调试信息，表示正在遍历i2c 从设备节点
dev_dbg(&adap->dev, "of_i2c: walking child nodes\n");
//尝试从i2c 控制器的设备树节点中获取名为"i2c-bus"的子节点
//如果没有找到该子节点，则直接使用i2c 控制器的设备树节点作为i2c 总线节点
bus = of_get_child_by_name(adap->dev.of_node, "i2c-bus");
if (!bus)
    bus = of_node_get(adap->dev.of_node);
//遍历i2c 总线节点的所有可用子节点
for_each_available_child_of_node(bus, node) {
    //检查该i2c 从设备节点是否已被标记为已注册
    //如果已被标记，则跳过该节点，避免重复注册
    if (of_node_test_and_set_flag(node, OF_POPULATED))
        continue;
    //使用of_i2c_register_device 函数注册i2c 从设备
    //该函数根据设备树节点的属性创建并注册i2c_client 结构体
    client = of_i2c_register_device(adap, node);
    if (IS_ERR(client)) {
        //如果注册失败，打印错误消息，并清除节点的"已注册"标志，以便下次可以重新尝试注册
        dev_err(&adap->dev,
            "Failed to create i2c device for %pOF\n",
            node);
        of_node_clear_flag(node, OF_POPULATED);
    }
}
//释放对i2c 总线节点的引用
of_node_put(bus);
}
```
of_i2c_register_devices() 函数核心代码是代码清单173.12 中第30 行的of_i2c_register_device()函数，该函数如代码清单173.10 所示。代码清单173.10

```c
static struct i2c_client *of_i2c_register_device(struct i2c_adapter *adap,
                           struct device_node *node)
{
    //声明i2c_client 和i2c_board_info 结构体指针变量
    struct i2c_client *client;
    struct i2c_board_info info;
    int ret;
    //打印调试信息,表示正在注册设备树节点
    dev_dbg(&adap->dev, "of_i2c: register %pOF\n", node);
    //根据设备树节点信息填充i2c_board_info 结构体
    ret = of_i2c_get_board_info(&adap->dev, node, &info);
    if (ret)
        //如果获取i2c_board_info 信息失败，返回错误指针
        return ERR_PTR(ret);
    //使用i2c_new_device()函数注册i2c_client 设备
    client = i2c_new_device(adap, &info);
    if (!client) {
        //如果注册失败，打印错误信息并返回错误指针
        dev_err(&adap->dev, "of_i2c: Failure registering %pOF\n", node);
        return ERR_PTR(-EINVAL);
    }
    //返回成功注册的i2c_client 指针
    return client;
}
```
of_i2c_register_device()函数中调用of_i2c_get_board_info()函数解析i2c 外设节点构建i2c_board_info 结构体，of_i2c_get_board_info()函数的具体内容如代码清单173.11 所示。代码清单173.11
```c
int of_i2c_get_board_info(struct device *dev, struct device_node *node,
               struct i2c_board_info *info)
{
    //声明地址和返回值变量
    u32 addr;
    int ret;
    //清空i2c_board_info 结构体
    memset(info, 0, sizeof(*info));
    //从设备树节点获取modalias 字符串
    if (of_modalias_node(node, info->type, sizeof(info->type)) < 0) {
```

```c
//如果获取modalias 失败，打印错误信息并返回
dev_err(dev, "of_i2c: modalias failure on %pOF\n", node);
return -EINVAL;
}
//从设备树节点获取reg 属性(设备地址)
ret = of_property_read_u32(node, "reg", &addr);
if (ret) {
//如果获取地址失败，打印错误信息并返回
dev_err(dev, "of_i2c: invalid reg on %pOF\n", node);
return ret;
}
//检查地址是否为10 位地址模式
if (addr & i2c_TEN_BIT_ADDRESS) {
//清除高位标志，并设置i2c_board_info 的flags
addr &= ~i2c_TEN_BIT_ADDRESS;
info->flags |= i2c_CLIENT_TEN;
}
//检查地址是否为从设备地址模式
if (addr & i2c_OWN_SLAVE_ADDRESS) {
//清除高位标志，并设置i2c_board_info 的flags
addr &= ~i2c_OWN_SLAVE_ADDRESS;
info->flags |= i2c_CLIENT_SLAVE;
}
//填充i2c_board_info 结构体的addr、of_node 和fwnode 字段
info->addr = addr;
info->of_node = node;
info->fwnode = of_fwnode_handle(node);
//检查设备树节点是否有"host-notify"属性
if (of_property_read_bool(node, "host-notify"))
// 如果有,设置i2c_board_info 的flags
info->flags |= i2c_CLIENT_HOST_NOTIFY;
//检查设备树节点是否有"wakeup-source"属性
if (of_get_property(node, "wakeup-source", NULL))
//如果有，设置i2c_board_info 的flags
info->flags |= i2c_CLIENT_WAKE;
//返回0，表示成功获取并填充i2c_board_info 结构体
return 0;
```

```c
}
```
i2c_board_info 结构体构建完成后调用i2c_new_device()函数创建i2c client。所以使用设备树编写的i2c client 会先被转换成i2c_board_info 结构体，然后由i2c_board_info 结构体在转换成i2c_client 结构体。但还有一个地方需要注意，在解析i2c 外设节点构建i2c_board_info 结构体时并没有解析设备树中的中断内容，而在编写ft5x06 的驱动时确用到了i2c_client 结构体中的中断信息，如代码清单178.11 中断相关代码，i2c_client 结构体中的中断号是在哪里进行的赋值呢？代码清单173.12
```c
ret = request_irq(client->irq, ft5x06_handler,
             IRQ_TYPE_EDGE_FALLING | IRQF_ONESHOT, "ft5x06 irq", NULL);
```
在执行ft5x06 驱动的probe()函数之前，会先执行drivers/i2c/i2c-core-base.c 文件中的i2c_device_probe()函数，该函数的具体内容如代码清单173.13 所示。代码清单173.13
```c
static int i2c_device_probe(struct device *dev)
{
    //从device 结构体中获取i2c_client 结构体
    struct i2c_client*client = i2c_verify_client(dev);
    //从device 结构体中获取i2c_driver 结构体
    struct i2c_driver*driver;
    int status;
    //如果client 不存在，则返回0
    if (!client)
        return 0;
    //将dev->driver 转换为i2c_driver 类型
    driver = to_i2c_driver(dev->driver);
    //如果client 没有中断号且driver 没有禁用i2c 核心中断映射，则尝试获取中断号
    if (!client->irq && !driver->disable_i2c_core_irq_mapping) {
        int irq = -ENOENT;
        //如果client 使用Host Notify 中断，则使用i2c_smbus_host_notify_to_irq 获取中断号
        if (client->flags & i2c_CLIENT_HOST_NOTIFY) {
            dev_dbg(dev, "Using Host Notify IRQ\n");
            //保持适配器处于活动状态，因为需要Host Notify 中断
            pm_runtime_get_sync(&client->adapter->dev);
            irq = i2c_smbus_host_notify_to_irq(client);
        }
```

```c
//如果设备有DT 节点，则尝试从DT 节点获取中断号
else if (dev->of_node) {
    irq = of_irq_get_byname(dev->of_node, "irq");
    if (irq == -EINVAL || irq == -ENODATA)
        irq = of_irq_get(dev->of_node, 0);
}
//如果设备有ACPI 关联，则尝试从ACPI 获取中断号
else if (ACPI_COMPANION(dev)) {
    irq = acpi_dev_gpio_irq_get(ACPI_COMPANION(dev), 0);
}
//如果获取中断号失败，则设置为0
if (irq == -Eprobe_DEFER)
    return irq;
if (irq < 0)
    irq = 0;
//将获取到的中断号设置到client 结构体中
client->irq = irq;
}
//如果driver 没有ID 表，且设备也没有匹配的OF 或ACPI ID 表，则返回-ENODEV
if (!driver->id_table &&
!i2c_acpi_match_device(dev->driver->acpi_match_table, client) &&
!i2c_of_match_device(dev->driver->of_match_table, client))
return -ENODEV;
//如果client 需要唤醒功能，则尝试设置唤醒中断
if (client->flags & i2c_CLIENT_WAKE) {
int wakeirq = -ENOENT;
//如果设备有DT 节点，则尝试从DT 节点获取唤醒中断号
if (dev->of_node) {
    wakeirq = of_irq_get_byname(dev->of_node, "wakeup");
    if (wakeirq == -Eprobe_DEFER)
        return wakeirq;
}
//启用设备的唤醒功能
device_init_wakeup(&client->dev, true);
//如果获取到了唤醒中断号，且与普通中断号不同，则设置专用唤醒中断
if (wakeirq > 0 && wakeirq != client->irq)
    status = dev_pm_set_dedicated_wake_irq(dev, wakeirq);
```

```c
//否则，使用普通中断作为唤醒中断
else if (client->irq > 0)
    status = dev_pm_set_wake_irq(dev, client->irq);
else
    status = 0;
//如果设置唤醒中断失败，则输出警告
if (status)
    dev_warn(&client->dev, "failed to set up wakeup irq\n");
}
dev_dbg(dev, "probe\n");
//设置设备的时钟默认值
status = of_clk_set_defaults(dev->of_node, false);
if (status < 0)
goto err_clear_wakeup_irq;
//附加PM 域
status = dev_pm_domain_attach(&client->dev, true);
if (status)
goto err_clear_wakeup_irq;
//调用driver 的probe_new 或probe 函数
if (driver->probe_new)
status = driver->probe_new(client);
else if (driver->probe)
status = driver->probe(client,
                       i2c_match_id(driver->id_table, client));
else
status = -EINVAL;
//如果probe 函数失败，则清除唤醒中断并分离PM 域
if (status) {
dev_pm_domain_detach(&client->dev, true);
goto err_clear_wakeup_irq;
}
return 0;
err_clear_wakeup_irq:
dev_pm_clear_wake_irq(&client->dev);
device_init_wakeup(&client->dev, false);
return status;
```

```c
}
```
其中代码清单173.13 第28 行-第36 行代码会解析设备树中的中断信息从而获取irq，在第45 行将获取到的中断号设置到client 结构体中。所以在ft5x06 的驱动中可以通过client->irq 使用中断号。

