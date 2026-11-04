# 第229章open和close函数分析

至此，CAN 控制器设备注册流程分析完毕。
本章节将分析open 和close 函数。对于网络设备而言，存在与之对应的函数操作集，例如在CAN 控制器的驱动程序中就有所体现，如代码清单229.1 所示：代码清单229.1
```c
static int rockchip_canfd_probe(struct platform_device *pdev) {
    ......
    // 设置网络设备的操作集
    ndev->netdev_ops = &rockchip_canfd_netdev_ops;
    // 设置网络设备的中断号
    ndev->irq = irq;
    // 设置网络设备的标志，添加IFF_ECHO 标志
    ndev->flags |= IFF_ECHO;
    .......
```
}在代码清单229.1 中第4 行设置网络设备的操作集，struct rockchip_canfd_netdev_ops 如代码清单229.2 所示，此结构体用于定义Rockchip CANFD 网络设备的操作方法，通过填充该结构体的各个成员，驱动程序可以为特定的网络设备提供定制化的操作实现。代码清单229.2
```c
static const struct net_device_ops rockchip_canfd_netdev_ops = {
    .ndo_open = rockchip_canfd_open,
    .ndo_stop = rockchip_canfd_close,
    .ndo_start_xmit = rockchip_canfd_start_xmit,
    .ndo_change_mtu = can_change_mtu,
};
```
代码清单229.2 中第2 行.ndo_open = rockchip_canfd_open 指定了打开网络设备时调用的函数为rockchip_canfd_open()。当网络设备需要开启，例如系统启动时加载设备或者用户手动启用设备，内核会调用此函数。函数实现如代码清单229.3 所示：代码清单229.3

```c
static int rockchip_canfd_open(struct net_device *ndev) {
        // 获取网络设备的私有数据，即rockchip_canfd 结构体指针rcan
        struct rockchip_canfd *rcan = netdev_priv(ndev);
        int err;
        // 执行CAN 设备通用打开操作
        err = open_candev(ndev);
    if (err) return err;
        // 获取设备电源管理运行时状态
        err = pm_runtime_get_sync(rcan->dev);
        if (err < 0) {
            // 记录获取电源管理状态失败的错误信息
            netdev_err(ndev, "%s: pm_runtime_get failed(%d)\n", __func__, err);
            goto exit;
    }
        // 启动CAN 设备
        err = rockchip_canfd_start(ndev);
        if (err) {
            // 记录无法启动CAN 外设的错误信息
            netdev_err(ndev, "could not start CAN peripheral\n");
            goto exit_can_start;
    }
        // 触发CAN 设备LED 打开事件
        can_led_event(ndev, CAN_LED_EVENT_OPEN);
        // 启动网络设备发送队列
        netif_start_queue(ndev);
        // 输出调试信息
        netdev_dbg(ndev, "%s\n", __func__);
        return 0;
    exit_can_start:
        // 释放设备电源管理运行时状态
        pm_runtime_put(rcan->dev);
    exit:
        // 执行CAN 设备通用关闭操作
        close_candev(ndev);
        return err;
}
```
其中，上述代码清单229.3 第16 行使用rockchip_canfd_start()函数启动CAN 设备，函数实现如代码清单229.4 所示：代码清单229.4

```c
// 定义rockchip_canfd_start 函数，用于启动网络设备，参数为指向网络设备结构体的指针ndev
static int rockchip_canfd_start(struct net_device *ndev) {
        // 获取网络设备ndev 的私有数据，即指向rockchip_canfd 结构体的指针rcan
        struct rockchip_canfd *rcan = netdev_priv(ndev);
        u32 val;
        // 进入复位模式，对CAN 设备进行复位操作
        set_reset_mode(ndev);
        // 清空CAN 中断屏蔽寄存器，使能所有中断
        rockchip_canfd_write(rcan, CAN_INT_MASK, 0);
        // 设置接收过滤器，接受所有消息
        rockchip_canfd_write(rcan, CAN_IDCODE, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK, CAN_RX_FILTER_MASK);
        rockchip_canfd_write(rcan, CAN_IDCODE0, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK0, CAN_RX_FILTER_MASK);
        rockchip_canfd_write(rcan, CAN_IDCODE1, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK1, CAN_RX_FILTER_MASK);
        rockchip_canfd_write(rcan, CAN_IDCODE2, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK2, CAN_RX_FILTER_MASK);
        rockchip_canfd_write(rcan, CAN_IDCODE3, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK3, CAN_RX_FILTER_MASK);
        rockchip_canfd_write(rcan, CAN_IDCODE4, 0);
        rockchip_canfd_write(rcan, CAN_IDMASK4, CAN_RX_FILTER_MASK);
        // 读取CAN_MODE 寄存器的值
    val = rockchip_canfd_read(rcan, CAN_MODE);
        // 使能接收FIFO
        rockchip_canfd_write(rcan, CAN_RXFC,rockchip_canfd_read(rcan, CAN_RXFC) | FIFO_ENABLE);
        // 设置模式相关位，启用FDOE 模式
    val |= MODE_FDOE;
        // 如果设备控制模式中设置了回环模式
        if (rcan->can.ctrlmode & CAN_CTRLMODE_LOOPBACK)
        // 设置自测试和回环模式相关位
    val |= MODE_SELF_TEST | MODE_LBACK;
        // 启用自动重传模式
    val |= MODE_AUTO_RETX;
        // 将修改后的模式值写回CAN_MODE 寄存器
    rockchip_canfd_write(rcan, CAN_MODE, val);
        // 设置CAN 设备的位定时
    rockchip_canfd_set_bittiming(ndev);
        // 设置设备为正常工作模式
```

