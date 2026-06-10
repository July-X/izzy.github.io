---
title: "Flow 粒子动画：UDP 消息流的可视化实现"
date: 2026-06-08 20:00:00
tags:
  - Go
  - Electron
  - 可视化
  - SSE
  - AntV X6
categories:
  - mingmo-server-go
top_img: false
---

<!-- more -->

游戏联机服务器的 UDP 流量监控一直是个麻烦事。日志里密密麻麻的 ingress/forward/drop 事件，运维人员需要在海量数据中肉眼定位异常包。我们给 `mingmo-server-go` 的 UDP Proxy 加了一个实时可视化面板，把消息流变成了拓扑图上的粒子动画。

这篇文章讲的是这个可视化系统的实现，重点放在 Flow 粒子动画的设计和踩坑。

## 问题：为什么需要消息流可视化

UDP Proxy 转发层每秒处理数万个包。传统的日志排查方式有两个痛点：

1. **延迟高**：从出问题到发现问题，中间隔着日志采集、落盘、查询，至少几分钟
2. **上下文丢失**：单条日志只能看到"收到了包"，看不到"这个包从哪来、到哪去、有没有丢"

我们需要一个实时的、有拓扑关系的可视化工具。

## 方案演进：从 strokeDashoffset 到 atConnectionRatio

我们在 X6 重写时经历了 5 次方案迭代，最终选定了 `atConnectionRatio` + 直接 SVG DOM 操作。

### Round5：strokeDashoffset 光流动画（已废弃）

最初的方案是用 X6 边的 `strokeDasharray` + `strokeDashoffset` 动画，让虚线"流动"：

```javascript
const edge = graph.addEdge({
  id: `flow-${Date.now()}`,
  source: [from.x, from.y],
  target: [to.x, to.y],
  attrs: {
    line: {
      stroke: color,
      strokeWidth: width,
      strokeDasharray: Math.max(4, Math.round(len / 14)),
    },
  },
});

edge.animate(
  { "attrs/line/strokeDashoffset": -len },
  { duration: 1200, iterations: Infinity, easing: "linear" },
);
```

**废弃原因**：双向网络通道共用一条动画，看不出上行/下行方向；视觉上"线条一直在流动"，空闲态和事件态分不清；持续动画耗 CPU。

### Round6（中间态）：自定义 NodeView 画粒子

继承 X6 的 `NodeView`，每帧 rAF 算 bezier 当前位置 + 画 trail。

**废弃原因**：X6 自动渲染不参与，节点管理和边生命周期复杂；trail 画完后留下"鬼影"。

### Round6（最终）：atConnectionRatio + SVG setAttribute（生产方案）

最终方案是在静态边的 markup 里注入一个 `<circle selector="marker">`，rAF 推它的 `atConnectionRatio` 让圆点沿 path 飞：

```javascript
// 取 X6 Edge 默认 markup（不破坏它的结构，只是在前面插入 marker）
const edgeMarkup = X6.Shape.Edge.getMarkup();

graph.addEdge({
    id: `e-player-${uid}`,
    source: { cell: "proxy", port: `p${slot}` },
    target: { cell: `player-${uid}`, anchor: { name: "center" } },
    router: { name: "manhattan", args: { startDirections: ["bottom"], endDirections: ["left"] } },
    connector: { name: "smooth" },
    // 注入自定义 marker（OSCP 官方例子 pattern）
    markup: [
        { tagName: "circle", selector: "marker", attrs: { stroke: "none", r: 5 } },
        ...edgeMarkup,
    ],
    attrs: {
        line: {
            stroke: roomLineColor(roomId),
            strokeWidth: 1.4,
            strokeOpacity: 0.7,
            sourceMarker: { name: "classic", size: 8, fill: roomLineColor(roomId) },
            targetMarker: { name: "classic", size: 8, fill: roomLineColor(roomId) },
        },
        marker: {
            fill: roomLineColor(roomId),
            atConnectionRatio: 0,  // 沿 path 位置（0=source, 1=target）
            opacity: 0,            // 平时隐藏
        },
    },
});
```

