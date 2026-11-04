# 第071章 device_node转换成platform_device

dtb 展开成device_node 这时候还并不能跟内核中的platform_driver 进行对接，为了让操作系统能够识别和管理设备，需要将设备节点转换为platform_device。
## 71.1 转换规则
在平台总线模型中，device 部分是用platform_device 结构体来描述硬件资源的，所以内核最终会将内核认识的device_node 树转换为platform_ device，但是并不是所有的device_node都会被转换成platform_ device，只有满足要求的才会转换成platform_ device。转换成platform_device 的节点可以在/sys/bus/platform/devices 下查看，那device_node 节点要满足什么要求才会被转换成platform_device 呢？规则1：根节点下有compatible 属性的子节点。规则2：节点中compatible 属性值是simple-bus、simple-mfd 和isa 其中之一，并且该节点下的子节点有compatible 属性。该节点下的子节点会被转换成platform_device。规则3：节点中compatible属性有arm或primecell，则对应的节点不会被转换成platform_device。接下来将通过几个设备树示例对上述规则进行实践。举例1 如代码清单71.1 所示。代码清单71.1
```c
/dts-v1/;
/ {
    model = "This is my devicetree!";
    #address-cells = <1>;
    #size-cells = <1>;
    chosen {
        bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0, 115200";
    };
    cpu1: cpu@1 {
        device_type = "cpu";
        compatible = "arm,cortex-a35", "arm,armv8";
        reg = <0x0 0x1>;
```

```c
};
aliases {
    led1 = "/gpio@22020101";
};
node1 {
    #address-cells = <1>;
    #size-cells = <1>;
    gpio@22020102 {
        reg = <0x20220102 0x40>;
    };
};
node2 {
    node1-child {
        pinnum = <01234>;
    };
};
gpio@22020101 {
    compatible = "led";
    reg = <0x20220101 0x40>;
    status = "okay";
};
};
```
在代码清单71.1 的设备树中，总共有chosen、cpu1: cpu@1、aliases、node1、node2、gpio@22020101 这六个节点，其中前五个节点都没有compatible 属性，所以并不会被转换为platform_device，而最后一个gpio@22020101 节点符合规则，在根节点下，且有compatible 属性，所以最后会转换为platform_device。举例2 如代码清单71.2 所示。代码清单71.2
```c
/dts-v1/;
/ {
    model = "This is my devicetree!";
    #address-cells = <1>;
    #size-cells = <1>;
    chosen {
        bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0, 115200";
```

```c
};
cpu1: cpu@1 {
    device_type = "cpu";
    compatible = "arm,cortex-a35", "arm,armv8";
    reg = <0x0 0x1>;
};
aliases {
    led1 = "/gpio@22020101";
};
node1 {
    #address-cells = <1>;
    #size-cells = <1>;
        compatible = "simple-bus";
    gpio@22020102 {
        reg = <0x20220102 0x40>;
    };
};
node2 {
    node1-child {
        pinnum = <01234>;
    };
};
gpio@22020101 {
    compatible = "led";
    reg = <0x20220101 0x40>;
    status = "okay";
};
};
```
相较于示例1 的设备树，这里在node1 节点中添加了compatible 属性，而且这个compatible 属性值为simple-bus，我们需要继续看他的子节点，子节点gpio@22020102 并没有compatible 属性值，所以这里的node1 节点不会被转换成platform_device。举例3 如代码清单71.3 所示。代码清单71.3
```c
/dts-v1/;
/ {
    model = "This is my devicetree!";
```

