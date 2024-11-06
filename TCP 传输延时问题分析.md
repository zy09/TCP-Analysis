# TCP 传输延时问题分析

## 描述:

​            当您使用 Wireshark捕获分析数据包时，它会对 TCP 连接进行分析，并能够通过 flag 标签了解 TCP 性能的某些行为。其中一些对应于特定的 TCP 消息，而另一些则是 Wireshark 突出显示连接状态。这些标志包括：

```
- TCP Window FullTCP 窗口已满
- TCP ZeroWindowTCP 零窗口
- TCP Retransmission (including variants)TCP 重传（包括变体）
- TCP Dup AckTCP 重复确认
```

​        本文旨在讨论负载再七层模式或者标准模式下的tcp 性能优化措施, 在 flastL4 模式下TCP 的协商由客户端和服务器来完成，只需要修改IP 头部的源目地址。负载在Standard type  模式下 数据也是按照流转发 ,针对http 较大的数据包例如 20M  http 报文也是按照流转发，不会等所有数据都接收再转发，切记。

## 问题原因:

​        TCP 术语:

1. **Receive Window** \- 接收窗口 - 也简称为 TCP 窗口，这是由 TCP 接收器向其对等方通告的。它是捕获中可见的窗口，并且在连接中的任一方向可能不同。接收窗口通常是可用接收缓冲区的大小。 TCP 接收者可以通过改变它们发送的 ACK 中的窗口值来调整接收窗口的大小。在负载上，最大大小由接收窗口 TCP 配置文件选项控制。
2. **Send Window** - 发送窗口 - 从技术上讲是拥塞窗口，这是 TCP 发送方的内部值，表示它可以在没有收到确认的情况下发送的最大数据量。这由 TCP 拥塞控制算法不断调整，在 BIG-IP 上，最大大小由发送缓冲区 TCP 配置文件选项控制。在捕获中无法观察到该值；可以推断，如果发送者停止发送数据，即使接收者的接收窗口未满，但实际值从未在网络上发送。
3. **Bytes In Flight** -Bytes In Flight - 这是 Wireshark 用来表示 TCP 发送方已传输的未确认数据量的术语。它总是小于或等于收件人的接收窗口。

**TCP Window Full:**

如果 TCP 发送方发送的数据包导致未收到确认字节数的报文（或传输中的字节数）与接收方的接收窗口相同，则称为填充窗口，Wireshark 报告 TCP 窗口已满。

​    Example:客户端配置在 WAN 上，服务器配置在 LAN 上

![windo_full](D:\网络\网络相关文章\tcp 参数说明\windo_full.png)

当您看到 TCP Window Full 标志时,表示您发送的数据等于接收方的窗口大小，相当于正在使用接收方的全部窗口大小。这完全受接收方窗口大小的影响。

**TCP Zero Window:**

当 TCP 接收器的缓冲区开始填满时，它可以减小其接收窗口。如果它填满，它可以将窗口减小到零，这会告诉 TCP 发送方停止发送。这被称为“关闭窗口”。通常，这表明网络传输流量的速度快于接收器处理它的速度。

当 负载关闭其接收窗口时，通常意味着 负载接收数据的速度快于它在对等流上发送数据的速度。这在某些情况下是正常的，例如:服务器端网络比客户端网络快，并且从服务器到客户端的传输量很大。负载缓冲一定数量的数据，然后在代理缓冲区达到配置的高水位标记时关闭其接收窗口。但是上述场景比较难见到，关闭的也是和服务端的窗口。

每个 TCP 标头中的窗口字段通告接收方可以接受的数据量。如果接收方不能再接受任何数据，它会将窗口值设置为零，这会告诉发送方暂停其传输。在某些特定情况下，这是正常的 — 例如，打印机可能会使用零窗口来暂停打印作业的传输，同时加载或反转一张纸。但是，在大多数情况下，这表明接收端存在性能或容量问题。恢复暂停的连接可能需要很长时间（有时是几分钟），即使导致零窗口的基本条件很快消失。

**TCP Retransmission:**

当 TCP 发送数据包时，它会等待确认以确认发送方发送的数据包是否已被接收方接收。如果在超时间隔（称为重传超时或 RTO）内未收到确认，则发送方重传数据包。

![retransmissions(1)](D:\网络\网络相关文章\tcp 参数说明\retransmissions(1).png)

TCP sender receives duplicate ACKs.

当 TCP 发送方收到重复的 ACK 时，也可以强制重新传输。DACK 快速重传场景。