为什么 markup 要插在 `getMarkup()` 前面？X6 渲染时按 markup 顺序生成 SVG 子元素。marker 在最前面 → 在 path 之下 → 圆点从 path 上方飘过时不会被 path 盖住。

burst 动画时，直接用 SVG `setAttribute` 推 ratio：

```javascript
function runMarkerBurst(edge, fromT, toT, duration, color, onComplete) {
    const view = graph.findViewByCell(edge);
    const markerEl = view.findOne("marker");

    const setMarkerAtRatio = (r) => {
        const tangent = view.getTangentAtRatio(r);
        const p = tangent.start;
        const angle = tangent.vector.vectorAngle({ x: 1, y: 0 });
        markerEl.setAttribute("transform",
            `matrix(1,0,0,1,${p.x},${p.y}) rotate(${angle})`);
    };

    markerEl.setAttribute("opacity", "1");
    markerEl.setAttribute("fill", color);
    setMarkerAtRatio(fromT);

    const start = performance.now();
    const tick = (now) => {
        const t = Math.min(1, (now - start) / duration);
        const eased = 1 - Math.pow(1 - t, 2);   // ease-out
        const ratio = fromT + (toT - fromT) * eased;
        setMarkerAtRatio(ratio);
        if (t < 1) requestAnimationFrame(tick);
        else { markerEl.setAttribute("opacity", "0"); onComplete(); }
    };
    requestAnimationFrame(tick);
}
```

**关键点**：`view.getTangentAtRatio(r)` 用 X6 内部的 Path 切线算法，知道 router+connector 算出的 path 长什么样。transform 直接用 SVG matrix 语法，不用 X6 的 attr 处理器（原因见下面的"核心 bug"章节）。

## 核心实现：flow 类型映射

UDP 流量事件有 9 种类型，每种有 3 种视觉表达之一（burst / 高亮 / toast）：

| kind | 来源 | 视觉表达 | 持续时间 | 顺序约束 |
|------|------|----------|----------|----------|
| ingress | 客户端 → proxy port | burst 绿点 player→port | 3s | 必须先于 forward |
| forward | proxy → 客户端广播 | burst 橙点 port→player | 6s | 必须后于 ingress |
| handshake | Hello 包到达 | burst 绿点（同 ingress） | 3s | 不参与 |
| bind | proxy 内部绑定 | 静态边加粗 300ms | 300ms | 不参与 |
| join | 玩家加入房间 | 静态边加粗 + toast | 300ms+1.8s | 不参与 |
| leave | 玩家离开房间 | toast | 1.8s | 不参与 |
| drop | 包被丢弃 | 静态边红色高亮 200ms | 200ms | 不参与 |
| send | Test 控件触发 | toast（不画粒子） | 1.8s | 不参与 |
| room | 房间元事件 | 仅计数 | — | 不参与 |

颜色按方向区分：ingress 绿 `#22c55e`，forward 橙 `#f97316`。drop 红 `#ef4444`。

`enqueueFlow` 是入口函数，根据 `flow.kind` 分发到不同的绘制逻辑：

```javascript
function enqueueFlow(flow) {
  if (!graph || !flow) return;
  const style = cfg.FLOW_STYLES[flow.kind] || cfg.FLOW_STYLES.ingress;

  // 计数（除 send 外）
  if (flow.kind !== "send") {
    if (Number.isInteger(flow.slot)) bumpPortActivity(flow.slot, flow.kind);
    if (flow.roomId) bumpRoomActivity(flow.roomId, flow.kind);
  }

  if (flow.kind === "send") {
    // send 是控制触发器，但前端要画出对应的 ingress + forward 粒子
    // 让用户能直观看到"消息从 sender 进 proxy，再被 proxy 广播给同房间其他人"的完整链路
    const eps = resolveEndpoints(flow);
    // ... 绘制上行和下行粒子
    return;
  }
  // ... 其他类型的处理
}
```

