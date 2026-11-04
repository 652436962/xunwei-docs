# 第123章 pinctrl_desc结构体中函数操作集

上一章对probe 函数进行了讲解，probe 函数的实际作用就是注册并启用pinctrl，并且在probe 函数中会对pinctrl_desc 结构体中的函数操作集进行实现。本章对pinctrl_desc 结构体中的三个函数操作集进行详细的讲解。
## 123.1 groups 和function
在讲解pinctrl_desc 结构体中函数操作集之前需要先了解两个关键概念：引脚组（groups）和功能（function），在介绍pinctrl 子系统函数操作集之前，首先对groups 和function 进行讲解。1.引脚组（Groups）引脚组是一组脚的集合，用于表示一种功能。每个引脚组通常与特定的硬件功能或外设相关联。例如，一个引脚组可以表示UART 功能或者SPI 功能。2.功能（Function）：某一个具体功能的不同配置叫做function。举例说明以rk3568-pinctrl.dtsi 设备树文件中的can0 和can1 两个功能为例对上面的内容进行举例，如代码清单123.1 所示。代码清单123.1
```c
can0 {
            /omit-if-no-ref/
            can0m0_pins: can0m0-pins {
                rockchip,pins =
                    /* can0_rxm0 */
                    <0 RK_PB4 2 &pcfg_pull_none>,
                    /* can0_txm0 */
                    <0 RK_PB3 2 &pcfg_pull_none>;
            };
            /omit-if-no-ref/
            can0m1_pins: can0m1-pins {
```

```c
rockchip,pins =
    /* can0_rxm1 */
    <2 RK_PA2 4 &pcfg_pull_none>,
    /* can0_txm1 */
    <2 RK_PA1 4 &pcfg_pull_none>;
};
};
can1 {
/omit-if-no-ref/
can1m0_pins: can1m0-pins {
rockchip,pins =
    /* can1_rxm0 */
    <1 RK_PA0 3 &pcfg_pull_none>,
    /* can1_txm0 */
    <1 RK_PA1 3 &pcfg_pull_none>;
};
/omit-if-no-ref/
can1m1_pins: can1m1-pins {
rockchip,pins =
    /* can1_rxm1 */
    <4 RK_PC2 3 &pcfg_pull_none>,
    /* can1_txm1 */
    <4 RK_PC3 3 &pcfg_pull_none>;
};
};
```
代码清单中的can0 和can1 节点对应两个不同的function，分别为CAN0 控制器和CAN1控制器的功能。can0 和can1 节点中的can0m0_pins: can0m0-pins、can0m1_pins: can0m1-pins、can1m0_pins: can1m0-pins 和can1m1_pins: can1m1-pins 表示group。详细介绍如下：（1）CAN0 控制器（function）中：引脚组can0m0-pins：这是CAN0 控制器的第一个引脚组，用于配置CAN0 的引脚。它定义了两个引脚：RK_PB4 和RK_PB3。其中，RK_PB4 用于CAN0 的接收引脚（can0_rxm0），RK_PB3 用于CAN0 的发送引脚（can0_txm0）。引脚组can0m1-pins：这是CAN0 控制器的第二个引脚组，也用于配置CAN0 的引脚。它定义了两个引脚：RK_PA2 和RK_PA1。其中，RK_PA2 用于CAN0 的接收引脚（can0_rxm1），RK_PA1 用于CAN0 的发送引脚（can0_txm1）。

（2）CAN1 控制器（function）中：引脚组can1m0-pins：这是CAN1 控制器的第一个引脚组，用于配置CAN1 的引脚。它定义了两个引脚：RK_PA0 和RK_PA1。其中，RK_PA0 用于CAN1 的接收引脚（can1_rxm0），RK_PA1 用于CAN1 的发送引脚（can1_txm0）。引脚组can1m1-pins：这是CAN1 控制器的第二个引脚组，也用于配置CAN1 的引脚。它定义了两个引脚：RK_PC2 和RK_PC3。其中，RK_PC2 用于CAN1 的接收引脚（can1_rxm1），RK_PC3 用于CAN1 的发送引脚（can1_txm1）。在struct pinctrl_desc 结构体中有groups、ngroups、functions 和nfunctions 四个结构体参数，分别表示functions，functions 的数量，groups 和groups 的数量。在dt_node_to _map 函数中会用到这些信息。如代码清单123.2 所示。代码清单123.2
```c
struct rockchip_pinctrl {
    struct regmap*regmap_base;// 基本寄存器映射指针
    intreg_size;// 寄存器大小
    struct regmap*regmap_pull;// 拉取寄存器映射指针
    struct regmap*regmap_pmu;// 电源管理单元寄存器映射指针
    struct device*dev;// 设备指针
    struct rockchip_pin_ctrl *ctrl;// 瑞芯微芯片引脚控制器指针
    struct pinctrl_descpctl;// 引脚控制器描述符
    struct pinctrl_dev*pctl_dev;// 引脚控制器设备指针
    struct rockchip_pin_group *groups;// 瑞芯微芯片引脚组指针
    unsigned intngroups;// 引脚组数量
    struct rockchip_pmx_func *functions;// 瑞芯微芯片引脚功能指针
    unsigned intnfunctions;// 引脚功能数量
};
```
groups、ngroups、functions 和nfunctions 四个结构体参数是如何赋值的呢？在probe 函数中会调用rockchip_pinctrl_register()函数，在rockchip_pinctrl_register()函数中，分别调用了rockchip_pinctrl_parse_dt()函数和devm_pinctrl_register()函数。并把struct rockchip_pinctrl *类型的结构体指针变量info 传递到了这俩个函数中。rockchip_pinctrl_parse_dt()函数如代码清单123.3 所示。代码清单123.3
```c
static int rockchip_pinctrl_parse_dt(struct platform_device *pdev,
                      struct rockchip_pinctrl *info)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
```

