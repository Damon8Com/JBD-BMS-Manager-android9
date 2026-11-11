# JBD BMS Manager — Android 9（API 28）兼容版

基于上游 `main` 分支 **v1.3.1（versionCode 50）** 改造，`applicationId` 保持
`com.bms.jbdmanager` 不变，版本号改为 **1.3.1-android9**，与原版同签名可直接覆盖安装。

---

## 一、核心结论

上游把 `minSdk` 设为 31（Android 12），**真正的硬阻断只有一处**，不是一堆坑：

> `hasBluetoothPermissions()` 写死检查 `BLUETOOTH_SCAN` / `BLUETOOTH_CONNECT`。
> 这两个权限是 **Android 12 才存在**的运行时权限，在 Android 9~11 上
> `checkSelfPermission()` 恒返回 `DENIED` —— 结果是 App 能打开，但**永远卡在
> "请授予附近设备权限"，扫不到也连不上**，不是崩溃而是"废掉"。

其余部分作者原本就写得规范：`BLE 读写`（`writeCharacteristic` / `writeDescriptor`）
在 `JbdBleManager` 里已有 `SDK_INT < TIRAMISU` 的 `@Suppress("DEPRECATION")` 旧 API
分支；全屏高温告警、通知权限、悬浮窗也都有 `TIRAMISU` / `UPSIDE_DOWN_CAKE` 版本判断。
存储全部走 `cacheDir` + SAF/FileProvider，没有 MediaStore 分区存储调用，Android 9 天然安全。

---

## 二、改动清单

### 1. `app/build.gradle.kts`

```kotlin
minSdk = 28      // 原 31
versionName = "1.3.1-android9"
```

**`compileSdk` / `targetSdk` 刻意保持 36 不动。**
只降 `minSdk` 是正确做法：若连带把 `compileSdk` 降到 28，Compose BOM 2025.05.01、
core-ktx 1.18、lifecycle 2.9.1 这批库会因 `compileSdk` 过低直接编译失败。
`compileSdk 36 + minSdk 28` 是官方支持的标准组合，老系统靠 AndroidX 向后兼容层兜底。

### 2. `MainActivity.kt` —— 关键修复

新增 `bleRuntimePermissions()`，按系统版本返回**该版本真实存在**的权限：

| 系统 | 申请的权限 | 说明 |
|---|---|---|
| Android 12+ (API 31+) | `BLUETOOTH_SCAN` + `BLUETOOTH_CONNECT` | 维持原逻辑 |
| Android 9~11 (API 28–30) | `ACCESS_COARSE_LOCATION` + `ACCESS_FINE_LOCATION` | BLE 扫描靠定位授权 |

`hasBluetoothPermissions()` 同步分流：旧系统只校验 `ACCESS_FINE_LOCATION`。
权限回调 `permissionLauncher` 本来就是重新调用 `hasBluetoothPermissions()` 实时复算，
所以授权后状态能正确翻转，无需额外改动。

### 3. `MainActivity.kt` —— 安装更新防崩溃

`Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES` 的 **per-app 页面在 Android 8~9 部分 ROM
上不存在**，`startActivity` 会抛 `ActivityNotFoundException` 直接崩。
改为候选 Intent 逐级降级：`MANAGE_UNKNOWN_APP_SOURCES` → `MANAGE_UNKNOWN_APPS` →
`SECURITY_SETTINGS` → 全都打不开则直接交给系统安装器（由系统弹权限提示），
不把用户卡死在更新流程里。

### 4. `AndroidManifest.xml`

