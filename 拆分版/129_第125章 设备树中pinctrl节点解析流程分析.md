# 第125章 设备树中pinctrl节点解析流程分析

本章学习学习设备树中pinctrl 节点解析流程。
## 125.1 dev_pin_info 结构体引入
以485 控制引脚的设备树节点和对应的pinctrl 设备树节点进行举例，如代码清单125.1 所示。代码清单125.1
```c
rk_485_ctl: rk-485-ctl {
            compatible = "topeet,rs485_ctl";
            gpios = <&gpio0 22 GPIO_ACTIVE_HIGH>;
            pinctrl-names = "default";
            pinctrl-0 = <&rk_485_gpio>;
};
&pinctrl {
        rk_485{
            rk_485_gpio:rk-485-gpio {
                rockchip,pins = <3 13 RK_FUNC_GPIO &pcfg_pull_none>;
            };
        };
};
```
当485 设节点备与对应485 驱动匹配成功后会执行485 驱动程序中的probe 函数，在probe函数中就可以对设备树中描述的485 使能引脚进行操作，即在执行probe 函数之前就已经设置好了引脚复用。所以从这里可以猜测出在执行485 驱动的probe 函数之前就已经使用pinctrl 子系统对引脚进行了设置。在学习设备模型时，我们已经知道了驱动中的probe 函数是在内核源码目录下的“drivers/base/dd.c”驱动文件中加载执行的，找到驱动文件中really probe()函数中与probe 函数加载相关的代码，关键代码如代码清单125.2 所示。代码清单125.2

```c
re_probe:
    dev->driver = drv;// 将设备的驱动程序指针设置为当前驱动
    /* 如果使用了pinctrl，在探测之前绑定引脚*/
    ret = pinctrl_bind_pins(dev);// 绑定设备的引脚
    if (ret)
        goto pinctrl_bind_failed;
......
    if (dev->bus->probe) {
        ret = dev->bus->probe(dev);// 如果总线的探测函数存在，则调用总线的探测函数
        if (ret)
             goto probe_failed;
    } else if (drv->probe) {
        ret = drv->probe(dev);// 否则调用驱动程序的探测函数
        if (ret)
             goto probe_failed;
    }
```
在代码清单125.2 中，如果在设备树中使用了pinctrl 就会调用第5 行的pinctrl_bind_pins()函数进行设备引脚的绑定，pinctrl_bind_pins() 函数定义在内核源码目录下的“drivers/base/pinctrl.c”文件中，如代码清单125.3 所示。代码清单125.3
```c
int pinctrl_bind_pins(struct device *dev)
{
    int ret;
    if (dev->of_node_reused)
        return 0;
    // 为设备的引脚分配内存空间
    dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
    if (!dev->pins)
        return -ENOMEM;
    // 获取设备的pinctrl 句柄
    dev->pins->p = devm_pinctrl_get(dev);
    if (IS_ERR(dev->pins->p)) {
        dev_dbg(dev, "没有pinctrl 句柄\n");
        ret = PTR_ERR(dev->pins->p);
        goto cleanup_alloc;
```

```c
}
// 查找设备的默认pinctrl 状态
dev->pins->default_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_DEFAULT);
if (IS_ERR(dev->pins->default_state)) {
    dev_dbg(dev, "没有默认的pinctrl 状态\n");
    ret = 0;
    goto cleanup_get;
}
// 查找设备的初始化pinctrl 状态
dev->pins->init_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_INIT);
if (IS_ERR(dev->pins->init_state)) {
    /* 不提供此状态是完全合法的*/
    dev_dbg(dev, "没有初始化的pinctrl 状态\n");
    // 选择默认的pinctrl 状态
    ret = pinctrl_select_state(dev->pins->p, dev->pins->default_state);
} else {
    // 选择初始化的pinctrl 状态
    ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
}
if (ret) {
    dev_dbg(dev, "无法激活初始的pinctrl 状态\n");
    goto cleanup_get;
}
#ifdef CONFIG_PM
/*
 * 如果启用了电源管理，还会寻找可选的睡眠和空闲的引脚状态，其语义在
 * <linux/pinctrl/pinctrl-state.h> 中定义
 */
dev->pins->sleep_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_SLEEP);
if (IS_ERR(dev->pins->sleep_state))
    /* 不提供此状态是完全合法的*/
    dev_dbg(dev, "没有睡眠的pinctrl 状态\n");
dev->pins->idle_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_IDLE);
if (IS_ERR(dev->pins->idle_state))
    /* 不提供此状态是完全合法的*/
    dev_dbg(dev, "没有空闲的pinctrl 状态\n");
#endif
```