```c
struct device_node *child;
int ret;
int i;
// 计算子节点数量并更新info 结构体中的计数器
rockchip_pinctrl_child_count(info, np);
dev_dbg(&pdev->dev, "nfunctions = %d\n", info->nfunctions);
dev_dbg(&pdev->dev, "ngroups = %d\n", info->ngroups);
// 为函数和组分配内存空间
info->functions = devm_kcalloc(dev,
                    info->nfunctions,
                    sizeof(struct rockchip_pmx_func),
                    GFP_KERNEL);
if (!info->functions)
    return -ENOMEM;
info->groups = devm_kcalloc(dev,
                 info->ngroups,
                 sizeof(struct rockchip_pin_group),
                 GFP_KERNEL);
if (!info->groups)
    return -ENOMEM;
i = 0;
// 遍历每个子节点，解析function 信息
for_each_child_of_node(np, child) {
    // 如果节点不是function 节点，则继续下一个节点
    if (!is_function_node(child))
        continue;
    // 解析function 信息并存储到info 结构体中
    ret = rockchip_pinctrl_parse_functions(child, info, i++);
    if (ret) {
        dev_err(&pdev->dev, "failed to parse function\n");
        of_node_put(child);
        return ret;
    }
}
return 0;
}
```

rockchip_pinctrl_parse_dt()函数关键部分介绍如下：第10-14 行：调用函数rockchip_pinctrl_child_count，计算设备节点np 的子节点数量，并更新info 结构体中的计数器，最后打印function 和组的数量。第17-22 行：使用devm_kcalloc ()函数为functions 分配内存空间并将指针保存在info->functions 中。第24-29 行：使用devm_kcalloc() 函数为组分配内存空间，并将指针保存在info->groups 中。第33-46 行使用for_each_child_of_node 宏遍历设备节点np 的每个子节点child。如果节点child 不是function 节点，则跳过当前节点，继续遍历下一个节点，在每一个循环中调用函数rockchip_pinctrl_parse_functions()解析function 信息，并将结果存储在info 结构体中的相应位置。这里首先对rockchip_pinctrl_child_count()函数进行讲解，该函数内容如代码清单123.4 所示。代码清单123.4
```c
static void rockchip_pinctrl_child_count(struct rockchip_pinctrl *info,
                          struct device_node *np)
{
    struct device_node *child;
    // 遍历设备节点的子节点
    for_each_child_of_node(np, child) {
        // 如果子节点不是function 节点，则跳过当前节点，继续遍历下一个节点
        if (!is_function_node(child))
             continue;
        // 子节点是function 节点，增加function 计数器
        info->nfunctions++;
        // 获取子节点的子节点数量，并增加到组计数器中
        info->ngroups += of_get_child_count(child);
    }
}
```
rockchip_pinctrl_child_count()函数首先会遍历function，从而得到系统设备树中实际使用的function 数量，并保存到nfunctions 参数中，而groups 变量保存的是function 中的引脚组数量，需要通过遍历function 子节点来进行获取，到这里就可以确定rockchip_pinctrl 结构体中的nfunctions 用来记录function 数量，ngroups 用来记录groups 数量。然后来看rockchip_pinctrl_parse_dt()函数的第40 行rockchip_pinctrl_parse_functions()函数，

通过该函数可以解析function 信息，如代码清单123.5 所示。代码清单123.5
```c
static int rockchip_pinctrl_parse_functions(struct device_node *np,
                          struct rockchip_pinctrl *info,
                          u32 index)
{
    struct device_node *child;
    struct rockchip_pmx_func *func;
    struct rockchip_pin_group *grp;
    int ret;
    static u32 grp_index;
    u32 i = 0;
    // 打印调试信息，显示正在解析的函数节点和索引
    dev_dbg(info->dev, "parse function(%d): %pOFn\n", index, np);
    // 获取当前函数在info->functions 数组中的指针
    func = &info->functions[index];
    /* 初始化函数*/
    func->name = np->name;
    // 获取函数节点的子节点数量，即关联的组数量
    func->ngroups = of_get_child_count(np);
    if (func->ngroups <= 0)
        return 0;
    // 为函数的组指针数组分配内存空间
    func->groups = devm_kcalloc(info->dev,
             func->ngroups, sizeof(char *), GFP_KERNEL);
    if (!func->groups)
        return -ENOMEM;
    // 遍历函数节点的每个子节点
    for_each_child_of_node(np, child) {
        // 将子节点的名称存储到函数的组指针数组中
        func->groups[i] = child->name;
        // 获取info->groups 数组中的对应组指针
        grp = &info->groups[grp_index++];
        // 解析组信息，并将结果存储到对应的组指针中
        ret = rockchip_pinctrl_parse_groups(child, grp, info, i++);
        if (ret) {
```

```c
of_node_put(child);
return ret;
}
}
return 0;
}
```
第6-7 行：定义了struct rockchip_pmx_func *类型的变量func和struct rockchip_pin_group*类型的变量grp，分别用来存放function 信息和groups 信息。第16 行：将info 的functions 地址赋值给了func，所以rockchip_pinctrl 的functions 参数的作用就是用来存放pinctrl 设备树中的function 信息。第37 行：将info 的groups 地址赋值给了grp，所以rockchip_pinctrl 的groups 参数的作用就是用来存放pinctrl 设备树中的groups 信息。在第40 行：使用rockchip_pinctrl_parse_groups()函数来解析group 信息，并将结果存储到对应的group 指针中。接下来分析rockchip_pinctrl_parse_groups()函数是如何解析group 信息的，rockchip_pinctrl_parse_groups()函数如代码清单123.6 所示。代码清单123.6
```c
static int rockchip_pinctrl_parse_groups(struct device_node *np,
                            struct rockchip_pin_group *grp,
                            struct rockchip_pinctrl *info,
                            u32 index)
{
    struct rockchip_pin_bank *bank;
    int size;
    const __be32 *list;
    int num;
    int i, j;
    int ret;
    // 打印调试信息，显示正在解析的组节点和索引
    dev_dbg(info->dev, "group(%d): %pOFn\n", index, np);
    // 初始化组信息
    grp->name = np->name;
    /*
     * 绑定格式为rockchip,pins = <bank pin mux CONFIG>，
```

