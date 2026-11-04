# 第127章 通过pinctrl状态设置引脚复用实验

本章分析pinctrl_bind_pins()函数从而讲解驱动是如何根据pinctrl 节点的设置去设置引脚复用。
## 127.1pinctrl_bind_pins 函数分析
pinctrl_bind_pins()函数如代码清单127.1 所示。代码清单127.1
```c
int pinctrl_bind_pins(struct device *dev)
{
    int ret;
    // 检查设备是否重用了节点
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
    }
    // 查找设备的默认pinctrl 状态
    dev->pins->default_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_DEFAULT);
    if (IS_ERR(dev->pins->default_state)) {
        dev_dbg(dev, "没有默认的pinctrl 状态\n");
        ret = 0;
        goto cleanup_get;
    }
    // 查找设备的初始化pinctrl 状态
```

```c
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
return 0;
/*
 * 如果对于此设备没有找到pinctrl 句柄或默认状态，
 * 明确释放设备中的引脚容器，因为保留它没有意义。
 */
cleanup_get:
devm_pinctrl_put(dev->pins->p);
cleanup_alloc:
devm_kfree(dev, dev->pins);
dev->pins = NULL;
```

```c
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
pinctrl_bind_pins()函数中第22-28 行代码使用pinctrl_lookup_state()函数查找设备的默认状态，即pinctrl-name 属性值为default 时对应的设置。pinctrl_lookup_state()函数定义在内核源码目录下的“drivers/pinctrl/core.c”文件中，如代码清单127.2 所示。代码清单127.2
```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                       const char *name)
{
struct pinctrl_state *state;
// 在状态链表中查找指定名称的状态对象
state = find_state(p, name);
if (!state) {
    if (pinctrl_dummy_state) {
        /* 创建虚拟状态*/
        dev_dbg(p->dev, "使用pinctrl 虚拟状态(%s)\n",
             name);
        // 如果找不到指定的状态对象，并且存在虚拟状态，则创建一个虚拟状态对象
        state = create_state(p, name);
    } else
        // 如果找不到指定的状态对象，并且不存在虚拟状态，则返回错误指针-ENODEV
        state = ERR_PTR(-ENODEV);
}
return state;
}
```
pinctrl_bind_pins()函数中第30-41 行代码使用pinctrl_lookup_state()函数查找设备的init 状态，即pinctrl-name 属性值为init 时对应的设置。pinctrl_bind_pins()函数中第48-56 行代码使用pinctrl_lookup_state()函数查找设备的sleep

状态，即pinctrl-name 属性值为sleep 时对应的设置。pinctrl_bind_pins()函数中第58-64 行代码使用pinctrl_lookup_state()函数查找设备的idle 状态，即pinctrl-name 属性值为idle 时对应的设置。pinctrl_bind_pins()函数中第37 行和第40 行会使用pinctrl_select_state ()函数选择并切换到指定的状态，该函数定义在内核源码目录下的“drivers/pinctrl/core.c”文件中，如代码清单127.3 所示。代码清单127.3
```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
{
    // 如果当前状态已经是要选择的状态，则无需进行任何操作，直接返回0 表示成功
    if (p->state == state)
        return 0;
    // 调用pinctrl_commit_state 函数切换状态
    return pinctrl_commit_state(p, state);
}
```
在pinctrl_select_state ()函数中调用的pinctrl_commit_state()函数切换到新的状态，该函数如代码清单127.4 所示。代码清单127.4
```c
static int pinctrl_commit_state(struct pinctrl *p, struct pinctrl_state *state)
{
    struct pinctrl_setting *setting, *setting2;
    struct pinctrl_state *old_state = p->state;
    int ret;
    if (p->state) {
        /*
         * 对于旧状态中的每个引脚复用设置，取消SW 记录的该引脚组的复用所有者。
         * 任何仍由新状态拥有的引脚组将在下面循环中的pinmux_enable_setting() 调用中重新获取。
         */
        list_for_each_entry(setting, &p->state->settings, node) {
             if (setting->type != PIN_MAP_TYPE_MUX_GROUP)
                 continue;
             pinmux_disable_setting(setting);
        }
    }
    p->state = NULL;
```

```c
/* 应用新状态的所有设置*/
list_for_each_entry(setting, &state->settings, node) {
    switch (setting->type) {
    case PIN_MAP_TYPE_MUX_GROUP:
        ret = pinmux_enable_setting(setting);
        break;
    case PIN_MAP_TYPE_CONFIGS_PIN:
    case PIN_MAP_TYPE_CONFIGS_GROUP:
        ret = pinconf_apply_setting(setting);
        break;
    default:
        ret = -EINVAL;
        break;
    }
    if (ret < 0) {
        // 如果应用设置失败，则回滚新状态的设置
        goto unapply_new_state;
    }
}
p->state = state;
return 0;
unapply_new_state:
// 回滚新状态的设置
list_for_each_entry_safe(setting, setting2, &state->settings, node) {
    switch (setting->type) {
    case PIN_MAP_TYPE_MUX_GROUP:
        pinmux_disable_setting(setting);
        break;
    case PIN_MAP_TYPE_CONFIGS_PIN:
    case PIN_MAP_TYPE_CONFIGS_GROUP:
        pinconf_remove_setting(setting);
        break;
    }
}
// 回滚完成后，将状态恢复为旧状态
p->state = old_state;
return ret;
}
```

pinctrl_commit_state()函数的核心操作是第22-42 行代码，第22-42 行代码遍历新状态的所有设置，并根据设置的类型执行相应的操作：1.对于PIN_MAP_TYPE_MUX_GROUP 类型调用pinmux_enable_setting()函数来启用该设置。2.对于PIN_MAP_TYPE_CONFIGS_PIN 或PIN_MAP_TYPE_CONFIGS_GROUP 类型调用pinconf_apply_setting()函数来应用该设置。3.对于其他类型的设置，返回错误码。然后通过代码p->state = state 记录这个状态。其中pinmux_enable_setting 函数定义在内核源码目录下的“drivers/pinctrl/pinmux.c”文件中，如代码清单127.5 所示。代码清单127.5
```c
int pinmux_enable_setting(const struct pinctrl_setting *setting)
{
    // 获取相关结构体指针
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinctrl_ops *pctlops = pctldev->desc->pctlops;
    const struct pinmux_ops *ops = pctldev->desc->pmxops;
    int ret = 0;
    const unsigned *pins = NULL;
    unsigned num_pins = 0;
    int i;
    struct pin_desc *desc;
    // 如果pctlops->get_group_pins 函数存在，则调用该函数获取组中的引脚信息
    if (pctlops->get_group_pins)
        ret = pctlops->get_group_pins(pctldev, setting->data.mux.group,&pins, &num_pins);
    if (ret) {
        const char *gname;
        gname = pctlops->get_group_name(pctldev,setting->data.mux.group);
        dev_warn(pctldev->dev, "could not get pins for group %s\n",gname);
        num_pins = 0;
    }
    // 逐个申请组中的引脚
    for (i = 0; i < num_pins; i++) {
        ret = pin_request(pctldev, pins[i], setting->dev_name, NULL);
        if (ret) {
            const char *gname;
```

```c
const char *pname;
desc = pin_desc_get(pctldev, pins[i]);
pname = desc ? desc->name : "non-existing";
gname = pctlops->get_group_name(pctldev,setting->data.mux.group);
dev_err(pctldev->dev,
        "could not request pin %d (%s) from group %s "
        " on device %s\n",
        pins[i], pname, gname,
        pinctrl_dev_get_name(pctldev));
goto err_pin_request;
}
}
for (i = 0; i < num_pins; i++) {
desc = pin_desc_get(pctldev, pins[i]);
if (desc == NULL) {
dev_warn(pctldev->dev,
         "could not get pin desc for pin %d\n",
         pins[i]);
continue;
}
desc->mux_setting = &(setting->data.mux);
}
// 调用ops->set_mux 函数设置复用
ret = ops->set_mux(pctldev, setting->data.mux.func,setting->data.mux.group);
if (ret)
goto err_set_mux;
return 0;
err_set_mux:
// 复用设置失败，清除复用设置
for (i = 0; i < num_pins; i++) {
desc = pin_desc_get(pctldev, pins[i]);
if (desc)
desc->mux_setting = NULL;
}
err_pin_request:
// 在错误发生时释放已申请的引脚
while (--i >= 0)
```

```c
pin_free(pctldev, pins[i], NULL);
return ret;
}
```
pinconf_apply_setting()函数定义在内核源码目录下的“drivers/pinctrl/pinconf.c”文件中，如代码清单127.6 所示。代码清单127.6
```c
int pinconf_apply_setting(const struct pinctrl_setting *setting)
{
        // 获取相关结构体指针
        struct pinctrl_dev *pctldev = setting->pctldev;
        const struct pinconf_ops *ops = pctldev->desc->confops;
        int ret;
        // 检查是否存在pinconf 操作函数集
        if (!ops) {
            dev_err(pctldev->dev, "missing confops\n");
            return -EINVAL;
        }
        // 根据设置类型选择相应的操作
        switch (setting->type) {
            case PIN_MAP_TYPE_CONFIGS_PIN:
                // 检查是否存在pin_config_set 操作函数
                if (!ops->pin_config_set) {
                    dev_err(pctldev->dev, "missing pin_config_set op\n");
                    return -EINVAL;
                }
                // 调用pin_config_set 函数设置单个引脚的配置
                ret = ops->pin_config_set(pctldev,
                                           setting->data.configs.group_or_pin,
                                           setting->data.configs.configs,
                                           setting->data.configs.num_configs);
                if (ret < 0) {
                    dev_err(pctldev->dev,
                             "pin_config_set op failed for pin %d\n",
                             setting->data.configs.group_or_pin);
                    return ret;
                }
                break;
```

```c
case PIN_MAP_TYPE_CONFIGS_GROUP:
    // 检查是否存在pin_config_group_set 操作函数
    if (!ops->pin_config_group_set) {
        dev_err(pctldev->dev, "missing pin_config_group_set op\n");
        return -EINVAL;
    }
    // 调用pin_config_group_set 函数设置引脚组的配置
    ret = ops->pin_config_group_set(pctldev,
                                     setting->data.configs.group_or_pin,
                                     setting->data.configs.configs,
                                     setting->data.configs.num_configs);
    if (ret < 0) {
        dev_err(pctldev->dev,
                "pin_config_group_set op failed for group %d\n",
                setting->data.configs.group_or_pin);
        return ret;
    }
    break;
default:
    return -EINVAL;
}
return 0;
}
```
pinmux_enable_setting()函数和pinconf_apply_setting()函数本质都是调用SDK 中原厂实现的函数操作集中的函数去设置引脚复用，这里不在赘述。