```c
return 0;
/*
 * 如果对于此设备没有找到pinctrl 句柄或默认状态，
 * 让我们明确释放设备中的引脚容器，因为保留它没有意义。
 */
cleanup_get:
devm_pinctrl_put(dev->pins->p);
cleanup_alloc:
devm_kfree(dev, dev->pins);
dev->pins = NULL;
/* 返回延迟*/
if (ret == -Eprobe_DEFER)
    return ret;
/* 返回严重错误*/
if (ret == -EINVAL)
    return ret;
/* 我们忽略诸如-ENOENT 的错误，表示没有pinctrl 状态*/
return 0;
}
```
代码清单125.3 中的函数有很多dev->pins 的结构体指针引用，然后跳转到struct device *结构体的定义，部分定义如代码清单125.4 所示。代码清单125.4
```c
struct device {
    struct device*parent;// 指向父设备的指针
    struct device_private*p;// 私有数据指针
    struct kobject kobj;// 内核对象，用于设备的管理
    const char*init_name;// 设备的初始名称
    const struct device_type *type;// 设备类型
    struct mutexmutex;// 互斥锁，用于同步对驱动程序的调用
    struct bus_type*bus;// 设备所在的总线类型
    struct device_driver *driver;// 分配该设备的驱动程序
    void*platform_data;// 平台特定的数据
    void*driver_data;// 驱动程序的数据，使用dev_set/get_drvdata 来设置和获取
    ....
#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
```

```c
struct irq_domain*msi_domain;// 设备的通用MSI IRQ 域
#endif
#ifdef CONFIG_PINCTRL
struct dev_pin_info *pins;// 设备的引脚信息
#endif
......
```
在代码清单125.4 中第21 行如果定义了CONFIG_PINCTRL 宏那记录设备引脚信息的结构体struct dev_pin_info*pins 就会被使能，struct dev_pin_info 结构体定义如代码清单125.5 所示。代码清单125.5
```c
struct dev_pin_info {
    struct pinctrl *p;
    struct pinctrl_state *default_state;// 默认状态指针
    struct pinctrl_state *init_state;// 初始化状态指针
    #ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;// 睡眠状态指针（仅在支持电源管理时可用）
    struct pinctrl_state *idle_state;// 空闲状态指针（仅在支持电源管理时可用）
    #endif
};
```
struct dev_pin_info 结构体详细介绍如下：（1）struct pinctrl *p;：用来表示器件引脚的控制状态持有者。（2）struct pinctrl_state *default_state;：默认状态指针。该指针指向设备的默认引脚配置状态。（3）struct pinctrl_state *init_state;：初始化状态指针。该指针指向设备的初始化引脚配置状态，。（4）struct pinctrl_state *sleep_state;：睡眠状态指针（。该指针指向设备的引脚配置状态，表示设备在进入睡眠状态时的引脚配置。（5）struct pinctrl_state *idle_state;：空闲状态指针。该指针指向设备的引脚配置状态，表示设备在空闲状态时的引脚配置。然后继续看485 的控制引脚这个例子，如代码清单125.6 所示。代码清单125.6
```c
rk_485_ctl: rk-485-ctl {
            compatible = "topeet,rs485_ctl";
            gpios = <&gpio0 22 GPIO_ACTIVE_HIGH>;
            pinctrl-names = "default";
            pinctrl-0 = <&rk_485_gpio>;
```

