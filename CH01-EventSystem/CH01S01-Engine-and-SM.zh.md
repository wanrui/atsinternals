# 引擎与状态机

事件（Event）可以通过三类API（EventProcessor，EThread，Event）放入这个引擎：

- EventProcessor
  - 创建一个 Event，将状态机（SM）包含进去
  - 然后，按照轮询算法从引擎里找到一个 EThread
  - 把 Event 交给这个 EThread 处理
- EThread
  - 直接把一个 Event 交给指定的 EThread 来处理
- Event
  - 把 Event 交回原来的 EThread 重新处理

引擎根据事件的类型执行不同的策略。

通过调用Event->Cont->handleEvent来驱动状态机（SM），状态机就可以获得CPU资源来完成当前状态的工作，然后进入下一个状态。

状态机在工作时，一旦遇到阻塞的情况，就要暂停运行，并迅速返回到引擎，等待下一次被引擎驱动的时候再继续之前未完成的任务。


这个引擎就如同汽车的发动机一样，每一个点火冲程，都会让轮子转一下，空调压缩机制冷一下。

但是唯一的一点小区别是：

- 这个引擎每次点火只能驱动一个功能
- 这一次是轮子转一下
- 下一次是空调压缩机制冷，等等。

但是引擎只有一个驱动轴：

- 所以轮子转完，就要让出位置，让驱动轴可以再带动其它的部件的运行
- 但是很多时候只转一下轮子车子无法到达目的地，只有不断的转动轮子才能到达目的地
- 如果引擎只带动轮子转动，就没法驱动空调，那么乘客就会抱怨

因此，我们需要把引擎的动力轮流分配给不同的功能，由各个功能自己判断目的是否已经达成：

- 如果还没有达到目的地，就要控制车轮持续转动
- 如果还没有达到舒适的温度，就让空调继续制冷

可是：

- 轮子怎么会知道我们还没有到达目的地？
- 空调怎么知道什么是舒适的问题？

所以：

- 我们要提供油门这个接口给司机，让司机可以控制轮子转动
- 我们要提供空调温度调整的按钮给乘客，让乘客可以设置一个舒适的温度

因此，在为这个引擎设计部件（Sub-System）时，必须按照以下方式来进行：

- 车轮控制子系统（Wheel Sub-System）
  - 能够让轮子持续运行的控制系统（Wheel SM）
  - 提供可以让车轮开始转动的接口（Wheel Procesor）
- 空调子系统（Cooling Sub-System）
  - 能够让空调持续改善温度的控制系统（Cooling SM）
  - 提供空调开启的开关和温度调整的按钮（Cooling Processor）

最后，乘客和司机则对应了应用系统的状态机：

- 司机会记得还没有到达目的地
  - 司机会控制油门，让车轮转动
- 乘客会做一些旅途上的其它事情
  - 开启空调并设定舒适的温度

在后面的章节将要介绍的NetHandler，就是由这个引擎驱动的状态机（属于网络子系统），这些由引擎驱动的状态机，我们把他们称之为引擎的用户。

为了保证整个系统流畅的运行，就要求每一个引擎的用户不能长时间运行，否则就会导致其它引擎用户不能良好的运行。

作为“用户”，不能 为了读取即将到来的数据而进行等待 或者 为了外部状态的改变而进行等待，此时需要：

- 立即返回到引擎
- 等待下一次引擎重新驱动状态机
- 然后再检查需要的数据是否已经到达 或者 外部状态是否已经改变

状态机在运行时必须是无阻塞的，凡是遇到阻塞时必须返回到引擎。

这跟我们停车之后，要挂空挡是一个道理，要让出引擎的驱动力给其它需要的组件，否则引擎就会熄火。

状态机在运行时不能出现循环状态，例如：

- 车内空气温度低于22度？
- No, 空调制冷运行一次, 回到第一行
- Yes, 重新调度该状态机在一分钟之后再运行, 返回

上面这个状态机的流程就是一种循环，但是整个运行过程没有阻塞。

这样的设计，让空调得到了最大化的运行效率，但是却让其它的系统全部都停止运行。
因为，它在达成温度控制目标之前都不会返回到引擎。
这样的设计是不符合引擎用户设计要求的，正确的设计应该是：

- 车内空气温度低于22度？
- No, 空调制冷运行一次, 重新调度该状态机立即再次运行, 返回
- Yes, 重新调度该状态机在一分钟之后再运行, 返回

这样的设计，让状态机每一次被引擎回调的时候，只进行一次制冷（有限运行），然后通过 Event 通知引擎：这个状态机还需要再次被驱动。
这样引擎就会在下一次驱动力分配中再次驱动这个状态机，这样就实现了制冷状态机的反复执行，直到车内温度达标。
 
最后，在达到温度控制目标之后，每分钟再检查一次车内温度。

## 组成系统的三要素

|     引擎     |   系统   |  EventSystem   |
|:------------:|:--------:|:--------------:|
|  燃油（能量）|   数据   |      Event     |
|    燃烧室    |   线程   |     EThread    |
| 燃烧控制系统 |  处理器  | EventProcessor |

我们把“引擎”与“系统”进行对比

- “燃烧控制系统” 把 ”燃油“ 放入 ”燃烧室“ 中点燃，然后将获得的动力输出驱动车子运行
- ”处理器“ 把 “数据“ 放入 “线程“ 中处理，然后根据”数据“内的指示驱动对应的”状态机“

再进一步的对比”事件处理系统“

- EventProcessor 把 Event 放入 EThread 中处理，然后根据 Event 的指示回调“状态机”
