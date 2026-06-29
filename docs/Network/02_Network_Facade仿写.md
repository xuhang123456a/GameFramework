# Network 网络 · Facade 精简仿写

> 目标：~150 行，复刻"TCP 两阶段拆包 + 收发双缓冲 + 心跳保活 + 序列化外置 + 按 Id 路由"。
> 取舍：用同步 socket 演示；序列化/协议交给注入 helper。重点是两阶段拆包状态机。

---

## 1. 设计映射表

| 原框架 | 精简版 | 是否保留 |
|--------|--------|----------|
| 两阶段拆包(包头→包体) | 同 | **保留**（粘包核心） |
| `IPacketHeader.PacketLength` | 同 | **保留** |
| `INetworkChannelHelper` 序列化外置 | 同 | **保留** |
| ReceiveState/SendState 双缓冲 | MemoryStream | 保留 |
| 心跳乐观计数 + 收包重置 | 同 | **保留** |
| 按 Id 路由 handler | 同 | 保留 |
| 异步/同步两频道 | 仅同步骨架 | 简化 |

---

## 2. 核心代码

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Net.Sockets;

namespace MiniNetwork
{
    public abstract class Packet { public abstract int Id { get; } }
    public interface IPacketHeader { int PacketLength { get; } }
    public interface IPacketHandler { int Id { get; } void Handle(object sender, Packet packet); }

    // 序列化/协议外置：框架只搬字节
    public interface INetworkChannelHelper
    {
        int PacketHeaderLength { get; }
        void Serialize(Packet packet, Stream dest);
        IPacketHeader DeserializePacketHeader(byte[] buffer, int length);
        Packet DeserializePacket(IPacketHeader header, byte[] buffer, int length);
        Packet BuildHeartBeat();
    }

    public sealed class NetworkChannel : IDisposable
    {
        private readonly INetworkChannelHelper m_Helper;
        private readonly Dictionary<int, IPacketHandler> m_Handlers = new();
        private Socket m_Socket;

        // 双缓冲
        private readonly byte[] m_ReceiveBuffer = new byte[1024 * 64];
        private int m_ReceivedLength;          // 当前缓冲已收字节
        private bool m_ReceivingHeader = true; // 拆包阶段标志
        private IPacketHeader m_CurrentHeader;
        private readonly MemoryStream m_SendStream = new(1024 * 64);

        // 心跳
        public float HeartBeatInterval = 30f;
        private float m_HeartBeatElapse;
        private int m_MissHeartBeatCount;
        public int MissHeartBeatCount => m_MissHeartBeatCount;
        public bool Connected => m_Socket != null && m_Socket.Connected;

        public NetworkChannel(INetworkChannelHelper helper) => m_Helper = helper;

        public void RegisterHandler(IPacketHandler h) => m_Handlers[h.Id] = h;

        public void Connect(IPAddress ip, int port)
        {
            m_Socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            m_Socket.Connect(ip, port);
            m_ReceivingHeader = true;
            m_ReceivedLength = 0;
            m_MissHeartBeatCount = 0; m_HeartBeatElapse = 0f;
        }

        public void Send(Packet packet)
        {
            m_SendStream.Position = 0; m_SendStream.SetLength(0);
            m_Helper.Serialize(packet, m_SendStream);     // helper 写包头+包体
            var data = m_SendStream.GetBuffer();
            m_Socket.Send(data, 0, (int)m_SendStream.Length, SocketFlags.None);
        }

        public void Update(float dt)
        {
            if (!Connected) return;
            ProcessReceive();
            ProcessHeartBeat(dt);
        }

