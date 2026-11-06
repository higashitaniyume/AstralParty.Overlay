# AstralParty.Overlay

WPF 进程外覆盖层（由 `CesiumLoader` 原生 `overlayctl` 拉起的独立 `exe`，已从原 `CesiumLoader/loader/CesiumLoader.Overlay` fork 出来单独维护）。

## 架构

```
游戏进程 (AstralParty_CN.exe pid=<gamePid>)
 └─ version.dll (CesiumLoader) ── overlayctl_start() ──> CreateProcess(CesiumOverlay.exe --pid <gamePid> --port <n> --cfg <overlay_config.json>)
                                                               │ Job Object(KILL_ON_CLOSE)
Mod (HybridCLR，OverlayFeedMod) ── UDP 127.0.0.1:CESIUM_OVERLAY_PORT ──> WPF 进程
Toys ── 写 overlay_config.json ── (文件) ──> WPF 进程（热重载）
游戏内拖动 ── 写 overlay_config.json ── (文件) ──> Toys 读回
```

* `version.dll` 只做三件事：分配临时端口、传 `pid/port/cfg` 给子进程、监管（最多重启 3 次）。
* `Mod` 只做一件事：`OverlayChannel.SendPeek()` 直接把文本快照推给`WPF`，不经过`version.dll`。
* `WPF` 只做两件事：跟随 `UnityWndClass`（同 PID）定位到右上、渲染`BBCode`并提供本地折叠/拖动/缩放。

## 配置

文件位置由 `CESIUM_OVERLAY_CFG` 注入，默认 `AstralParty_ModLoader/overlay/overlay_config.json`。

```json
{
  "enabled": true,
  "scale": 1.0,
  "fontSize": 14,
  "opacity": 0.85,
  "toggleKey": "Backslash",
  "toggleVk": 220,
  "offsetX": 20,
  "offsetY": 70,
  "panelWidth": 340
}
```

`Toys -> 覆盖层` 页直接改同一文件，也可在游戏中按反斜杠关掉点击穿透后拖动工具条，松手写回。

## 启动

```powershell
# 加载器模式：doorstop_config.json 里 overlayEnabled=true，启动游戏即起
CesiumOverlay.exe --pid 1234 --port 56789 --cfg overlay_config.json

# 本地调试
CesiumOverlay.exe --demo
```

## 交互

* `反斜杠(\)` 切换点击穿透（默认开），穿透开时鼠标直达游戏，关时可点标题栏折叠/拖动
* 点标题可单卡折叠，点工具条「收起」压成迷你条
* 在穿透关闭状态下拖工具条改 `offsetX/Y`，松手 persist

## 协议

`PeekProtocol.cs`：一条 datagram = 一个 JSON，`kind=peek`，`panel=shop|relic`。文本是`Mod`的`Tracker.BuildXXXOverlayText`原样透传的`BBCode`，`WPF`侧`BbCodeParser.cs`解析。

## 现实限制

* 独占全屏下被压住，窗口/无边框模式正常。
* 跟随 120ms 轮询 `EnumWindows` 找 `UnityWndClass`，抖动 <1px。

## 分发

自 `overlay-*` 标签起，CI 会发布`CesiumOverlay.exe + dll + overlay_config.json`的`zip`，`CesiumLoader` 的打包脚本从这里取产物，不再本地编译本仓库。

## 授权

AGPL-3.0（见上游 `CesiumLoader`）。
