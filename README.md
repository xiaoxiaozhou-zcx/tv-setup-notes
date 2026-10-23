# 小米电视 L55M7-ES 精简改造 · 配置与恢复手册

> 记录时间：2026-10-23
> 用途：电视改造后若需恢复原状、或重装应用，按本文操作。

## 设备信息

| 项目 | 值 |
|---|---|
| 型号 | 小米电视 ES55 2022 款（L55M7-ES） |
| 系统 | 国行 MIUI for TV（Android 底层，MT9632 四核，2GB + 32GB） |
| 电视 IP | 192.168.5.29（ADB 端口 5555） |
| 电脑 adb | `C:\Users\32614\platform-tools\adb.exe`（v37.0.1） |
| 连接命令 | `adb connect 192.168.5.29:5555` |

## 当前已完成的改造

### 1. Aerial 干净桌面（替代原厂广告桌面）
- 包名：`app.orchard.tv`，版本 1.38（standard 版）
- 项目主页：https://github.com/pengchujin/aerial-tv
- 官方安装/还原说明：https://aerial.qust.me/agent.md
- 无障碍服务 `HomeGuardService` 已注册；按 **HOME 键**进入 Aerial

### 2. 开机直达 Aerial 桌面（2026-10-23 晚调整后）
- 当前值：`adb shell settings put system start_3rd_app app.orchard.tv`（开机直达 Aerial 桌面）
- 历史：曾设为 `com.chinablue.tv`（开机直达天光云影），后经多轮排查确认 PatchWall 的 `BootActivity` 通过 `AppUtil.openAutoApp` 读取该键执行开机直达；`settings delete` 清空后曾出现缓存残留拉起，改写成存在的包名 `app.orchard.tv` 后重启验证生效（开机 12 秒拉起 Aerial，返回桌面不再拉起天光云影）
- 天光云影：包名 `com.chinablue.tv`，v3.3.5，主 Activity = `top.yogiczy.mytv.tv.MainActivity`
- 注意：Aerial 桌面自带「开机时打开」功能（长按 App →「开机时打开」），与系统键无关，会在开机后由 Aerial 自行拉起配置的应用；要彻底不自动启动，需在 Aerial 桌面长按该 App 取消勾选
- 重新启用直达天光云影：`adb shell settings put system start_3rd_app com.chinablue.tv`

### 3. 精简停用 12 个系统包
以下全部用 `adb shell pm disable-user --user 0 <包名>` 停用（可逆）：

| 类别 | 包名 |
|---|---|
| 遥测统计 | com.miui.tv.analytics |
| 遥测统计 | com.miui.analytics |
| 遥测统计 | com.xiaomi.statistic |
| 广告推送 | com.xiaomi.mitv.tvpush.tvpushservice |
| 商城 | com.xiaomi.mitv.shop |
| 商城插件 | com.mitv.shoplugin |
| 锁屏广告位 | com.xiaomi.mibox.lockscreen |
| 屏保 | com.mitv.screensaver |
| 健康诊断 | com.xiaomi.mitv.healthdiagnosis |
| 调试 | com.xiaomi.mitv.debug |
| 媒体测试 | com.mstar.android.media.tests |
| 演示 | fusion.android.tv.demo |

注：`com.mitv.healthdiagnosis` 是错误包名（不存在），正确包名为 `com.xiaomi.mitv.healthdiagnosis`。

另有 5 个原本就处于停用状态的包（非本次操作，勿动）：
`com.xiaomi.eetv.exhibition`、`com.xiaomi.eetv.wizard`、`com.xiaomi.eetv.tv911`、`com.xiaomi.eetv.home`、`com.xiaomi.mitv.remotecontroller.service`

## 恢复方案

### 恢复原厂桌面 / 取消开机直达
当前 `start_3rd_app = app.orchard.tv`（开机直达 Aerial）。想完全取消开机直达任何应用：
```
adb shell settings delete system start_3rd_app
```
删除后开机不再直达任何应用，恢复为进原桌面（Aerial 守护接管）。

### 恢复被停用的应用（单个）
```
adb shell pm enable <包名>
```
例：恢复广告推送 `adb shell pm enable com.xiaomi.mitv.tvpush.tvpushservice`

### 卸载天光云影之后想恢复
1. 卸载本身无风险：开机时系统找不到该包会静默跳过，正常进桌面。
2. 卸载会同时删除其配置数据（直播源、设置）。
3. 恢复方法：
   - 若已清 `start_3rd_app`：重装同包名 APK（com.chinablue.tv）后，重新执行
     `adb shell settings put system start_3rd_app com.chinablue.tv`
   - 若未清：重装同包名 APK 即自动恢复开机直达。
4. 若换成**其他直播 App**（包名不同）：需把 `start_3rd_app` 改成新包名，否则开机直达失效。

### 严禁触碰（停用会出问题）
- `com.mitv.tvhome`（原桌面 PatchWall，Aerial 守护依赖）
- `com.mediatek.*`（芯片驱动）
- `com.xiaomi.mitv.settings` / `com.xiaomi.mitv.systemui`（设置 / 系统界面）
- 投屏三件套：`com.duokan.airkan.tvbox`、`com.xiaomi.miplay`、`com.xiaomi.mi_connect_service`
- `com.xiaomi.voicecontrol`（小爱语音）
- `com.sohu.inputmethod.sogou.tv`（搜狗输入法）

## 常用验证命令
```
adb shell settings get system start_3rd_app      # 查看开机直达值
adb shell pm list packages -d                     # 列出已停用应用
adb shell dumpsys window | grep mCurrentFocus     # 查看当前前台应用
adb shell ps -A | grep -E 'orchard|chinablue'     # 查看关键进程
```

## 已知限制（避免重复踩坑）
- 小米电视固件**不认**手机 MIUI 的自启机制：`appops AUTO_START` / `APP_AUTO_START` / `miui.permission.AUTO_START` 全部报 Unknown operation，不要走这条路。
- 天光云影 App 内部虽有"开机自启"开关（键名 appBootLaunch），但本机无设置入口，无法从 App 内开启。
- Aerial 官网 APK 下载站（aerial.qust.me/download/...）可能长时间 0 字节，改用 GitHub releases 镜像。
- `adb reboot` 在某些环境下会被安全机制拦截，重启可改用遥控器（电源键 → 重启）。