```c
* 进行合法性检查并计算引脚数量
*/
list = of_get_property(np, "rockchip,pins", &size);
// 对返回值不进行检查，因为传递的节点是安全的
size /= sizeof(*list);
if (!size || size % 4) {
   dev_err(info->dev, "wrong pins number or pins and configs should be by 4\n");
   return -EINVAL;
}
// 计算组的引脚数量
grp->npins = size / 4;
// 为组的引脚数组和数据数组分配内存空间
grp->pins = devm_kcalloc(info->dev, grp->npins, sizeof(unsigned int),
                    GFP_KERNEL);
grp->data = devm_kcalloc(info->dev,
                grp->npins,
                sizeof(struct rockchip_pin_config),
                GFP_KERNEL);
if (!grp->pins || !grp->data)
   return -ENOMEM;
// 遍历group 中的每个元素，每4 个元素表示一个引脚的信息
for (i = 0, j = 0; i < size; i += 4, j++) {
   const __be32 *phandle;
   struct device_node *np_config;
   // 获取引脚号
   num = be32_to_cpu(*list++);
   bank = bank_num_to_bank(info, num);
   if (IS_ERR(bank))
       return PTR_ERR(bank);
   // 计算引脚编号
   grp->pins[j] = bank->pin_base + be32_to_cpu(*list++);
   // 获取引脚对应的功能选择值
   grp->data[j].func = be32_to_cpu(*list++);
   // 获取与引脚相关的配置信息
   phandle = list++;
   if (!phandle)
       return -EINVAL;
```

```c
// 通过句柄查找配置节点
np_config = of_find_node_by_phandle(be32_to_cpup(phandle));
// 解析配置信息，并将结果存储到组的数据数组中
ret = pinconf_generic_parse_dt_config(np_config, NULL,
        &grp->data[j].configs, &grp->data[j].nconfigs);
if (ret)
    return ret;
}
return 0;
}
```
rockchip_pinctrl_parse_groups()函数关键部分介绍如下：第17 行：初始化group 信息，将group 的名称设置为节点的名称。第23-29 行：通过of_get_property()函数获取设备树group 节点的属性"rockchip,pins"的值，该属性的格式为"bank pin mux CONFIG"。如果属性值为空或者数量不是4 的倍数，将返回错误代码EINVAL。第31-42 行：首先计算得到的group 中的引脚数量，然后根据计算得到的引脚数量为引脚分配内存空间。如果内存分配失败，函数将返回错误代码ENOMEM。第45 行：遍历group 中的每个元素，每4 个元素表示一个引脚的信息。第50-53 行：获取引脚号，并使用bank_num_to_bank()函数将引脚号转换为对应struct rockchip_pin_bank 结构体类型的指针。如果转换失败，函数将返回相应的错误代码。第56 行：根据引脚基地址（pin_base）中的和节点的属性rockchip,pins 中第二个属性值计算引脚的编号，并将其存储在引脚数组（grp->pins）中。第58 行：从节点属性rockchip,pins 中获取与当前引脚相关的功能选择值（即节点属性rockchip,pins 中第三个属性值），并将其存储在数据数组（grp->data）中的相应位置。第60-72 行获取节点的属性rockchip,pins 中第四个属性值对应的配置信息对应的节点，如rockchip,pins=<2 RK_PA2 4 &pcfg_pull_none>属性中的pcfg_pull_none 节点。并使用函数数使用pinconf_generic_parse_dt_config()把配置属性解析为通用引脚配置值。至此，关于rockchip_pinctrl 结构体的groups、ngroups、functions 和nfunctions 参数的是如何赋值的就讲解完成了。

## 123.2 函数操作集对应的结构体
在pinctrl_desc 结构体中总共有三个函数操作集，分别是struct pinctrl_ops、structpinmux_ops 和struct pinconf_ops 结构体，如代码清单123.7 所示。代码清单123.7
```c
const struct pinctrl_ops *pctlops;// 引脚控制操作函数指针
const struct pinmux_ops *pmxops;// 引脚复用操作函数指针
const struct pinconf_ops *confops;// 引脚配置操作函数指针
```
struct pinctrl_ops、struct pinmux_ops 和struct pinconf_ops 结构体介绍如下：（1）struct pinctrl_ops 结构体pinctrl_ops 结构体定义在内核源码的“include/linux/pinctrl/pinctrl.h”文件中，如代码清单123.8 所示。代码清单123.8
```c
struct pinctrl_ops {
    int (*get_groups_count)(struct pinctrl_dev *pctldev); //获取指定的Pin Control 设备支持的引脚组数量
    const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                       unsigned selector);// 获取指定引脚组对应的引脚组名称
    int (*get_group_pins)(struct pinctrl_dev *pctldev,
                   unsigned selector,
                   const unsigned **pins,
                   unsigned *num_pins);// 获取指定引脚组对应的引脚
    void (*pin_dbg_show)(struct pinctrl_dev *pctldev, struct seq_file *s,
                  unsigned offset);//在调试信息中输出指定引脚对应的引脚信息
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                   struct device_node *np_config,
                   struct pinctrl_map **map, unsigned *num_maps);// 根据给定的设备树节点，创建与之
相关联的Pin Control 映射
    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                 struct pinctrl_map *map, unsigned num_maps);//释放之前通过dt_node_to_map 创建的
Pin Control 映射
};
```
pinctrl_ops 结构体定义了一组操作函数，以下是对每个操作函数的简要说明：（1）get_groups_count：获取引脚组的数量。（2）get_group_name：获取引脚组的名称。（3）get_group_pins：获取引脚组的引脚列表。（4）pin_dbg_show：在调试信息中输出引脚信息。

