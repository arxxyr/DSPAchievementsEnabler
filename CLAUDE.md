# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**AchievementsEnabler** 是戴森球计划（Dyson Sphere Program）的 BepInEx 插件，通过 Harmony 运行时补丁禁用游戏的"异常检测"（Abnormality / integrity check），使 mod 玩家在本地仍能解锁成就。默认仍然阻止把成就上报给 Steam / RAIL 平台和上传到 Milky Way（官方数据服务器），可通过配置项 `EnablePlatformAchievements` 打开真实平台成就。

- 目标框架：`netstandard2.0`
- 运行环境：DSP（Unity / IL2CPP 之前的 Mono 运行时）+ BepInEx 5 (`xiaoye97-BepInEx 5.4.17`)
- 发布渠道：GitHub Releases + Thunderstore（`PhantomGamers/AchievementsEnabler`）

## 构建与本地调试

```bash
dotnet build -c Release
```

- `AchievementsEnabler.csproj` 中定义了 `<GameDir>C:\Program Files (x86)\Steam\steamapps\common\Dyson Sphere Program</GameDir>`；**当该目录存在时**，`OutDir` 会自动指向 `<GameDir>\BepInEx\plugins\PhantomGamers-AchievementsEnabler\`，构建产物直接部署到游戏目录，启动游戏即可验证。
- 若本机 DSP 装在其它路径，请在本地修改 `<GameDir>`（不要提交该修改），或 `dotnet build -c Release -o <自定义输出目录>`。
- NuGet 源：`NuGet.Config` 额外添加 `https://nuget.bepinex.dev/v3/index.json`，用于拉取 `BepInEx.*` 与 `DysonSphereProgram.GameLibs` 引用程序集。
- 没有单元测试工程；验证方式是"构建 → 让 BepInEx 加载 → 进游戏看成就是否解锁、看 BepInEx 日志"。

## CI / 发布流程

- `.github/workflows/workflow.yml` 手动触发（`workflow_dispatch`），输入一个 SemVer 版本号。
- 流程：校验版本号 → `dotnet build -c Release /p:Version=<version>` → 打包 zip → 同时发到 GitHub Release（draft）和 Thunderstore（`tcli publish`）。
- 发布 Thunderstore 需要仓库 Secret `TCLI_AUTH_TOKEN`。
- 工作流环境变量中 `BundleBepInEx=false`：产物里**不包含** BepInEx 运行时，用户须自行安装 BepInEx。

## 代码结构

整个插件只有一个源文件 `Plugin.cs`，两个类：

1. **`Plugin : BaseUnityPlugin`** — BepInEx 入口。`Awake()` 里读取配置 `General/EnablePlatformAchievements`，然后 `Harmony.CreateAndPatchAll` 应用全部补丁。
2. **`Patches`** — 所有 Harmony 补丁集中在此，按用途分四组：
   - `Skip()`（Prefix 返回 `false`，阻断原方法）：
     - 禁用异常检测相关：`ABN.GameAbnormalityData_0925.NotifyOnAbnormalityChecked` / `TriggerAbnormality`、`AbnormalityLogic.GameTick`
     - 禁用 Milky Way 上报：`MilkyWayWebClient.SendUploadLoginRequest` / `SendUploadRecordRequest`
     - 禁用 Steam 排行榜：`STEAMX.UploadScoreToLeaderboard`
   - `SkipPlatform()`（Prefix，根据 `EnablePlatformAchievements` 决定是否放行）：
     - 拦截 `STEAMX.UnlockAchievement`、`SteamAchievementManager.{UnlockAchievement,Update,Start}`
     - 拦截 `RAILX.UnlockAchievement`、`RailAchievementManager.{UnlockAchievement,Update,Start}`
     - 默认 `false` → 阻断，不让真实平台成就触发；用户开启配置后返回 `true` → 放行。
   - `AbnormalityRuntimeData_Import_Postfix`：读档后把 `triggerTime` / `protoId` / `evidences` 重置，擦掉旧的异常标记。
   - `AlwaysTrue` / `AlwaysFalse`：把几个 getter 和判断函数（`AchievementLogic.active`、`isSelfFormalGame`、`PropertyLogic.isSelfFormalGame`、`NothingAbnormal`、`IsAbnormalTriggerred`）强制返回"一切正常"的值。

> **游戏符号兼容性**：类名 `GameAbnormalityData_0925` 中的 `_0925` 是 DSP 版本号片段（0.9.25）。DSP 更新大版本时，该类名经常变化（历史上曾多次出现 README Changelog 里"Fixed compatibility with DSP 0.9.XX"条目）。升级兼容性时通常要改这里的类型引用并相应提升版本号。

## 修改补丁时的注意事项

- 新增补丁前务必弄清被 Hook 的方法**在哪个上下文被调用**，以及它返回 `false` 会不会让后续依赖其副作用的游戏逻辑出错。`Skip()`（直接阻断）和 `AlwaysTrue`/`AlwaysFalse`（覆盖返回值）语义不同，按需选择。
- 如果要新增一个依赖配置项的补丁，照 `EnablePlatformAchievements` 的写法：在 `Plugin.Awake()` 里 `Config.Bind<...>()` 读一次，把 `.Value` 赋给 `Patches` 上的静态属性；不要在补丁方法内直接访问 `Config`。
- 修改后本地构建（见上文自动部署到 `GameDir`），进游戏开/关对应成就场景验证；BepInEx 控制台会打出 `Plugin {GUID} is loaded!`。
