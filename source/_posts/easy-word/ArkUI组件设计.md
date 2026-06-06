---
title: ArkUI 自定义组件设计：翻转卡片与热力图的实现
date: 2026-06-05 13:00:00
tags:
  - easy-word
  - 鸿蒙
  - ArkUI
  - 组件设计
categories:
  - 鸿蒙开发
top_img: false
---

<!-- more -->

## 引言

easy-word 是一个鸿蒙 NEXT 原生的单词学习应用，UI 层全部用 ArkUI 声明式语法实现。项目拆出了 15 个以上的自定义 `@Component`，每个组件职责单一。

这篇文章聚焦三个核心组件的设计：翻转闪卡、GitHub 风格热力图、浮动宠物。它们分别展示了 ArkUI 中状态管理、数据驱动渲染、手势动画的典型用法。

## 翻转闪卡：三层状态管理

### 组件结构

`WordFlashCard` 是学习页面的核心交互组件，用户点击卡片翻转查看中文释义。文件位于 `entry/src/main/ets/components/WordFlashCard.ets`。

```typescript
@Component
export struct WordFlashCard {
  @Prop word: Word;                           // 外部传入，只读
  @State private flipped: boolean = false;    // 内部翻转状态
  @Prop speakHandler: SpeakHandler | null = null;
  @StorageLink('appThemeMode') themeMode: string = 'light';
```

这里用了三种状态修饰符：

| 修饰符 | 用途 | 数据来源 |
|--------|------|---------|
| `@Prop` | 只读数据，父组件传入 | 外部 |
| `@State` | 组件内部状态 | 内部 |
| `@StorageLink` | 全局共享状态 | AppStorage |

`@StorageLink('appThemeMode')` 让组件自动响应全局主题切换——当用户在设置页切换深色模式时，所有绑定了 `appThemeMode` 的组件会自动重新渲染。

### 条件渲染

翻转逻辑没有用 3D 旋转动画，而是通过 `if/else` 切换正面和背面内容：

```typescript
if (!this.flipped) {
  Column({ space: 10 }) {
    Text(this.word.word).fontSize(36).fontWeight(FontWeight.Bold)
    // 音标、发音按钮
  }
} else {
  Column({ space: 10 }) {
    Text(this.word.translation).fontSize(22)
    Scroll() {
      Column({ space: 10 }) {
        SentenceExampleBlock({ sentence: this.word.example, ... })
      }
    }
  }
}
```

点击触发翻转：

```typescript
.onClick(() => {
  this.flipped = !this.flipped;
})
```

卡片尺寸固定 300vp 高度、16vp 圆角、带阴影。这种"切换内容"的方式比 3D 翻转更可靠——鸿蒙 NEXT 的 3D 变换在部分设备上还有兼容性问题，用 `if/else` 切换是最稳妥的方案。

### 子组件嵌套

卡片背面的例句展示用了一个独立子组件 `SentenceExampleBlock`，负责高亮显示句子中的目标单词：

```typescript
SentenceExampleBlock({
  sentence: this.word.example,
  targetWord: this.word.word,
  speakHandler: this.speakHandler
})
```

组件拆分的原则是：每个子组件只负责一件事。`WordFlashCard` 只管翻转和布局，`SentenceExampleBlock` 只管例句渲染和高亮。

## 热力图：纯 ForEach 网格

### 为什么不用 Canvas

GitHub 风格的热力图通常用 Canvas 绘制，但在 ArkUI 中 Canvas 需要手动管理绘制上下文和坐标计算。easy-word 选择用纯 `Column`/`Row` + `ForEach` 实现——30 天 × 3 行的网格，总共 90 个格子，用声明式布局更直观。

组件位于 `entry/src/main/ets/components/HeatmapChart.ets`。

### 数据结构

```typescript
const TOTAL_DAYS = 30;
const HEATMAP_ROWS = 3;
const CELL_SIZE = 20;
const CELL_GAP = 6;

const LIGHT_LEVELS = ['#EBEDF0', '#C6E48B', '#7BC96F', '#239A3B', '#196127'];
const DARK_LEVELS = ['#2B3645', '#284C3A', '#2F6B4C', '#419F67', '#6FD08C'];
```

5 级颜色对应 5 个活跃度等级，深色模式有独立的色板。颜色值直接参考了 GitHub 的贡献图配色。

### @Watch 响应数据变化

```typescript
@Component
export struct HeatmapChart {
  @StorageLink('appThemeMode') themeMode: string = 'system';
  @Prop @Watch('onRecordsChange') records: CheckIn[] = [];
  @State private weeks: WeekData[] = [];
  @State private selectedDay: DayData = this.createEmptyDay();
```

`@Watch('onRecordsChange')` 是关键——当外部传入的 `records` 变化时，自动触发 `onRecordsChange` 方法重新计算周数据：

```typescript
onRecordsChange() {
  this.weeks = this.buildWeekData(this.records);
}
```

这比手动在父组件中监听变化再传值要简洁得多。

### 颜色映射

```typescript
private levelColor(day: DayData): string {
  const levels = this.isDark() ? DARK_LEVELS : LIGHT_LEVELS;
  const threshold = Math.max(this.maxCount, 20);
  const step = Math.max(1, Math.ceil(threshold / 4));
  if (day.count <= 0)       return levels[0]; // 空白
  if (day.count <= step)    return levels[1]; // 浅绿
  if (day.count <= step*2)  return levels[2];
  if (day.count <= step*3)  return levels[3];
  return levels[4];                           // 深绿
}
```