```c
};
&pinctrl {
rk_485{
    rk_485_gpio:rk-485-gpio {
        rockchip,pins = <3 13 RK_FUNC_GPIO &pcfg_pull_none>;
    };
};
};
```
代码清单125.6 中第4 行代码pinctrl-names 属性指定了设备的default 状态，default 状态对应pinctrl-0 属性中rk_485_gpio 节点所描述的引脚复用，所以struct pinctrl_state *default_state 这一默认状态结构体指针会用来存放该信息，这个信息是如何被保存的呢？代码清单125.6 中rk_485 节点会转换为pinctrl_map 结构体，那么struct pinctrl_state *default_state 必然会跟pinctrl_map 结构体建立联系，如果想搞清楚是如何建立关系的，需要继续分析pinctrl_bind_pins()函数。
## 125.2 create_pinctrl 函数
pinctrl_bind_pins()函数中会按照代码清单125.7 中的调用关系调用create_pinctrl()函数。代码清单125.7
```c
devm_pinctrl_get()
    pinctrl_get()
        create_pinctrl()
```
重点看create_pinctrl()函数，在create_pinctrl()函数中使用了struct pinctrl 结构体和structpinctrl_maps 结构体。struct pinctrl 结构体表示创建pinctrl，该结构体定义在内核源码目录下的“drivers/pinctrl/core.h”文件中，如代码清单125.8 所示。代码清单125.8
```c
struct pinctrl {
    struct list_head node;// 用于将pinctrl 添加到全局列表的链表节点
    struct device *dev;// 关联的设备
    struct list_head states;// 存储引脚配置状态的链表，用于跟踪不同的引脚配置状态
    struct pinctrl_state *state; // 当前应用的引脚配置状态
    struct list_head dt_maps;// 存储设备树中定义的引脚映射信息的链表
    struct kref users;// 引脚控制器的引用计数，用于跟踪引脚控制器的引用数量
};
```

struct pinctrl_maps 结构体用来管理pinctrl_map，该结构体定义在内核源码目录下的“drivers/pinctrl/core.h”文件中，如代码清单125.9 所示。代码清单125.9
```c
struct pinctrl_maps {
    struct list_head node;// 链表节点
    const struct pinctrl_map *maps;// 指向映射数组的指针
    unsigned num_maps;// 映射数组中的映射数量
};
```
create_pinctrl()函数如代码清单125.10 所示。代码清单125.10
```c
static struct pinctrl *create_pinctrl(struct device *dev,
                       struct pinctrl_dev *pctldev)
{
    struct pinctrl *p;
    const char *devname;
    struct pinctrl_maps *maps_node;
    int i;
    const struct pinctrl_map *map;
    int ret;
    p = kzalloc(sizeof(*p), GFP_KERNEL);
    if (!p)
        return ERR_PTR(-ENOMEM);
    p->dev = dev;
    INIT_LIST_HEAD(&p->states);
    INIT_LIST_HEAD(&p->dt_maps);
    ret = pinctrl_dt_to_map(p, pctldev);
    if (ret < 0) {
        kfree(p);
        return ERR_PTR(ret);
    }
    devname = dev_name(dev);
    mutex_lock(&pinctrl_maps_mutex);
    /* 遍历引脚控制映射以定位正确的映射*/
    for_each_maps(maps_node, i, map) {
        /* 映射必须适用于此设备*/
        if (strcmp(map->dev_name, devname))
             continue;
```

```c
/*
 * 如果pctldev 不为空，我们正在声明它的独占使用权，
 * 这意味着它自己提供了该设置。
 *
 * 因此，我们必须跳过适用于此设备但由其他设备提供的映射。
 */
if (pctldev &&
    strcmp(dev_name(pctldev->dev), map->ctrl_dev_name))
    continue;
ret = add_setting(p, pctldev, map);
if (ret == -Eprobe_DEFER) {
    pinctrl_free(p, false);
    mutex_unlock(&pinctrl_maps_mutex);
    return ERR_PTR(ret);
}
}
mutex_unlock(&pinctrl_maps_mutex);
if (ret < 0) {
pinctrl_free(p, false);
return ERR_PTR(ret);
}
kref_init(&p->users);
/* 将pinctrl 添加到全局列表*/
mutex_lock(&pinctrl_list_mutex);
list_add_tail(&p->node, &pinctrl_list);
mutex_unlock(&pinctrl_list_mutex);
return p;
}
```
create_pinctrl()函数中的核心操作有三部分。第三部分放在讲解add _setting()函数函数中讲解，第三部分对应24 到62 行代码。本节先详解第一和第二部分。第一部分：第14 行代码将dev 赋值给p->dev，即struct pinctrl 结构体中保存了对应的dev。第15-16 行代码使用INIT_LIST_HEAD 宏初始化p->states 和p->dt_maps 链表头。第二部分：第18 行代码调用pinctrl_dt_to_map ()函数将设备树中pinctrl 定义的引脚复用信息转换为struct pinctrl_map 结构体，并将其添加到p->dt_maps 链表中。

