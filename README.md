# IEC 60870-5-103 子站 (Go)

基于 **IEC 60870-5-103:1997 / DL/T 667-1999** 实现的继电保护设备信息接口子站 (Slave / IED) Go 语言库。

如果需要技术支持，可以联系dHNjbGFiemhhbmdAZm94bWFpbC5jb20=

> 标准全称: *IEC 60870-5 第 103 节 - 继电保护设备信息接口配套标准*。

## 协议覆盖

| 层级 | 内容 | 实现状态 |
|------|------|----------|
| 物理层 | RS-485 / 光纤 / TCP 透传 | TCP, 任意 `io.ReadWriteCloser` |
| 链路层 | FT1.2 帧格式 + 不平衡传输 | 完整 |
| 应用层 | TYP/VSQ/COT/CommonAddr + Information Object | 完整 |
| 时间格式 | CP32Time2a, CP56Time2a | 完整 |
| 监视方向 ASDU | 1,3,5,6,8,9,11,23,26,27,28,29,30,31 | 完整 |
| 控制方向 ASDU | 6 (时钟同步), 7 (总召唤), 20 (一般命令), 24/25 (录波) | 完整解析 |
| 应用功能 | 站初始化、总召唤、时钟同步、命令传输、循环测量 | 完整 |
| 故障录波 (ASDU 23-31) | 完整状态机 + 分片传输 + 中止/否定 ACK 处理 | 完整 |
| 通用分类服务 (ASDU 10/21) | -- | 预留接口, 未实现 |

## 项目结构

```
stand103/
├── go.mod                      # Go module
├── iec103/                     # 协议核心库
│   ├── const.go                # 帧/控制域/COT/类型标识/TOO 等常量
│   ├── frame.go                # FT1.2 帧编解码 (含同步与校验)
│   ├── time.go                 # CP32/CP56Time2a 时间格式
│   ├── asdu.go                 # ASDU 基础结构与编解码
│   ├── monitor_asdu.go         # 监视方向 ASDU 构造器 (常规)
│   ├── control_asdu.go         # 控制方向 ASDU 解析 (常规)
│   ├── disturbance.go          # 录波数据模型 + ASDU 23/24/25/26/27/28/29/30/31
│   ├── disturbance_session.go  # 录波传输会话状态机
│   ├── link.go                 # 链路层不平衡传输状态机
│   ├── transport.go            # TCP/conn 传输适配
│   ├── slave.go                # 子站核心 (状态机+回调)
│   └── *_test.go               # 单元测试
├── examples/
│   ├── slave/main.go           # TCP 子站示例
│   └── master/main.go          # 主站联调客户端
└── tests/
    └── integration_test.go     # 端到端集成测试 (基于 net.Pipe)
```

## 快速开始

### 子站

```go
package main

import (
    "context"
    "net"
    "time"

    "github.com/stand103/iec103/iec103"
)

type myHandler struct{}

func (myHandler) OnInitialize(s *iec103.Slave) {
    asdu := iec103.BuildIdentification(iec103.COTPowerOn, s.CommonAddress(), 2, "IED-001")
    s.EnqueueClass1(asdu)
}

func (myHandler) OnTimeSync(s *iec103.Slave, msg iec103.TimeSync) {
    // 在此应用主站时间到本机 RTC
}

func (myHandler) OnGeneralInterrogation(s *iec103.Slave, scan uint8) {
    // 应用层将本站全部状态量逐条 Enqueue 后调用 FinishGeneralInterrogation
    s.FinishGeneralInterrogation()
}

func (myHandler) OnGeneralCommand(s *iec103.Slave, cmd iec103.GeneralCommand) bool {
    return true // 接受命令
}

func (myHandler) OnUnknownASDU(s *iec103.Slave, a iec103.ASDU) {}

func main() {
    listener, _ := net.Listen("tcp", ":2404")
    for {
        conn, _ := listener.Accept()
        go func(c net.Conn) {
            slave := iec103.NewSlave(iec103.WrapConn(c), iec103.SlaveOptions{
                LinkAddress: 1, CommonAddr: 1, Identifier: "IED-001",
            }, myHandler{})
            defer slave.Close()
            ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
            defer cancel()
            slave.Run(ctx)
        }(conn)
    }
}
```
