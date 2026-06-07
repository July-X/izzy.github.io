---
title: godot-game-one 的 HUD 与游戏内 UI 实现
date: 2026-06-06 12:00:00
tags:
  - godot-game-one
  - Godot 4
  - 游戏UI
  - HUD
categories:
  - Godot 游戏开发
top_img: false
---

<!-- more -->

## 引言

HUD（Heads-Up Display）是玩家在战斗中唯一能看到的 UI 层。godot-game-one 的 HUD 需要同时展示血量、分数、技能冷却、Boss 状态、排行榜等信息，还要处理多人模式下的玩家标识。

这篇文章梳理 HUD 的布局设计、Boss 血条的双层实现、分数动画、以及多人玩家标识。

## HUD 布局

HUD 的场景文件在 `scenes/ui/hud.tscn`，核心节点结构：

```
HUD (CanvasLayer)
├── ScoreLabel          ← 左上 (16,12)     "总分: 0"
├── LevelLabel          ← 左上 (16,42)     "等级 1"
├── PlayerScoresLabel   ← 左上 (16,60)     "P1: 0 | P2: 0"
├── HealthBar           ← 左上 (16,84→220,110)  ProgressBar
│   └── HPNum           ← 居中 "200/200"
├── PowerupDisplay      ← 左上 (16,156)    VBoxContainer（属性条）
├── ControlsLabel       ← 右上锚定         操作提示
├── FpsLabel            ← 右上锚定         "FPS: 60"
├── SkillBar            ← 右下锚定         HBoxContainer（技能槽）
├── LeaderboardPanel    ← 右侧 (宽213)     半透明排行榜
└── GameOverPanel       ← 居中 (隐藏)      结算面板
```

所有 HUD 元素都在 `CanvasLayer` 上，确保不随游戏世界摄像机移动。布局上采用固定像素定位 + 锚点定位混合：左侧信息用固定像素，右侧技能栏和排行榜用锚点适配屏幕宽度。

## Boss 血条：双层设计

Boss 血条有两层：Boss 实体上的内联血条 + HUD 层的全局状态条。

### 内联血条

Boss 场景（`scenes/entities/boss.tscn`）自带一个 ProgressBar，跟随 Boss 移动：

```
[node name="HealthBar" type="ProgressBar" parent="."]
offset_left = -90.0  offset_top = -76.0  offset_right = 90.0  offset_bottom = -64.0
theme_override_styles/fill = Color(0.6, 0.1, 0.6, 1)  # 亮紫
```

初始化时设置最大值，每帧同步当前血量：

```python
# scripts/boss.gd
if _health_bar:
    _health_bar.max_value = _max_health
    _health_bar.value = _health
if _label:
    _label.text = "裂隙母舰 Lv%d" % _level
```

内联血条的优点是直观——玩家看到血条在 Boss 头上，空间关系明确。缺点是 Boss 移动时血条也跟着动，不容易精确判断血量。

### HUD 层状态条

HUD 层在屏幕顶部居中显示一个固定位置的状态条，包含百分比和阶段名：

```python
# scripts/hud.gd
func show_boss_status(current: float, maximum: float, phase_name: String) -> void:
    _ensure_boss_status_ui()
    _boss_status_root.visible = true
    var ratio: float = clampf(current / maxf(maximum, 1.0), 0.0, 1.0)
    _boss_status_fill.size.x = 520.0 * ratio
    _boss_status_label.text = "裂隙母舰  %d%%" % int(round(ratio * 100.0))
    _boss_status_phase.text = phase_name
```

状态条是动态创建的：

```python
func _ensure_boss_status_ui() -> void:
    _boss_status_root = Control.new()
    _boss_status_root.position = Vector2(
        maxf((viewport_size.x - 560.0) * 0.5, 16.0), 16.0)
    _boss_status_root.size = Vector2(560.0, 48.0)
    _boss_status_root.z_index = 45
    # 深色底条 + 红色填充 + 百分比标签 + 阶段名标签
```

Boss 有三个阶段，每个阶段有独立名称：

```python
func get_phase_name() -> String:
    match _phase:
        BossPhase.SUPPRESSION: return "压制校准"
        BossPhase.RIFT:        return "裂隙展开"
        BossPhase.OVERLOAD:    return "核心过载"
```

双层血条的设计意图是：内联血条提供空间感（Boss 在哪里），HUD 层状态条提供精确信息（还剩多少血、当前什么阶段）。

## 分数系统

### 基础分数

分数通过信号驱动更新：

```python
# scripts/hud.gd
func _on_score_changed(new_score: int) -> void:
    _update_score(new_score)

func _update_score(score: int) -> void:
    _score_label.text = "总分: %d" % score
    _cached_score = score
```

HUD 每帧检查分数变化，避免不必要的重绘：

```python
func _refresh_score_if_needed() -> void:
    var current_score: int = GameState.score
    if current_score != _cached_score:
        _update_score(current_score)
```

### 多人分数

多人模式下，分数按存活人数均分。HUD 用竖线分隔显示每个玩家的分数：

```python
func _update_player_scores(scores: Dictionary, total_score: int) -> void:
    var rows: Array[String] = []
    for key in keys:
        var pid: int = int(String(key))
        rows.append("%s: %d" % [_peer_label(pid), int(scores[key])])
    _player_scores_label.text = "  |  ".join(rows)
```

默认显示格式：`P1: 1200  |  P2: 800`

### 属性条动画