pinctrl_dt_to_map ()函数定义在内核源码目录下的“drivers/pinctrl/devicetree.c”文件中，如代码清单125.11 所示代码清单125.11
```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
{
        struct device_node *np = p->dev->of_node;// 获取pinctrl 关联设备的设备树节点
        int state, ret;
        char *propname;
        struct property *prop;
        const char *statename;
        const __be32 *list;
        int size, config;
        phandle phandle;
        struct device_node *np_config;
        /* 如果CONFIG_OF 启用，且p->dev 不是从设备树实例化而来*/
        if (!np) {
            if (of_have_populated_dt())
                dev_dbg(p->dev, "no of_node; not parsing pinctrl DT\n");
            return 0;
        }
        /* 节点内部存储属性名称的指针*/
        of_node_get(np);
        /* 便利每个定义的状态*/
        for (state = 0;; state++) {
            /* 获取pinctrl-* 属性*/
            propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);//构造属性名称，如pinctrl-0
            prop = of_find_property(np, propname, &size);//查找对应的属性，如pinctrl-0
            kfree(propname);
            if (!prop) {
                if (state == 0) {
                    of_node_put(np);
                    return -ENODEV;
                }
                break;
            }
            list = prop->value;//获取属性值
            size /= sizeof(*list);//计算属性值的大小
            /* 判断pinctrl-names 属性是否命名了该状态*/
```

```c
ret = of_property_read_string_index(np, "pinctrl-names", state, &statename);
/*
 * 如果未命名，则statename 仅是整数状态ID。但是，为了避免动态分配和之后要释放的
麻烦，
 * 可以直接将statename 指向属性名称的一部分。
 */
if (ret < 0) {
    /* strlen("pinctrl-") == 8 */
    statename = prop->name + 8;
}
/* 遍历pinctrl-n（n=0，1，2...）属性中每个引用的引脚配置节点*/
for (config = 0; config < size; config++) {
phandle = be32_to_cpup(list++);//假设属性pinctrl-0 的值为<&rk_485_gpio>，此时这个phandle
的值为rk_485_gpio 节点对应的phandle 的值。
    /* 查找引脚配置节点*/
    np_config = of_find_node_by_phandle(phandle);//通过phandle 找到对应的节点，即
rk_485_gpio 节点
    if (!np_config) {
        dev_err(p->dev, "prop %s index %i invalid phandle\n", prop->name, config);
        ret = -EINVAL;
        goto err;
    }
    /* 解析节点*/
    ret = dt_to_map_one_config(p, pctldev, statename, np_config);
    of_node_put(np_config);
    if (ret < 0)
        goto err;
}
/* 如果在设备树中没有条目，则生成一个虚拟状态表条目*/
if (!size) {
    ret = dt_remember_dummy_state(p, statename);
    if (ret < 0)
        goto err;
}
}
return 0;
err:
pinctrl_dt_free_maps(p);
```