## 时序控制：BURST 和 STAGGER

单个粒子太单调。我们用 BURST（5个粒子）+ STAGGER（180ms 间隔）来模拟"一串数据包"的感觉：

```javascript
const BURST = 5;      // 每次事件触发的粒子数
const STAGGER = 180;  // 粒子错开的启动时间（ms）
const T_INBOUND = 1200;  // 上行传输时间
const T_WAIT = 350;      // Proxy 处理延迟
const T_OUTBOUND = 1000; // 下行传输时间
```

上行粒子先出发，等待 T_WAIT 后下行粒子才开始。这模拟了真实的数据流时序：包到了 Proxy，处理一下，再转发出去。

```javascript
// 上行：sender → proxy port（蓝色 ingress 粒子，BURST 个）
for (let i = 0; i < cfg.BURST; i++) {
  setTimeout(() => {
    spawnFlowEdge({
      from: eps.sender,
      to: eps.target,
      color: cfg.FLOW_STYLES.ingress.color,
      width: 2.4,
      duration: cfg.T_INBOUND,
    });
  }, i * cfg.STAGGER);
}

// 下行：proxy → 同 room 其他玩家（绿色 forward 粒子）
// 等上行 + proxy 处理完成后再画下行
const startAfter = cfg.T_INBOUND + cfg.T_WAIT;
others.forEach((p, i) => {
  setTimeout(() => {
    // ... 绘制下行粒子
  }, startAfter + i * 100);
});
```

## 活动衰减：滑动窗口算法

端口和房间的活动计数需要"慢慢衰减"，而不是瞬间归零。我们用指数衰减实现：

```javascript
function decayActivity() {
  const decay = 0.96;
  for (const p of portActivity) {
    if (p) { p.rx *= decay; p.fwd *= decay; p.drop *= decay; }
  }
  for (const a of MM.store.roomActivity.values()) {
    a.rx *= decay; a.fwd *= decay;
  }
}
```

每 200ms 执行一次衰减。0.96 的衰减系数意味着：如果一个端口在某一秒有 100 次 ingress，10 秒后这个计数会降到 66（100 × 0.96^50）。

这个衰减直接驱动端口颜色变化——活动越多，端口颜色越亮。

## 后端：EventHub 手写 JSON 序列化

后端的 `EventHub` 负责把 FlowEvent 广播给所有 SSE 订阅者。为了压榨性能，我们手写了 JSON 序列化，避免 `json.Marshal` 的反射开销：

```go
func (ev FlowEvent) appendJSON(dst []byte) []byte {
  dst = append(dst, '{')
  first := true
  writeStr := func(key, val string) {
    if val == "" { return }
    if !first { dst = append(dst, ',') }
    first = false
    dst = append(dst, '"')
    dst = append(dst, key...)
    dst = append(dst, '"', ':', '"')
    // 转义特殊字符
    for i := 0; i < len(val); i++ {
      switch val[i] {
      case '"':  dst = append(dst, '\\', '"')
      case '\\': dst = append(dst, '\\', '\\')
      case '\n': dst = append(dst, '\\', 'n')
      default:   dst = append(dst, val[i])
      }
    }
    dst = append(dst, '"')
  }
  // ... 其他字段
  writeStr("type", ev.Type)
  writeStr("uid", ev.UID)
  // ...
  return dst
}
```

实测比 `json.Marshal` 快约 1 微秒，少 4 次内存分配。在高 PPS 场景下，这个差距会累积。

## 采样控制：高频事件降级

生产环境的 PPS 可能达到数万，全量推送到前端会卡死。EventHub 支持按环境变量配置采样率：

```go
func (h *EventHub) Publish(ev FlowEvent) {
  // 采样：高频事件（ingress/forward）按 sampleRate 跳过
  if h.sampleRate > 0 && (ev.Type == "ingress" || ev.Type == "forward") {
    count := h.sampleCounter.Add(1)
    if count%h.sampleRate != 0 {
      return
    }
  }
  // ...
}
```

