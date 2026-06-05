---
title: "UE5 分布式编译选型：FASTBuild 还是 Horde？"
date: 2026-06-05 22:00:00
tags:
  - Unreal Engine
  - 构建系统
  - CI/CD
  - Horde
  - FASTBuild
categories:
  - 开发日志
top_img: false
---

<!-- more -->

"UE5 多机分布式编译打包，从哪个版本开始有？应该怎么用？推荐什么配置硬件？有什么短板？"

这几个问题，来自一个正在搭 UE5 项目的朋友。当时他一个人扛着全栈——引擎编译 40 分钟、Cook 2 小时起步，每次改完代码去倒杯水，回来发现还没结束。

但回答他的过程，远比我想的复杂。

## 不是"从哪个版本开始有"这么简单

UE5 的分布式编译方案，其实有三代：

```
                  UE4.5         UE4.20        UE 5.2
                    │              │              │
XGE/IncrediBuild ───┤──────────────┤──────────────┤────→ 需要商业 License
                    │              │              │
FASTBuild ──────────┼──────────────┤──────────────┤────→ 开源免费，轻量
                    │              │              │
Horde ──────────────┼──────────────┼──────────────┤────→ Epic 原生全栈
                                    (内部使用)    (5.2 对外公开)
```

XGE 最早，但需要买 IncrediBuild License，Epic 自己都不主推了。

FASTBuild 从 UE4.20 就有了，开源免费，UE5 一直支持。配置简单——一个 XML 文件加一个网络共享目录，半小时能搭完。

Horde 是 Epic 内部跑 Fortnite 产线的方案，UE 5.2 起才对外公开。它是完整的构建编排系统，不只是编译分发，还管 Cook、Pak、Stage、Archive、测试，外加 Dashboard、Artifact 管理、Agent 池弹性伸缩。

所以"从哪个版本开始"的答案取决于你问的是哪个方案：FASTBuild 从 UE4.20，Horde 从 UE 5.2。

## FASTBuild：一天能搭完的方案

如果你的团队不到 10 个人、项目 C++ 代码量不算天文数字，FASTBuild 可能就够了。

原理很简单：UBT 生成 FASTBuild 的 `.bff` 配置文件，由 FBuild.exe 分发到各 worker 编译。Worker 机器上只需要装相同版本的 VS Build Tools，跑一个 `FBuildWorker.exe`，能访问 Broker 网络共享就行。

```xml
<!-- Engine/Saved/UnrealBuildTool/BuildConfiguration.xml -->
<Configuration>
    <BuildConfiguration>
        <bAllowFASTBuild>true</bAllowFASTBuild>
    </BuildConfiguration>
    <FASTBuild>
        <FBuildExecutablePath>D:\FASTBuild\FBuild.exe</FBuildExecutablePath>
        <FBuildBrokeragePath>\\192.168.1.100\fastbuild</FBuildBrokeragePath>
        <bEnableCaching>true</bEnableCaching>
    </FASTBuild>
</Configuration>
```

但 FASTBuild 有硬伤——只做 C++ 编译分发，Cook 和 Pak 不在它的职责范围内。而且 worker 基本只支持 Windows。

## Horde：工业化流水线

如果你的团队 10 人以上、需要自动化 CI 流水线、或者需要 Mac/Linux 多平台同时构建，Horde 是正解。

Horde 分三层：Server（ASP.NET Core + MongoDB）、Agent（每台 build 机器）、BuildGraph 脚本（XML 定义流水线）。架构长这样：

{% mermaid %}
graph TD
    CI["CI 触发 Jenkins/GitLab"] --> HS["Horde Server"]
    subgraph "Horde Server"
        HS --> DB["MongoDB 主库"]
        HS --> ART["Artifact Storage MinIO S3"]
    end
    HS --> A1["Agent Win64 #1"]
    HS --> A2["Agent Win64 #2"]
    HS --> A3["Agent Linux #1"]
    HS --> A4["Agent Mac #1"]
    A1 --> ZEN["Zen Server DDC 共享"]
    A2 --> ZEN
    A3 --> ZEN
{% endmermaid %}

BuildGraph 脚本是这套系统的灵魂。一个合格的模板大概是这样的：

