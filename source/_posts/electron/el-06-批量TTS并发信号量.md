---
title: Electron 实战：批量 TTS 并发信号量池的实现
date: 2026-06-04
categories:
  - [Electron实战]
tags:
  - Electron
  - 并发控制
  - 信号量
  - JavaScript
  - TTS
  - 问答
top_img: linear-gradient(135deg, #e65c00 0%, #F9D423 100%)
---

<!-- more -->

> 用户点击"全部生成"后，可能有几十条甚至上百条台词等待合成。如果同时发起所有请求，TTS API 会返回 429（限流错误）。MiMo TTS Studio 用一个 **手写的信号量池** 控制最大并发数为 3——不用任何第三方库。

---

## Q1：为什么并发数限制为 3？不是越大越好吗？

**A：**

| 并发数 | 效果 |
|--------|------|
| **1（串行）** | 安全但太慢，100 条台词可能要 10+ 分钟 |
| **3（当前设置）** | 平衡点：充分利用 API 吞吐，基本不触发限流 |
| **10+** | 大概率触发 429 限流 → 重试 → 更慢 |
| **无限制（全并行）** | API 直接封禁或返回大量错误 |

3 这个数字来自实测：
- MiMo API 单次请求平均耗时 **3~8 秒**
- 并发 3 时总吞吐约 **每分钟 20~25 条**
- 并发超过 5 后 429 错误率急剧上升

> 具体数字取决于你的 API 套餐和服务器负载。这个值做成常量 `MAX_CONCURRENT` 就是为了方便调整。

---

## Q2：信号量池的核心实现只有 20 行？

**A：** 核心逻辑确实非常简洁：

```javascript
// ttsStore.js — 并发控制的完整实现

const _activeRequests = ref(0);     // 当前正在执行的请求数
const MAX_CONCURRENT = 3;            // 最大并发数
const _pendingQueue = [];           // 等待队列（存 resolve 函数）

function _acquireSlot() {
  if (_activeRequests.value < MAX_CONCURRENT) {
    _activeRequests.value++;
    return Promise.resolve();       // 有空位，立即通过
  }
  // 没有空位了，返回一个 pending 的 Promise
  return new Promise(resolve => {
    _pendingQueue.push(resolve);    // 排队等
  });
}

function _releaseSlot() {
  _activeRequests--;
  if (_pendingQueue.length > 0) {
    _activeRequests.value++;         // 队列里有人等着，直接给它用
    const next = _pendingQueue.shift();
    next();                           // 解除那个 pending Promise
  }
}
```

### 工作原理

```
初始状态: activeRequests=0, queue=[]

任务A 到来: acquireSlot()
  activeRequests=0 < 3 → 立即通过, activeRequests=1 ✅

任务B 到来: acquireSlot()
  activeRequests=1 < 3 → 立即通过, activeRequests=2 ✅

任务C 到来: acquireSlot()
  activeRequests=2 < 3 → 立即通过, activeRequests=3 ✅ (满了!)

任务D 到来: acquireSlot()
  activeRequests=3 不 < 3 → 返回 PendingPromise, queue=[D] ⏳

任务E 到来: acquireSlot()
  返回 PendingPromise, queue=[D, E] ⏳⏳

... 任务A 完成 ...
releaseSlot():
  activeRequests=2, queue 非空 → 取出 D, activeRequests=3, D 被唤醒! 🎉

... 任务B 完成 ...
releaseSlot():
  activeRequests=2, queue 非空 → 取出 E, activeRequests=3, E 被唤醒! 🎉
```

### 为什么不直接用 async/await + for 循环？

```javascript
// ❌ 天真的写法——所有任务同时开始！
for (const line of lines) {
  await generateLine(line.id);     // 虽然 await 了，
}                                   // 但它们是同一个 microtask 里启动的
                                    // 实际上会并行执行（因为 generateLine 内部不阻塞）

// ✅ 正确的写法——每个任务先"取号"，完成后再"还号"
const tasks = lines.map(line => async () => {
  await _acquireSlot();             // 先拿信号量
  try {                             // ← 拿到了才能往下走
    return await generateLine(line.id);
  } finally {
    _releaseSlot();                 // 不管成败都要释放
  }
});
await Promise.all(tasks.map(fn => fn()));  // 同时启动所有包装函数
```

关键区别：`_acquireSlot()` 在没有空位时会 **return 一个未 resolved 的 Promise**，这会让当前 `async` 函数**挂起**在那里——直到有其他任务调用 `_releaseSlot()` 把它唤醒。

---

## Q3：单角色批量生成 generateAllLines() 的完整流程？

**A：**

```javascript
async function generateAllLines(charId) {
  const charStore = useCharacterStore();
  const ch = charStore.characters.find(c => c.id === charId);
  if (!ch) return { success: false, error: 'Character not found' };

  // 1. 校验克隆配置
  const cloneValidation = validateCloneConfig(ch);
  if (!cloneValidation.ok) return cloneValidation;

  // 2. 过滤出需要生成的台词
  const pending = ch.lines.filter(l => l.status !== 'done');
  if (pending.length === 0) return { success: true };  // 全部完成了

  // 3. 为每条台词创建带信号量的异步任务
  const tasks = pending.map(line => async () => {
    await _acquireSlot();              // 👈 取号
    try {
      const result = await generateLine(ch.id, line.id);
      if (!result?.success) {
        errors.push({ lineId: line.id, error: result?.error });
      }
      return result;
    } finally {
      _releaseSlot();                  // 👈 还号
    }
  });

  // 4. 同时启动所有任务（受信号量约束）
  await Promise.all(tasks.map(fn => fn()));

  // 5. 汇报结果
  if (errors.length > 0) {
    return { success: false, error: `${errors.length}/${pending.length} 条失败`, errors };
  }
  return { success: true, count: results.length };
}
```

**时序示意（5 条待生成，MAX=3）：**

```
时间轴 →

L1: [====acquire====][===generate===]----release---->
L2: [====acquire====][===generate===]----------release-->
L3: [====acquire====][===generate===]-------------release->
L4: ...............[==wait==][====acquire====][===generate==]--release-->
L5: ......................[==wait==][====acquire====][===gen==]-release->

↑ 槽位占用:  L1+L2+L3 同时运行（满载）
               L4 在排队等
                     L4 开始（L1 释放了一个槽）
                          L5 开始（L2 释放了一个槽）
```

---

## Q4：跨角色批量生成 generateAllCharacters() 有什么不同？

**A：** 核心逻辑相同，区别在于**收集任务的来源更广**：

```javascript
async function generateAllCharacters() {
  const charStore = useCharacterStore();

  // 1. 收集所有角色的所有未完成台词
  let allPending = [];
  for (const ch of charStore.characters) {
    for (const line of ch.lines) {
      if (line.status !== "done") {
        allPending.push({ charId: ch.id, lineId: line.id });
      }
    }
  }

  if (allPending.length === 0) return { success: true, count: 0 };

  // 2. 进度追踪（UI 显示进度条用）
  batchProgress.value = {
    total: allPending.length,
    done: 0,
    failed: 0,
    running: true
  };

  // 3. 同样使用信号量池
  const tasks = allPending.map(({ charId, lineId }) => async () => {
    await _acquireSlot();
    try {
      const result = await generateLine(charId, lineId);
      if (result?.success) batchProgress.value.done++;
      else batchProgress.value.failed++;
      return result;
    } catch (e) {
      batchProgress.value.failed++;
    } finally {
      _releaseSlot();
    }
  });

  await Promise.all(tasks.map(fn => fn()));
  batchProgress.value.running = false;

  // 4. 返回详细结果（包含哪些失败了、哪些成功了）
  // ...
}
```

### 进度数据怎么驱动 UI 更新？

```
batchProgress = ref({ total: 20, done: 0, failed: 0, running: true })

UI 层:
  <n-progress type="line" :percentage="done/total*100" />
  <span>{{ done }}/{{ total }} 完成</span>
  <span v-if="failed">, {{ failed }} 失败</span>

每次 generateLine 完成后:
  done++ 或 failed++ → ref 变化 → Vue 自动更新 UI
```

不需要手动操作 DOM —— Pinia ref 的响应式自动处理一切。

---

## Q5：失败重试 regenerateFailedLines() 是怎么做的？

**A：** 重试的策略非常简单——**把失败项的状态重置为 pending，然后重新跑一遍批量生成**：

```javascript
async function regenerateFailedLines(charId) {
  const charStore = useCharacterStore();
  const ch = charStore.characters.find(c => c.id === charId);
  if (!ch) return { success: false, error: "Character not found" };

  // 找出所有状态为 'error' 的台词
  const failed = ch.lines.filter(l => l.status === "error");
  if (failed.length === 0) return { success: true, count: 0 };

  // 重置状态为 'pending'（让 generateAllLines 能再次选中它们）
  failed.forEach(l => { l.status = "pending"; });

  // 复用已有的批量生成逻辑（同样的信号量控制）
  return generateAllLines(ch.id);
}
```

为什么这么简单就够用了？

1. **错误大多是暂时的** — 限流（429）、网络抖动、超时，过一会儿再试通常就好了
2. **不需要智能排序** — 失败的台词重新入队即可，FIFO 公平
3. **不会无限循环** — `_callApi` 本身已经有最多 2 次重试（指数退避），regenerate 是第二层重试

如果连续两次 regenerate 都还有失败的项，那大概率是**台词本身有问题**（如特殊字符导致 API 报错）或 **API Key 过期**，这时应该提示用户检查而非继续重试。

---

## Q6：为什么不直接用 p-limit 库？

**A：** 这是一个好问题。`p-limit` 是成熟的并发控制库，API 也更优雅：

```javascript
// p-limit 的用法（对比）
import pLimit from 'p-limit';
const limit = pLimit(3);

const inputs = Array(100).fill('task');
const result = await Promise.all(
  inputs.map(input => limit(() => fetchSomething(input)))
);
```

选择手写的原因：

| 因素 | 手写 | p-limit |
|------|------|---------|
| **依赖数** | 0 | +1 个 npm 包 |
| **代码量** | ~20 行核心逻辑 | import + 初始化 |
| **调试友好度** | 完全可控，可加日志 | 黑盒，出问题要看源码 |
| **功能覆盖** | 刚好满足需求 | 提供了很多用不到的功能（concurrency 动态调整等） |
| **包体积** | 0 | ~1KB gzipped |

对于个人项目来说，**20 行代码换 0 依赖是值得的**。如果未来需求变复杂（比如需要动态调整并发数、优先级队列），再引入 `p-limit` 也不迟。

---

## 总结

| 组件 | 作用 | 代码行数 |
|------|------|---------|
| `_acquireSlot()` | 获取执行许可（有空位立即通过，否则排队） | ~8 行 |
| `_releaseSlot()` | 释放许可并唤醒下一个等待者 | ~6 行 |
| `_pendingQueue` | 存储等待中的 Promise.resolve | 1 行 |
| `MAX_CONCURRENT` | 可调的最大并发常数 | 1 行 |
| `generateAllLines()` | 单角色批量（带信号量） | ~30 行 |
| `generateAllCharacters()` | 跨角色批量（含进度追踪） | ~40 行 |
| `regenerateFailedLines()` | 失败重试（重置状态 + 复用批量逻辑） | ~12 行 |

**核心洞察：** JavaScript 的 Promise + async/await 天然适合写并发控制——一个未 resolved 的 Promise 就是天然的"锁"。不需要 mutex、semaphore 这些传统原语，语言本身已经提供了足够的抽象能力。

下一篇深入 **Canvas 波形可视化** —— 从 ArrayBuffer 到柱状图的完整渲染流水线。
