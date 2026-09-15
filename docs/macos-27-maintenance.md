# macOS 27 自用修复版

本仓库基于 [Mac Mouse Fix](https://github.com/noah-nuebling/mac-mouse-fix)，保留原项目许可证和原作者署名。

## 分支与来源

- `master`：官方基线，当前为 `ab46c4368`。
- `codex/macos-27-gestures`：自用修复分支。
- 已通过 cherry-pick 引入 [PR #1950](https://github.com/noah-nuebling/mac-mouse-fix/pull/1950)，来源提交 `e6db0ffa9b657cbcae53ea0b25ded9ccb86484fc`，保留作者信息。
- `origin` 指向个人 Fork，`upstream` 指向官方仓库；默认推送目标为 `origin`。

## 行为与限制

macOS 27 下，“空间与调度中心”的拖动改用系统快捷键触发离散操作，不能保持原来的随鼠标连续拖动动画。水平默认阈值为 220 px，垂直为 150 px；另外包含快速甩动检测和冷却时间。

补丁还修改了 macOS 27 手势事件的 HID 数据挂载方式，以及结束速度的方向处理。它依赖系统私有接口和窗口信息判断；后续系统更新仍可能影响行为。PR 尚未合并到官方项目，代码中的平台机制说明属于补丁作者的判断，并未在本机独立验证。

## 已完成验证

环境：macOS 27.0 (26A428)，Xcode 27.0 (27A266a)。

- 官方基线 Debug 构建成功。
- 集成 PR 后 Debug 构建成功。
- 构建使用 `CODE_SIGNING_ALLOWED=NO`：产物未完成可分发签名、公证或运行验证。
- Xcode 对两个第三方依赖的 macOS 10.12 最低部署版本发出警告；未阻止构建。
- 原始补丁带有行尾空白，未为此修改社区提交。

可复现构建：

```sh
xcodebuild -project 'Mouse Fix.xcodeproj' \
  -scheme 'Fast Build' -configuration Debug \
  -derivedDataPath /private/tmp/mmf-build \
  CODE_SIGNING_ALLOWED=NO build
```

## 已修复：启用崩溃（无正式签名导致）

用 `CODE_SIGNING_ALLOWED=NO` 编译出的 App 只有链接器自动附加的 ad-hoc 签名，没有真正的 Apple 开发者签名。这足以让主程序本身启动，但**不足以让 `SMAppService` 注册特权 Helper 登录项**——点击"启用 Mac Mouse Fix"时会在 `+[HelperServices enableHelperAsUserAgent:onComplete:]` 的回调里崩溃，日志显示：

```
[ServiceManagement] Unable to validate code signature on plist for <private>. Code: -67056
Failed to register Helper with error: Error Domain=SMAppServiceErrorDomain Code=3
```

`GeneralTabController.swift` 里对这类"未预期的 SMAppService 错误"直接 `assert(false)` 崩溃整个 App，而不是优雅提示——这是 Debug 构建专属的安全网（Release 下 `assert` 是空操作），原作者显然没预料到 macOS 27 会返回这种新错误码。

修复方式：`HelperServices.m` 里其实保留着 Ventura（macOS 13）之前的旧实现——手写 `launchd.plist` + `launchctl bootstrap`，完全不经过 `SMAppService` 的签名校验。既然是纯本地自用、不追求官方合规，就强制走这条旧路径（`enableHelperAsUserAgent:` 里 `if (@available(macOS 13.0, *) && NO)`），同时去掉旧路径里两处"macOS 13+ 不应该走到这里"的自杀式断言。

顺带修了另一处连带断言崩溃：许可证试用计数功能（`SecureStorage.swift`）用 `kSecAttrSynchronizable` 做 iCloud 钥匙串同步，这同样需要正式开发者签名+配置文件，ad-hoc 签名必然失败；原代码失败即 `assert(false)`。现在改为失败仅记录日志、不崩溃（不影响鼠标手势功能，只是许可证/试用状态不会云同步）。

验证环境：macOS 27.0 (26A428)。点击"启用"后 Helper 正常通过 `launchctl` 注册、常驻运行，功能确认可用。

## 已知限制：空间与调度中心手势变成"阶梯式"而非跟手

不是新引入的 bug，是 macOS 27 的硬限制，`ModifiedDragOutputThreeFingerSwipe.m` 里的补丁注释已写明：WindowServer 会丢弃第三方合成的 dockSwipe 手势事件（`CGXSenderCanSynthesizeEvents()` 比对发送方 PID 是否等于 WindowServer 自身，签名/权限/挂载 IOHIDEvent 都绕不过去）。补丁改用累积拖动距离达到阈值后触发 `SymbolicHotKeys`（系统级快捷键）来切换空间/开合调度中心，本质是"攒够距离就整段播放系统自带动画"，无法做到旧版本那种逐帧跟随鼠标的连续动画。

阈值可调（默认水平 220px、垂直 150px，对应配置 `Other.threeFingerSwipeSHKThresholdHorizontal` / `Other.threeFingerSwipeSHKThresholdVertical`），调低可以让触发更快，但改变不了"跳一下"的本质。当前保留默认值。

## 下一阶段：手动功能验证

尚未替换 `/Applications/Mac Mouse Fix.app`，也未更改系统权限或鼠标配置。避免官方 Helper 和自编译 Helper 同时处理鼠标。

待手动验证：

- 按住侧键向左/右拖动及快速甩动，切换一个或多个桌面。（已知：阈值式触发，非跟手）
- 向上打开调度中心、向下打开应用窗口，再反向关闭。
- 桌面与启动台、普通滚动、滚动与导航、前进/后退、缩放。
- 自然方向开关、多显示器和全屏空间。
- 长时间运行观察 Helper 是否稳定（冷启动时观察到一次 `SLEventTapEnable` 相关的偶发崩溃，`launchd` 的 `KeepAlive` 自动重启后运行正常，尚未复现）。

以后更新时，先获取 upstream，独立审查官方变化；每个新增功能使用独立分支。不要直接把所有社区 PR 混合应用到自用分支。
