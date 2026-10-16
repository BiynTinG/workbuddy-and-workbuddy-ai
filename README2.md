# WorkBuddy 国内版与国际版共用安装目录,自动更新时互相覆盖共享资源层

## 摘要

「WorkBuddy（国内版,workbuddy.cn）」与「WorkBuddy AI（国际版,workbuddy.ai）」是两个**使用不同产品 ID 的独立应用**，但默认安装到**同一个目录**。二者共用同一份 `resources\app.asar` 及整个共享资源层，而各自的自解压安装包又**只包含自己的主程序**。结果是：任何一方触发自动更新，都会把共享资源层整体覆盖成对方版本的产物，导致另一个产品出现「exe 是 A 版、内核是 B 版」的错乱状态并**静默启动失败**。

---

## 环境

| 项 | 值 |
|---|---|
| 操作系统 | Windows 11 |
| WorkBuddy（国内版） | 5.8.3，构建 `db0e9b51b7fa0b383f41f5b0b83dd442c8557918` |
| WorkBuddy AI（国际版） | 5.8.4，构建 `73952edb48da46bd0b244661efe22f4f247c4977` |
| Electron / Node | Electron 37.10.3 / Node 22.21.1 |
| 版本文件 `version` | `37.10.3-42`（两版相同） |

---

## 症状

1. 无论先安装哪一个，**启动其中一个就会提示更新，更新完成后另一个直接变成国际版**。
2. 国内版的快捷方式还在，但点开后实际上是国际版界面。
3. 反复重装无法解决，两个版本永远只剩一个可用。
4. 最终状态：国内版**完全无法启动**——进程启动后立即退出，`uptimeSec=0`，无任何报错弹窗。

---

## 关键证据

### 1. 两个产品在注册表中指向同一个安装目录

`HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\` 下同时存在两条独立记录：

```
[BFD312E9-1019-4F57-9F44-F86246833B50]
DisplayName     : WorkBuddy 5.8.3
DisplayVersion  : 5.8.3
HelpLink        : https://www.workbuddy.cn
UninstallString : "C:\workbuddy_cn\Uninstall WorkBuddy.exe" /currentuser

