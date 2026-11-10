# 当贝 X7 Pro 投影仪适配版 (Syncthing-Fork Dangbei Build)

本仓库 fork 自 [researchxxl/syncthing-android](https://github.com/researchxxl/syncthing-android)（Syncthing-Fork 正主仓库），针对 **当贝 X7 Pro 投影仪**（MT9681 / arm64-v8a / 当贝OS 6.0，遥控器 D-pad 操作，无触屏）做适配构建。

## 相对上游的改动

1. **`FoldersAdapter` TV 焦点修复**：文件夹列表行的 `android:descendantFocusability="blocksDescendants"` 导致行内「打开文件夹 / Override（发送覆盖）/ Revert（撤销本地变更）」按钮在遥控器 D-pad 下永远无法聚焦。TV 模式下（`UI_MODE_TYPE_TELEVISION`）恢复子项焦点（`FOCUS_BEFORE_DESCENDANTS`），并为无边框的打开文件夹按钮加上可见的焦点涟漪背景。手机/平板行为不变。
2. 上游已有的 TV 支持（LEANBACK_LAUNCHER、TV banner、Onboarding/Settings/Drawer 的 TV 焦点适配）保持原样，本 fork 在当贝 OS 上验证。

构建出的 APK 与上游同名同包名（`com.github.catfriend1.syncthingfork`），但用自己的 keystore 签名——**升级必须用同一签名的 APK**，不能直接装 F-Droid 官方签名版（除非先卸载）。

## 安装（投影仪上）

当贝 OS 开发者选项：**设置 → 关于 → 连点「版本号」7 次**，然后开启 **网络调试**（会显示 `IP:端口`）。

在局域网任一台机器上：

```bash
adb connect <投影仪IP>:5555        # 端口以屏幕显示为准
adb install -r syncthingfork_release_v2.1.7.1_arm64-v8a.apk
```

装好后可在当贝 OS「我的应用」里看到 Syncthing-Fork（带 TV banner）。

## 必做：后台保活（否则当贝 OS 会杀后台导致同步中断）

```bash
# 1. 电池优化白名单（Doze 豁免）
adb shell dumpsys deviceidle whitelist +com.github.catfriend1.syncthingfork

# 2. 所有文件访问权限（跳过首启向导里手动授予）
adb shell appops set com.github.catfriend1.syncthingfork MANAGE_EXTERNAL_STORAGE allow
```

投影仪本体设置：

- **当贝助手 / 设置 → 应用管理 → 自启动管理**：允许 Syncthing-Fork 开机自启（当贝 OS 的自启动白名单）。
- 若有「一键清理 / 内存加速」，把 Syncthing-Fork 加入保护名单。
- 应用内「设置 → 运行条件」保持默认（充电时运行——投影仪常供电，等效常开）。

## 配对：免输入 Device ID（TV 打长 ID 太痛苦）

两台设备都保持**本地发现（Local Discovery）开启**（默认开）：

1. 投影仪端打开 Syncthing-Fork，等它出现在家里其他 syncthing 设备（如 pi5）的 Web UI「设备」页的发现列表里，**在 pi5 端添加它**（pi5 端有键盘，操作舒服）。
2. 投影仪端会弹出「新设备连接请求」，遥控器 OK 确认即可。
3. 共享文件夹同理：pi5 端添加共享 → 选投影仪设备 → 投影仪端 OK 确认并选本地存储路径（SAF 文件选择器支持遥控器）。

## 遥控器操作要点

- **上/下**：列表与左侧抽屉导航项之间移动
- **左/右**：从列表行进入行内按钮（打开文件夹 / Override / Revert）
- **OK**：确认 / 打开；**返回键**：返回 / 关闭抽屉
- **菜单键**（若有）：呼出右上角菜单（添加文件夹、全部重扫描等）
- 顶栏 Tab（文件夹/设备/状态）可用上键聚焦后左右切换

## 从源码构建

需要 x86_64 Linux（NDK 工具链只有 x86_64 版）、JDK 21、Android SDK（licenses 已接受），Go 1.27.3 由 `syncthing/build-syncthing.py` 自动下载安装。

```bash
git clone --recursive https://github.com/alan-w-255/syncthing-android.git
cd syncthing-android
cat > local.properties <<EOF
sdk.dir=$HOME/android-sdk
SYNCTHING_RELEASE_STORE_FILE=/path/to/keystore.jks
SIGNING_PASSWORD=****
SYNCTHING_RELEASE_KEY_ALIAS=syncthingfork
EOF
./gradlew buildNative assembleRelease
# 产物: app/build/outputs/apk/release/*arm64-v8a*.apk  (另有 universal/armeabi-v7a/x86/x86_64)
```

> 注：`buildNative` 用 NDK clang 做 CGO 交叉编译，在 ARM 主机（如树莓派）上无法直接跑，请在 x86_64 机器或 CI 上构建。
