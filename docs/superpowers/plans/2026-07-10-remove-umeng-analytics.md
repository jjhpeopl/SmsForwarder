# 移除友盟统计与自动更新检查实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 从 APK 中物理移除友盟统计，并删除应用启动时的自动更新和远程提示请求，同时保留手动更新及用户主动配置的联网功能。

**Architecture:** 在构建层删除友盟依赖与 AppKey，在应用层删除初始化和页面统计调用；在主界面删除整个自动联网触发链，并同步移除对应偏好项和界面开关。手动更新仍由“关于”页面按钮直接调用 `XUpdateInit.checkUpdate`，其他用户主动触发的联网能力不变。

**Tech Stack:** Android、Kotlin、Groovy Gradle、XML 资源、R8/ProGuard、Gradle Wrapper

## Global Constraints

- 应用启动时不得初始化统计 SDK，也不得自动访问友盟、更新或远程提示服务。
- APK 中不得包含 `com.umeng`、`umsdk` 或友盟 AppKey。
- 保留“关于”页面的手动更新按钮及预览版本选择能力。
- 不修改用户配置的转发渠道、FRPC 或其他主动触发的联网功能。
- 不新增替代统计 SDK 或统一网络拦截层。

---

### Task 1: 物理移除友盟依赖和运行时代码

**Files:**
- Modify: `app/build.gradle:105-174, 314-317`
- Modify: `app/proguard-rules.pro:255-263`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/App.kt:51, 340-341`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/core/BaseFragment.kt:14, 132-141`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/core/BaseContainerFragment.kt:7, 77-86`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/core/BaseSimpleListFragment.kt:7, 47-56`
- Delete: `app/src/main/kotlin/cn/ppps/forwarder/utils/sdkinit/UMengInit.kt`

**Interfaces:**
- Consumes: Android 应用启动流程和现有 Fragment 生命周期。
- Produces: 不含友盟依赖、初始化或页面埋点的可编译应用代码。

- [ ] **Step 1: 运行移除前的静态检查，确认检查能够发现现有友盟代码**

Run:

```bash
rg -n -i 'com\.umeng|umsdk|UMConfigure|MobclickAgent|UMengInit|APP_ID_UMENG|60254fc7425ec25f10f4293e' app/build.gradle app/proguard-rules.pro app/src/main/kotlin
```

Expected: 命令输出 `app/build.gradle`、`UMengInit.kt`、`App.kt` 和三个 Fragment 基类中的匹配项，作为删除前的失败基线。

- [ ] **Step 2: 建立修改前的编译基线**

Run:

```bash
bash ./gradlew :app:compileDebugKotlin
```

Expected: `BUILD SUCCESSFUL`。如果失败，先记录并处理基线问题，不把既有失败归因于本次隐私改动。

- [ ] **Step 3: 删除 Gradle 中的友盟 AppKey 和依赖**

将 `release` 和 `debug` 中的签名选择分别收敛为以下结构，不再读取 `APP_ID_UMENG`：

```groovy
if (isNeedPackage.toBoolean()) {
    signingConfig signingConfigs.release
} else {
    signingConfig signingConfigs.debug
}
```

从 `dependencies` 中删除以下整个区块：

```groovy
//友盟统计
implementation 'com.umeng.umsdk:common:9.8.9'
implementation 'com.umeng.umsdk:asms:1.8.7.2'
implementation 'com.umeng.umsdk:uyumao:1.1.4'
```

- [ ] **Step 4: 删除友盟初始化和页面访问统计**

从 `App.kt` 删除：

```kotlin
import cn.ppps.forwarder.utils.sdkinit.UMengInit
```

以及：

```kotlin
// 运营统计数据
UMengInit.init(this)
```

删除 `UMengInit.kt`。从三个 Fragment 基类中删除：

```kotlin
import com.umeng.analytics.MobclickAgent
```

以及各自完整的以下生命周期覆写：

```kotlin
override fun onResume() {
    super.onResume()
    MobclickAgent.onPageStart(pageName)
}

override fun onPause() {
    super.onPause()
    MobclickAgent.onPageEnd(pageName)
}
```

- [ ] **Step 5: 删除友盟专用混淆规则**

从 `app/proguard-rules.pro` 删除以下完整区块：