（5）dt_node_to_map：根据设备树节点创建与之相关联的映射。（6）dt_free_map：释放通过dt_node_to_map 创建的映射。
（2）struct pinmux_ops 结构体pinmux_ops 结构体定义在内核源码的“include/linux/pinctrl/pinctrl.h”文件中，如代码清单123.9 所示。代码清单123.9
```c
struct pinmux_ops {
    int (*request) (struct pinctrl_dev *pctldev, unsigned offset);//查看是否可以将某个引脚设置为可用于复用
    int (*free) (struct pinctrl_dev *pctldev, unsigned offset);//request() 回调的反向函数，在请求后释放引脚
    int (*get_functions_count) (struct pinctrl_dev *pctldev);//返回function 的数量
    const char *(*get_function_name) (struct pinctrl_dev *pctldev,
                       unsigned selector);//返回function 的名称
    int (*get_function_groups) (struct pinctrl_dev *pctldev,
                   unsigned selector,
                   const char * const **groups,
                   unsigned *num_groups);//返回与function 关联的group
    int (*set_mux) (struct pinctrl_dev *pctldev, unsigned func_selector,
             unsigned group_selector);//设置引脚的复用功能
    int (*gpio_request_enable) (struct pinctrl_dev *pctldev,
                     struct pinctrl_gpio_range *range,
                     unsigned offset);//请求并启用gpio
    void (*gpio_disable_free) (struct pinctrl_dev *pctldev,
                    struct pinctrl_gpio_range *range,
                    unsigned offset);//在特定引脚上释放GPIO 复用
    int (*gpio_set_direction) (struct pinctrl_dev *pctldev,
                    struct pinctrl_gpio_range *range,
                    unsigned offset,
                    bool input);//设置gpio 方向
    bool strict;
};
```
struct pinmux_ops 结构体中定义了引脚复用相关函数。下面是对该结构体成员的详细解释：（1）request: 判断是否可以将某个引脚设置为可用于复用。（2）free: 该函数是request()函数的反向操作，用于在引脚请求后释放引脚。（3）get_functions_count: 返回function 的数量。（4）get_function_name: 返回function 的名称。

（5）get_function_groups: 返回与function 关联的group。（6）set_mux: 设置引脚的复用功能。（7）gpio_request_enable: 在特定引脚上请求并启用GPIO。（8）gpio_disable_free: 释放特定引脚上的GPIO 复用。（9）gpio_set_direction: 根据GPIO 配置将引脚设置为输入或输出。（3）struct pinconf_ops 结构体pinconf_ops 结构体定义在内核源码的“include/linux/pinctrl/pinconf.h”文件中，如代码清单123.10 所示。代码清单123.10
```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic; // 是否为通用引脚配置操作
#endif
    int (*pin_config_get) (struct pinctrl_dev *pctldev,
                    unsigned pin,
                    unsigned long *config); // 获取引脚配置信息
    int (*pin_config_set) (struct pinctrl_dev *pctldev,
                    unsigned pin,
                    unsigned long *configs,
                    unsigned num_configs); // 设置引脚配置信息
    int (*pin_config_group_get) (struct pinctrl_dev *pctldev,
                      unsigned selector,
                      unsigned long *config); // 获取引脚组配置信息
    int (*pin_config_group_set) (struct pinctrl_dev *pctldev,
                      unsigned selector,
                      unsigned long *configs,
                      unsigned num_configs); // 设置引脚组配置信息
    int (*pin_config_dbg_parse_modify) (struct pinctrl_dev *pctldev,
                         const char *arg,
                         unsigned long *config); // 解析和修改引脚配置的调试函数
    void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev,
                      struct seq_file *s,
                      unsigned offset); // 调试函数，显示引脚配置信息
    void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev,
                         struct seq_file *s,
                         unsigned selector); // 调试函数，显示引脚组配置信息
    void (*pin_config_config_dbg_show) (struct pinctrl_dev *pctldev,
                          struct seq_file *s,
                          unsigned long config); // 调试函数，显示引脚配置的具体信息
```

```c
};
```
structpinconf_ops 结构体中定义了引脚配置相关函数，。下面是对该结构体成员的详细解释。（1）is_generic: 一个布尔值，表示是否为通用引脚配置操作。在配置文件中定义了CONFIG_GENERIC_PINCONF 时可用。（2）pin_config_get: 获取引脚配置信息。（3）pin_config_set: 设置引脚配置信息。（4）pin_config_group_get: 获取引脚组配置信息。（5）pin_config_group_set: 设置引脚组配置信息。（6）pin_config_dbg_parse_modify: 解析和修改引脚配置的调试函数。（7）pin_config_dbg_show: 显示引脚配置信息的调试函数。（8）pin_config_group_dbg_show: 显示引脚组配置信息的调试函数。（9）pin_config_config_dbg_show: 显示引脚配置具体信息的调试函数。
## 123.4 函数操作集的具体实现
在probe 函数中把struct rockchip_pinctrl *类型的结构体指针类型的变量info 按照rockchip_pinctrl_register()->rockchip_pinctrl_parse_dt()->devm_pinctrl_register()->pinctrl_register()函数调用关系传入到了pinctrl_register()函数，pinctrl_register()函数内容如代码清单123.11 所示。代码清单123.11
```c
struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc,
                     struct device *dev, void *driver_data)
{
    struct pinctrl_dev *pctldev;
    int error;
    // 初始化pinctrl 控制器
    pctldev = pinctrl_init_controller(pctldesc, dev, driver_data);
    if (IS_ERR(pctldev))
        return pctldev;
    // 启用pinctrl 控制器
    error = pinctrl_enable(pctldev);
```

