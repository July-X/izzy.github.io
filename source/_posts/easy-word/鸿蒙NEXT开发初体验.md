---
title: "鸿蒙 NEXT 原生开发初体验：ArkTS + ArkUI 的实践经验"
date: 2026-06-04 12:00:00
tags:
  - 鸿蒙
  - ArkTS
  - ArkUI
  - 移动开发
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`easy-word` 是一个鸿蒙 NEXT 原生的小学英语单词学习应用。从零开始用 ArkTS + ArkUI 开发，踩了不少和 Android/iOS 完全不同的坑。本文记录开发过程中的关键经验。

## ArkTS 和 TypeScript 的差异

ArkTS 基于 TypeScript，但不是 TypeScript。几个关键差异：

**1. 不支持动态类型**：不能用 `any`、`as` 强制转换受限、运行时类型检查被限制。这是为了 AOT 编译优化。

**2. 装饰器语法不同**：`@Component`、`@State`、`@Prop`、`@StorageLink` 是 ArkUI 的声明式 UI 装饰器，和 Angular/React 的装饰器语义完全不同。

```typescript
@Component
struct WordCard {
  @State word: string = ''
  @State flipped: boolean = false
  @Prop meaning: string = ''  // 单向数据流

  build() {
    Column() {
      Text(this.flipped ? this.meaning : this.word)
        .fontSize(24)
        .onClick(() => this.flipped = !this.flipped)
    }
  }
}
```

**3. API 兼容性陷阱**：`AppStorage.GetShared()` 在 API 22 不存在，`SafeAreaType` 是全局 declare enum 而不是 `@kit.ArkUI` 导出。版本间 API 差异大，需要做运行时兼容。

{% mermaid %}
graph TD
    A["ArkTS 开发"] --> B["声明式 UI"]
    A --> C["RDB 数据库"]
    A --> D["分布式同步"]
    B --> B1["@Component struct"]
    B --> B2["@State 状态管理"]
    B --> B3["build() 布局"]
    C --> C1["relationalStore"]
    C --> C2["BaseDao 抽象"]
    D --> D1["setDistributedTables"]
    D --> D2["sync 跨设备"]
{% endmermaid %}

## 深色模式适配：一个隐藏的坑

鸿蒙的系统栏颜色需要在 `EntryAbility.onWindowStageCreate()` 中设置，但这里不能直接读取 `AppStorage`——时序问题。

```typescript
// EntryAbility.ets
onWindowStageCreate(windowStage: window.WindowStage) {
  // 错误：此时 AppStorage 可能还没初始化
  // syncSystemBarByMode('light')

  // 正确：从 ThemeUtils 读取，带默认值
  const mode = ThemeUtils.getThemeMode()
  this.syncSystemBar(mode)
}
```

`ThemeUtils` 是自建的主题管理工具，因为 API 22 的 `AppStorage.on()` 不支持监听，需要自己注册回调：

```typescript
class ThemeUtils {
  private static callbacks: Array<(mode: string) => void> = []

  static registerCallback(cb: (mode: string) => void) {
    this.callbacks.push(cb)
  }

  static setThemeMode(mode: string) {
    AppStorage.setOrCreate('appThemeMode', mode)
    this.callbacks.forEach(cb => cb(mode))
  }
}
```

## RDB 异步初始化的时序问题

鸿蒙的 `relationalStore` 是异步初始化的，但页面组件可能在数据库就绪前就开始渲染。解决方案是 **BaseDao + 懒加载**：

```typescript
abstract class BaseDao<T> {
  private store: relationalStore.RdbStore | null = null

  protected async getStore(): Promise<relationalStore.RdbStore> {
    if (this.store) return this.store
    this.store = await relationalStore.getRdbStore(getContext(), {
      name: 'easy_word.db',
      securityLevel: relationalStore.SecurityLevel.S1
    })
    return this.store
  }

  async query(sql: string): Promise<T[]> {
    const store = await this.getStore()
    const resultSet = await store.querySql(sql)
    return this.mapResultSet(resultSet)
  }
}
```

每个 DAO 调用 `getStore()` 时才初始化，第一次调用会 await，后续直接返回缓存。避免了全局初始化的时序依赖。

## 安全区域布局

鸿蒙的刘海屏/挖孔屏适配用 `expandSafeArea`：

```typescript
Column() {
  // 内容
}
.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
```

关键点：`expandSafeArea` 必须在最外层 Column 统一处理，不能分散在内层组件上。否则会出现内容重叠或间距异常。

## 组件化架构

```
entry/src/main/ets/
├── components/
│   ├── HeatmapChart.ets      # 学习热力图
│   ├── WordFlashCard.ets     # 闪卡组件
│   └── ProgressBar.ets       # 进度条
├── pages/
│   ├── HomePage.ets           # 首页
│   ├── FlashCardPage.ets     # 闪卡学习
│   ├── SpellingPage.ets      # 拼写练习
│   └── QuizPage.ets          # 选择题
├── dao/
│   ├── BaseDao.ets           # DAO 基类
│   ├── WordDao.ets           # 单词数据
│   └── RecordDao.ets         # 学习记录
└── utils/
    └── ThemeUtils.ets        # 主题管理
```

## 和 Android/iOS 开发的对比

| 维度 | Android (Kotlin) | iOS (Swift) | 鸿蒙 (ArkTS) |
|------|------------------|-------------|---------------|
| UI 框架 | Jetpack Compose | SwiftUI | ArkUI |
| 状态管理 | ViewModel + LiveData | @State + Observable | @State + @StorageLink |
| 数据库 | Room | Core Data | relationalStore |
| 异步模型 | Coroutines | async/await | async/await |
| 生态成熟度 | 高 | 高 | 中（快速追赶中） |

最大的差异不在语法，在生态。鸿蒙的第三方库少，很多功能需要自己实现。但反过来，这也逼迫你更深入理解底层机制。

> 新平台的开发体验不是"好"或"坏"，是"不同"。理解差异、适应差异，才是跨平台开发者的核心能力。