```c
return ret;
}
```
pinctrl_dt_to_map ()函数首先通过第3 行代码获取与pinctrl 关联设备的设备树节点。然后通过第24-76 行解析设备树节点中的pinctrl-n(n=0,1,2...)和pinctrl-names 属性。然后通过phandle 的值找到对应的配置节点，如代码清单1.38 中的rk_485_gpio 配置节点。最后调用dt_to_map_one_config 函数把function 和group 转成pinctrl_map，在dt_to_map_one_config 函数会调用函数操作集中的dt_node_to_map 函数，dt_to_map_one_config 函数如代码清单125.12 所示。代码清单125.12
```c
static int dt_to_map_one_config(struct pinctrl *p,
                 struct pinctrl_dev *hog_pctldev,
                 const char *statename,
                 struct device_node *np_config)
{
    struct pinctrl_dev *pctldev = NULL;
    struct device_node *np_pctldev;
    const struct pinctrl_ops *ops;
    int ret;
    struct pinctrl_map *map;
    unsigned num_maps;
    bool allow_default = false;
    /* 通过np_config 找到对应的设备节点*/
    np_pctldev = of_node_get(np_config);
    for (;;) {
        /* 如果不允许默认配置，则读取pinctrl-use-default 属性*/
        if (!allow_default)
             allow_default = of_property_read_bool(np_pctldev,
                                    "pinctrl-use-default");
        /* 获取np_pctldev 的父节点*/
        np_pctldev = of_get_next_parent(np_pctldev);
        /* 如果没有父节点或者父节点是根节点，则释放np_pctldev 引用并返回*/
        if (!np_pctldev || of_node_is_root(np_pctldev)) {
             of_node_put(np_pctldev);
             /* 检查是否延迟探测驱动程序状态*/
             ret = driver_deferred_probe_check_state(p->dev);
             /* 如果启用了模块并且不允许默认配置，并且返回值是-ENODEV，则延迟探测*/
             if (IS_ENABLED(CONFIG_MODULES) && !allow_default && ret == -ENODEV)
                 ret = -Eprobe_DEFER;
             return ret;
```

```c
}
/* 如果正在创建一个hog，可以使用传递的pctldev */
if (hog_pctldev && (np_pctldev == p->dev->of_node)) {
    pctldev = hog_pctldev;
    break;
}
/* 通过np_pctldev 获取pinctrl_dev 结构体*/
pctldev = get_pinctrl_dev_from_of_node(np_pctldev);
/* 如果获取到了pinctrl_dev 结构体，则跳出循环*/
if (pctldev)
    break;
/* 不要推迟探测hogs（循环）*/
if (np_pctldev == p->dev->of_node) {
    of_node_put(np_pctldev);
    return -ENODEV;
}
}
of_node_put(np_pctldev);
/*
* 调用pinctrl 驱动程序解析设备树节点，并生成映射表
*/
ops = pctldev->desc->pctlops;
/* 检查pinctrl 驱动程序是否支持设备树*/
if (!ops->dt_node_to_map) {
dev_err(p->dev, "pctldev %s doesn't support DT\n",
    dev_name(pctldev->dev));
return -ENODEV;
}
/* 调用pinctrl 驱动程序的dt_node_to_map 方法*/
ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);
if (ret < 0)
return ret;
/* 将映射表块存储起来以供后续使用*/
return dt_remember_or_free_map(p, statename, pctldev, map, num_maps);
}
```
代码清单125.12 中核心操作是第64 行代码调用dt_node_to_map 函数，即代码ops->dt_node_to_map。dt_node_to_map 函数即上一章节中讲解的函数。在第69 行代码中会嗲用dt_remember_or_free_map()函数保存map 信息。下一小节来分析这个函数。

