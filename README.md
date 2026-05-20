# BHTwitter — no-immersive fork

> **中文**: 基于 [BandarHL/BHTwitter](https://github.com/BandarHL/BHTwitter) 的 fork,补全"禁用沉浸式短视频流"功能。上游 master 这个开关是**死代码**(写 `NSUserDefaults` 但没人读 + 设置 UI 也没渲染),这个 fork 把它接上线。
>
> **English**: Fork of [BandarHL/BHTwitter](https://github.com/BandarHL/BHTwitter) that **restores the dead "Disable Immersive Player" toggle**. Upstream master writes the `NSUserDefaults` key but no hook reads it, and the settings cell was never rendered. This fork wires it all up.

---

## 改了什么 (一眼看 / At a Glance)

| File 文件 | Lines 行数 | What changed 改动 |
|---|---|---|
| `Tweak.x` | **+30 / -3** | 3 个新 hook:tap GR 禁用 + push/present 拦截 / 3 new hooks: disable tap GR + push/present intercept |
| `BHTManager.h` + `.m` | **+6** | 加 `disableImmersivePlayer` reader,默认 YES / add reader for `disable_immersive_player`, defaults to YES |
| `SettingsViewController.m` | **+3** | 渲染本来缺的 toggle cell / render the missing toggle cell |
| `Makefile` | **+1 / -1** | TARGET 指 16.5 SDK 不用 Xcode 默认 / TARGET pinned to 16.5 SDK (avoid Xcode default missing PrivateFrameworks) |
| `FORK_CHANGES.md` | **new** | 完整代码 diff 说明 + FLEX 验证的真类名表 / full diff explanation + FLEX-verified class name table |
| `README.md` | **rewritten** | (this file) — fork 主入口 |
| `README_UPSTREAM.md` | **moved** | 原版 README 在这 / original upstream README preserved here |

---

## TL;DR

| | 中文 | English |
|---|---|---|
| 一句话 | 上游有半成品 toggle,我把它做完 | Upstream has a half-finished toggle; this fork finishes the job |
| 解决什么问题 | X 点视频进沉浸式短视频流 → 屏蔽 | Tap a video → X opens TikTok-style swipe feed → blocked |
| 不解决什么 | 视频不 hide,inline 还能看,声音按钮还能用 | Doesn't hide videos, inline still plays, sound button still works |

---

## 测试环境 / Test Environment

| | Value |
|---|---|
| 越狱 / Jailbreak | Dopamine **roothide** |
| X iOS app | **v10.66** |
| 构建 / Build scheme | **rootless** |
| 产物 / Output | `packages/BHTwitter-rootless.deb` (~8.55 MB) |
| 安装方式 / Install via | Sileo 拖入 / drag into Sileo |

---

## 安装 / Install

**中文**: 
1. 下 [release](https://github.com/Zichao-xu/BHTwitter/releases/tag/v0.1-x10.66) 里的 `BHTwitter-rootless.deb`
2. AirDrop 或 scp 到 iPhone
3. Sileo 打开 deb → Install → 自动 respring
4. 进 X → 左侧菜单 → Settings and Support → BHTwitter → "禁用沉浸式播放器" 默认开

**English**: 
1. Grab `BHTwitter-rootless.deb` from the [release](https://github.com/Zichao-xu/BHTwitter/releases/tag/v0.1-x10.66)
2. AirDrop or scp to iPhone
3. Open with Sileo → Install → respring
4. In X → side menu → Settings and Support → BHTwitter → "Disable Immersive Player" is ON by default

---

## 工作原理 / How It Works

### 上游的死开关 / The Dead Switch Upstream Left Behind

**中文**: master 里 `Tweak.x:56` 会在 FirstRun 给 `disable_immersive_player` 写 `true`,本地化字符串 `DISABLE_IMMERSIVE_PLAYER_TITLE` 也在。但:
- BHTManager 没有 reader 方法
- SettingsViewController 没注册这个 cell
- Tweak.x 没有任何 `%hook` 真的去屏蔽 immersive

**English**: Upstream `Tweak.x:56` writes `disable_immersive_player = true` on FirstRun, and the `DISABLE_IMMERSIVE_PLAYER_TITLE` localization string is bundled. But:
- BHTManager has no reader method
- SettingsViewController doesn't register the cell
- Tweak.x has no `%hook` that actually blocks immersive

### 这个 fork 补的部分 / What This Fork Adds

| 层 / Layer | 改动 / Change |
|---|---|
| **Data** | `BHTManager.disableImmersivePlayer` reader,key 没设时默认 YES(老用户也生效)/ reader defaults to YES so existing users get it |
| **UI** | Settings 里渲染开关(本地化已存在)/ render the toggle cell (localization already existed) |
| **Behavior 1: tap 拦截** | `T1StatusPhotoVideoForwardView` 上 disable UITapGR / disable UITapGR on inline video container |
| **Behavior 2: 兜底** | `UINavigationController.pushViewController` + `UIViewController.presentViewController` 精确拦 `T1ImmersiveFullScreenViewController` + `T1ImmersiveViewController` / exact-class intercept |

详细代码 diff:见 **[FORK_CHANGES.md](./FORK_CHANGES.md)** / Full code diff: see **[FORK_CHANGES.md](./FORK_CHANGES.md)**

---

## 当前效果 / Current Behavior

| 行为 / Behavior | 状态 / Status |
|---|---|
| 点视频进 immersive / Tap to enter immersive | ✅ 完全被拦 / Fully blocked |
| 上下滑切下一个视频 / Swipe to next video | ✅ 没机会发生(根本进不去)/ N/A (never enters immersive) |
| 声音按钮 / Sound button | ✅ 正常 / Works |
| inline 视频自动播 / Inline video autoplay | ✅ 正常 / Works |
| 点视频时短暂高亮反馈 / Brief tap highlight | ⚠️ 仍有 / Still present — `touchesBegan` 路径未 hook |
| BHTwitter 其他功能 / Other BHTwitter features | ✅ 全保留 / All retained |

---

## 关键类名(X 10.66,FLEX 验证)/ Key Classes (X 10.66, FLEX-verified)

| 旧名 (DLTwitter 2024 era) | 真名 / Real name in X 10.66 |
|---|---|
| `T1ImmersiveExploreViewController` | `T1ImmersiveFullScreenViewController` (外/outer) + `T1ImmersiveViewController` (内/inner) |
| paging 容器 / pager | `T1TwitterSwift.ImmersiveAccessibleContainerView` (plain UIView,**not** UIScrollView) |
| 视频播放 view / video player view | `TAVPlayerView` + `TAVPlayerLayerOutputView` (Twitter 自定义,**not** AVPlayer subclass) |
| 点击入口 view / tap entry view | `T1StatusPhotoVideoForwardView` (frame matches video media exactly) |

---

## 构建 / Build

**注意 / Note**: Apple Silicon Mac 上 brew 的 `fakeroot` 二进制有 arch 兼容问题(`arm64` vs `arm64e`),`make package` 在 stage 阶段会崩。**推荐手工 stage** / On Apple Silicon Macs, brew's `fakeroot` has an arch mismatch; **use manual stage** instead.

```bash
export THEOS=$HOME/theos
export THEOS_PACKAGE_SCHEME=rootless

make clean && make
# 产出 .theos/obj/debug/arm64/BHTwitter.dylib (~18MB)

# 手工 stage(避开 fakeroot)/ manual stage (bypass fakeroot)
rm -rf .theos/_
mkdir -p ".theos/_/var/jb/Library/MobileSubstrate/DynamicLibraries"
mkdir -p ".theos/_/var/jb/Library/Application Support/BHT"
mkdir -p .theos/_/DEBIAN
cp .theos/obj/debug/arm64/BHTwitter.dylib ".theos/_/var/jb/Library/MobileSubstrate/DynamicLibraries/"
cp BHTwitter.plist ".theos/_/var/jb/Library/MobileSubstrate/DynamicLibraries/"
cp -R "layout/Library/Application Support/BHT/BHTwitter.bundle" ".theos/_/var/jb/Library/Application Support/BHT/"

cat > ".theos/_/DEBIAN/control" << 'EOF'
Package: com.bandarhl.bhtwitter
Name: BHTwitter
Version: 4.4-12+notap
Architecture: iphoneos-arm64
Depends: mobilesubstrate, ws.hbang.common
Section: Tweaks
EOF

dpkg-deb --root-owner-group -Zgzip --build .theos/_ packages/BHTwitter-rootless.deb
```

theos SDK(含 PrivateFrameworks)从 [theos/sdks](https://github.com/theos/sdks) sparse-checkout 拿:

```bash
cd $THEOS
git clone --depth 1 --filter=blob:none --sparse https://github.com/theos/sdks.git
cd sdks && git sparse-checkout set iPhoneOS16.5.sdk
```

---

## 兼容性 / Compatibility

| | 测试过 / Tested | 大概率行 / Likely works | 没试过 / Untested |
|---|---|---|---|
| Dopamine roothide + X 10.66 | ✅ | — | — |
| Dopamine rootless (非 roothide) | — | ✅ rootless deb 通用 / generic rootless | — |
| 完整越狱(rootful, e.g. checkra1n) | — | — | 需要 rootful 构建 / requires rootful build |
| TrollStore (无越狱)/ TrollStore (no jailbreak) | — | — | 需注入 IPA / requires IPA injection |
| X 10.67+ | — | ⚠️ 类名可能漂 / class names may drift | — |

---

## 已知坑 / Known Gaps

1. **点视频仍有视觉反馈** / **Tap still produces visual feedback** — tap GR 被禁但 `touchesBegan/Moved/Ended` 路径未 hook。要完全静默需要 swizzle 这些方法 / would require swizzling `touchesBegan/Moved/Ended` to fully suppress.
2. **没替换成系统播放器** / **System player replacement not implemented** — 框架在 commit history(`bht_extractVideoURL` helper),KVC 路径覆盖率未做完 / scaffolding exists in commit history but KVC paths weren't exhaustively tested.
3. **X 10.67+ 未验证** / **X 10.67+ not validated** — Twitter 频繁重命名内部类,新版本可能漂 / Twitter renames internal classes often.

---

## 致谢 / Credits

- 上游 / Upstream: [BandarHL/BHTwitter](https://github.com/BandarHL/BHTwitter)
- 参考实现 / Reference impl: [brynts/DLTwitter](https://github.com/brynts/DLTwitter)
- 调试工具 / Debug tool: FLEX(in-app inspector,用来确认 X 10.66 真类名 / used to verify X 10.66 class names)
- 工具链 / Toolchain: [theos](https://github.com/theos/theos) + [theos/sdks](https://github.com/theos/sdks)

---

## License

继承上游 / Inherits from upstream — see original [`README_UPSTREAM.md`](./README_UPSTREAM.md).
