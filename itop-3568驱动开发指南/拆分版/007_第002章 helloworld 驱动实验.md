# 第002章 helloworld 驱动实验

本章编写第一个驱动程序helloworld，并讲解Linux 驱动基本框架。
## 2.1 驱动编写
本小节来编写一个最简单的驱动——helloworld 驱动。其程序helloworld.c 代码如代码清单2.1 所示.代码清单2.1
```c
#include <linux/module.h>
#include <linux/kernel.h>
static int __init helloworld_init(void)//驱动入口函数
{
      printk(KERN_EMERG "helloworld_init\r\n");//注意：内核打印用printk 而不是printf
      return 0;
}
static void __exit helloworld_exit(void)//驱动出口函数
{
       printk(KERN_EMERG "helloworld_exit\r\n");
}
module_init(helloworld_init);//注册入口函数
module_exit(helloworld_exit);//注册出口函数
MODULE_LICENSE("GPL v2");//同意GPL 开源协议
MODULE_AUTHOR("topeet");//作者信息
```
## 2.2 驱动基本框架
Linux 驱动的基本框架由模块加载函数，模块卸载函数，模块许可证声明，模块参数，模块导出符号，模块作者信息等几部分组成，我们先来看一下这几个部分的作用。1 模块加载函数（驱动入口函数）当加载驱动模块时，内核会执行模块加载函数，完成模块加载函数中的初始化工作。2 模块卸载函数（驱动出口函数）当卸载驱动模块时，内核会执行模块卸载函数，完成模块卸载函数中的退出工作。3 模块许可证声明

许可证声明描述了内核模块的许可权限，如果不声明模块许可，模块在加载的时候，会收到“内核被污染（kernel tainted）”的警告。可接受的内核模块声明许可有“GPL”“GPL v2”等。4 模块参数（可选择）内核模块参数是模块被加载的时候可以传递给它的值。5 模块导出符号（可选择）内核模块可以导出的符号，导出符号以后其他内核模块可以使用本模块中的变量或函数。6 模块作者信息等说明（可选择）其中模块参数，模块导出符号，模块作者信息是可选的部分。其余部分是必须有的。
接下来通过驱动基本框架分析helloworld 驱动。（1）模块加载函数：代码清单2.2
```c
static int __init helloworld_init(void)//驱动入口函数
{
      printk(KERN_EMERG "helloworld_init\r\n");
      return 0;
}
module_init(helloworld_init);//注册入口函数
```
（2）模块卸载函数：代码清单2.3
```c
static void __exit helloworld_exit(void)//驱动出口函数
{
     printk(KERN_EMERG "helloworld_exit\r\n");
}
module_exit(helloworld_exit);//注册出口函数
```
（3）模块许可证声明代码清单2.4
```c
MODULE_LICENSE("GPL v2");//同意GPL 开源协议
```
（4）模块作者信息代码清单2.5
```c
MODULE_AUTHOR("topeet");//作者信息
```
（5）头文件代码清单2.6
```c
#include <linux/module.h> //模块加载函数和卸载函数需要的头文件
```

