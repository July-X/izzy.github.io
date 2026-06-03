---
title: "Hub 模式：游戏后端 WebSocket 实时联机的架构设计"
date: 2025-06-04 11:00:00
tags:
  - 游戏后端
  - WebSocket
  - Go
  - 实时联机
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`game-baas` 的实时联机系统基于 `gorilla/websocket`，采用 **Hub 模式**管理房间和玩家连接。所有操作通过 channel 串行化到一个主事件循环中，避免并发竞态。

## Hub 模式：中央事件循环

核心设计：Hub 是唯一的事件处理中心，所有房间管理、消息广播、匹配池扫描都在 Hub 的主循环里串行执行：

```go
type Hub struct {
    rooms       map[string]*Room
    players     map[string]*Player
    matches     map[string]*AuthoritativeMatch
    register    chan *Player        // 新玩家连接
    unregister  chan *Player        // 玩家断开
    broadcast   chan *BroadcastMsg  // 房间广播
    pingTicker  *time.Ticker       // 心跳检测
    poolTicker  *time.Ticker       // 匹配池扫描
    stopCh      chan struct{}
}

func (h *Hub) Run() {
    for {
        select {
        case player := <-h.register:
            h.handleRegister(player)
        case player := <-h.unregister:
            h.handleUnregister(player)
        case msg := <-h.broadcast:
            h.handleBroadcast(msg)
        case <-h.pingTicker.C:
            h.handlePing()
        case <-h.poolTicker.C:
            h.scanMatchPool()
        case <-h.stopCh:
            h.shutdown()
            return
        }
    }
}
```

{% mermaid %}
graph TD
    A["WebSocket 连接"] --> B["Hub.register channel"]
    B --> C["Hub 主循环"]
    C --> D{"事件类型"}
    D -->|"register"| E["加入房间"]
    D -->|"unregister"| F["离开房间"]
    D -->|"broadcast"| G["房间广播"]
    D -->|"ping"| H["心跳检测"]
    D -->|"pool"| I["匹配池扫描"]
{% endmermaid %}

为什么用 channel 串行化而不是 mutex？因为 Hub 需要处理的事件类型多（注册、注销、广播、心跳、匹配），用 channel 可以保证每个事件的处理是原子的，不需要担心锁的粒度和死锁。

## Room：房间广播

每个 Room 维护一个玩家集合，广播时遍历所有连接写入消息：

```go
type Room struct {
    id      string
    players map[string]*Player
    hub     *Hub
}

func (r *Room) Broadcast(msg []byte) {
    for _, p := range r.players {
        select {
        case p.send <- msg:
            // 写入成功
        default:
            // 发送缓冲区满，断开连接
            r.hub.unregister <- p
        }
    }
}
```

`select` + `default` 的模式避免阻塞——如果玩家的发送缓冲区满（网络慢），直接断开而不是拖慢整个房间。

## 权威 Match：服务端权威

对于需要反作弊的对战场景，game-baas 提供了权威 Match 模式。服务端维护游戏状态，客户端只发送输入：

```go
type AuthoritativeMatch struct {
    id        string
    players   map[string]*Player
    state     *GameState
    inputCh   chan *PlayerInput
    tickRate  time.Duration  // 默认 20Hz
}

func (m *AuthoritativeMatch) Run() {
    ticker := time.NewTicker(m.tickRate)
    defer ticker.Stop()

    for {
        select {
        case input := <-m.inputCh:
            m.state.ApplyInput(input)  // 应用客户端输入
        case <-ticker.C:
            m.state.Tick()             // 服务端物理模拟
            m.broadcastState()         // 广播最新状态
        }
    }
}
```

服务端以固定频率（20Hz）tick，每 tick 收集所有客户端输入，应用到游戏状态，然后广播最新状态。客户端收到状态后做插值显示。

## 连接生命周期

```go
// 握手阶段
func (h *Hub) handleRegister(p *Player) {
    h.players[p.id] = p
    // 自动加入默认房间
    room := h.getOrCreateRoom("lobby")
    room.players[p.id] = p
}

// 心跳检测
func (h *Hub) handlePing() {
    for _, p := range h.players {
        if time.Since(p.lastPong) > 30*time.Second {
            h.unregister <- p  // 超时断开
            continue
        }
        p.send <- pingMessage
    }
}

// 断线处理
func (h *Hub) handleUnregister(p *Player) {
    delete(h.players, p.id)
    for _, room := range h.rooms {
        delete(room.players, p.id)
    }
    close(p.send)
    p.conn.Close()
}
```

心跳间隔 10 秒，超时阈值 30 秒（3 次心跳未响应）。断线时从所有房间移除，关闭发送 channel，释放连接资源。

## 跨节点通信：Redis Pub/Sub

单机 Hub 只能管理本机连接。多机部署时，房间广播需要跨节点同步：

```go
// 跨节点广播
func (h *Hub) publishToRedis(roomID string, msg []byte) {
    h.rdb.Publish(context.Background(), "room:"+roomID, msg)
}

// 订阅其他节点的消息
func (h *Hub) subscribeRedis() {
    pubsub := h.rdb.Subscribe(context.Background(), "room:*")
    for msg := range pubsub.Channel() {
        roomID := extractRoomID(msg.Channel)
        if room, ok := h.rooms[roomID]; ok {
            room.localBroadcast([]byte(msg.Payload))  // 只广播给本地连接
        }
    }
}
```

本地广播直接写 WebSocket，跨节点广播走 Redis Pub/Sub。收到订阅消息后只广播给本节点的连接，避免重复。

## 适用场景

Hub 模式适合：
- 房间人数 < 100（单房间广播是 O(n)）
- 低频状态更新（< 50Hz）
- 需要房间管理功能（聊天、匹配、观战）

不适合：
- 大规模 MMO（需要空间分区、AOI 算法）
- 超高频状态同步（> 60Hz，需要专用二进制协议）

> Hub 模式的核心价值不是"性能最优"，是"架构最简"。一个 channel 驱动的事件循环，覆盖了房间管理、消息广播、心跳检测、匹配池扫描的全部需求。