```c
#address-cells = <1>;
#size-cells = <1>;
chosen {
    bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0, 115200";
};
cpu1: cpu@1 {
    device_type = "cpu";
    compatible = "arm,cortex-a35", "arm,armv8";
    reg = <0x0 0x1>;
};
aliases {
    led1 = "/gpio@22020101";
};
node1 {
    #address-cells = <1>;
    #size-cells = <1>;
        compatible = "simple-bus";
    gpio@22020102 {
        compatible = "gpio";
        reg = <0x20220102 0x40>;
    };
};
node2 {
    node1-child {
        pinnum = <01234>;
    };
};
gpio@22020101 {
    compatible = "led";
    reg = <0x20220101 0x40>;
    status = "okay";
};
};
```
相较于示例2 的设备树，node1 节点的compatible 属性值为simple-bus，其子节点gpio@22020102 的compatible 属性值为gpio，所以这里的gpio@22020102 节点会被转换成platform_device。

举例4 如代码清单71.4 所示。代码清单71.4
```c
/dts-v1/;
/ {
    model = "This is my devicetree!";
    #address-cells = <1>;
    #size-cells = <1>;
    chosen {
        bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";
    };
    cpul: cpu@1 {
        device_type = "cpu";
        compatible = "arm,cortex-a35", "arm,armv8";
        reg = <0x0 0x1>;
        amba {
            compatible = "simple-bus";
            #address-cells = <2>;
            #size-cells = <2>;
            ranges;
            dmac_peri: dma-controller@ff250000 {
                compatible = "arm,p1330", "arm,primecell";
                reg = <0x0 0xff250000 0x0 0x4000>;
                interrupts = <GIC_SPI 2 IRQ_TYPE_LEVEL_HIGH>,
                              <GIC_SPI 3 IRQ_TYPE_LEVEL_HIGH>;
                #dma-cells = <1>;
                arm,pl330-broken-no-flushp;
                arm,p1330-periph-burst;
                clocks = <&cru ACLK DMAC_PERI>;
                clock-names = "apb_pclk";
            };
            dmac_bus: dma-controller@ff600000 {
                compatible = "arm,p1330", "arm,primecell";
                reg = <0x0 0xff600000 0x0 0x4000>;
                interrupts = <GIC_SPI 0 IRQ_TYPE_LEVEL_HIGH>,
                              <GIC_SPI 1 IRQ_TYPE_LEVEL_HIGH>;
                #dma-cells = <1>;
                arm,pl330-broken-no-flushp;
                arm,pl330-periph-burst;
```

```c
clocks = <&cru ACLK_DMAC_BUS>;
clock-names = "apb_pclk";
};
};
};
};
```
amba 节点的compatible 值为simple-bus，不会被转换为platform_device，而是作为父节点用于组织其他设备，所以需要来查看他的子节点。dmac_peri: dma-controller@ff250000 节点: 该节点的compatible 属性包含arm,p1330 和arm,primecell，所以该节点不会被转换为platform_device。dmac_bus: dma-controller@ff600000 节点: 该节点的compatible 属性包含arm,p1330 和arm,primecell，所以该节点不会被转换为platform_device。
## 71.2 转换流程源码分析
在内核启动时会执行of_platform_default_populate_init()函数，该函数就是将设备树节点（device_node）转换成平台设备（platform_device）的入口函数。该函数会遍历设备树中的设备节点，并为每个符合条件的设备节点创建一个对应的platform_device 结构，然后将其注册到内核中，使得设备驱动程序能够识别和操作这些设备。该函数的具体内容如代码清单71.5 所示。代码清单71.5
```c
static int __init of_platform_default_populate_init(void)
    {
        struct device_node *node;
        device_links_supplier_sync_state_pause();
        // 如果设备树尚未填充，则返回错误码
        if (!of_have_populated_dt())
            return -ENODEV;
        //遍历与reserved_mem_matches 匹配的device_node（如/reserved-memory 中具有compatible 属性
的节点）
        for_each_matching_node(node, reserved_mem_matches)
            of_platform_device_create(node, NULL, NULL);
        // 查找节点"/firmware"
        node = of_find_node_by_path("/firmware");
```