```proguard
# umeng统计
-keep class com.umeng.** {*;}
-keepclassmembers class * {
   public <init> (org.json.JSONObject);
}
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

- [ ] **Step 6: 验证友盟代码引用消失并编译 Kotlin**

Run:

```bash
rg -n -i 'com\.umeng|umsdk|UMConfigure|MobclickAgent|UMengInit|APP_ID_UMENG|60254fc7425ec25f10f4293e' app/build.gradle app/proguard-rules.pro app/src/main/kotlin
```

Expected: 无输出，退出码为 `1`。

Run:

```bash
bash ./gradlew :app:compileDebugKotlin
```

Expected: `BUILD SUCCESSFUL`。

- [ ] **Step 7: 提交友盟运行时代码删除**

```bash
git add app/build.gradle app/proguard-rules.pro app/src/main/kotlin/cn/ppps/forwarder/App.kt app/src/main/kotlin/cn/ppps/forwarder/core/BaseFragment.kt app/src/main/kotlin/cn/ppps/forwarder/core/BaseContainerFragment.kt app/src/main/kotlin/cn/ppps/forwarder/core/BaseSimpleListFragment.kt app/src/main/kotlin/cn/ppps/forwarder/utils/sdkinit/UMengInit.kt
git commit -m "privacy: remove umeng analytics"
```

### Task 2: 删除应用启动时的自动联网链路

**Files:**
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/activity/MainActivity.kt:48-49, 64, 182-191`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/fragment/AboutFragment.kt:68-71`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/utils/SettingUtils.kt:10-11`
- Modify: `app/src/main/kotlin/cn/ppps/forwarder/utils/Constants.kt:31`
- Modify: `app/src/main/res/layout/fragment_about.xml:79-105`
- Modify: `app/src/main/res/values/strings.xml:597`
- Modify: `app/src/main/res/values-en/strings.xml:567`
- Modify: `app/src/main/res/values-zh-rCN/strings.xml:568`
- Modify: `app/src/main/res/values-zh-rTW/strings.xml:568`

**Interfaces:**
- Consumes: `MainActivity.initData()`、`AboutFragment` 和现有 `XUpdateInit.checkUpdate` 手动更新接口。
- Produces: 启动时不自动发起更新或远程提示请求，但 `btnUpdate` 仍能触发手动更新。

- [ ] **Step 1: 运行移除前检查，确认自动联网链路和设置项存在**

Run:

```bash
rg -n 'autoCheckUpdate|AUTO_CHECK_UPDATE|scb_auto_check_update|@string/auto_check|showTips\(this\)|XUpdateInit\.checkUpdate\(this, false' app/src/main/kotlin app/src/main/res
```

Expected: 输出 `MainActivity.kt`、`AboutFragment.kt`、`SettingUtils.kt`、`Constants.kt`、布局和四套字符串资源中的匹配项。

- [ ] **Step 2: 删除主界面的自动更新和远程提示请求**

从 `MainActivity.kt` 删除以下导入：

```kotlin
import cn.ppps.forwarder.utils.sdkinit.XUpdateInit
import cn.ppps.forwarder.widget.GuideTipsDialog.Companion.showTips
import com.xuexiang.xutil.net.NetworkUtils
```

将 `initData()` 改为只加载本地菜单资源：

```kotlin
private fun initData() {
    mMenuTitles = ResUtils.getStringArray(this, R.array.menu_titles)
    mMenuIcons = ResUtils.getDrawableArray(this, R.array.menu_icons)
}
```

此步骤同时删除启动时的更新请求和 `showTips(this)` 远程 GET 请求。设置页中的 `GuideTipsDialog.showTipsForce` 保留，因为它由用户主动触发。

- [ ] **Step 3: 删除自动更新偏好项和设置界面**

从 `AboutFragment.initViews()` 删除：

```kotlin
binding!!.scbAutoCheckUpdate.isChecked = SettingUtils.autoCheckUpdate
binding!!.scbAutoCheckUpdate.setOnCheckedChangeListener { _, isChecked ->
    SettingUtils.autoCheckUpdate = isChecked
}
```

从 `SettingUtils.kt` 删除：

```kotlin
//是否启动时检查更新
var autoCheckUpdate: Boolean by SharedPreference(AUTO_CHECK_UPDATE, true)
```

