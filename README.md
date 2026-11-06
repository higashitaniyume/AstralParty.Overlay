# CesiumOverlay (WPF Overlay)

进程外 WPF 覆盖层，和 `version.dll` 并列分发，由原生加载器 `overlayctl` 拉起。

## 架构

```
游戏进程 (AstralParty_CN.exe)
 └─ version.dll (loader) ── overlayctl_start() ──> CreateProcess(CesiumOverlay.exe --pid <gamePid> --port <cesiumOverlayPort>)
                                                               │  Job Object(KILL_ON_CLOSE)
Mod (HybridCLR, ShopRelicPeek) ── UDP 127.0.0.1:CESIUM_OVERLAY_PORT ──> WPF 进程
```

* `version.dll` 只做三件事：分配 UDP 临时端口、起子进程、监管（看护线程最多重启 3 次）。
* `Mod` 只做一件事：`OverlayChannel.SendPeek()` 把 FairyGUI 风格的 BBCode 文本推过来。
* `WPF` 只做两件事：跟随 `UnityWndClass`（同 PID）定位到右上、渲染 BBCode 并提供本地折叠。

## 启动

```bash
# 加载器模式（推荐）：doorstop_config.json 里 overlayEnabled=true，启动游戏即起
CesiumOverlay.exe --pid 1234 --port 56789

# 本地调试模式：无参时开一个随机端口并写 port.txt，或直接双击 exe 跑
CesiumOverlay.exe
CesiumOverlay.exe --demo   # 直接画示例，不依赖游戏/Mod
```

## 折叠

* 点每张卡片标题栏（🛒 商店看板 / ✦ 筹码看板）可收起/展开，箭头 `▾/▸` 表示状态。
* 点工具条「收起」会把所有卡片压进一条迷你条，点击「展开」再弹回。
* `Insert` 切换点击穿透（默认开）。穿透开时鼠标直达游戏；关时可点标题栏。标题栏点击和 `Insert` 都不经过游戏输入。

## 协议

`PeekProtocol.cs`：一条 datagram = 一个 JSON，`kind=peek`，`panel=shop|relic`，字段 `text/fontSize/panelWidth/offsetX/offsetY/ts`。文本是 `ShopRelicState.BuildShopOverlayText/BuildRelicOverlayText` 原样透传的 BBCode（`[color=...][b]...[/b][/color]`），WPF 侧 `BbCodeParser` 解析。

## 现实限制

* 独占全屏下 `TopMost` 会被压住（和高刷 Steam Overlay 不一样），窗口/无边框模式正常。
* 坐标是屏幕像素，游戏移动/缩放时每 120ms 重定位一次，抖动 <1px。