```c
if (node) {
    // 使用找到的节点进行设备树平台设备的填充
    of_platform_populate(node, NULL, NULL, NULL);
    of_node_put(node);
}
// 填充设备树中其他设备
fw_devlink_pause();
of_platform_default_populate(NULL, NULL, NULL);
fw_devlink_resume();
return 0;
}
```
代码清单71.5 中要着重关注的是第29 行的of_platform_default_populate(NULL, NULL, NULL)函数，三个NULL 参数分别对应root、lookup 与parent，找到该函数的定义如代码清单71.6 所示。代码清单71.6
```c
int of_platform_default_populate(struct device_node *root,
                  const struct of_dev_auxdata *lookup,
                  struct device *parent)
{
    return of_platform_populate(root, of_default_bus_match_table, lookup,
                     parent);
}
```
of_platform_default_populate()函数会调用of_platform_populate ()函数，根据device_node信息创建并注册platform_device，并通过of_default_bus_match_table 匹配表指定仅处理如simple-bus 等兼容的device_node，设备匹配表内容如代码清单71.7 所示。代码清单71.7
```c
const struct of_device_id of_default_bus_match_table[] = {
    { .compatible = "simple-bus", },
    { .compatible = "simple-mfd", },
    { .compatible = "isa", },
#ifdef CONFIG_ARM_AMBA
    { .compatible = "arm,amba-bus", },
#endif /* CONFIG_ARM_AMBA */
    {} /* Empty terminated list */
};
```
代码清单71.7 中的设备匹配表就是我们在上一小节中提到的第2 条规则。接下来找到

of_platform_populate()函数的定义具体内容如代码清单71.8 所示。代码清单71.8
```c
int of_platform_populate(struct device_node *root,
             const struct of_device_id *matches,
             const struct of_dev_auxdata *lookup,
             struct device *parent)
    {
    struct device_node *child;
    int rc = 0;
    //获取根节点的引用（为空则查找根路径）
    root = root ? of_node_get(root) : of_find_node_by_path("/");
    if (!root)
        return -EINVAL;
    pr_debug("%s()\n", __func__);
    pr_debug(" starting at: %pOF\n", root);
    device_links_supplier_sync_state_pause();
    //遍历root 节点的所有子节点（查找与平台设备相关的节点，这些节点通常具有compatible 属性，用
于匹配设备驱动程序）
    for_each_child_of_node(root, child) {
        //创建平台设备并添加到设备树总线
        rc = of_platform_bus_create(child, matches, lookup, parent, true);
        if (rc) {
             of_node_put(child);
             break;
        }
    }
    device_links_supplier_sync_state_resume();
    //设置root 节点的OF_POPULATED_BUS 标志
    of_node_set_flag(root, OF_POPULATED_BUS);
    //释放root 节点的引用计数
    of_node_put(root);
    return rc;
}
```
of_platform_populate() 函数会遍历给定节点的所有子节点，并为每个子节点调用

of_platform_bus_create()。在of_platform_bus_create()中若子节点符合匹配规则，则递归地为其子节点继续创建platform_device，从而完成整棵设备子树的转换与注册，其中matches 为上面的of_default_bus_match_table 设备匹配表。of_platform_bus_create()函数的具体定义如代码清单71.9 所示。代码清单71.9
```c
static int of_platform_bus_create(struct device_node *bus,
                   const struct of_device_id *matches,
                   const struct of_dev_auxdata *lookup,
                   struct device *parent, bool strict)
{
    const struct of_dev_auxdata *auxdata;
    struct device_node *child;
    struct platform_device *dev;
    const char *bus_id = NULL;
    void *platform_data = NULL;
    int rc = 0;
    /* 确保设备节点具有compatible 属性*/
    if (strict && (!of_get_property(bus, "compatible", NULL))) {
        pr_debug("%s() - skipping %pOF, no compatible prop\n",
              __func__, bus);
        return 0;
    }
    /* 跳过不想创建设备的节点*/
    if (unlikely(of_match_node(of_skipped_node_table, bus))) {
        pr_debug("%s() - skipping %pOF node\n", __func__, bus);
        return 0;
    }
    // 如果节点已被填充，避免重复创建平台设备
    if (of_node_check_flag(bus, OF_POPULATED_BUS)) {
        pr_debug("%s() - skipping %pOF, already populated\n",
             __func__, bus);
        return 0;
    }
    auxdata = of_dev_lookup(lookup, bus);
    if (auxdata) {
        bus_id = auxdata->name;
        platform_data = auxdata->platform_data;
```

