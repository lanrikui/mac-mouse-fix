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

## 下一阶段：运行验证

准备签名、备份现有应用和配置后，再切换测试。避免官方 Helper 和自编译 Helper 同时处理鼠标。尚未替换 `/Applications/Mac Mouse Fix.app`，也未更改系统权限或鼠标配置。

待手动验证：

- 按住侧键向左/右拖动及快速甩动，切换一个或多个桌面。
- 向上打开调度中心、向下打开应用窗口，再反向关闭。
- 桌面与启动台、普通滚动、滚动与导航、前进/后退、缩放。
- 自然方向开关、多显示器和全屏空间。

以后更新时，先获取 upstream，独立审查官方变化；每个新增功能使用独立分支。不要直接把所有社区 PR 混合应用到自用分支。