从 `Constants.kt` 删除：

```kotlin
const val AUTO_CHECK_UPDATE = "auto_check_update"
```

从 `fragment_about.xml` 删除 `scb_auto_check_update` 和显示 `@string/auto_check` 的 `TextView`，并删除 `btn_update` 上不再需要的：

```xml
android:layout_marginStart="5dp"
```

从四套 `strings.xml` 分别删除以下准确条目：

```xml
<!-- app/src/main/res/values/strings.xml -->
<string name="auto_check">启动时检查</string>

<!-- app/src/main/res/values-en/strings.xml -->
<string name="auto_check">Auto check</string>

<!-- app/src/main/res/values-zh-rCN/strings.xml -->
<string name="auto_check">启动时检查</string>

<!-- app/src/main/res/values-zh-rTW/strings.xml -->
<string name="auto_check">啟動時檢查</string>
```

- [ ] **Step 4: 验证自动联网入口消失，手动更新入口仍存在**

Run:

```bash
rg -n 'autoCheckUpdate|AUTO_CHECK_UPDATE|scb_auto_check_update|@string/auto_check|showTips\(this\)|XUpdateInit\.checkUpdate\(this, false' app/src/main/kotlin app/src/main/res
```

Expected: 无输出，退出码为 `1`。

Run:

```bash
rg -n 'btnUpdate\.setOnClickListener|XUpdateInit\.checkUpdate\(requireContext\(\), true, SettingUtils\.joinPreviewProgram\)' app/src/main/kotlin/cn/ppps/forwarder/fragment/AboutFragment.kt
```

Expected: 输出手动更新按钮监听器及 `needErrorTip = true` 的更新调用。

Run:

```bash
bash ./gradlew :app:compileDebugKotlin
```

Expected: `BUILD SUCCESSFUL`，证明 ViewBinding 不再引用已删除控件。

- [ ] **Step 5: 提交自动联网链路删除**

```bash
git add app/src/main/kotlin/cn/ppps/forwarder/activity/MainActivity.kt app/src/main/kotlin/cn/ppps/forwarder/fragment/AboutFragment.kt app/src/main/kotlin/cn/ppps/forwarder/utils/SettingUtils.kt app/src/main/kotlin/cn/ppps/forwarder/utils/Constants.kt app/src/main/res/layout/fragment_about.xml app/src/main/res/values/strings.xml app/src/main/res/values-en/strings.xml app/src/main/res/values-zh-rCN/strings.xml app/src/main/res/values-zh-rTW/strings.xml
git commit -m "privacy: disable automatic update requests"
```

### Task 3: 更新公开和内置隐私说明

**Files:**
- Modify: `README.md:39`
- Modify: `README_en.md:39`
- Modify: `PRIVACY:1-7`
- Modify: `app/src/main/assets/protocol/privacy_protocol.txt:1-7, 71-78`

**Interfaces:**
- Consumes: Task 1 和 Task 2 的实际联网行为。
- Produces: 与本分支行为一致的中英文隐私说明。

- [ ] **Step 1: 确认旧文案仍声明启动时向友盟发送信息**

Run:

```bash
rg -n '启动时发送版本信息|umeng\.com for stats|友盟\+SDK需要收集|友盟统计SDK' README.md README_en.md PRIVACY app/src/main/assets/protocol/privacy_protocol.txt
```

Expected: 四个文件均有匹配项。

- [ ] **Step 2: 用精确、不过度承诺的文案替换旧隐私声明**

将 `README.md` 的隐私声明整行替换为：

```text
* 隐私声明：本分支不集成友盟统计 SDK，也不会在应用启动时自动访问友盟、更新或提示服务。只有在用户主动配置转发渠道、手动检查更新或主动使用其他联网功能时，应用才会向对应服务发起请求。
```

将 `README_en.md` 的隐私声明整行替换为：

```text
* Privacy: This branch does not include the UMeng analytics SDK and does not contact analytics, update, or tips services automatically at startup. Network requests occur only when the user configures a forwarding channel, manually checks for updates, or explicitly uses another network-dependent feature.
```

将 `PRIVACY` 的全部内容替换为：

