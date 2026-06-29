# Network 网络 · 考题（输出倒逼输入）

> 先合上文档独立作答。🟢概念 🟡机制 🔴架构/陷阱

---

## 一、概念题 🟢

1. 一条"网络频道"(NetworkChannel) 对应什么？一个 NetworkManager 能管几条？
2. `IPacketHeader` 里最关键的字段是什么？它的作用是什么？
3. 消息序列化/反序列化由谁负责？框架核心负责什么？
4. `IPacketHandler` 如何路由消息？与 EventPool 的路由有何相似？
5. 心跳 `MissHeartBeatCount` 表示什么？

---

## 二、机制题 🟡

6. 描述 TCP 两阶段拆包的完整流程（包头阶段 → 包体阶段）。
7. `ReceiveState` 如何用 MemoryStream 在"包头长度"和"包体长度"之间切换目标？
8. 一次 socket recv 收到一个半包，框架如何处理那半个包？
9. 一次 recv 收到 3 个完整包，如何保证 3 个都被处理？
10. 心跳的"乐观计数"逻辑是什么？什么时候清零？
11. `ResetHeartBeatElapseSecondsWhenReceivePacket` 开启后有什么效果？

---

## 三、架构 / 陷阱题 🔴

12. 为什么 TCP 需要"定长包头 + 变长包体"才能切包？直接按 recv 边界切会怎样？
13. 拆包时若"没收满目标长度就尝试解析"，会发生什么 bug？
14. 为什么用应用层心跳而不依赖 TCP 自带的 keepalive？
15. Network 是少数不依赖 TaskPool/ObjectPool 的模块，为什么它能/需要自管 socket？
16. `TcpNetworkChannel`(异步) 与 `TcpWithSyncReceiveNetworkChannel`(同步) 共享什么、不同什么？体现什么设计模式？
17. 序列化为什么必须外置给 helper 而不写死在网络层？举一个换协议的场景。
18. 收发为什么用两个独立的 MemoryStream 缓冲？共用一个会有什么问题？

---

## 四、实操题 ✍️

19. 给精简版加"断线重连"：检测到掉线后延迟 N 秒重新 Connect，重试上限 M 次。
20. 设计一个粘包测试：模拟 socket 把两个包的字节拼在一起一次性到达，验证拆包正确。
21. 实现一个 helper：协议格式为 `[2字节协议号][4字节包体长][包体]`，写出 PacketHeaderLength 与反序列化。

---

## 参考答案要点

<details>
<summary>点开核对</summary>

- **1**：一条 socket 连接；可管多条（连不同服务器）。
- **2**：PacketLength(包体长度)；让接收方知道要读多少字节才是一个完整包体。
- **3**：helper(INetworkChannelHelper)；核心只管 socket 字节流收发与切包。
- **4**：按 packet.Id 查注册的 handler 分发；与 EventPool 的 Id 路由同构。
- **6**：收满 PacketHeaderLength → 解析包头得 PacketLength → 切包体阶段 → 收满 PacketLength → 反序列化 → 分发 → 回包头阶段。
- **8**：半包留在缓冲，Position 标记已收量，等下次 recv 补齐到目标长度才前进。
- **9**：while 循环处理，每处理完一个包重置到包头阶段，直到缓冲不足一个完整单元。
- **10**：发心跳即 MissHeartBeatCount++（先假设丢失）；收到回包/任意包时清零；超阈值判掉线。
- **12**：TCP 是字节流无消息边界；按 recv 边界切会粘包(多包黏一起)或拆包(一包被切断)。
- **13**：解析到半个包头/包体 → 长度字段错乱 → 后续所有包错位 → 连接数据流彻底损坏。
- **14**：TCP keepalive 默认超时以分钟计，太慢；应用层心跳可秒级感知断线。
- **15**：网络是 IO 密集的字节流搬运，有自己的状态机(连接/收发/心跳)，与对象复用/任务调度正交，故自管。
- **16**：共享 NetworkChannelBase 的拆包/心跳/分发；不同在"从 socket 取字节"(异步 BeginReceive vs 同步线程 recv)；模板方法模式。
- **17**：核心搬字节、编码外置 → 可换 Protobuf/JSON/私有协议而不动网络层。例：从自定义二进制换 Protobuf，只改 helper。
- **18**：收发并发进行，共用一个流会让发送数据与接收数据互相覆盖/错位。

</details>