补充 Android 9~11 必需的 **install-time** 蓝牙权限（声明即生效，无需运行时申请）：

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
```

`BLUETOOTH_SCAN/CONNECT`（31）、`POST_NOTIFICATIONS`（33）、
`FOREGROUND_SERVICE_LOCATION`（34）、`USE_FULL_SCREEN_INTENT`（34）、
`POST_PROMOTED_NOTIFICATIONS`（35）**全部保留**——低版本系统会安全忽略未知权限，
删掉反而会让新系统丧失正确行为。

### 5. 已核查、**无需修改**的点（避免误改）

| 项 | 结论 |
|---|---|
| `foregroundServiceType="location"` | API 29 引入的属性，Android 9 安全忽略；`startForeground(id, notif)` 用的是两参旧签名，兼容 |
| `ContextCompat.startForegroundService` | 只处理 Android 8+ 后台启动限制，API 28 正常 |
| `canRequestPackageInstalls()` | API 26 加入，Android 9 可用 |
| `NotificationChannel` | API 26 加入，可用 |
| `PdfRenderer` / `ParcelFileDescriptor` | API 21 / 9 就有，健康报告预览不受影响 |
| `dataExtractionRules.xml` | API 31 才读，低版本自动回退 `allowBackup`，无需删 |
| `networkSecurityConfig` + 明文 | 已全站 HTTPS，Android 9 默认禁明文也不受影响 |
| 在线更新最低版本门槛 0.5.0 (versionCode 30) | 本版 versionCode 仍为 50，**高于门槛，可正常在线升级** |

---

## 三、怎么编译出 APK

### 方式 A：GitHub Actions 云端出包（推荐，免装环境）

仓库已附带 `.github/workflows/build-android9.yml`，**不依赖任何 Secret**
（调试 keystore 在 Runner 上现场生成，Fork 后开箱即用）：

1. 把本目录内容推到你自己的 Fork 的 `android9` 分支；
2. 进 `Actions` → **构建 Android 9 兼容 APK** → `Run workflow`；
3. 约 5–8 分钟后，在本次运行的 `Artifacts` 下载
   `JBD-BMS-Manager-Android9-APK.zip`，内含 `app-debug.apk` 重命名版。

工作流内置一道**防呆校验**，会自动执行：

```bash
aapt2 dump badging app-debug.apk | grep "sdkVersion:'28'"
```

若 minSdk 没真正生效，流水线直接 **失败退出**，不会给你一个"以为是 Android 9、
其实装不上"的包。同时会打印 `uses-permission` 清单供核对。

### 方式 B：本地编译

```bash
# 需 JDK 17 + Android SDK（platform;android-36, build-tools;36.0.0）
export ANDROID_HOME=/path/to/Android/Sdk
./gradlew testDebugUnitTest assembleDebug
```

产物：`app/build/outputs/apk/debug/app-debug.apk`

安装：

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

---

## 四、Android 9 实机验收清单

装完后请按这个顺序验，前两项是本次改动的核心：

1. **权限页** — 首次点扫描，弹的是**位置权限**（不是"附近设备"）。允许后
   主界面不再显示"权限未授予"。← 这一条过了，说明核心修复生效。
2. **扫描 + 连接** — 能列出 JBD 设备，连上后 1 秒刷新电压/电流/SOC，
   `设置 → 通信信息` 里能看到 BLE 服务与识别到的协议（FF00/FF01/FF02 或 FFE0/FFE1）。
3. **断线重连** — 走远再回来，按 2/5/10/30 秒退避自动重连，数据不卡在旧值
   （>5 秒标过期、>10 秒重连的逻辑与系统版本无关）。
4. **行程/GPS** — 开一段行程，通知栏出现前台服务，杀掉 App 后服务能补启动。
   *注意 Android 9 必须系统定位开关为开，否则 `requestLocationUpdates` 会失败并提示。*
5. **CSV / 备份 / 健康 PDF** — 走系统文件选择器保存与分享，无需存储权限。
6. **在线更新** — 有新版时能下载并拉起安装器；若在"未知来源"页被弹回，
   兜底逻辑不会卡死。

### 已知限制（Android 9 固有，非本补丁缺陷）

- BLE 扫描期间**系统定位必须开启**，这是 Android 10 以下的硬性要求，无法绕过。
- 锁屏全屏高温警报依赖 `USE_FULL_SCREEN_INTENT`（API 34）的专用授权页在
  Android 9 不存在；旧系统走 `showWhenLocked` + `turnScreenOn` 起
  `TemperatureEmergencyActivity`，效果可用，但**悬浮窗兜底需手动授权**。
- 画中画（PiP）在 Android 9 可用（API 26+），但部分国产 ROM 裁剪了该特性，
  不显示入口属正常。

---

## 五、回滚

改动仅限 3 个文件 + 1 个新增工作流，回滚极干净：

```bash
git checkout -- app/build.gradle.kts app/src/main/AndroidManifest.xml \
               app/src/main/java/com/bms/jbdmanager/MainActivity.kt
rm .github/workflows/build-android9.yml
```

`@Suppress` / 版本判断全部是加法式改动，没有重构原有逻辑，不会影响 Android 12+ 行为。