Wireshark 区分了几类 TCP 重传；有关详细信息，请参阅 Wireshark TCP 分析文档。

**TCP Duplicate ACK** 

TCP 重复 ACK

当 TCP 接收器接收到乱序的数据包时，它会将其解释为数据丢失，它会发送一个 ACK 指示预期的序列号。在两个重复的 ACK 之后，TCP 发送方开始重新传输。

## 优化建议

当您在排除 TCP 性能故障时观察到上述行为时，请使用以下准则。

**TCP Window Full**

检查负载配置。大多数负载的默认 TCP 配置文件的发送缓冲区和接收窗口设置为 65535，这也将远程端点（即客户端和服务器）的接收窗口限制为 65535，因为它禁用了窗口缩放。如果您认为 负载窗口设置限制了吞吐量，通常可以安全地通过将它们加倍进行试验，直到您不再看到性能改进为止。如果传输到 负载的带宽受到限制，则应增加 Receive Window 设置，如果从 负载传输到另一台设备的带宽受到限制，则应增加 Send Window 设置。对应增加接收和发送的窗口窗口大小。

<!--*注意：增加发送缓冲区和接收窗口会增加每个连接消耗的最大内存，因此建议在繁忙的虚拟服务器上调整这些设置时监控 CPU 内存使用情况。但是，此内存仅在需要时使用，并且连接仅在最大吞吐量期间最大化缓冲区限制。*-->

**TCP Zero Window**

如果在负载端通告零窗口：

检查对等流 - 例如:如果您在负载和服务器端看到零窗口，请检查客户端流。零窗口通常表示吞吐量受到对等流的限制。也就是看到负载向服务端发送zero window  ,相当于客户端的接收窗口沾满了。因为负载时七层负载，要考虑客户端的问题。

如果负载需要很长时间来处理数据，如果它负载过重并执行繁重的处理，就会发生这种情况，负载本身就会引入延迟。发送tcp zero window，表示他的窗口满了。

 如果在负载重新打开其接收窗口后远程系统需要很长时间（相对）才能再次开始传输，您可能需要通过增加代理缓冲区设置来提高性能，该设置允许负载在客户端之间缓冲数据和服务器端 TCP 流。这些没有固定的指导方针；较高的设置可能会在接收延迟较高的情况下提高性能，但过大的代理缓冲区可能会导致在高性能网络上传输数据时出现延迟。有关代理缓冲区设置及其操作的更多详细信息，请参阅 https://support.f5.com/csp/article/K3422

*注意：增加代理缓冲区设置会增加每个连接消耗的最大内存，因此 建议在繁忙的虚拟服务器上调整这些设置时监控 CPU内存使用情况。但是，此内存仅在需要时使用，并且连接仅在最大吞吐量期间最大化缓冲区限制。*

**TCP Dup Ack and TCP Retransmissions**

这些行为通常是相关的。这些通常表示网络存在丢包行为。损失可能是问题，也可能不是问题； TCP 旨在增加吞吐量，直到观察到丢失数据报的情况以估计重新估算网络容量，因此如果您偶尔看到 duplicate ACKs 和retransmissions但是网络具有稳定吞吐量，您可能是正常的 TCP 行为。

**Additional Information**附加信息

如果您了解端到端网络的特征，则可以使用带宽延迟乘积 (BDP) 估算所需的发送窗口和接收缓冲区设置（即所需的 TCP 窗口）。这是带宽（通常以每秒位数衡量）和客户端往返时间（RTT 或延迟）的乘积，以字节表示。请注意，RTT 通常以毫秒表示，带宽通常以每秒千位、兆位或千兆位表示，因此有必要相应地调整单位。

例如，对于可以承载每秒 40 兆比特 (Mbit/s) 和 50 毫秒 RTT 的网络，BDP 为：

```
    40,000,000 bit/s * 0.05 s = 2,000,000 bit ÷ 8 bit/byte = 250,000 bytes
```

为了占满这条链路，也就是在理想条件下使带宽完全饱和，TCP 窗口必须至少为 250000 字节。

## Related Content相关内容

