# MQTT over QUIC 的特性与优势

EMQX 5.0 为 MQTT over QUIC 设计了独特的消息传输机制和管理方式，为在现代复杂网络中传输 MQTT 消息提供了一种更高效、更安全的方式，从而在某些场景下提高了 MQTT 的性能。

EMQX 目前的实现将传输层换成 QUIC 流，由客户端发起连接和创建流，EMQX 和客户端在一个双向流上实现交互。考虑到复杂的网络环境，如果客户端因某种原因未能通过 QUIC 握手，客户端将自动退回到传统 TCP 上，避免系统无法建立跟服务器的通信。客户端和 EMQX 的交互分为两种模式：[单流模式](#单流模式)和[多流模式](#多流模式)。下面的章节将介绍两种模式各自的特性和优势。

<img src="./assets/mqtt-over-quic.png" alt="mqtt-over-quic" style="zoom:25%;" />

## 单流模式

这是一种基本模式，将 MQTT 数据包封装在一个双向 QUIC 流中。客户端需要在 QUIC 连接中从客户端向 EMQX 发起一个双向流。所有 MQTT 数据包交换都在这个流上完成。

### 特性和优势

- 快速握手

  客户端和 EMQX 之间的 QUIC 连接可以在一个往返中建立。

- 有序数据传输

  与 TCP 类似，MQTT 数据包的传输也遵循在流中接收顺序与发送顺序一致，即使底层 UDP 包的收发顺序不一致。

- 连接恢复，0-RTT

  使用 0-RTT 方法，客户端能够恢复连接并在第一个或紧接其后的 QUIC 数据包中向服务器发送应用程序数据。无需等待 EMQX 的回复完成一个往返。 这个特性对于快速恢复因网络干扰而关闭的连接并将应用业务重新上线非常有用。

- 客户端地址迁移

  在不断开和建立新的 QUIC 连接的情况下，客户端可以主动或被动地将其本地地址迁移到由于网络地址转换 (NAT) 重新绑定而产生的新地址。QUIC 连接可以保持，MQTT 及以上层面不会受到重大干扰。

- 包丢失检测和恢复

  相对于其他协议，QUIC 更具响应性，能够更快地检测到数据包丢失并进行恢复。这些行为可以针对每个用例进行特定的定制调优。

## 多流模式

单个 MQTT 连接通常会携带不同业务的多个主题数据。这些主题可能相关或不相关。多流模式通过利用 QUIC 的流多路复用特性扩展了单流模式，允许 MQTT 数据包通过多个流进行传输。MQTT 客户端与 EMQX 之间的 QUIC 连接握手与单流模式相同。

从客户端到 EMQX 建立的初始流称为控制流，其目的是处理 MQTT 连接的维护或更新。在此之后，客户端可以启动一个或多个数据流来发布主题或订阅主题。

客户端可以自由选择如何映射流，例如：

- 每个主题使用一个流。
- QoS 1 和 QoS 0 各使用一个流。
- 发布和订阅各使用一个流。（控制流上也允许发布/订阅。）

作为消息代理，EMQX 执行流数据包绑定：

- 将 PUBACK 数据包发送到接收 QoS 1 的 PUBLISH 的流中，对于 QoS 2 数据包也是如此。
- 将 PUBLISH 数据包发送到相对的订阅（subscribe）流中，并期望从同一流中收到 QoS 1 的 PUBACK。

::: tip

数据收发的顺序在每个流（Stream）中保证。跨流的数据顺序不做保证。  因此，如果有多个主题（topics），他们在业务上是相关的且对数据收发一致性（ordering）有要求，则它们应映射到同一个流中， 或者说用相同的流做传输。

:::

### 特性和优势

- 分离连接控制和 MQTT 数据交换

  使用控制流（control stream）承载连接控制包（如 CONNECT、CONNACK 和 PING），而用数据流（data stream）承载发布和订阅等数据。即使数据流（data stream）处理较慢，也可以通过在控制流上收发 PINGREQ 和 PINGRESP 来保活连接，减少断连的几率。

- 避免主题之间的顺序阻塞问题

  MQTT over QUIC 允许为不同主题创建多个数据流，使不同主题的消息能够独立传递。

- 分离上行（发布）和下行（订阅）数据

  例如，客户端可以使用一个数据流发布 QoS1 消息并在同一流上处理 PUBACK，同时通过另一个流从EMQX接收 QoS 0 消息。

- 对不同数据进行优先级排序

  MQTT over QUIC 还提供了通过多流优先处理不同 MQTT 主题数据的功能。这意味着可以根据需要对主题数据进行优先排序和传递，从而提高连接的整体性能和响应速度。

- 提高客户端和EMQX 端的并行处理能力

  EMQX 和客户端能够利用数据流并行处理多个流，提高系统的整体效率和资源使用率，进而降低延时、提高应用层响应速度。

- 减少错误发生的开销

  如果由于应用程序错误导致单个数据流中断，它将不会导致整个连接关闭。相反，应用程序可以重新创建流并恢复数据。这可以实现更可靠和弹性的 MQTT 通信。

- 控制数据流的流量

  可以对每个数据流应用流量控制，从而为不同的应用主题或 QoS 级别设置不同的流量控制策略。

- 减少订阅延迟

  客户端在发送订阅或发布包之前不需要等待 MQTT CONNACK 来确认连接成功与否。

  不过，EMQX 只会在客户端建立连接且允许连接时开始处理这些数据包。