通过 `MM_PROXY_EVENT_SAMPLE_RATE=10` 可以只发送十分之一的 ingress/forward 事件。drop、handshake、debug 这些低频事件始终 100% 发送。

## SSE 双通道：events 和 metrics

前端维护两个独立的 SSE 连接：

- `/proxy/events`：推送 FlowEvent（粒子动画的数据源）
- `/proxy/metrics`：推送 PPS、Rooms、Players、Drop Rate、Memory

两个通道独立重连，互不影响。`sse.js` 里实现了去抖逻辑——如果 3 秒内连续触发 error，只执行一次重连：

```javascript
eventSource.onerror = () => {
  const now = performance.now();
  // 去抖：距上次 error 不满 RECONNECT_MS 则丢弃
  if (now - sseLastErrorAt < cfg.RECONNECT_MS) return;
  sseLastErrorAt = now;
  scheduleReconnect("events");
};
```

## 核心 bug：atConnectionRatio 在 rAF 中不同步 DOM

这是我们踩过的最深的坑，花了两天才定位到根因。

表现是这样的：rAF 循环里用 `edge.attr('attrs/marker/atConnectionRatio', t)` 改了 model，但 view（DOM）的 `transform` 长时间不更新。肉眼看到 burst 斑点卡在初始位置不动。

根因是 AntV X6 3.x 的 view 是 async render（内部用 `queue.queueFlush`）。即使你整对象替换 `edge.attr('attrs/marker', {...})`，X6 也不保证下一帧 DOM 已更新。

最初我们以为是自己的代码问题，试了各种"强制刷新"的 hack，都不管用。最后翻 X6 源码才发现这是框架层面的限制。

**解法**：绕开 X6 的 attr 处理器，直接操作 SVG DOM：

```javascript
function runMarkerBurst(edge, fromT, toT, duration, color, onComplete) {
    const view = graph.findViewByCell(edge);
    const markerEl = view.findOne("marker"); // 直接拿注入的 <circle>

    const setMarkerAtRatio = (r) => {
        const tangent = view.getTangentAtRatio(r);
        const p = tangent.start;
        const angle = tangent.vector.vectorAngle({ x: 1, y: 0 });
        markerEl.setAttribute("transform",
            `matrix(1,0,0,1,${p.x},${p.y}) rotate(${angle})`);
    };

    markerEl.setAttribute("opacity", "1");
    markerEl.setAttribute("fill", color);    // 直接 SVG setAttribute
    setMarkerAtRatio(fromT);

    const start = performance.now();
    const tick = (now) => {
        const t = Math.min(1, (now - start) / duration);
        const eased = 1 - Math.pow(1 - t, 2);
        const ratio = fromT + (toT - fromT) * eased;
        setMarkerAtRatio(ratio);             // 直接 SVG DOM，零延迟
        if (t < 1) requestAnimationFrame(tick);
        else { markerEl.setAttribute("opacity", "0"); onComplete(); }
    };
    requestAnimationFrame(tick);
}
```

三个关键点：
1. `view.getTangentAtRatio(r)` 用 X6 内部 Path 切线算法，知道 router+connector 算出的 path 长什么样
2. transform 直接用 SVG matrix 语法，不用 X6 的 `translate(x,y) rotate(angle)`——matrix 更稳，避免 X6 attr 处理器二次解析
3. `setAttribute` 同步生效，不走 X6 的 render queue

burst 染色也有同样的坑：用 `edge.attr('attrs/marker/fill', color)` 改颜色，结果改完颜色不变，或者跑完一次 burst 后颜色回滚到初始的 room color。X6 的 attr 处理器在 rAF 推 atConnectionRatio 时可能"擦掉"其他 attr 子项。同样改成 `markerEl.setAttribute("fill", color)` 解决。

**一句话口诀**：改静态样式用 `cell.attr`；改动画瞬时样式用 `setAttribute`。