[C02C88CB-5DAE-4330-AEC1-572CFD048A59]
DisplayName     : WorkBuddy AI 5.8.4
DisplayVersion  : 5.8.4
HelpLink        : https://www.workbuddy.ai
UninstallString : "C:\workbuddy_cn\Uninstall WorkBuddyAI.exe" /currentuser
```

⇒ **产品 ID 不同、域名不同、版本不同，但 `InstallLocation` 完全相同。**

### 2. 该目录下两套主程序并存

```
C:\workbuddy_cn\WorkBuddy.exe     204,590,360 字节  2026-10-15 10:04:12  (v5.8.3)
C:\workbuddy_cn\WorkBuddyAI.exe   204,590,360 字节  2026-10-15 19:24:28  (v5.8.4)
```

两者字节数相同但 **SHA256 不同**，是两个独立构建（各自内嵌不同的产品配置）：

```
WorkBuddy.exe    SHA256 = 5EA5D8BA6F6A435C27C239115647886A9646F76FF5A721E82F84F0FF16AC5E15
WorkBuddyAI.exe  SHA256 = 031F3A0E4A9588CC07883CB4B458CBF3EA95271F6423FF06A3E65AEF9A4B9CC4
```

### 3. 共享资源层被国际版覆盖

```
WorkBuddy.exe                              2026-10-15 10:04:12   ← 国内版主程序
WorkBuddyAI.exe                            2026-10-15 19:24:28
resources\app.asar                         2026-10-15 19:24:28   ← 已被 5.8.4 覆盖
resources\app.asar.unpacked\cli\product.json 2026-10-15 19:24:28
resources\install-manifest.json            2026-10-15 19:24:28
version                                    2026-10-15 19:24:28
```

被覆盖后的 `resources\app.asar.unpacked\cli\product.json` 内容为国际版配置：

```json
{
  "applicationName": "workbuddy-ai",
  "isOversea": true,
  "win32ExecutableName": "WorkBuddyAI",
  "dataFolderName": ".workbuddy-ai",
  "updateUrl": "https://www.workbuddy.ai",
  "genieVersion": "5.8.4",
  "win32MutexName": "workbuddy-ai",
  "win32x64UserAppId": "{C02C88CB-5DAE-4330-AEC1-572CFD048A59}",
  "deepLinkSchemes": ["workbuddy-ai"]
}
```

⇒ 国内版启动时读到 `isOversea: true`，产品身份被彻底掉包。

### 4. 国内版因此启动即退出

国内版本机日志 `AppStartup.log` 中，CN 构建的启动记录全部 `uptimeSec=0`：

```
[AppStartup] appName=WorkBuddy build=db0e9b51... productConfigType=CLI uptimeSec=1 source=app_startup
[AppStartup] appName=WorkBuddy build=db0e9b51... productConfigType=CLI uptimeSec=0 source=app_startup
[AppStartup] appName=WorkBuddy build=db0e9b51... productConfigType=CLI uptimeSec=0 source=app_startup
[AppStartup] appName=WorkBuddy build=db0e9b51... productConfigType=CLI uptimeSec=0 source=app_startup
```

对照：使用**同版本** `app.asar` 的国际版则正常，`total_marks=69~74`、启动成功。

### 5. 安装包结构：两边都不含对方的主程序

直接解包两个安装包（它们实为 ZIP 自解压格式）后核对根目录：

| 安装包 | 根目录主程序 | 是否含对方主程序 |
|---|---|---|
| CN 5.8.3 (`WorkBuddy-win32-x64-user-5.8.3.*.exe`) | `WorkBuddy.exe` | **否** |
| AI 5.8.4 (`WorkBuddy-Setup-5.8.4.*.exe`) | `WorkBuddyAI.exe` | **否** |

⇒ 所以**谁也删不掉对方的 exe**，破坏只发生在**共享资源层**。

### 6. 静默升级路径

`resources\scripts\` 中存在升级专用脚本，配合 NSIS 静默安装：

```
resources\scripts\launch-update-progress.vbs
resources\scripts\update-progress.ps1
```

更新包缓存在：

```
%LOCALAPPDATA%\@genieworkbuddy-desktop-updater\installer.exe                    (474.7 MB)
%TEMP%\workbuddy-update-x64\WorkBuddy-Setup-5.8.4.41366630.exe                 (474.7 MB)
%TEMP%\workbuddy-update-x64-workbuddy-ai\                                       (同名缓存目录)
```

注意最后两个缓存目录名：`workbuddy-update-x64` 与 `workbuddy-update-x64-workbuddy-ai`
——**两版各自有缓存目录，却共享同一个安装目标目录**。

---

## 根因定位（代码层）

从 `resources\app.asar` 中提取打包后的 JS，定位到 `update-service.win32.js`：

**① 安装目标目录由自身 `execPath` 推导，而非硬编码**

```js
getWorkbuddyInstallDir() {
  return process.env.WORKBUDDY_INSTALL_DIR?.trim() || path.dirname(process.execPath)
}

// quitAndInstall()
const n = path.dirname(process.execPath);
const i = ['/S', '--updated', '/UPDATE=1', `/D=${n}`];
spawn(e, i, { detached: true, stdio: ['ignore','ignore','ignore'] })
```

⇒ 更新器把**整个 `dirname(execPath)`** 当作自己的地盘交给安装程序覆盖。
两个产品放在同一目录时，就必然互相覆盖。

**② 更新缓存目录却按产品名区分**

```js
const i = (productConfig?.applicationName || basename(process.execPath))
            .toLowerCase().replace(/[^a-z0-9._-]+/g, '-');