```c
if (error)
    return ERR_PTR(error);
return pctldev;
}
```
在pinctrl_register()函数中struct rockchip_pinctrl *类型的结构体指针类型的变量info 会作为driver_data 函数参数传递到pinctrl_init_controller()函数中，pinctrl_init_controller()函数如代码清单123.12 所示。代码清单123.12
```c
static struct pinctrl_dev *pinctrl_init_controller(struct pinctrl_desc *pctldesc, struct device *dev,void
*driver_data)
{
struct pinctrl_dev *pctldev;
int ret;
if (!pctldesc)
    return ERR_PTR(-EINVAL);
if (!pctldesc->name)
    return ERR_PTR(-EINVAL);
pctldev = kzalloc(sizeof(*pctldev), GFP_KERNEL);
if (!pctldev)
    return ERR_PTR(-ENOMEM);
/* 初始化引脚控制设备结构体*/
pctldev->owner = pctldesc->owner; // 设置所有者
pctldev->desc = pctldesc; // 设置描述符
pctldev->driver_data = driver_data; // 设置驱动程序数据
INIT_RADIX_TREE(&pctldev->pin_desc_tree, GFP_KERNEL); // 初始化引脚描述符树
#ifdef CONFIG_GENERIC_PINCTRL_GROUPS
INIT_RADIX_TREE(&pctldev->pin_group_tree, GFP_KERNEL); // 初始化引脚组树
#endif
#ifdef CONFIG_GENERIC_PINMUX_FUNCTIONS
INIT_RADIX_TREE(&pctldev->pin_function_tree, GFP_KERNEL); // 初始化引脚功能树
#endif
INIT_LIST_HEAD(&pctldev->gpio_ranges); // 初始化GPIO 范围链表
INIT_LIST_HEAD(&pctldev->node); // 初始化节点链表
pctldev->dev = dev; // 设置设备指针
mutex_init(&pctldev->mutex); // 初始化互斥锁
/* 检查核心操作函数的有效性*/
ret = pinctrl_check_ops(pctldev);
```

```c
if (ret) {
    dev_err(dev, "pinctrl ops lacks necessary functions\n");
    goto out_err;
}
/* 如果实现了引脚复用功能，检查操作函数的有效性*/
if (pctldesc->pmxops) {
    ret = pinmux_check_ops(pctldev);
    if (ret)
        goto out_err;
}
/* 如果实现了引脚配置功能，检查操作函数的有效性*/
if (pctldesc->confops) {
    ret = pinconf_check_ops(pctldev);
    if (ret)
        goto out_err;
}
/* 注册所有引脚*/
dev_dbg(dev, "try to register %d pins ...\n",pctldesc->npins);
ret = pinctrl_register_pins(pctldev, pctldesc->pins, pctldesc->npins);
if (ret) {
    dev_err(dev, "error during pin registration\n");
    pinctrl_free_pindescs(pctldev, pctldesc->pins,
                   pctldesc->npins);
    goto out_err;
}
return pctldev;
out_err:
mutex_destroy(&pctldev->mutex);
kfree(pctldev);
return ERR_PTR(ret);
}
```
代码清单123.12 中第19 行代码pctldev->driver_data = driver_data 其右值driver_data 是从probe 函数一步一步传递过来的struct rockchip_pinctrl *类型的结构体指针变量info，左值pctldev 为要注册的引脚控制设备（pin controller device），至此两个数据结构建立起了关联，因此接可以通过pctldev 对rockchip_pinctrl 中的数据进行访问。记住此结论即可。在pinctrl_desc 结构体中的struct pinctrl_ops、struct pinmux_ops 和struct pinconf_ops 结构

体（函数操作集）由原厂工程师实现，以iTOP-RK3568 开发板的SDK 为例。打开内核源码目录下的“ /drivers/pinctrl/pinctrl-rockchip.c ” 文件，在该文件中的rockchip_pinctrl_register()函数中对三个结构体（函数操作集）进行了填充，如代码清单123.13所示。代码清单123.13
```c
ctrldesc->pctlops = &rockchip_pctrl_ops;
ctrldesc->pmxops = &rockchip_pmx_ops;
ctrldesc->confops = &rockchip_pinconf_ops;
```
接下来对每一个函数操作集的具体实现进行讲解。
### 123.4.1 pinctrl_ops 结构体实现
瑞芯微在SDK 源码中实现了其中五个函数，如代码清单123.14 所示。代码清单123.14
```c
static const struct pinctrl_ops rockchip_pctrl_ops = {
    .get_groups_count = rockchip_get_groups_count, // 获取group 数量的函数
    .get_group_name= rockchip_get_group_name,// 获取group 名称的函数
    .get_group_pins= rockchip_get_group_pins,// 获取group 中引脚的函数
    .dt_node_to_map= rockchip_dt_node_to_map,// 将设备树节点转换为引脚控制器映射的函数
    .dt_free_map= rockchip_dt_free_map,// 释放引脚控制器映射资源的函数
};
```
接下来讲解代码清单123.14 中五个函数的具体实现：（1）rockchip_get_groups_count()函数rockchip_get_groups_count() 函数首先使用pinctrl_dev_get_drvdata(pctldev) 函数从pinctrl_dev 设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了ngroups 的值，所以直接返回info->ngroups 的值即可获得groups 的值。如代码清单123.15所示。代码清单123.15
```c
static int rockchip_get_groups_count(struct pinctrl_dev *pctldev)
    {
    // 从pinctrl_dev 结构中获取私有数据指针，将其转换为rockchip_pinctrl 结构
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    // 返回rockchip_pinctrl 结构中存储的引脚组数量
    return info->ngroups;
}
```