```text
SmsForwarder 本分支不集成友盟统计 SDK，也不会在应用启动时自动访问友盟、更新或提示服务。

只有在用户主动配置转发渠道、手动检查更新或主动使用其他联网功能时，应用才会向对应服务发起请求。
```

将 `privacy_protocol.txt` 第 1 至 7 行替换为同样两段中文。保留通用协议第 1 至 6 节，并将原第 7、8 节替换为：

```text
7. 统计与遥测说明

本版本未接入友盟统计SDK，不会为友盟运营统计目的收集或上报设备信息。
```

- [ ] **Step 3: 验证文案不再描述友盟收集行为**

Run:

```bash
rg -n '启动时发送版本信息|umeng\.com for stats|友盟\+SDK需要收集|友盟统计SDK\(com\.umeng\)' README.md README_en.md PRIVACY app/src/main/assets/protocol/privacy_protocol.txt
```

Expected: 无输出，退出码为 `1`。

Run:

```bash
rg -n '不集成友盟统计 SDK|does not include the UMeng analytics SDK|未接入友盟统计SDK' README.md README_en.md PRIVACY app/src/main/assets/protocol/privacy_protocol.txt
```

Expected: 四个文件均包含新的行为说明。

- [ ] **Step 4: 提交隐私文案更新**

```bash
git add README.md README_en.md PRIVACY app/src/main/assets/protocol/privacy_protocol.txt
git commit -m "docs: update privacy disclosures"
```

### Task 4: 验证依赖树、最终 APK 和改动范围

**Files:**
- Verify: `app/build/outputs/apk/debug/*universal_debug.apk`
- Verify: 当前分支全部改动

**Interfaces:**
- Consumes: Task 1 至 Task 3 的所有提交。
- Produces: 可安装且不包含友盟、启动时不自动联网的调试版 APK 验证证据。

- [ ] **Step 1: 扫描运行时代码和构建配置**

Run:

```bash
rg -n -i 'com\.umeng|umsdk|UMConfigure|MobclickAgent|UMengInit|APP_ID_UMENG|60254fc7425ec25f10f4293e' app/build.gradle app/proguard-rules.pro app/src/main/kotlin
```

Expected: 无输出，退出码为 `1`。

Run:

```bash
rg -n 'autoCheckUpdate|AUTO_CHECK_UPDATE|scb_auto_check_update|@string/auto_check|showTips\(this\)|XUpdateInit\.checkUpdate\(this, false' app/src/main/kotlin app/src/main/res
```

Expected: 无输出，退出码为 `1`。

- [ ] **Step 2: 检查 Gradle 依赖树**

Run:

```bash
bash ./gradlew :app:dependencyInsight --dependency umeng --configuration debugRuntimeClasspath
```

Expected: 输出 `No dependencies matching given input were found`，且任务成功。

- [ ] **Step 3: 完整构建调试版 APK**

Run:

```bash
bash ./gradlew :app:assembleDebug
```

Expected: `BUILD SUCCESSFUL`，并在 `app/build/outputs/apk/debug/` 生成 universal 及各 ABI 的 APK。

- [ ] **Step 4: 扫描 universal APK 中的 DEX 字符串**

Run:

```bash
find app/build/outputs/apk/debug -type f -name '*universal_debug.apk' -exec unzip -p {} 'classes*.dex' \; | strings | rg -n -i 'com[/\.]umeng|umsdk|uyumao'
```

Expected: 无输出，退出码为 `1`。

- [ ] **Step 5: 最终核对手动功能、格式和工作区**

Run:

```bash
rg -n 'XUpdateInit\.checkUpdate\(requireContext\(\), true, SettingUtils\.joinPreviewProgram\)' app/src/main/kotlin/cn/ppps/forwarder/fragment/AboutFragment.kt
```

Expected: 输出唯一的手动更新调用。

Run:

```bash
rg -n 'GuideTipsDialog\.showTipsForce\(requireContext\(\)\)' app/src/main/kotlin/cn/ppps/forwarder/fragment/SettingsFragment.kt
```

Expected: 输出用户主动触发的远程提示入口。

Run:

```bash
git diff --check main...HEAD
```

Expected: 无输出，退出码为 `0`。

Run:

```bash
git status --short --branch
```

Expected: 位于 `codex/remove-umeng-analytics`，没有未提交文件。