const a = `workbuddy-update-${process.arch}${i === 'workbuddy' ? '' : '-' + i}`;
this.cachePath = path.join(os.tmpdir().trim(), a);
```

⇒ 这解释了为什么 `%TEMP%` 下有两个缓存目录：产品身份是分开的，
**唯独安装目标目录是共用的**。

**③ 存在一个"安装目录守卫"，但只在特定条件下触发**

```js
var i = new Set(['bin','locales','resources','tools','log','logs','_','share-target','assets']);
function isKnownAppDirectory(e) {
  let t = e.toLowerCase();
  return !!(i.has(t) || t.startsWith('$') || t.startsWith('nsis'));
}
function hasUserContentInInstallDir() {
  let e = getInstallDir();
  if (!e) return false;
  try {
    return fs.readdirSync(e, { withFileTypes: true })
             .some(e => e.isDirectory() && !isKnownAppDirectory(e.name));
  } catch { return false }
}
```

⇒ 该守卫用于保护用户项目目录，**并不会因为"存在另一个 WorkBuddy 产品"而阻止更新**，
因此拦不住本问题。

**④ 启动器/自启项按产品名区分，唯独目录不区分**

```js
function writeOsAutoLaunch(e) {
  if (process.platform === 'win32') {
    electron.app.setLoginItemSettings({
      openAtLogin: e,
      name: require_index.E(),                    // win32MutexName
      path: path.win32.normalize(process.execPath)
    });
  }
}
```

注册表实际生成两个不同名字的项：

```
WorkBuddy.WorkBuddy    = D:\多智能体\WorkBuddy\WorkBuddy.exe
WorkBuddy.WorkBuddyAI  = D:\Ai_tools\workbuddy-ai\WorkBuddyAI\WorkBuddyAI.exe
```

⇒ 命名空间是分开的，**再次印证两个产品本该是两个独立安装**。

---

## 复现步骤

1. 全新安装「WorkBuddy 国内版」5.8.3，安装到默认目录 `C:\workbuddy_cn\`。
2. 确认可正常使用，版本号 5.8.3。
3. 安装「WorkBuddy AI 国际版」5.8.4 —— 观察它**同样装入 `C:\workbuddy_cn\`**。
4. 启动其中一个，触发自动更新。
5. 检查 `resources\app.asar` 的修改时间已被更新为最新版本；
   另一个产品的 `product.json` 被替换为更新方的配置。
6. 启动被打包覆盖的那个产品：进程启动后立即退出，无错误提示。

---

## 影响

- 两个产品**无法在同一台机器上共存**，除非手工做目录隔离。
- 被覆盖方表现为**静默启动失败**（退出码 0、无日志、无弹窗），
  用户几乎不可能自行定位原因。
- 若被覆盖的版本较早，其 `app.asar` 与自身主程序版本不匹配，
  可能带来未定义的运行时行为。

---

## 建议修复

1. **让两个产品使用不同的默认安装目录**（最直接）。
   例如 `%LOCALAPPDATA%\WorkBuddy` 与 `%LOCALAPPDATA%\WorkBuddyAI`，
   或在安装时按 `win32x64UserAppId` 派生目录名。
   目前 `product.json` 中已有 `defaultFolderName`（国际版为 `workbuddy-ai`），
   但安装目标并未使用它。

2. **安装程序加冲突检测**：写入前检查目标目录是否已有**另一个产品**的
   `product.json`（`applicationName` 不同），若有则提示用户更换目录，
   而不是直接覆盖。

3. **把 `resources/` 整个资源层按产品隔离**：即使目录共用，也不应共享
   `app.asar` / `app.asar.unpacked` / `product.json`。

4. **更新器增加校验**：安装前比对目标目录 `product.json` 的
   `applicationName` 与自身是否一致，不一致则中止更新并给出明确提示。

5. **启动失败不要静默**：`app.asar` 与主程序版本不匹配时应给出可见提示，
   而不是以退出码 0 静默结束。

---

## 临时规避方案（用户侧）

既然安装目录即产品身份，把两者拆到各自目录即可，**无需等待官方修复**：

1. 从各自官方安装包中**直接解包**（这些包是 ZIP 自解压格式，
   `tar -xf <包> -C <目标目录>` 即可，不会触发覆盖），
   分别放入两个独立目录；
2. 修正注册表 `Uninstall` 项的 `UninstallString` / `DisplayIcon` / `InstallLocation`
   指向新目录，并把对应卸载器一并复制过去；
3. 把 `HKCU\...\Run` 中指向旧目录的自启项改指到各目录下的启动脚本；
4. 重建快捷方式；
5. **阻断自动更新**（任一即可，建议叠加）：
   - 在安装目录放置 `.env`，内容 `WORKBUDDY_UPDATE_URL=http://127.0.0.1:1`
     （更新基址支持该环境变量覆盖）；
   - 启动脚本中显式设置 `WORKBUDDY_CONFIG_DIR` 与 `WORKBUDDY_UPDATE_URL`；
   - 删除 `%LOCALAPPDATA%\@genieworkbuddy-desktop-updater\` 下的缓存安装包。

两版数据目录本就天然隔离（分别由 `dataFolderName` 决定：`.workbuddy` / `.workbuddy-ai`），
因此拆开后**会话、记忆、登录态不会互相污染**。

> 附：`product.json` 中 `win32MutexName` 分别为 `workbuddy` / `workbuddy-ai`，
> 互斥体不同，两版**可以同时运行**，进一步说明它们本设计为两个独立安装。
