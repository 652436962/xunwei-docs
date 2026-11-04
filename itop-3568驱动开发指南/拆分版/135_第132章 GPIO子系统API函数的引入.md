# 第132章 GPIO子系统API函数的引入

事实上，在前面中断课程中，已经简单接触到了GPIO 子系统中的API 函数，如用来获取gpio 中断编号的gpio_to_irq()函数就属于GPIO 子系统中的API 函数。目前GPIO 子系统的API 函数存在两个版本，这里将两个版本区分为新版本和旧版本。新版本GPIO 子系统的API 函数是基于descriptor-based 来实现的，而旧版本的GPIO 子系统的API函数是基于integer-based 来实现的，Linux 内核为了保持向下的兼容性，旧版本的接口在最新的内核版本中仍然得到支持，而随着时间的推移，新版本的GPIO 子系统的API 函数会越来越完善，最终完全取代旧版本，所以在本课程中主要讲解新版本的GPIO 子系统的API 函数。新版本的GPIO 子系统的API 函数需要与设备树结合使用。使用设备树和新版本的GPIO 子系统的API 函数可以更加灵活地配置和管理系统中的GPIO 资源，提供了更好的可扩展性和可移植性。所以如果没有设备树，就无法使用新版本的GPIO 子系统的API 函数。那要如何对新旧GPIO 子系统的API 函数进行区分呢，一个明显的区别是新版本的GPIO 子系统API 函数使用以"gpiod_"作为前缀的函数命名约定，而旧版本的GPIO 子系统API 函数使用以"gpio_"作为前缀的函数命名约定。如代码清单132.1 和代码清单132.2 所示。新版本GPIO 子系统API 函数示例如代码清单132.1 所示。代码清单132.1
```c
gpiod_set_value()
gpiod_direction_input()
gpiod_direction_output()
gpiod_get_value()
gpiod_request()
```
旧版本GPIO 子系统API 函数示例如代码清单132.2 所示。代码清单132.2
```c
gpio_set_value()
gpio_direction_input()
gpio_direction_output()
gpio_get_value()
gpio_request()
```
新版本GPIO 子系统API 函数是基于descriptor-based 来实现的，使用struct gpio_desc 结构体表示一个GPIO，该结构体定义在内核源码drivers/gpio/gpiolib.h 文件下，具体内容如代码清单132.3 所示。

代码清单132.3
```c
struct gpio_desc {
    struct gpio_device gdev;// GPIO 设备结构体
    unsigned long flags;// 标志位，用于表示不同的属性
    /* 标志位符号对应的位号*/
    #define FLAG_REQUESTED 0// GPIO 已请求
    #define FLAG_IS_OUT 1// GPIO 用作输出
    #define FLAG_EXPORT 2// 受sysfs_lock 保护的导出标志
    #define FLAG_SYSFS 3// 通过/sys/class/gpio/control 导出的标志
    #define FLAG_ACTIVE_LOW 6// GPIO 值为低电平时激活
    #define FLAG_OPEN_DRAIN 7// GPIO 为开漏类型
    #define FLAG_OPEN_SOURCE 8// GPIO 为开源类型
    #define FLAG_USED_AS_IRQ 9// GPIO 连接到中断请求（IRQ）
    #define FLAG_IS_HOGGED 11// GPIO 被独占占用
    #define FLAG_TRANSITORY 12// GPIO 在休眠或复位时可能失去值
    /*连接标签*/
    const char *label;
    /*GPIO 的名称*/
    const char *name;
};
```
代码清单132.3 中需要重点关注的是然后struct gpio_device 结构体，该结构体同样定义在内核源码的drivers/gpio/gpiolib.h 文件中，具体内容如代码清单132.4 所示。代码清单132.4
```c
struct gpio_device {
    int id;// GPIO 设备ID
    struct device *dev;//对应的设备结构体指针
    struct cdev chrdev;//字符设备结构体
    struct device *mockdev;//模拟设备结构体指针
    struct module *owner;//拥有该GPIO 设备的内核模块指针
    struct gpio_chip *chip;//对应的GPIO 芯片结构体指针
    struct gpio_desc *descs;// GPIO 描述符数组指针
    int base;// GPIO 编号的起始值
    u16 ngpio;// GPIO 的数量
    const char *label;// GPIO 设备的标签
    void *data;//与GPIO 设备相关的数据指针
    struct list_head list;//用于将GPIO 设备结构体连接到链表中
#ifdef CONFIG_PINCTRL
    /*
     * 如果启用了CONFIG_PINCTRL 选项，GPIO 控制器可以选择描述它们在SoC 中服务的实际引脚范围。
     * 此信息将由pinctrl 子系统用于配置相应的GPIO 引脚。
     */
```

```c
struct list_head pin_ranges;// 描述GPIO 控制器引脚范围的链表
#endif
};
```
在代码清单132.4 中的参数，要重点关注的是struct gpio_chip *chip 这一结构体，该结构体表示与GPIO 关联的GPIO 控制器对应的结构体，该结构体定义在内核源码目录下的include/linux/gpio/driver.h 文件中，具体内容如代码清单132.5 所示。代码清单132.5
```c
struct gpio_chip {
    const char *label;// GPIO 芯片标签
    struct gpio_device gpiodev;// GPIO 设备
    struct device *parent;//父设备指针
    struct module *owner;//拥有者模块指针
    int (*request)(struct gpio_chip *chip, unsigned offset);//请求GPIO
    void (*free)(struct gpio_chip *chip, unsigned offset);//释放GPIO
    int (*get_direction)(struct gpio_chip *chip, unsigned offset); //获取GPIO 方向
    int (*direction_input)(struct gpio_chip *chip, unsigned offset); //设置GPIO 为输入
    int (*direction_output)(struct gpio_chip *chip, unsigned offset, int value); //设置GPIO 为输出
    int (*get)(struct gpio_chip chip, unsigned offset);//获取GPIO 值
    int (*get_multiple)(struct gpio_chip chip, unsigned long *mask, unsigned long *bits); //获取多个GPIO 的
值
    void (*set)(struct gpio_chip chip, unsigned offset, int value);//设置GPIO 值
    void (*set_multiple)(struct gpio_chip chip, unsigned long mask, unsigned long *bits); //设置多个GPIO 的
值
    int (*set_config)(struct gpio_chip *chip, unsigned offset, unsigned long config); //设置GPIO 配置
    int (*to_irq)(struct gpio_chip chip, unsigned offset);//将GPIO 转换为中断
    void (*dbg_show)(struct seq_file *s, struct gpio_chip chip);//在调试信息中显示GPIO
    int base;// GPIO 编号的基准值
    u16 ngpio;// GPIO 的数量
    const char *const *names;// GPIO 的名称数组
..........
};
```
代码清单132.5 中struct gpio_chip *chip 这一结构体用于描述GPIO 控制器的属性和操作函数，需要注意的是这个结构体中的一系列函数都不需要我们来填充，这些工作都是由芯片原厂工程师来完成的，我们只需要学会新版本的GPIO 子系统API 函数的使用即可。