升级道具拾取后，属性条有 Tween 动画反馈：

```python
func _animate_powerup_row(row, bg, fill, val_lbl, prev_fill_w, fill_w, bar_h, bar_color):
    # 缩放弹跳：0.98 → 1.0
    row_tween.tween_property(row, "scale", Vector2(1, 1), 0.18)
    if fill_w > prev_fill_w:
        # 升级：白色拖尾效果
        trail_tween.tween_property(trail, "modulate:a", 0.0, 0.22)
    else:
        # 降级：边缘闪烁效果
        flash_tween.tween_property(edge_flash, "modulate:a", 0.0, 0.16)
```

升级和降级有不同的视觉反馈——升级是白色拖尾向上，降级是边缘闪烁。这种细微差异帮助玩家不看文字也能感知变化方向。

## 多人玩家标识

### 标签映射

ENet 的 peer_id 不固定，项目限制 1 个 Client，所以 UI 层按角色槽位显示：

```python
func _peer_label(peer_id: int) -> String:
    if peer_id == 1:
        return "P1"
    return "P2"
```

### 玩家节点的 authority 设置

每个玩家节点的 `multiplayer_authority` 设为该玩家的 peer_id：

```python
# scripts/player.gd
var peer_id: int = 1

func _init_multiplayer() -> void:
    if name.is_valid_int():
        peer_id = name.to_int()
    set_multiplayer_authority(peer_id)
```

### 本地玩家解析

HUD 需要区分当前本地玩家和远程玩家：

```python
func _resolve_local_player() -> Node:
    for p in get_tree().get_nodes_in_group("player"):
        if not NetworkManager.is_online():
            return p
        if int(maybe_peer_id) == multiplayer.get_unique_id():
            return p
        if p.is_multiplayer_authority():
            return p
    return nil
```

单机模式下直接返回唯一的玩家节点，在线模式下通过 `multiplayer_authority` 匹配。

## 动态 UI 元素

### 中央横幅

等级提升和 Boss 出现时显示全屏横幅：

```python
func show_center_banner(text: String, hold: float = 2.0, color: Color = ...):
    # 淡入 → 保持 → 淡出 → queue_free
    tween.tween_property(banner, "modulate:a", 1.0, 0.18)
    tween.tween_interval(maxf(hold, 0.2))
    tween.tween_property(banner, "modulate:a", 0.0, 0.28)
```

### 受击红屏

玩家受伤时全屏红色闪烁：

```python
func _setup_damage_flash() -> void:
    _damage_flash = ColorRect.new()
    _damage_flash.z_index = 200  # 覆盖全屏
    # health 减少时触发: 0→0.18 红色→0 红色 (0.05s + 0.25s)
```

### 拾取 Toast

道具拾取时在世界坐标位置显示上浮淡出的文字提示，跟随道具位置消失。

### 技能冷却遮罩

技能图标上有半透明暗色遮罩，冷却完成时变为全透明：

```python
# scripts/cooldown_overlay.gd
func set_ready_progress(value: float) -> void:
    _progress = clamp(value, 0.0, 1.0)
    color = Color(0.0, 0.0, 0.0, 0.5 * (1.0 - _progress))
```

## 技能栏的响应式布局

技能栏在右下角定位，移动端和桌面端有不同的边距：

```python
const SKILL_SLOT_DESKTOP: Vector2 = Vector2(144, 144)
const SKILL_SLOT_MOBILE: Vector2 = Vector2(116, 116)
const SKILL_BAR_MOBILE_MARGIN: Vector2 = Vector2(30, 38)
const SKILL_BAR_DESKTOP_MARGIN: Vector2 = Vector2(20, 18)

func _layout_skill_bar() -> void:
    _skill_bar.anchor_left = 1.0  _skill_bar.anchor_top = 1.0
    _skill_bar.anchor_right = 1.0  _skill_bar.anchor_bottom = 1.0
    if OS.has_feature("android") or OS.has_feature("ios"):
        _skill_bar.offset_left = -bar_size.x - SKILL_BAR_MOBILE_MARGIN.x
    else:
        _skill_bar.offset_left = -bar_size.x - SKILL_BAR_DESKTOP_MARGIN.x
```

移动端技能格更小（116px vs 144px），边距更大（30px vs 20px），避免被系统手势区域遮挡。

## 排行榜

右侧排行榜实时显示分数排名和历史记录：

```python
func _refresh_leaderboard() -> void:
    label.text = "#%d  %s : %d" % [i + 1, _peer_label(int(row.peer_id)), int(row.score)]
    history.text = "历史 %s  总分:%d  %s  Lv.%d" % [
        str(entry.time), int(entry.score), players_text, int(entry.level)]
```

排行榜是半透明面板，不遮挡游戏画面，但足够清晰。

## 设计取舍

这套 HUD 方案有几个明确的取舍：

1. **固定像素定位为主**：没有用 Godot 的锚点布局系统做全适配。对于一个目标分辨率明确的弹幕游戏，固定像素定位更可控。
2. **动态创建 vs 场景预设**：Boss 状态条和受击红闪是代码动态创建的，方便控制生命周期；基础元素用场景预设，方便编辑器调整。
3. **CanvasLayer 单层**：所有 HUD 元素在同一个 CanvasLayer，没有分层。如果未来需要支持 HUD 透明度调节或分层显示，需要重构。

你在做游戏 HUD 时，更倾向于全部用场景编辑器拖拽布局，还是像这样用代码动态创建？