[K1893](https://support.f5.com/csp/article/K1893): Packet trace analysis

[K56401167](https://support.f5.com/csp/article/K56401167): Standard virtual servers do not use TCP Window Scaling with certain profile settings

K56401167：标准虚拟服务器不使用具有某些配置文件设置的 TCP 窗口缩放

注意；在设置发送和接收窗口大于65535 时，会自动启动 TCP window scaling

- [Wireshark manual chapter 7.5: TCP AnalysisWireshark 手册第 7.5 章：TCP 分析](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)

# TCP 针对大时延，长距离传输问题分析

针对f5-tcp-wan 进行描述:

  https://support.f5.com/csp/article/K10281257 

## Description描述

如果您将虚拟服务器配置为处理严格基于 WAN 的流量，则可以使用 f5-tcp-wan 配置文件来增强基于 WAN 的流量也就是client-side 。例如，在许多情况下，客户端通过 WAN 链接连接到负载虚拟服务器，这通常比负载系统和池成员服务器之间的连接要慢。因此，负载系统可以更快地接受数据，从而使池成员服务器上的资源保持可用。通过将虚拟服务器配置为使用 f5-tcp-wan 配置文件，您可以增加负载系统在等待远程客户端接受时缓冲的数据量。此外，您可以通过减少负载系统在网络上发送的短 TCP 段的数量来增加网络吞吐量。

**f5-tcp-wan 配置文件中的优化设置和定义**

| Setting环境                              | Value价值                                                                                          | Description描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Congestion Control拥塞控制                 | Woodside伍德赛德                                                                                     | Specifies that the system uses a TCP algorithm that is especially targeted at high-speed, long-distance networks with enhanced congestion recovery.指定系统使用 TCP 算法，该算法特别针对具有增强拥塞恢复的高速、长距离网络。                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Early Retransmit提前重传                   | Enabled启用                                                                                        | Specifies, when selected (enabled), that the system uses early retransmit (as specified in RFC 5827) to reduce the recovery time for connections that are receive- buffer or user-data limited. Enabling this setting allows TCP to act as though a packet is lost, after fewer than the standard number of duplicate ACKs, if there is no way to send new data and generate more duplicate ACKs.指定在选中（启用）时，系统使用提前重传（如 RFC 5827 中指定）来减少接收缓冲区或用户数据受限的连接的恢复时间。如果无法发送新数据并生成更多重复 ACK，则启用此设置允许 TCP 在重复 ACK 少于标准数量后就像丢失数据包一样。                                                                                                                                                  |
| Explicit Congestion Notification显式拥塞通知 | Enabled启用                                                                                        | Specifies, when enabled, that the system uses the TCP flags CWR (Congestion Window Reduced) and ECE (ECN-Echo, used to indicate that the TCP peer is Explicit Congestion Notification capable during the three-way handshake) to notify its peer of congestion and congestion counter-measures.指定启用时，系统使用 TCP 标志 CWR（减少拥塞窗口）和 ECE（ECN-Echo，用于指示 TCP 对等方在三次握手期间具有显式拥塞通知能力）来通知其对等方拥塞和拥塞对策。  Note: When this option is enabled, it is used only when two hosts signal that they want to use it. It allows a host to set the ECN bit within the IP header to notify the transmitter that the link is congested.注意：启用此选项后，仅当两个主机表示要使用它时才使用它。它允许主机设置 IP 报头中的 ECN 位，以通知发送器链路拥塞。 |
| Initial Congestion Window Size初始拥塞窗口大小 | 10 MSS units10 个 MSS 单元                                                                          | Specifies the initial congestion window size for connections to this destination. Actual window size is this value multiplied by the MSS (Maximum Segment Size) for the same connection. Valid values range from 0 to 16.指定到此目的地的连接的初始拥塞窗口大小。实际窗口大小是该值乘以同一连接的 MSS（最大段大小）。有效值范围为 0 到 16。                                                                                                                                                                                                                                                                                                                                                                                  |
| Initial Receive Window Size初始接收窗口大小    | 10 MSS units10 个 MSS 单元                                                                          | Specifies the initial receive window size for connections to this destination. Actual window size is this value multiplied by the MSS (Maximum Segment Size) for the same connection. Valid values range from 0 to 16.指定到此目标的连接的初始接收窗口大小。实际窗口大小是该值乘以同一连接的 MSS（最大段大小）。有效值范围为 0 到 16。                                                                                                                                                                                                                                                                                                                                                                                      |
| Minimum RTO最小 RTO                      | 500 ms500 毫秒                                                                                     | Specifies the minimum length of time that the system waits for acknowledgements of data sent, before resending the data. The value must be equal to or less than 5000. If the Packet Loss Ignore Rate and Packet Loss Ignore Burst settings contain values other than 0, this value drops to 203 ms. If MPTCP is enabled, the value drops to 200 ms.指定系统在重新发送数据之前等待确认已发送数据的最短时间。该值必须等于或小于 5000。如果 Packet Loss Ignore Rate 和 Packet Loss Ignore Burst 设置包含 0 以外的值，则该值下降到 203 ms。如果启用 MPTCP，则该值下降到 200 毫秒。                                                                                                                                                                 |
| Rate Pace速率步伐                          | Enabled启用                                                                                        | The system paces the egress packets to avoid dropping packets, allowing for optimum goodput. This option mitigates bursty behavior in mobile networks.系统调整出口数据包的速度以避免丢弃数据包，从而实现最佳吞吐量。此选项可减轻移动网络中的突发行为。                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Tail Loss Probe尾部丢失探头                  | Enabled启用                                                                                        | Specifies, when selected (enabled), that the system uses Tail Loss Probe to reduce the number of retransmission timeouts. Retransmission timeouts occur when TCP segments are lost from the tail of a transaction, or a window of data/ACKs are lost. Enabling this setting allows TCP to send a probe segment to trigger fast recovery instead of recovering a loss by way of a retransmission timeout.指定在选中（启用）时，系统使用 Tail Loss Probe 来减少重新传输超时的次数。当 TCP 段从事务的尾部丢失，或者数据/ACK 窗口丢失时，会发生重传超时。启用此设置允许 TCP 发送探测段以触发快速恢复，而不是通过重新传输超时来恢复丢失。                                                                                                                                   |
| Proxy Buffer Low代理缓冲区低                 | 196608                                                                                           | This setting specifies the proxy buffer level at which the receive window was opened. For more information, refer to [K3422: Overview of content spooling](https://support.f5.com/csp/article/K3422).此设置指定打开接收窗口的代理缓冲区级别。有关详细信息，请参阅 K3422：内容假脱机概述。                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Proxy Buffer High代理缓冲区高                | 262144                                                                                           | This setting specifies the proxy buffer level at which the receive window is no longer advanced. For more information, refer to [K3422: Overview of content spooling](https://support.f5.com/csp/article/K3422).此设置指定接收窗口不再高级的代理缓冲区级别。有关详细信息，请参阅 K3422：内容假脱机概述。                                                                                                                                                                                                                                                                                                                                                                                                          |
| Proxy Options代理选项                      | Disabled已禁用                                                                                      | Specifies, when checked (enabled), that the system advertises an option (such as time stamps) to the server only when the option is negotiated with the client.指定检查（启用）时，系统仅在与客户端协商该选项时向服务器宣传选项（例如时间戳记）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Send Buffer发送缓冲区                       | 262144                                                                                           | This setting causes the BIG-IP system to send the buffer size in bytes. To optimize LAN-based traffic, this setting should be at least 64 K in order to allow the BIG-IP system to output more data at a time, if it is allowed by the congestion window.此设置使 BIG-IP 系统以字节为单位发送缓冲区大小。为了优化基于 LAN 的流量，此设置应至少为 64 K，以允许 BIG-IP 系统在拥塞窗口允许的情况下一次输出更多数据。                                                                                                                                                                                                                                                                                                                       |
| Receive Window接收窗口                     | 131072                                                                                           | This setting causes the BIG-IP system to receive the window size in bytes. If this setting is set too low in a LAN environment it can cause delays, as some systems inhibit data transfers if the receive window is too small.此设置使 BIG-IP 系统接收以字节为单位的窗口大小。如果在 LAN 环境中将此设置设置得太低，可能会导致延迟，因为如果接收窗口太小，某些系统会禁止数据传输。                                                                                                                                                                                                                                                                                                                                                           |
| Selective ACKs选择性 ACK                  | Enabled启用                                                                                        | When this setting is enabled, the BIG-IP system can inform the data sender about all segments that it has received, allowing the sender to retransmit only the segments that have been lost.启用此设置后，BIG-IP 系统可以通知数据发送方它已收到的所有段，允许发送方仅重新传输丢失的段。                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Nagle's Algorithm纳格尔算法                 | Auto (BIG-IP 13.1.0 and later)自动（BIG-IP 13.1.0 及更高版本） Enabled (BIG-IP 13.0.x)已启用 (BIG-IP 13.0.x) | When this setting is enabled, the BIG-IP system applies Nagle's algorithm to reduce the number of short segments on the network by holding data until the peer system acknowledges outstanding segments.启用此设置后，BIG-IP 系统应用 Nagle 算法通过保持数据直到对等系统确认未完成的段来减少网络上短段的数量。                                                                                                                                                                                                                                                                                                                                                                                                       |