## 125.3 pinctrl_register_mappings 函数
在保存map 信息时需要使用pinctrl_dt_map 结构体，struct pinctrl_dt_map 结构体如代码清单125.13 所示。代码清单125.13
```c
struct pinctrl_dt_map {
    struct list_head node;
    struct pinctrl_dev *pctldev;
    struct pinctrl_map *map;
    unsigned num_maps;
};
```
pinctrl_dt_map 结构体的作用是保存从设备树中解析出来信息。dt_to_map_one_config()函数中会按照dt_remember_or_free_map->pinctrl_register_mappings 该函数调用关系调用pinctrl_register_mappings()函数。dt_remember_or_free_map()函数有五个函数参数，分别是p：指向struct pinctrl 结构体的指针。statename：指向状态名称的指针，表示要设置的状态的名称。pctldev：指向struct pinctrl_dev 结构体的指针。map：指向struct pinctrl_map 结构体数组的指针。num_maps：表示pinctrl_map 的数量。dt_remember_or_free_map()函数如代码清单125.14 所示。代码清单125.14
```c
static int dt_remember_or_free_map(struct pinctrl *p, const char *statename,
                    struct pinctrl_dev *pctldev,
                    struct pinctrl_map *map, unsigned num_maps)
{
    int i;
    struct pinctrl_dt_map *dt_map;
    for (i = 0; i < num_maps; i++) {
        const char *devname;
        /* 复制设备名称*/
        devname = kstrdup_const(dev_name(p->dev), GFP_KERNEL);
        if (!devname)
             goto err_free_map;
        /* 设置pinctrl_map 的设备名称、状态名称和控制器设备名称*/
        map[i].dev_name = devname;
        map[i].name = statename;
        if (pctldev)
             map[i].ctrl_dev_name = dev_name(pctldev->dev);
```

```c
}
dt_map = kzalloc(sizeof(*dt_map), GFP_KERNEL);
if (!dt_map)
    goto err_free_map;
dt_map->pctldev = pctldev;
dt_map->map = map;
dt_map->num_maps = num_maps;
list_add_tail(&dt_map->node, &p->dt_maps);
return pinctrl_register_mappings(map, num_maps);
err_free_map:
*/
dt_free_map(pctldev, map, num_maps);
return -ENOMEM;
}
```
dt_remember_or_free_map()函数的核心操作为第23-30 行代码，即先struct pinctrl_dt_map 结构体分配内存，然后填充结构体成员然后将其添加到p->dt_maps 链表中。此时pinctrl_dt_map 结构体中记录了pinctrl_map 的信息。在dt_remember_or_free_map()函数的第32 行代码中使用pinctrl_register_mappings ()函数注册pinctrl_map。pinctrl_register_mappings ()函数定义在内核源码目录下的”drivers/pinctrl/core.c“文件中，如代码清单125.15 所示。代码清单125.15
```c
int pinctrl_register_mappings(const struct pinctrl_map *maps,
                   unsigned num_maps)
    {
    int i, ret;
    struct pinctrl_maps *maps_node;
    pr_debug("add %u pinctrl maps\n", num_maps);
    /* 首先对pinctrl_map 进行合法性检查*/
    for (i = 0; i < num_maps; i++) {
        // 检查设备名称是否存在
        if (!maps[i].dev_name) {
             pr_err("failed to register map %s (%d): no device given\n",
                    maps[i].name, i);
             return -EINVAL;
```

```c
}
// 检查pinctrl_map 名称是否存在
if (!maps[i].name) {
    pr_err("failed to register map %d: no map name given\n",
           i);
    return -EINVAL;
}
// 对于pinctrl_map 类型和配置映射类型，检查引脚控制设备名称是否存在
if (maps[i].type != PIN_MAP_TYPE_DUMMY_STATE &&
        !maps[i].ctrl_dev_name) {
    pr_err("failed to register map %s (%d): no pin control device given\n",
           maps[i].name, i);
    return -EINVAL;
}
switch (maps[i].type) {
case PIN_MAP_TYPE_DUMMY_STATE:
    // 对于虚拟状态映射类型，不进行验证
    break;
case PIN_MAP_TYPE_MUX_GROUP:
    // 对于复用组映射类型，进行引脚复用映射验证
    ret = pinmux_validate_map(&maps[i], i);
    if (ret < 0)
        return ret;
    break;
case PIN_MAP_TYPE_CONFIGS_PIN:
case PIN_MAP_TYPE_CONFIGS_GROUP:
    // 对于配置映射类型，进行引脚配置映射验证
    ret = pinconf_validate_map(&maps[i], i);
    if (ret < 0)
        return ret;
    break;
default:
    // 对于无效的映射类型，返回错误
    pr_err("failed to register map %s (%d): invalid type given\n",
           maps[i].name, i);
    return -EINVAL;
}
}
// 分配pinctrl_map 内存
maps_node = kzalloc(sizeof(*maps_node), GFP_KERNEL);
```