```c
}
if (of_device_is_compatible(bus, "arm,primecell")) {
    /*
     * 在此处不返回错误以保持与旧设备树文件的兼容性。
     */
    of_amba_device_create(bus, bus_id, platform_data, parent);
    return 0;
}
dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
if (!dev || !of_match_node(matches, bus))
    return 0;
for_each_child_of_node(bus, child) {
    pr_debug("create child: %pOF\n", child);
    rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
    if (rc) {
        of_node_put(child);
        break;
    }
}
of_node_set_flag(bus, OF_POPULATED_BUS);
return rc;
}
```
of_platform_bus_create()函数的作用是根据device_node 创建platform_device，并递归的对子节点创建platform_device。在代码清单719 第14 行中判断节点是否具有compatible 属性，因为compatible 属性用于匹配设备驱动程序，对应我们在上一小节的第1 条规则，没有的话不做处理。在代码清单71.9 第21 行判断of_match_node()函数，根据of_skipped_node_table 表查看节点属性，判断节点是否需要创建设备，如果节点中的compatible 属性的值为operating-points-v2，则跳过。在代码清单71.9 第27 行判断节点是否已被展开（设备节点是否设置OF_POPULATED_BUS 标志），如果是则跳过，这个条件判断用于避免重复创建已经填充的设备节点。在代码清单71.9 第33 行-第37 行由于lookup 这里传入的参数为NULL，所以下面的条件判断并不会被执行。在代码清单71.9 第39 行判断如果设备节点bus 兼容于arm,primecell，则调用of_amba_device_create() 函数创建AMBA 设备，对应我们在上一小节学习的第3 条规则。

在代码清单71.9 第47 行调用of_platform_device_create_pdata()函数将节点转化为platform_device。在代码清单71.9 第51 行-第58 行遍历设备节点的每个子节点child，并递归调用of_platform_bus_create()函数将子节点转化为platform_device。在对子节点进行操作时，根据matchs传入的数组进行判断，如果包括simple-bus、simple-mfd、isa 等属性时，会执行该部分代码。最后在代码清单71.9 第59 行将在总线上的节点转化完毕后，设置标志OF_POPULATED_BUS，设置该标志的作用是标记该总线或节点已经被遍历处理过，从而在后续调用中避免重复注册。其中of_platform_device_create_pdata()函数的具体定义如代码清单71.10 所示。代码清单71.10
```c
static struct platform_device *of_platform_device_create_pdata(
                     struct device_node *np,
                     const char *bus_id,
                     void *platform_data,
                     struct device *parent)
{
    struct platform_device *dev;
    /* 检查设备节点是否可用或已填充*/
    if (!of_device_is_available(np) ||
        of_node_test_and_set_flag(np, OF_POPULATED))
        return NULL;
    /* 分配平台设备结构体*/
    dev = of_device_alloc(np, bus_id, parent);
    if (!dev)
        goto err_clear_flag;
    /* 设置平台设备的一些属性*/
    dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);
    if (!dev->dev.dma_mask)
        dev->dev.dma_mask = &dev->dev.coherent_dma_mask;
    dev->dev.bus = &platform_bus_type;
    dev->dev.platform_data = platform_data;
    of_msi_configure(&dev->dev, dev->dev.of_node);
    of_reserved_mem_device_init_by_idx(&dev->dev, dev->dev.of_node, 0);
    /* 将平台设备添加到设备模型中*/
    if (of_device_add(dev) != 0) {
```