        // ---- 两阶段拆包核心 ----
        private void ProcessReceive()
        {
            while (m_Socket.Available > 0)
            {
                int target = m_ReceivingHeader
                    ? m_Helper.PacketHeaderLength            // 包头阶段: 目标=定长包头
                    : m_CurrentHeader.PacketLength;          // 包体阶段: 目标=包头声明的包体长

                int need = target - m_ReceivedLength;
                int got = m_Socket.Receive(m_ReceiveBuffer, m_ReceivedLength, need, SocketFlags.None);
                if (got <= 0) break;
                m_ReceivedLength += got;

                if (m_ReceivedLength < target) break;        // 未收满 → 留缓冲等下次

                if (m_ReceivingHeader)                        // 收满包头 → 解析长度，切到包体阶段
                {
                    m_CurrentHeader = m_Helper.DeserializePacketHeader(m_ReceiveBuffer, target);
                    m_ReceivingHeader = false;
                    m_ReceivedLength = 0;
                }
                else                                          // 收满包体 → 反序列化 + 分发
                {
                    var packet = m_Helper.DeserializePacket(m_CurrentHeader, m_ReceiveBuffer, target);
                    Dispatch(packet);
                    m_ReceivingHeader = true;                 // 回到包头阶段，准备下一个包
                    m_ReceivedLength = 0;
                    m_MissHeartBeatCount = 0;                 // 收到包 → 心跳计数清零
                    m_HeartBeatElapse = 0f;
                }
            }
        }

        private void Dispatch(Packet packet)
        {
            if (m_Handlers.TryGetValue(packet.Id, out var h)) h.Handle(this, packet);
            else Console.WriteLine($"No handler for packet id {packet.Id}");
        }

        private void ProcessHeartBeat(float dt)
        {
            m_HeartBeatElapse += dt;
            if (m_HeartBeatElapse < HeartBeatInterval) return;
            m_HeartBeatElapse = 0f;
            Send(m_Helper.BuildHeartBeat());
            m_MissHeartBeatCount++;                          // 乐观+1, 等回包清零
            if (m_MissHeartBeatCount > 3)
                Console.WriteLine("连接疑似掉线（连续丢失心跳）");
        }

        public void Close() { m_Socket?.Close(); m_Socket = null; }
        public void Dispose() { Close(); m_SendStream.Dispose(); }
    }
}
```

---

## 3. 使用示例（伪 helper）

```csharp
// 协议: [4字节包体长][1字节协议号][包体]
sealed class ChatPacket : Packet { public override int Id => 100; public string Text; }

sealed class DemoHelper : INetworkChannelHelper
{
    public int PacketHeaderLength => 4;
    public void Serialize(Packet p, Stream dest)
    {
        var body = System.Text.Encoding.UTF8.GetBytes($"{p.Id}:{(p as ChatPacket)?.Text}");
        var lenBytes = BitConverter.GetBytes(body.Length);
        dest.Write(lenBytes, 0, 4); dest.Write(body, 0, body.Length);
    }
    public IPacketHeader DeserializePacketHeader(byte[] buf, int len)
        => new Header(BitConverter.ToInt32(buf, 0));
    public Packet DeserializePacket(IPacketHeader h, byte[] buf, int len)
    {
        var s = System.Text.Encoding.UTF8.GetString(buf, 0, len);
        return new ChatPacket { Text = s };
    }
    public Packet BuildHeartBeat() => new ChatPacket { Text = "ping" };
    sealed class Header : IPacketHeader { public Header(int l) => PacketLength = l; public int PacketLength { get; } }
}

sealed class ChatHandler : IPacketHandler
{
    public int Id => 100;
    public void Handle(object sender, Packet p) => Console.WriteLine($"收到: {(p as ChatPacket)?.Text}");
}

var ch = new NetworkChannel(new DemoHelper());
ch.RegisterHandler(new ChatHandler());
ch.Connect(IPAddress.Loopback, 9000);
ch.Send(new ChatPacket { Text = "hello" });
// 帧循环: while(true) { ch.Update(0.016f); }
```

---

## 4. 取舍自检

- ✅ 保留：两阶段拆包(收满才前进、剩余留缓冲、循环处理多包)、收发双缓冲、心跳乐观计数+收包重置、序列化外置 helper、按 Id 路由。
- ❌ 砍掉：异步接收(BeginReceive)、多频道管理、Packet 的 ReferencePool 复用、5 个网络事件、自定义错误数据。
- ⚠️ 提醒：拆包必须"收满目标长度才解析"，半包绝不能解析；一次可读多个包要 while 循环处理。心跳要用应用层秒级探测，别指望 TCP 自带保活。序列化写死在网络层是反模式——传输与编码必须解耦。

---

> 配套考题见 `03_Network_考题.md`。
