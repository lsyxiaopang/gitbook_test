---
layout: editorial
---

# Controller库文档

## 关于`Controller`

`Controller`库是一个运行在任意系统上的一个用以控制Grbl操作移动平台进行物理量测量的Python接口

其提供了将测量设备的代码完全隔离的手段,并且有着较为完善的日志系统

其由以下部分组成

* `Controller`:主要程序接口,将移动设备和测量设备整合
* `Measuredevices`:主要由用户定义,只要满足一定的条件即可实现测量
* `MoveController`:Grbl控制接口,可以异步地控制电机运作

在目前的版本中,我们还集成了`adaptive`库,可以实现高效的测量

{% content-ref url="adaptive-ji-cheng.md" %}
[adaptive-ji-cheng.md](adaptive-ji-cheng.md)
{% endcontent-ref %}

## `Controller`是如何运作的

`Controller`通过记录用户的移动命令,而后_异步_传输至grbl运行

同时提供了回调函数以使用户实现对电机等装置实时状态的获取

## 快速安装与使用

如果你想快速尝试一下该库的安装与使用,可以参考一下快速开始

{% content-ref url="an-zhuang-yu-ru-men.md" %}
[an-zhuang-yu-ru-men.md](an-zhuang-yu-ru-men.md)
{% endcontent-ref %}

## Want to deep dive?

Dive a little deeper and start exploring our API reference to get an idea of everything that's possible with the API:

{% content-ref url="reference/api-reference/" %}
[api-reference](reference/api-reference/)
{% endcontent-ref %}