（2）rockchip_get_group_name()函数rockchip_get_group_name()函数首先使用pinctrl_dev_get_drvdata(pctldev)函数从pinctrl_dev 设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了groups 的名称，所以直接返回info->groups[selector].name 的值即可获得groups 的名称。如代码清单123.16 所示。代码清单123.16
```c
static const char *rockchip_get_group_name(struct pinctrl_dev *pctldev,
                              unsigned selector)
{
    // 从pinctrl_dev 结构中获取私有数据指针，将其转换为rockchip_pinctrl 结构
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    // 返回指定引脚组的名称
    return info->groups[selector].name;
}
```
（3）rockchip_get_group_pins()函数rockchip_get_group_pins()函数首先使用pinctrl_dev_get_drvdata(pctldev)函数从pinctrl_dev 设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了groups 中的引脚信息，通过代码*pins = info->groups[selector].pins 返回该group 中的引脚的地址。通过代码*npins =info->groups[selector].npins 返回该group 中包含的引脚数量。如代码清单123.17 所示。代码清单123.17
```c
static int rockchip_get_group_pins(struct pinctrl_dev *pctldev,
                       unsigned selector, const unsigned **pins,
                       unsigned *npins)
{
    // 从pinctrl_dev 结构中获取私有数据指针，将其转换为rockchip_pinctrl 结构
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    // 如果超出group 的范围，则返回错误码-EINVAL
    if (selector >= info->ngroups)
        return -EINVAL;
    // 将指向group 中的引脚的指针赋值给传入的pins 指针
    *pins = info->groups[selector].pins;
    // 将group 中的引脚数量赋值给传入的npins 变量
    *npins = info->groups[selector].npins;
    // 返回成功的状态码0
    return 0;
}
```

