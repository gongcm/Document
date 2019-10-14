# Input 子系统

在linux系统中 input设备主要有 evdev、mouse、keybord等，对于这个几种设备linux提供了
input子系统来驱动这些设备。

在input子系统中主要有两个链表：

input_handler_list : 主要是对于不同的设备提供不同的操作和配置。
input_dev_list     : 主要是记录系统中存在多少个struct input_dev 设备。

# input驱动编写

```
//分配一个 struct platform_driver
//分配一个 struct input_dev

// 将 input_register_device(input_dev)

//定义 probe 函数初始化硬件，中断等

// platform_register_driver(platform_driver)



```


# 事件上报

```
//在设备中断函数中调用input_event函数
// input_event 最终会调用到 hander->event \ events 函数

// 用户空间打开相应的 /dev/event0设备读取事件
```


# 注意input
  
  在该子系统主要是handler 和 input_dev 的两个链表中进行匹配。