```xml
<!-- Template: StandardBuild.xml -->
<BuildGraph>
    <Option Name="ProjectName" DefaultValue="MyGame"/>
    <Option Name="TargetPlatforms" DefaultValue="Win64;Linux"/>

    <!-- 一个平台一个 Agent，全部并行 -->
    <ForEach Name="Platform" Values="$(TargetPlatforms)">
        <Agent Name="Build $(Platform) Editor" Type="$(Platform)"
               Pool="editor-build">
            <Node Name="Compile $(Platform)">
                <Compile Target="UnrealEditor" Platform="$(Platform)"
                         Configuration="Development"/>
            </Node>
        </Agent>
    </ForEach>

    <!-- 编译完了再 Cook，Cook 完了再 Pak -->
    <Agent Name="Cook Win64" Type="Win64" Pool="client-cook">
        <Node Name="Cook Win64 Client"
              Requires="Compile Win64">
            <Cook Project="$(ProjectName)" Platform="Win64"
                  Cultures="en;zh-CN"/>
        </Node>
        <Node Name="Pak and Archive"
              Requires="Cook Win64 Client">
            <Pak Project="$(ProjectName)" Platform="Win64"/>
            <Archive Files="..." To="$(ArchiveDir)"/>
        </Node>
    </Agent>
</BuildGraph>
```

几个关键模式：`Requires` 定义依赖链、`ForEach` 做参数化、`Pool` 隔离不同用途的 Agent 机器。

## 什么硬件才够用？

这也是朋友问得最多的。UE5 编译是三重密集型——CPU、内存、IO 都要硬。

| 角色 | CPU | RAM | 存储 | 网络 |
|------|-----|-----|------|------|
| Build Agent | AMD 9950X 16C 起步 | 64GB 底线，128GB 舒适 | NVMe PCIe 4.0 2TB+ | 10GbE |
| Horde Server | 4C 够用 | 16GB | 256GB SSD | 10GbE |
| NAS Broker | 低配 | 32GB | NVMe RAID 4TB+ | 25GbE 推荐 |

一个常被忽视的坑：**网络**。3 台以上 agent 走 1GbE，Broker 分发 `.obj` 和 `.pch` 文件的时候就是瓶颈。10GbE 起步，中型团队直接 25GbE。

另一个坑：**RAM**。UE5 全量编译的链接阶段能吃 50-80GB 内存。如果 agent 只有 32GB，链接时 OOM 是迟早的事。

## 最被低估的问题：分布式编译不能解决什么

搭分布式编译最大的误区，是以为"多机 = 快"。但实际项目里，编译往往不是瓶颈：

```
典型 UE5 项目的构建时间拆解（一个 AAA 项目的数据）：
├── C++ 编译         15-30 分钟    ← FASTBuild/Horde 能加速
├── Shader 编译      20-45 分钟    ← 不靠多机，靠 DDC 缓存
├── Cook 资产烘焙    60-120 分钟   ← Horde 能编排，但本质是单机 IO
└── Pak/Stage/Archive 10-20 分钟   ← Horde 能分发
```

Shader 和 Cook 占了整体时间的大头，但它们很难跨机器分发。Shader 编译依赖于 GPU 和 DDC（派生数据缓存），Cook 受限于资产依赖图的复杂性。

**花 50 万买 build 机器集群，不如花 5 万优化 DDC 策略。** 具体做法：部署 Zen Server（UE 5.4+ 推荐），所有 Agent 直连同一个 Zen 实例，DDC 命中率从 40% 提到 90% 以上，Cook 时间直接腰斩。

## 决策框架

| 场景 | 推荐方案 |
|------|----------|
| 3-5 人，编译 20 分钟内 | 先优化 DDC 和 Unity Build，可能不需要分布式 |
| 5-10 人，C++ 编译瓶颈 | FASTBuild + 2-3 台 agent |
| 10-50 人，需要 CI 流水线 | Horde 全套 |
| 50+ 人，AAA 项目 | Horde + Zen Server + 云弹性扩容 |

---

**更新记录**

- 2026-06-05：初稿发布。

---

> 分布式编译是把"CPU 瓶颈"分摊到多台机器，但它不解决 IO 瓶颈、不解决 DDC 命中率、不解决流水线编排的复杂度。你投入之前，真的搞清楚瓶颈在哪里了吗？