## 踩过的坑

### Port 系统接入：position.absolute 不同步到 cx/cy

X6 的 `position.absolute.args.{x, y}` 不会自动同步到 port 元素的 `cx/cy`。症状是 port 元素用 `position: { name: "absolute", args: { x: 0, y: 0 } }`，结果 port 跑回 (0,0)。

解法：用 `position: { name: "bottom-line" }`，这是 X6 内置的 port 位置算法，自动按 proxy 底边均匀分布。

另一个问题：X6 没有公开 API 直接给"port 在画布上的 (x, y)"，因为 port 是 proxy 的子元素，proxy 移动 port 就跟着移动。我们必须在 `ensurePorts()` 里同步算一次，存到 `portCenters[slot]` 缓存里：

```javascript
const proxyPos = proxyCell.getPosition();
const proxyTransform = proxyCell.getTransform();
portCenters[slot] = {
    x: proxyPos.x + (relativeX + portR) * scaleX,
    y: proxyPos.y + proxyHeight * scaleY,
};
```

其他模块（x6-flow 算连线端点）只读这个缓存，不重新从 graph 查。

### HTML 节点（room-card）的 effect 双订阅

X6 HTML 节点用 `effect: ["data", "size"]` 订阅。`cell.resize()` 改尺寸后，HTML 元素的 `height` 不会自动同步——X6 只更新 `<foreignObject>` 的尺寸，div 需要 `height: 100%` + effect 重渲才能跟上。

代码必须这么写：
1. `effect` 加上 `"size"` 订阅
2. HTML 内部 div 用 `height: 100%`
3. 改 size 的代码要**先 `cell.resize()` 再 `cell.setData()`**，触发 effect 跑一次完整的 html() 重渲

### connector:"smooth" 导致动画不启动

最初用 `edge.animate` 配合 `view.selector("line")` 拿 `<line>` 元素做动画。但 `connector: "smooth"` 在 X6 3.1.7 里渲染为 `<path>`，WAAPI 永远找不到目标元素。

解法：改成 `edge.animate` 走 X6 的 attrs 系统，让 X6 自己去找正确的元素。后来发现 attrs 系统也有 async render 的问题，最终才改成直接 SVG DOM 操作。

### send 事件的复合绘制

`send` 事件是控制触发器，前端需要画出完整的 ingress → proxy → forward 链路。我们把它拆成两部分：先画上行粒子（sender → proxy），等 T_INBOUND + T_WAIT 后再画下行粒子（proxy → 其他玩家）。这样视觉上能清楚看到"消息从哪来、到哪去"。

### 顺序约束：同一条边的 burst 必须串行播放

ingress（玩家→proxy 上行）必须先播完，forward（proxy→其他玩家下行）才能播。否则会丢失"消息先收后发"的因果链——视觉上看起来像 proxy 在没收到消息时就广播了。

实现方式是在每条边上挂一个队列：

```javascript
function enqueueMarkerBurst(edge, fromT, toT, duration, color) {
    if (!edge.__burstQueue) edge.__burstQueue = [];
    edge.__burstQueue.push({ fromT, toT, duration, color });
    if (!edge.__burstActive) startNextBurst(edge);
}
function startNextBurst(edge) {
    if (edge.__burstQueue.length === 0) {
        edge.__burstActive = false;
        return;
    }
    const job = edge.__burstQueue.shift();
    edge.__burstActive = true;
    runMarkerBurst(edge, job.fromT, job.toT, job.duration, job.color, () => {
        edge.__burstActive = false;
        startNextBurst(edge);
    });
}
```

删边前必须清理队列，否则 rAF 还会引用已死的 cell。

## 文件结构

