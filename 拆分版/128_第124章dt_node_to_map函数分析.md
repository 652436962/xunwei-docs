# 第124章dt_node_to_map函数分析

本章分析dt_node _to_map 函数。
## 124.1 pinctrl_map 结构体
代码清单124.1 中设备树里的function 和group 节点只是描述性信息，本身没有任何作用，只有当驱动程序使用这些信息时候才会起作用。驱动程序会把代码清单124.1中描述的function和group 节点信息通过dt_node_to_map 函数转换成pinctrl_map 结构体,或者说被映射成pinctrl_map 结构体。代码清单124.1
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
                rockchip,pins =
                    /* can0_rxm1 */
                    <2 RK_PA2 4 &pcfg_pull_none>,
                    /* can0_txm1 */
                    <2 RK_PA1 4 &pcfg_pull_none>;
            };
        };
    ......
};
```
struct pinctrl_map 结构体定义在内核源码的“include/linux/pinctrl/machine.h”目录下，如代码清单124.2 所示。代码清单124.2
```c
struct pinctrl_map {
```

```c
const char *dev_name;// 设备名称
const char *name;// 映射名称
enum pinctrl_map_type type; // 映射类型
const char *ctrl_dev_name;// 控制设备名称
//特别注意，这里是union 类型，说明pinctrl_map 结构体可以保存/映射复用信息或者是配置信息
union {
    struct pinctrl_map_mux mux;// 复用数据
    struct pinctrl_map_configs configs; // 配置数据
} data;
};
```
1. 结构体成员type 取值type 成员可以有如下取值：PIN_MAP_TYPE_INVALID：无效的映射类型。PIN_MAP_TYPE_DUMMY_STATE：虚拟映射（不需要对具体硬件操作时使用，比如默认状态的空实现）引脚进行复用功能配置。PIN_MAP_TYPE_MUX_GROUP：引脚组复用功能映射类型。PIN_MAP_TYPE_CONFIGS_PIN：单个引脚电气属性配置映射类型。PIN_MAP_TYPE_CONFIGS_GROUP：引脚组电气属性配置映射类型。2. 联合体中struct pinctrl_map_mux 结构体原型struct pinctrl_map_mux 结构体原型如代码清单124.3 所示。代码清单124.3
```c
struct pinctrl_map_mux {
    const char *group;
    const char *function;
};
```
3.联合体中struct pinctrl_map_configs 结构体原型struct pinctrl_map_configs 结构体原型如代码清单124.4 所示.代码清单124.4
```c
struct pinctrl_map_configs {
    const char *group_or_pin;
    unsigned long *configs;
    unsigned num_configs;
};
```

## 124.2 dt_node_to_map 函数分析
dt_node_to_map 函数在pinctrl_ops 函数操作集中已经进行了简单的讲解。该函数一般由芯片原厂在SDK 中实现，如iTIP-RK356 开发板的SDK 中的rockchip_dt_node_to_map()函数。rockchip_dt_node_to_map()函数函数的具体实现在内核源码的“drivers/pinctrl/pinctrl-rockchip.c”文件中，rockchip_dt_node_to_map()函数如代码清单124.5所示。代码清单124.5
```c
static int rockchip_dt_node_to_map(struct pinctrl_dev *pctldev,
                  struct device_node *np,
                  struct pinctrl_map **map, unsigned *num_maps)
{
    struct rockchip_pinctrl *info = pinctrl_dev_get_drvdata(pctldev); // 获取pctldev 的私有数据指针
    const struct rockchip_pin_group *grp;
    struct device *dev = info->dev;
    struct pinctrl_map *new_map; // 用来保存被映射的pinctrl_map
    struct device_node *parent;
    int map_num = 1;
    int i;
    /* 通过节点名称获取对应的group，本质获取了struct rockchip_pinctrl *info,结构体中的group，因为这
个结构体中保存了group，func 等这些信息*/
    grp = pinctrl_name_to_group(info, np->name);
    if (!grp) {
        dev_err(dev, "unable to find group for node %pOFn\n", np); // 如果找不到group，打印错误信息
        return -EINVAL;
    }
    map_num += grp->npins; // 把group 中的引脚数量保存到map_num 中
    new_map = kcalloc(map_num, sizeof(*new_map), GFP_KERNEL); // 分配内存空间用于存储映射的
pinctrl_map
    if (!new_map)
        return -ENOMEM;
    *map = new_map; // 将分配好内存的pinctrl_map 赋值给输出参数
    *num_maps = map_num; // 将映射数量,即pinctrl_map 数量赋值给输出参数
    /* 创建复用映射*/
    parent = of_get_parent(np); // 获取节点的父节点，即function
    if (!parent) {
        kfree(new_map);
```

```c
return -EINVAL;
}
new_map[0].type = PIN_MAP_TYPE_MUX_GROUP; // 设置映射类型为复用映射
new_map[0].data.mux.function = parent->name; // 复用功能名称为父节点的名称，即function 的名称
new_map[0].data.mux.group = np->name; //group 名称为节点的名称，即group 的名称
of_node_put(parent); // 释放父节点的引用计数
/* 创建配置映射*/
new_map++; // 指针向后移动一个位置，当成数组去操作
for (i = 0; i < grp->npins; i++) {
new_map[i].type = PIN_MAP_TYPE_CONFIGS_PIN; // 设置映射类型为配置映射
new_map[i].data.configs.group_or_pin =
    pin_get_name(pctldev, grp->pins[i]); //group 或引脚名称为group 中的引脚名称
new_map[i].data.configs.configs = grp->data[i].configs; // 配置信息数组为group 中该引脚的配置信
息
new_map[i].data.configs.num_configs = grp->data[i].nconfigs; // 配置信息数量为group 中该引脚的
配置数量
}
dev_dbg(dev, "maps: function %s group %s num %d\n",
(*map)->data.mux.function, (*map)->data.mux.group, map_num); // 打印调试信息，显示创建的引脚
映射的功能名称、组名和数量
return 0; // 返回成功标志
}
```
以代码清单124.6 中的function 和group 为例讲解dt_node_to_map 函数。代码清单124.6
```c
rk_485{
    rk_485_gpio:rk-485-gpio {
        rockchip,pins = <3 13 RK_FUNC_GPIO &pcfg_pull_none>;
    };
};
```
在rockchip_dt_node_to_map()函数中“第14-19 行：根据设备节点的名称使用pinctrl_name_to_group()函数查找与该节点对应的group。如果找不到group，则函数打印错误消息并返回EINVAL 错误代码。这里查找到的group即代码清单1.33 中的rk-485-gpio。第20 行：函数根据group 中的引脚数量计算需要创建的映射数量（即pinctrl_map 结构体的数量）。映射数量包括复用映射和配置映射。代码清单1.33 中group 中引脚数量为1，而map_num 变量默认为1，所以map_num += grp->npins 的计算结果为2。为什么map_num 变量

默认为1 呢，因为pinctrl_map[0]是配置group 的所有引脚复用。pinctrl_map[1~n]是配置group下的每个引脚的属性。第22-24 行：使用kcalloc()函数为映射数组（即pinctrl_map 数组）分配内存空间。如果内存分配失败，函数将返回ENOMEM 错误代码。（因为后面当成了数组去操作，所以这里称呼分配的地址为映射数组）第26-27 行：将分配的映射数组（即pinctrl_map 数组）和映射数量（map_num）通过函数参数map 和num_maps 返回给调用者。第30-38 行：获取设备树中group 节点的父节点，即function 节点，并把function 节点名称作为复用功能名称。即代码清单1.33 中的rk_485 节点。然后设置第一个映射（即pinctrl_map[0]）的类型为PIN_MAP_TYPE_MUX_GROUP，并记录该复用功能的名称（即function 名称）和group 名称。最后，使用of_node_put()函数释放父节点的引用计数。第41-58 行：函数遍历group 中的引脚，并为每个引脚创建一个配置映射（即pinctrl_map）。映射的类型为PIN_MAP_TYPE_CONFIGS_PIN，同时，将group 中该引脚的配置信息和配置信息数量以及名称设置为映射后的配置数据。

