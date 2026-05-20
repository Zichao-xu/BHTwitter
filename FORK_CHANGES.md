# Fork 改动说明 (Zichao-xu)

这是 [BandarHL/BHTwitter](https://github.com/BandarHL/BHTwitter) 的 fork,目标:**为 X iOS app v10.66 + Dopamine roothide 越狱补回"屏蔽 immersive 短视频流"功能**。

上游的 `disable_immersive_player` toggle 是个**死开关**:Tweak.x:56 在 FirstRun 默认写 `true`,但整个项目**没有任何 hook 读这个 key**,UI 也没渲染 cell。这个 fork 把它补完。

## 测试环境

- iOS 越狱:**Dopamine roothide**(rootless 路径方案)
- X app:**v10.66**
- 构建:**rootless deb**(`THEOS_PACKAGE_SCHEME=rootless`)
- 实测:`dpkg -i` 后 respring,BHTwitter 设置面板正常出现新 toggle,immersive 入口被拦

## 改动清单

### 1. `Makefile` — 让 theos 用本地 SDK

```diff
-TARGET := iphone:clang:latest:14.0
+TARGET := iphone:clang:16.5:14.0
```

**原因**:macOS 上 Xcode 默认 SDK 不带 PrivateFrameworks(链接 `Preferences.framework` 失败)。改用 16.5 让 theos 走 `$THEOS/sdks/iPhoneOS16.5.sdk`([theos/sdks repo](https://github.com/theos/sdks) 里 sparse-checkout 拿)。

### 2. `BHTManager.h` / `BHTManager.m` — 补 reader

```objc
// .h
+ (BOOL)disableImmersivePlayer;

// .m
+ (BOOL)disableImmersivePlayer {
    NSUserDefaults *d = [NSUserDefaults standardUserDefaults];
    if ([d objectForKey:@"disable_immersive_player"] == nil) {
        return YES;  // 默认开(不依赖 FirstRun_4.3 — 老用户也生效)
    }
    return [d boolForKey:@"disable_immersive_player"];
}
```

### 3. `SettingsViewController.m` — 补 UI toggle

本地化字符串 `DISABLE_IMMERSIVE_PLAYER_TITLE = "禁用沉浸式播放器"` 已经在 `zh_CN/zh-Hant/en/es/ja/...` lproj 里就位,只是没渲染 cell。

补 PSSpecifier 注册到 `_specifiers` 数组:

```objc
PSSpecifier *disableImmersivePlayer = [self newSwitchCellWithTitle:
    [[BHTBundle sharedBundle] localizedStringForKey:@"DISABLE_IMMERSIVE_PLAYER_TITLE"]
    detailTitle:nil
    key:@"disable_immersive_player"
    defaultValue:true
    changeAction:nil];
// 然后 disableImmersivePlayer 加进 _specifiers 数组,位置在 videoLayerCaption 之后
```

### 4. `Tweak.x` — 3 个 hook(入口拦截 + 兜底)

#### a) `T1StatusPhotoVideoForwardView.layoutSubviews` — timeline 视频 tap 入口

```objc
%hook T1StatusPhotoVideoForwardView
- (void)layoutSubviews {
    %orig;
    if ([BHTManager disableImmersivePlayer]) {
        UIView *v = (UIView *)self;
        for (UIGestureRecognizer *gr in v.gestureRecognizers ?: @[]) {
            if ([gr isKindOfClass:[UITapGestureRecognizer class]]) {
                gr.enabled = NO;
            }
        }
    }
}
%end
```

#### b) + c) `UINavigationController.pushViewController` + `UIViewController.presentViewController` — 兜底拦 immersive VC

```objc
static BOOL bht_isImmersiveVC(NSString *cls) {
    return [cls isEqualToString:@"T1ImmersiveFullScreenViewController"]
        || [cls isEqualToString:@"T1ImmersiveViewController"];
}

%hook UINavigationController
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    if ([BHTManager disableImmersivePlayer]
        && bht_isImmersiveVC(NSStringFromClass([viewController class]))) {
        return;
    }
    %orig;
}
%end

%hook UIViewController
- (void)presentViewController:(UIViewController *)viewControllerToPresent
                       animated:(BOOL)flag
                     completion:(void (^)(void))completion {
    if ([BHTManager disableImmersivePlayer]
        && bht_isImmersiveVC(NSStringFromClass([viewControllerToPresent class]))) {
        if (completion) completion();
        return;
    }
    %orig;
}
%end
```

**精确类名过滤**(`isEqualToString:` 不是 `containsString:`)— 防止误伤其他 `*Immersive*` 命名的合法导航 VC。

## 关键 X 10.66 类名(FLEX 验证)

`brynts/DLTwitter` 老 POC 引用的 `T1ImmersiveExploreViewController` 在 X 10.66 **已改名**。当前真名:

| 旧名 (DLTwitter 2024) | X 10.66 真名 (FLEX 验证) |
|---|---|
| `T1ImmersiveExploreViewController` | `T1ImmersiveFullScreenViewController`(外层) + `T1ImmersiveViewController`(内层) |
| (paging) | `T1TwitterSwift.ImmersiveAccessibleContainerView`(plain UIView,**不是** UIScrollView) |
| (player) | `TAVPlayerView` + `TAVPlayerLayerOutputView`(Twitter 自定义,**不是** AVPlayer 子类) |
| (tap 容器) | `T1StatusPhotoVideoForwardView`(frame 跟视频媒体框完全对应) |

## 当前状态 / 已知问题

| 行为 | 状态 |
|---|---|
| Sileo 装 deb + respring | ✅ |
| 设置面板出现"禁用沉浸式播放器"开关 | ✅ |
| 开关默认开(老用户也生效) | ✅ |
| 点视频后进入 immersive | ✅ **完全被拦** |
| 进 immersive 后上下滑切下一个 | ✅(根本进不去,自然没有) |
| 声音按钮(speaker icon) | ✅ 正常 |
| inline 视频自动播 | ✅ 正常(没动 TAVPlayerView) |
| 点视频时是否有视觉反馈 | ⚠️ 仍有短暂高亮闪烁(tap 实际触发了,只是后续 push 被拦)— 要完全静默需 hook `touchesBegan/Moved/Ended`,**当前空白** |
| 偶发某些页面无返回按钮(早期 `containsString` 过激过滤导致) | ✅ 已修(改为精确类名匹配) |

## 构建步骤

Apple Silicon Mac 上 brew 装的 `fakeroot` 有 arch 兼容问题(`arm64` vs `arm64e`),`make package` 会在 stage 阶段崩。改用**手工 stage**:

```bash
export THEOS=$HOME/theos
export THEOS_PACKAGE_SCHEME=rootless

make clean && make
# 编出 .theos/obj/debug/arm64/BHTwitter.dylib (~18MB)

# 手工 stage(替代 make stage)
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
Description: BHTwitter (no-immersive fork)
Maintainer: Bandar Alruwaili
Author: Bandar Alruwaili
Section: Tweaks
Depends: mobilesubstrate, ws.hbang.common
Architecture: iphoneos-arm64
Version: 4.4-12+notap
Installed-Size: 19000
EOF

dpkg-deb --root-owner-group -Zgzip --build .theos/_ packages/BHTwitter-rootless.deb
```

产物:`packages/BHTwitter-rootless.deb` (~8.55 MB)。

## 致谢

- 上游:[BandarHL/BHTwitter](https://github.com/BandarHL/BHTwitter)
- 参考:[brynts/DLTwitter](https://github.com/brynts/DLTwitter) — 提供 immersive hook 旧名时代的思路与代码结构
- 工具链调试:FLEX 在 X 内查 view hierarchy,确定 X 10.66 真类名

## 未做 / 留给后人

- 完全静默化 tap(需要 hook `touchesBegan/Moved/Ended` 或 hitTest 层)— 二轮盲改没收敛,留作 TODO
- 把 immersive 视频替换成系统 `AVPlayerViewController` 的方案(可从 BHTwitter download 代码复用 video URL 提取路径)— 框架在 commit history,KVC 路径覆盖率未做完