```
frontend/proxy-visualizer/
├── index.html          # 入口，加载 X6 CDN
├── styles.css          # 节点/动画样式
├── x6-app.js           # 应用入口
└── src/
    ├── config.js       # 时序常量、颜色配置
    ├── store.js        # 全局状态
    ├── sse.js          # SSE 双通道连接
    ├── flow-events.js  # 事件分发
    ├── x6-graph.js     # X6 Graph 初始化
    ├── x6-topology.js  # 拓扑布局、节点增删
    ├── x6-flow.js      # Flow 粒子动画（核心）
    ├── x6-rooms.js     # 房间状态同步
    └── x6-ui.js        # 侧边栏、指标卡
```

## Burst 动画演化时间线

这个系统的 burst 动画经历了 5 次迭代才稳定下来。记录一下每次迭代的决策过程。

| 迭代 | 方案 | 结果 | 根因 |
|------|------|------|------|
| Round5 | strokeDashoffset 光流 | 废弃 | 上下行共用一条动画看不出方向；持续动画耗 CPU |
| Round6 中间态 | 自定义 NodeView 画 trail | 返工 | X6 自动渲染不参与，trail 留下"鬼影" |
| Round6 最终 | atConnectionRatio + edge.attr | 失败 | X6 async render，attr 不同步 DOM |
| Round6 修正 | atConnectionRatio + SVG setAttribute | 采纳 | 直接操作 SVG DOM 绕过 X6 |
| Round6 染色 | 方向分色 + 双箭头 | 定稿 | 绿/橙区分上下行，sourceMarker+targetMarker 固定装饰 |

Round5 → Round6 的转折点是发现"虚线流动"无法表达方向。ingress 是玩家→proxy，forward 是 proxy→玩家，方向相反但视觉上一样，运维分不清。

Round6 中间态到最终的转折点就是那个核心 bug——atConnectionRatio 在 rAF 里不同步 DOM。我们一开始以为是自己的代码问题，后来翻 X6 源码才发现是框架层面 async render 的限制。

最终定稿的参数：ingress 3 秒、forward 6 秒（下行比上行长一倍，因为是 1-to-N 广播）。颜色用 tailwind 色板：ingress 绿 `#22c55e`，forward 橙 `#f97316`。

## 命名约定

节点/边 ID 的命名规则：

| 类型 | ID 模板 | 示例 |
|------|---------|------|
| Proxy 节点 | `proxy` | `proxy` |
| Room 节点（HTML） | `room-${roomId}` | `room-A`, `room-7f3a` |
| Player pin | `player-${uid}` | `player-70310000abcdef` |
| 静态边（proxy → room） | `e-proxy-${roomId}` | `e-proxy-A` |
| Player 边（port → player pin） | `e-player-${uid}` | `e-player-70310000abcdef` |

规则很简单：节点用 `${type}-${业务key}`，边加 `e-` 前缀区分。

模块公共 API 边界也值得注意——跨层只通过三个对象通信：

| 模块 | 公共 API |
|------|----------|
| MM.x6Graph | `LAYOUT`, `slotColor()`, `roomLineColor()`, `bootstrap()` |
| MM.x6Topology | `attach()`, `sync()`, `getRoomNode()`, `triggerPlayerLineAnimation()` |
| MM.flowEngine | `enqueueFlow()`, `decayActivity()`, `showToast()` |

加新功能时先想"这个方法属于哪个模块"——是 layout（x6-topology）、事件分发（x6-flow）、还是数据（store）？放错模块会增加跨模块耦合。

## 性能数据

在本地测试环境（10 个模拟玩家，3 个房间）：

| 指标 | 值 |
|------|-----|
| FPS | 稳定 60 |
| 同时活跃 flow 边数 | 最多 ~50 |
| SSE 消息吞吐 | ~200 条/秒 |
| 前端内存增量 | < 10MB |

每个 flow 边存活 1~1.5 秒后自动清理，所以同时存在的边数有上限。即使 PPS 很高，前端也不会被撑爆。

---

下一步打算做什么？也许是给 flow 粒子加上丢包重试的视觉反馈，或者接入 Prometheus 指标做更细粒度的监控。你觉得还有什么值得加？