（4）rockchip_dt_node_to_map(）函数rockchip_dt_node_to_map()是pinctrl 中的重要函数之一，该函数可以把一个function 对应的节点转换成pinctrl_map 结构体。如代码清单123.18 所示。代码清单123.18
```c
static int rockchip_dt_node_to_map(struct pinctrl_dev *pctldev,
                  struct device_node *np,
                  struct pinctrl_map **map, unsigned *num_maps)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev); // 获取引脚控制器的私有数据指针
    const struct rockchip_pin_group *grp; // 引脚组指针
    struct device *dev = info->dev; // 设备指针
    struct pinctrl_map *new_map;
    struct device_node *parent; // 父节点指针
    int map_num = 1; // 映射数量，默认为1
    int i;
    /* 查找group */
    grp = pinctrl_name_to_group(info, np->name); // 根据节点名称查找对应的group
    if (!grp) {
        dev_err(dev, "unable to find group for node %pOFn\n", np); // 如果找不到group，打印错误信息
        return -EINVAL;
    }
    map_num += grp->npins; // 计算要转换的数量。
    new_map = kcalloc(map_num, sizeof(*new_map), GFP_KERNEL); // 分配内存空间用于存储转换后的
pinctrl_map
    if (!new_map)
        return -ENOMEM;
    *map = new_map;
    *num_maps = map_num;
    parent = of_get_parent(np); // 获取节点的父节点
    if (!parent) {
        kfree(new_map); // 如果父节点不存在，释放内存空间
        return -EINVAL;
    }
    new_map[0].type = PIN_MAP_TYPE_MUX_GROUP; // 设置类型为复用类型
    new_map[0].data.mux.function = parent->name; // 复用功能名称为父节点的名称
    new_map[0].data.mux.group = np->name; // 设置gpoup 名称为节点的名称
    of_node_put(parent); // 释放父节点的引用计数
    /* 创建pinctrl_map 结构体*/
```

```c
new_map++; /指针向后移动一个位置
for (i = 0; i < grp->npins; i++) {
    new_map[i].type = PIN_MAP_TYPE_CONFIGS_PIN; // 设置类型为配置映射
    new_map[i].data.configs.group_or_pin =
        pin_get_name(pctldev, grp->pins[i]); // group 或引脚名称为group 组中的引脚名称
    new_map[i].data.configs.configs = grp->data[i].configs; // 配置信息数组为group 中该引脚的配置信
息
    new_map[i].data.configs.num_configs = grp->data[i].nconfigs; // 配置信息数量为group 中该引脚的
配置数量
}
dev_dbg(dev, "maps: function %s group %s num %d\n",
    (*map)->data.mux.function, (*map)->data.mux.group, map_num); // 打印调试信息
return 0; // 返回成功标志
}
```
（5）rockchip_dt_free_map()函数rockchip_dt_free_map()函数用来释放pinctrl_map。如代码清单123.19 所示。代码清单123.19
```c
static void rockchip_dt_free_map(struct pinctrl_dev *pctldev,
                     struct pinctrl_map *map, unsigned num_maps)
{
    kfree(map);// 释放pinctrl_map。
}
```
### 123.4.2 pinmux_ops 结构体实现
瑞芯微在SDK 源码中实现了其中四个函数，如代码清单123.20 所示。代码清单123.20
```c
static const struct pinmux_ops rockchip_pmx_ops = {
    .get_functions_count= rockchip_pmx_get_funcs_count,// 获取引脚复用功能数量的函数
    .get_function_name = rockchip_pmx_get_func_name,// 获取引脚复用功能名称的函数
    .get_function_groups= rockchip_pmx_get_groups,// 获取引脚复用功能对应的group 的函数
    .set_mux= rockchip_pmx_set,// 设置引脚复用功能的函数
};
```
接下来讲解代码清单123.20 中四个函数的具体实现。（1）rockchip_pmx_get_funcs_count()函数rockchip_pmx_get_funcs_count()函数首先使用pinctrl_dev_get_drvdata(pctldev)函数从pinctrl_dev设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了nfunctions 的值，所以直接返回info->nfunc 的值即可获得nfunctions 的值。如代码清单123.21 所示。

代码清单123.21
```c
static int rockchip_pmx_get_funcs_count(struct pinctrl_dev *pctldev)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    return info->nfunctions;// 返回引脚复用功能的数量
}
```
（2）rockchip_pmx_get_func_name()函数rockchip_pmx_get_func_name()函数首先使用pinctrl_dev_get_drvdata(pctldev)函数从pinctrl_dev 设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了function 的name 的值，所以直接返回info->functions[selector].name 的值即可。如代码清单123.22 所示。代码清单123.22
```c
static const char *rockchip_pmx_get_func_name(struct pinctrl_dev *pctldev,
                       unsigned selector)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    return info->functions[selector].name;// 返回引脚复用功能的名称
}
```
（3）rockchip_pmx_get_groups()函数rockchip_pmx_get_groups() 函数首先使用pinctrl_dev_get_drvdata(pctldev) 函数从pinctrl_dev 设备结构中获得rockchip_pinctrl 结构体类型的指针变量info。指针变量info 中存放了groups 和ngroups 的值，所以直接返回值即可。如代码清单123.23 所示。代码清单123.23
```c
static int rockchip_pmx_get_groups(struct pinctrl_dev *pctldev,
                 unsigned selector, const char * const **groups,
                 unsigned * const num_groups)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    *groups = info->functions[selector].groups;// 返回引脚复用功能对应的引脚组数组
    *num_groups = info->functions[selector].ngroups;// 返回引脚组的数量
    return 0;// 返回成功
}
```
（4）rockchip_pmx_set()函数rockchip_pmx_set()函数中会调用rockchip_set_mux()函数设置引脚复用。如代码清单123.24 所示。代码清单123.24
```c
static int rockchip_pmx_set(struct pinctrl_dev *pctldev, unsigned selector,
            unsigned group)
```

```c
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    const unsigned int *pins = info->groups[group].pins;
    const struct rockchip_pin_config *data = info->groups[group].data;
    struct device *dev = info->dev;
    struct rockchip_pin_bank *bank;
    int cnt, ret = 0;
    dev_dbg(dev, "enable function %s group %s\n",
        info->functions[selector].name, info->groups[group].name);
    /*
     * 针对所选的引脚组中的每个引脚，将相应的引脚功能号码编程到配置寄存器中。
     */
    for (cnt = 0; cnt < info->groups[group].npins; cnt++) {
        bank = pin_to_bank(info, pins[cnt]);
        ret = rockchip_set_mux(bank, pins[cnt] - bank->pin_base,
                        data[cnt].func);
        if (ret)
             break;
    }
    if (ret && cnt) {
        /* 恢复已经设置的引脚设置*/
        for (cnt--; cnt >= 0 && !data[cnt].func; cnt--)
             rockchip_set_mux(bank, pins[cnt] - bank->pin_base, 0);
        return ret;
    }
    return 0;// 返回成功
}
```
### 123.4.3 pinconf_ops 结构体实现
瑞芯微在SDK 源码中实现了其中两个函数，如代码清单123.25 所示。代码清单123.25
```c
static const struct pinconf_ops rockchip_pinconf_ops = {
    .pin_config_get= rockchip_pinconf_get, // 获取引脚配置的函数
    .pin_config_set= rockchip_pinconf_set,// 设置引脚配置的函数
    .is_generic= true,// 表示是通用的引脚配置操作
};
```

接下来讲解代码清单123.25 中的这俩个函数。（1）rockchip_pinconf_set()函数rockchip_pinconf_set()函数是设置引脚配置函数，如代码清单123.26 所示。代码清单123.26
```c
static int rockchip_pinconf_set(struct pinctrl_dev *pctldev, unsigned int pin,
                 unsigned long *configs, unsigned num_configs)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    struct rockchip_pin_bank *bank = pin_to_bank(info, pin);
    struct gpio_chip *gpio = &bank->gpio_chip;
    enum pin_config_param param;
    u32 arg;
    int i;
    int rc;
    for (i = 0; i < num_configs; i++) {
        param = pinconf_to_config_param(configs[i]);
        arg = pinconf_to_config_argument(configs[i]);
        if (param == PIN_CONFIG_OUTPUT || param == PIN_CONFIG_INPUT_ENABLE) {
             /*
              * 检查GPIO 驱动程序是否已经探测到。
              * 锁定确保GPIO 探测完成或者GPIO 驱动程序尚未探测到。
              */
             mutex_lock(&bank->deferred_lock);
             if (!gpio || !gpio->direction_output) {
                 // 如果驱动程序尚未探测到，则将配置信息延迟处理并返回。
                 rc = rockchip_pinconf_defer_pin(bank, pin - bank->pin_base, param, arg);
                 mutex_unlock(&bank->deferred_lock);
                 if (rc)
                     return rc;
                 break;
             }
             mutex_unlock(&bank->deferred_lock);
        }
        switch (param) {
        case PIN_CONFIG_BIAS_DISABLE:
             // 禁用上下拉电阻
             rc = rockchip_set_pull(bank, pin - bank->pin_base, param);
             if (rc)
```

```c
return rc;
break;
case PIN_CONFIG_BIAS_PULL_UP:
case PIN_CONFIG_BIAS_PULL_DOWN:
case PIN_CONFIG_BIAS_PULL_PIN_DEFAULT:
case PIN_CONFIG_BIAS_BUS_HOLD:
// 检查上下拉电阻是否有效
if (!rockchip_pinconf_pull_valid(info->ctrl, param))
return -ENOTSUPP;
if (!arg)
return -EINVAL;
// 设置上下拉电阻
rc = rockchip_set_pull(bank, pin - bank->pin_base, param);
if (rc)
return rc;
break;
case PIN_CONFIG_OUTPUT:
// 设置引脚复用功能为GPIO
rc = rockchip_set_mux(bank, pin - bank->pin_base, RK_FUNC_GPIO);
if (rc != RK_FUNC_GPIO)
return -EINVAL;
// 设置引脚为输出模式
rc = gpio->direction_output(gpio, pin - bank->pin_base, arg);
if (rc)
return rc;
break;
case PIN_CONFIG_INPUT_ENABLE:
// 设置引脚复用功能为GPIO
rc = rockchip_set_mux(bank, pin - bank->pin_base, RK_FUNC_GPIO);
if (rc != RK_FUNC_GPIO)
return -EINVAL;
// 设置引脚为输入模式
rc = gpio->direction_input(gpio, pin - bank->pin_base);
if (rc)
return rc;
break;
case PIN_CONFIG_DRIVE_STRENGTH:
// 仅支持某些芯片（如rk3288）的每个引脚独立的驱动强度设置
if (!info->ctrl->drv_calc_reg)
return -ENOTSUPP;
```

```c
// 设置引脚的驱动强度
rc = rockchip_set_drive_perpin(bank, pin - bank->pin_base, arg);
if (rc < 0)
    return rc;
break;
case PIN_CONFIG_INPUT_SCHMITT_ENABLE:
// 仅支持某些芯片的施密特触发设置
if (!info->ctrl->schmitt_calc_reg)
    return -ENOTSUPP;
// 设置引脚的施密特触发模式
rc = rockchip_set_schmitt(bank, pin - bank->pin_base, arg);
if (rc < 0)
    return rc;
break;
case PIN_CONFIG_SLEW_RATE:
// 仅支持某些芯片的引脚驱动速率设置
if (!info->ctrl->slew_rate_calc_reg)
    return -ENOTSUPP;
// 设置引脚的驱动速率
rc = rockchip_set_slew_rate(bank, pin - bank->pin_base, arg);
if (rc < 0)
    return rc;
break;
default:
// 不支持的配置参数
return -ENOTSUPP;
}
} /* for each config */
return 0;
}
```
（2）rockchip_pinconf_get()函数rockchip_pinconf_get()函数是获取指定引脚配置函数，如代码清单123.27 所示。代码清单123.27
```c
static int rockchip_pinconf_get(struct pinctrl_dev *pctldev, unsigned int pin,
                 unsigned long *config)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);
    struct rockchip_pin_bank *bank = pin_to_bank(info, pin);
    struct gpio_chip *gpio = &bank->gpio_chip;
```

```c
enum pin_config_param param = pinconf_to_config_param(*config);
u16 arg;
int rc;
switch (param) {
case PIN_CONFIG_BIAS_DISABLE:
    // 检查上下拉电阻是否禁用
    if (rockchip_get_pull(bank, pin - bank->pin_base) != param)
        return -EINVAL;
    arg = 0;
    break;
case PIN_CONFIG_BIAS_PULL_UP:
case PIN_CONFIG_BIAS_PULL_DOWN:
case PIN_CONFIG_BIAS_PULL_PIN_DEFAULT:
case PIN_CONFIG_BIAS_BUS_HOLD:
    // 检查上下拉电阻是否有效，并获取当前的上下拉电阻配置
    if (!rockchip_pinconf_pull_valid(info->ctrl, param))
        return -ENOTSUPP;
    if (rockchip_get_pull(bank, pin - bank->pin_base) != param)
        return -EINVAL;
    arg = 1;
    break;
case PIN_CONFIG_OUTPUT:
    // 检查引脚是否配置为GPIO 输出模式
    rc = rockchip_get_mux(bank, pin - bank->pin_base);
    if (rc != RK_FUNC_GPIO)
        return -EINVAL;
    if (!gpio || !gpio->get) {
        arg = 0;
        break;
    }
    // 获取引脚的输出状态
    rc = gpio->get(gpio, pin - bank->pin_base);
    if (rc < 0)
        return rc;
    arg = rc ? 1 : 0;
    break;
case PIN_CONFIG_DRIVE_STRENGTH:
```

```c
// 仅支持某些芯片（如rk3288）的每个引脚独立的驱动强度设置
if (!info->ctrl->drv_calc_reg)
    return -ENOTSUPP;
// 获取引脚的驱动强度配置
rc = rockchip_get_drive_perpin(bank, pin - bank->pin_base);
if (rc < 0)
    return rc;
arg = rc;
break;
case PIN_CONFIG_INPUT_SCHMITT_ENABLE:
// 仅支持某些芯片的施密特触发设置
if (!info->ctrl->schmitt_calc_reg)
    return -ENOTSUPP;
// 获取引脚的施密特触发配置
rc = rockchip_get_schmitt(bank, pin - bank->pin_base);
if (rc < 0)
    return rc;
arg = rc;
break;
case PIN_CONFIG_SLEW_RATE:
// 仅支持某些芯片的引脚驱动速率设置
if (!info->ctrl->slew_rate_calc_reg)
    return -ENOTSUPP;
// 获取引脚的驱动速率配置
rc = rockchip_get_slew_rate(bank, pin - bank->pin_base);
if (rc < 0)
    return rc;
arg = rc;
break;
default:
// 不支持的配置参数
return -ENOTSUPP;
}
*config = pinconf_to_config_packed(param, arg);
return 0;
}
```