`threshold` 取 `maxCount` 和 20 的较大值，确保即使某天打卡次数很少，颜色梯度也能正常展开。

### 双层 ForEach 渲染网格

```typescript
Row({ space: CELL_GAP }) {
  ForEach(this.weeks, (week: WeekData) => {
    Column({ space: CELL_GAP }) {
      ForEach(week.days, (day: DayData) => {
        Column()
          .width(CELL_SIZE).height(CELL_SIZE)
          .backgroundColor(this.levelColor(day))
          .borderRadius(GRID_RADIUS)
          .border({
            width: this.selectedDay.date === day.date ? 1 : 0,
            color: this.cellBorderColor(day)
          })
          .onClick(() => { this.handleSelect(day); })
      })
    }
  })
}
```

外层 `ForEach` 遍历周，内层 `ForEach` 遍历天。选中的日期用 1px 边框高亮。点击格子后 `selectedDay` 更新，底部摘要区域显示当天的学习数据。

整个组件没有用 Canvas，也没有引入第三方图表库，90 个格子的渲染性能完全没问题。

## 浮动宠物：手势组合与动画

### 点击弹跳

浮动宠物是一个始终悬浮在页面上的小图标，点击时有弹跳反馈：

```typescript
.onClick(() => {
  animateTo({ duration: 150, curve: Curve.EaseOut }, () => {
    this.clickScale = 1.2;
  });
  setTimeout(() => {
    animateTo({ duration: 250, curve: Curve.EaseIn }, () => {
      this.clickScale = 1.0;
    });
  }, 150);
})
```

两段动画通过 `setTimeout` 串联：先放大到 1.2，150ms 后缩回 1.0。`animateTo` 是 ArkUI 的内置动画 API，会自动对目标属性做插值。

### 长按拖拽

拖拽用手势链实现——先长按 600ms 触发拖拽模式，再用 `PanGesture` 跟踪手指移动：

```typescript
.gesture(
  GestureGroup(GestureMode.Sequence,
    LongPressGesture({ duration: 600, repeat: false })
      .onAction(() => {
        this.dragging = true;
        animateTo({ duration: 150, curve: Curve.EaseOut }, () => {
          this.dragScale = 1.2;
        });
      }),
    PanGesture({ direction: PanDirection.All, distance: 1 })
      .onActionUpdate((event: GestureEvent) => {
        this.posX = this.startX + (event.offsetX / 360) * 100;
        this.posY = this.startY + (event.offsetY / 780) * 100;
      })
      .onActionEnd(() => {
        this.dragging = false;
        animateTo({ duration: 200, curve: Curve.EaseOut }, () => {
          this.dragScale = 1.0;
        });
      })
  )
)
```

`GestureGroup(GestureMode.Sequence, ...)` 表示手势按顺序触发——必须先完成长按，才能开始拖拽。这避免了普通点击被误判为拖拽。

位置用百分比计算（`posX`、`posY` 是 0-100 的值），这样在不同屏幕尺寸下宠物都能正确定位：

```typescript
.position({ x: `${this.posX}%`, y: `${this.posY}%` })
.scale({ x: this.clickScale * this.dragScale, y: this.clickScale * this.dragScale })
```

缩放是点击缩放和拖拽缩放的乘积——长按拖拽时宠物放大，松手回弹，点击时也有放大反馈，两个效果互不干扰。

## 主题系统

### ThemeProvider

项目用 `@StorageLink('appThemeMode')` 实现全局主题切换。`ThemeProvider` 组件封装了这个逻辑：

```typescript
@Component
export struct ThemeProvider {
  @StorageLink('appThemeMode') themeMode: string = 'light';
  @Builder child: () => void = () => {};
  build() { this.child() }
}
```

用 `@Builder` 插槽模式，子组件直接嵌套在 `ThemeProvider` 内部。所有子组件通过 `ThemeUtils.isDarkModeFor(this.themeMode)` 读取当前主题，决定使用哪套颜色。

### 组件拆分原则

easy-word 拆出了 15+ 个自定义组件，按职责划分：

| 组件 | 职责 |
|------|------|
| `WordFlashCard` | 闪卡翻转 |
| `HeatmapChart` | 学习热力图 |
| `FloatingPet` | 浮动宠物（手势+动画） |
| `BadgeCelebration` | 徽章庆祝弹层 |
| `PetEvolutionOverlay` | 宠物进化动画覆盖层 |
| `ThemeProvider` | 主题注入 |
| `SentenceExampleBlock` | 例句高亮 |
| `SpellingKeyboardCover` | 拼写键盘 |
| `SyncStatusBar` | 同步状态指示 |

每个组件只管自己的渲染和交互，不直接操作其他组件的状态。组件间通过 `@Prop` 传数据，通过回调函数传事件。

## 设计总结

这三个组件展示了 ArkUI 自定义组件的几个关键模式：

1. **状态分层**：`@StorageLink` 管全局、`@Prop` 管外部输入、`@State` 管内部状态，职责清晰
2. **数据驱动**：`@Watch` 监听数据变化自动触发重算，避免手动管理更新
3. **声明式渲染**：用 `ForEach` 和 `if/else` 替代命令式 DOM 操作
4. **手势组合**：`GestureGroup(Sequence, ...)` 组合多种手势，避免冲突

ArkUI 的声明式语法和 SwiftUI、Jetpack Compose 思路一致。如果你有这两个框架的经验，上手 ArkUI 的学习曲线会比较平缓。你觉得 ArkUI 相比 SwiftUI 或 Compose，在游戏化 UI 场景下有哪些独特的优势或不足？
