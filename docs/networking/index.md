# **网络通信**(`Networking`)

服务器与客户端之间的通信是成功实现模组的基石。

网络通信有两个主要目标：

1. 确保客户端视图与服务器视图**同步**(`in sync`)
    - 坐标(X, Y, Z)处的花刚刚生长
2. 让客户端能够告知服务器玩家状态的改变
    - 玩家按下了某个按键

实现这些目标的最常见方式是在客户端和服务器之间传递消息。这些消息通常是结构化的，包含特定排列的数据，以便于发送和接收。

NeoForge提供了一种主要构建于[netty]之上的技术来促进通信。该技术可通过监听`RegisterPayloadHandlersEvent`事件使用，然后向**注册器**(`registrar`)注册特定类型的[载荷(`payloads`)][payloads]、其**读取器**(`reader`)及其**处理函数**(`handler function`)。

[netty]: https://netty.io "Netty 官网"
[payloads]: payload.md "注册自定义载荷"