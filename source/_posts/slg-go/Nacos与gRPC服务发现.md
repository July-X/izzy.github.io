---
title: "Nacos + gRPC：游戏服务的注册与发现"
date: 2026-06-04 15:30:00
tags:
  - 游戏后端
  - Go
  - Nacos
  - gRPC
  - 服务发现
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`slg-go` 的三层服务分布在不同机器上，Gate 需要知道 Logic 在哪，Logic 需要知道 Battle 在哪。服务发现用 Nacos，层间通信用 gRPC。

## Nacos 注册与发现

每个服务启动时向 Nacos 注册，关闭时注销。其他服务通过 Nacos 查询可用节点：

```go
// internal/discovery/nacos.go
type Discovery struct {
    client naming_client.INamingClient
}

func (d *Discovery) Register(serviceName string, ip string, port int, metadata map[string]string) error {
    return d.client.RegisterInstance(constant.RegisterInstanceParam{
        Ip:          ip,
        Port:        port,
        ServiceName: serviceName,
        Weight:      10,
        Enable:      true,
        Healthy:     true,
        Metadata:    metadata,
    })
}

func (d *Discovery) Discover(serviceName string) ([]model.Instance, error) {
    result, err := d.client.SelectInstances(constant.SelectInstancesParam{
        ServiceName: serviceName,
        HealthyOnly: true,
    })
    return result, err
}
```

{% mermaid %}
graph TD
    A["Gate 启动"] --> B["注册到 Nacos"]
    C["Logic 启动"] --> D["注册到 Nacos"]
    E["Battle 启动"] --> F["注册到 Nacos"]
    G["Gate 需要转发"] --> H["查询 Nacos 获取 Logic 列表"]
    H --> I["选择实例（一致性哈希）"]
    I --> J["gRPC 调用"]
{% endmermaid %}

## 一致性哈希：玩家路由

Gate 转发玩家请求到 Logic 时，需要保证同一玩家总是路由到同一 Logic 实例。用一致性哈希：

```go
// internal/discovery/hashring.go
type HashRing struct {
    ring     *consistent.Consistent
    instances []model.Instance
}

func (h *HashRing) Locate(playerID int64) string {
    node, err := h.ring.Locate(strconv.FormatInt(playerID, 10))
    if err != nil {
        return h.instances[0].Ip + ":" + strconv.Itoa(h.instances[0].Port)
    }
    return node.String()
}
```

Nacos 推送节点变更时，一致性哈希只重新映射少量玩家，大部分玩家的路由不变。

## gRPC 服务定义

```protobuf
// proto/logic.proto
service LogicService {
    rpc HandleMessage (MessageRequest) returns (MessageResponse);
    rpc BatchSync (BatchSyncRequest) returns (BatchSyncResponse);
    rpc PlayerOnline (OnlineRequest) returns (OnlineResponse);
}

message MessageRequest {
    int64 player_id = 1;
    uint32 route = 2;
    bytes payload = 3;
}

message MessageResponse {
    int32 code = 1;
    bytes payload = 2;
}
```

Go 端实现：

```go
// internal/logic/grpc.go
func (s *Server) HandleMessage(ctx context.Context, req *pb.MessageRequest) (*pb.MessageResponse, error) {
    shard := s.shards[req.PlayerId % int64(len(s.shards))]
    result, err := shard.Handle(ctx, req.PlayerId, uint16(req.Route), req.Payload)
    if err != nil {
        return &pb.MessageResponse{Code: -1}, err
    }
    return &pb.MessageResponse{Code: 0, Payload: result}, nil
}
```

## gRPC vs HTTP

| 维度 | HTTP/JSON | gRPC/Protobuf |
|------|-----------|---------------|
| 序列化 | JSON（文本） | Protobuf（二进制） |
| 体积 | 大 | 小 3-10 倍 |
| 速度 | 慢 | 快 5-10 倍 |
| 浏览器支持 | 原生 | 需要 gRPC-Web |
| 定义语言 | 无 | .proto 文件 |
| 流式支持 | WebSocket | 原生 gRPC Stream |

层间通信用 gRPC（性能优先），客户端到 Gate 用 TCP/WebSocket（兼容性优先）。

## 健康检查

Nacos 通过心跳检测服务健康。服务每 5 秒发送一次心跳，15 秒未收到心跳则标记为不健康：

```go
// 服务端心跳（自动）
// Nacos SDK 内部维护，无需手动实现

// 客户端健康检查
func (d *Discovery) Watch(serviceName string, callback func([]model.Instance)) {
    d.client.Subscribe(&vo.SubscribeParam{
        ServiceName: serviceName,
        Callback: func(services []model.Instance, err error) {
            if err == nil {
                callback(services)
            }
        },
    })
}
```

节点变更时 Nacos 推送事件，Gate 收到后更新一致性哈希环。整个过程对玩家透明，不需要断线重连。

## 部署拓扑

```
Nacos 集群（3 节点）
    ├── Gate × 4（注册到 Nacos）
    ├── Logic × 40（注册到 Nacos）
    ├── Battle × 15（注册到 Nacos）
    ├── Chat × 5（注册到 Nacos）
    └── Redis Cluster（3 主 3 从）
```

Nacos 集群 3 节点保证高可用。所有服务通过 Nacos 发现彼此，不需要硬编码 IP 地址。

> 服务发现的本质是"解耦"。Gate 不需要知道 Logic 有多少台，只需要问 Nacos "Logic 在哪"。