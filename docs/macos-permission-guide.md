# macOS WeChat 密钥提取：权限与签名完全指南

> 基于多台机器 (macOS 10.15 ~ 15.x, Intel + Apple Silicon) 的实测经验总结。

## 核心结论

能否从微信进程提取加密密钥，取决于 **两个独立问题**：

| 问题 | 控制什么 | 关键因素 |
|------|---------|---------|
| `task_for_pid()` 能否成功 | 读取进程内存 | **目标 App 的代码签名** |
| `codesign` 能否重签名 | 修改 App 文件 | **调用者的完全磁盘访问** |

---

## 一、task_for_pid 权限（读取微信内存）

### 决定因素：微信 App 的 Hardened Runtime

```bash
# 检查微信签名状态
codesign -dv /Applications/WeChat.app 2>&1 | grep -E "Signature|flags"
```

#### 情况 A：Ad-hoc 签名（无 Hardened Runtime）

```
flags=0x2(adhoc)
Signature=adhoc
TeamIdentifier=not set
```

**原因**: 安装过防撤回补丁等第三方修改工具，App 被重新签名。

**权限要求**: 只需 `sudo`，任何上下文（Terminal、SSH、cron）都能成功。

```bash
# SSH 远程直接可用
sudo ./find_all_keys_macos
```

#### 情况 B：Apple 官方签名（有 Hardened Runtime）

```
flags=0x10000(runtime)
Signature size=9092
Authority=...Apple...
```

**原因**: App Store 下载或官方 DMG 安装，未经修改。

**权限要求**: `sudo` + 本机 GUI 终端 + TCC "开发者工具"授权。SSH **不可行**。

```
taskgated 检查流程:
  目标有 hardened runtime?
    YES → 检查调用者的"负责应用"是否有 TCC DeveloperTool 授权
          SSH 的负责应用是 sshd → 无法获得 TCC 授权 → 拒绝
          Terminal.app 可以弹窗获得授权 → 允许
    NO  → root (sudo) 即可 → 允许
```

### 实测数据

| 机器 | macOS | WeChat 签名 | 本机 Terminal sudo | SSH sudo |
|------|-------|------------|-------------------|---------|
| MacBook (macOS 15.x) | 15.x | **ad-hoc** (防撤回补丁) | ✅ | ✅ |
| Mac mini (Catalina) | 10.15.8 | Apple 官方 runtime | ✅ | ❌ |
| MacBook Pro (Big Sur) | 11.1 | Apple 官方 runtime | ✅ | ❌ |

### SSH 下穷举过的所有方法（Apple 签名时全部失败）

| 方法 | 结果 | 错误信息 |
|------|------|---------|
| `sudo frida -p <pid>` | ❌ | unable to access process |
| `lldb -p <pid>` | ❌ | non-interactive debug session |
| `sudo gcore <pid>` | ❌ | insufficient privilege |
| 带 debugger entitlement 的 C 程序 | ❌ | KERN_FAILURE=5 |
| `launchctl asuser` (用户会话) | ❌ | task_for_pid=5 |
| LaunchAgent (Aqua GUI 会话) | ❌ | 非 root，需要 sudo |
| LaunchDaemon (root) | ❌ | 系统域无 GUI 上下文 |
| `launchctl submit` (root) | ❌ | 同上 |
| `osascript` 操控 Terminal.app | ❌ | 需要辅助功能权限/挂起 |
| 修改 TCC.db 给 sshd 授权 | ❌ | SIP 保护，restricted 只读 |
| `vmmap` / `heap` | ⚠️ | 只能看元数据，无法读内存 |

---

## 二、codesign 权限（重签名微信 App）

如果微信是 Apple 官方签名，需要重签名为 ad-hoc 来解锁 SSH 提取。

### 问题：SSH 下 codesign 可能失败

```
$ sudo codesign --force --deep --sign - /Applications/WeChat.app
/Applications/WeChat.app: Operation not permitted
In subcomponent: /Applications/WeChat.app/Contents/MacOS/WeChatAppEx.app
```

**原因**: SSH 进程没有「完全磁盘访问」(Full Disk Access, FDA) 权限，无法修改 `/Applications` 下的 App bundle 文件。

### 给 SSH 授予完全磁盘访问

在目标机器的 **GUI** 上操作：

```
系统偏好设置 → 安全性与隐私 → 隐私 → 完全磁盘访问
点击 🔒 解锁 → 点 + 号 → Cmd+Shift+G 输入路径
```

**必须添加这两个**（缺一不可）：

| 路径 | 说明 |
|------|------|
| `/usr/sbin/sshd` | SSH 守护进程 |
| `/usr/libexec/sshd-keygen-wrapper` | SSH 的实际执行进程（负责应用） |

> ⚠️ 添加后必须**断开 SSH 重新连接**！TCC 权限在进程启动时检查，不会热更新。

### 验证 FDA 是否生效

```bash
# 重连 SSH 后执行
cat ~/Library/Application\ Support/com.apple.TCC/TCC.db > /dev/null 2>&1 && echo "FDA: YES" || echo "FDA: NO"
```

TCC.db 是受保护文件，只有 FDA 进程能读取。

### 完整流程：SSH 远程重签名微信

```bash
# 0. 前提：SSH 已有 FDA（上面的步骤）

# 1. 确认微信已退出
kill $(pgrep -x WeChat) 2>/dev/null
sleep 2
pgrep -x WeChat && echo "还在运行！" || echo "已退出"

# 2. 清除扩展属性（可选，防止干扰）
sudo xattr -cr /Applications/WeChat.app

# 3. Ad-hoc 重签名
sudo codesign --force --deep --sign - /Applications/WeChat.app

# 4. 验证签名
codesign -dv /Applications/WeChat.app 2>&1 | grep -E "Signature|flags"
# 期望: flags=0x2(adhoc), Signature=adhoc

# 5. 用户需在 GUI 上重新打开微信并登录
# （或者 SSH 执行 open，但用户仍需在 GUI 上完成登录）
open /Applications/WeChat.app
```

### 注意事项

| 事项 | 说明 |
|------|------|
| 微信必须先退出 | 运行中的 App，其 dylib/binary 被占用，codesign 会报 `internal error` |
| **重签名后必须重启微信** | 已运行的进程仍使用旧签名的内存映像，task_for_pid 仍会失败。必须 kill 后重新启动 |
| 重签名后需重新登录微信 | 签名变更会使登录态失效 |
| 自动更新可能覆盖签名 | 微信更新后变回 Apple 签名，需要再次重签 |
| 小程序可能受影响 | 部分小程序校验签名，ad-hoc 可能报安全错误 |

---

## 三、权限矩阵总结

| 操作 | 需要的权限 | SSH 需要额外配置 |
|------|-----------|-----------------|
| 读取微信数据库文件 | 文件系统权限（通常有） | 无 |
| `task_for_pid` (ad-hoc App) | sudo | 无 |
| `task_for_pid` (Apple 签名 App) | sudo + TCC DeveloperTool | **不可行**，必须本机 Terminal |
| `codesign` 重签名 App | sudo + FDA | SSH 需添加 sshd + sshd-keygen-wrapper 到 FDA |
| 修改 TCC.db | sudo + 关闭 SIP | **不推荐** |

### 完全远程操作清单（一次性 GUI 配置）

只需在目标机器 GUI 上做一次，之后 SSH 永久可用：

1. **完全磁盘访问** → 添加 `/usr/sbin/sshd` 和 `/usr/libexec/sshd-keygen-wrapper`
2. SSH 连入 → `sudo codesign --force --deep --sign - /Applications/WeChat.app`
3. 用户在 GUI 重开微信并登录
4. 之后 SSH 永久可以 `sudo` 提取密钥，微信重启也不影响（除非更新覆盖签名）

---

## 四、常见误区

| 误区 | 真相 |
|------|------|
| "需要给终端完全磁盘访问才能调试" | ❌ FDA 控制文件访问，不控制进程调试 |
| "需要给终端开发者工具权限" | ⚠️ 仅当目标 App 有 hardened runtime 时才需要 |
| "SSH 下永远无法提取密钥" | ❌ 目标 App 是 ad-hoc 签名时，SSH sudo 可以 |
| "macOS 版本决定了能否 SSH 调试" | ❌ 主要取决于目标 App 的签名状态 |
| "SIP 阻止了调试微信" | ❌ SIP 只保护系统进程，微信不受 SIP 保护 |
| "加了 sshd 到 FDA 就行" | ❌ 还需要加 `sshd-keygen-wrapper`，且要重连 SSH |
| "微信开着也能重签名" | ❌ 运行中的 binary/dylib 被占用，codesign 会失败 |

---

## 五、重签名后微信权限 silent 失效

### 现象

完成 ad-hoc 重签名后，微信任意以下功能都可能"看起来已授权但实际被拒绝"：

- 截图 / 屏幕共享（`ScreenCapture`）
- 视频通话 / 扫码（`Camera`）
- 语音消息 / 通话（`Microphone`）
- 自动化、第三方输入法（`AppleEvents`）
- 同步通讯录（`AddressBook`）
- 文件发送 / 接收（`SystemPolicyDocumentsFolder` / `Downloads` / `Desktop`）

System Settings 里通常仍看到"微信.app"开关是 ON，但运行时权限校验失败。微信会反复弹"需要开启 X 权限"。

### 根因（第一性原理）

macOS TCC（Transparency, Consent, and Control）按 **bundle id + csreq** 联合校验权限。`csreq`（code requirement）是从 app 的 code signature 推导出的二进制 blob，存在 `/Library/Application Support/com.apple.TCC/TCC.db` 的 `access` 表里，每条 ~160 字节。

`codesign --force --deep --sign -` 把 WeChat 从官方签名换成 ad-hoc 签名（甚至 ad-hoc → ad-hoc 重签也会变），新进程的 csreq 跟旧记录里那条对不上 —— tccd 拒绝。

System Settings UI 只按 client 显示开关、不重算 csreq，所以视觉上是"已授权"，运行时实际拒绝。这是 silent drift。

### 修复步骤

把 WeChat 在 TCC 里的旧记录全部抹掉，让 macOS 在下次微信请求权限时按新签名重新生成 csreq：

```bash
for s in ScreenCapture Camera Microphone AppleEvents AddressBook \
         SystemPolicyDocumentsFolder SystemPolicyDownloadsFolder SystemPolicyDesktopFolder; do
  tccutil reset "$s" com.tencent.xinWeChat
done
```

`tccutil` 对没有授权过的 service 会报 "No such bundle identifier"，这是 no-op，不影响其他 service 的 reset。

之后退出并重新打开微信，按 GUI 提示重新允许：

```bash
killall WeChat
open /Applications/WeChat.app
```

> 这一步**应当由用户/agent 手动执行**，不在 `wx init` 里自动跑——TCC 重置会让用户的现有授权失效，需要由人决定时机。

#### macOS 26 的 UI 拆分

在 macOS 26 上，**隐私与安全 → 录屏与系统录音** 显示为两块，容易踩坑：

| 区域 | 作用 |
|------|------|
| **录屏与系统录音**（上半区） | 录制屏幕内容 + 系统音频；微信截图、屏幕共享需要这一项 |
| **仅系统录音**（下半区） | 只录系统音频；只打开这一项**不能**修复微信截图 |

把 WeChat 加进上半区；只勾下半区的"仅系统录音"无效。

### 验证

确认 WeChat 当前是 ad-hoc 签名（这是修复前提）：

```bash
codesign -dv --verbose=4 /Applications/WeChat.app 2>&1 | grep -E "Signature|flags|TeamIdentifier"
```

期望看到：

```text
flags=0x2(adhoc)
Signature=adhoc
TeamIdentifier=not set
```

最直接的功能验证：在微信里使用截图、视频通话、麦克风等功能，按 GUI 弹窗的"允许"重新授权一次，之后正常工作。

---

## 六、`"微信" 想访问其他 App 的数据` 弹窗

### 现象

执行过 `wx init`、对 `/Applications/WeChat.app` 做过 ad-hoc 重签名之后，再使用微信时会比较频繁地看到 macOS 弹出：

```
"微信" 想访问其他 App 的数据。
单独存放 App 数据可让你更容易管理隐私和安全。
[ 不允许 ]   [ 允许 ]
```

最常见的触发面是**在微信里打开公众号文章**，但这只是高频触发面，不是根因。

### 根因（第一性原理）

这弹窗是 macOS Ventura+ / 14 / 15 对 **app data container 跨身份访问** 的保护：当前进程（"微信"）正在读取另一个 code identity 的 app 留下的数据。

我们当前 macOS 方案为了让 `task_for_pid` 能拿到 WeChat 的 task port、读取进程内存里的 raw key，要求用户执行：

```bash
codesign --force --deep --sign - /Applications/WeChat.app
```

这一步把 WeChat 从 Apple 官方签名换成 ad-hoc 身份。对用户来说它仍然是"微信"；对 macOS 安全模型来说，**重签前的 WeChat** 和 **重签后的 WeChat** 已经不是同一个 app identity。

之后当（重签后的）微信访问它原本的 `~/Library/Containers/com.tencent.xinWeChat/...`、缓存、app group 等数据时，系统看到的是"一个新身份在读旧身份留下的 container 数据"，于是按隐私保护策略弹这个对话框。公众号文章里的 webview / cookie / 缓存路径刚好踩到了这条访问路径，所以"打开公众号就弹"会非常容易复现，但**本质不是公众号页面的问题**，而是 code identity + container access。

> 注意：这**不是** "wx-cli 在偷偷读别的 App 的数据"，wx-cli 进程本身对 WeChat container 是只读访问；但**要求用户重签 WeChat** 这一步本身就是这类弹窗的直接诱因。所以这是当前 macOS invasive init 路径的已知副作用，不是与 wx-cli 无关的系统行为。

### 应对

短期缓解：
- 点"允许"通常只是放行**当前这次** WeChat 进程；下一次 WeChat 启动权限会 reset，可能还会再弹
- 该授权一般不会在 System Settings 里留下显式开关，因为它绑定的是动态的 code identity

彻底不弹：
- 把 `/Applications/WeChat.app` 恢复成官方签名（重装官方 WeChat 包），不再执行 `codesign --force --deep --sign -`
- 这一步只是放弃**当前依赖 ad-hoc 重签的默认路径**，并不等于放弃 macOS memory-scan：在本机 GUI Terminal 下、对 Terminal.app 授予「开发者工具」TCC 权限后，`task_for_pid` 对 Apple 官方签名（hardened runtime）的 WeChat 应当仍能走通——参考 §一 实测表里的"Apple 签名 + 本机 Terminal sudo = ✅"
- ⚠️ 实测覆盖范围说明：§一 实测表里 "Apple 签名 + 本机 Terminal sudo ✅" 的两条实证只覆盖 macOS 10.15 (Catalina) 与 11.1 (Big Sur)；macOS 14 (Sonoma) / 15 (Sequoia) 上是否仍走通**未在本项目内实测**。如果你按这条路恢复官方签名后发现 init 走不通，请回到重签路径并接受本节描述的弹窗副作用
- 真正受限的场景是 SSH 远程 + Apple 签名 WeChat：`sshd` 拿不到 TCC 开发者工具授权，这时才必须走重签路径

长期方向：
- 这条副作用的真正修复是把 `wx init` 重新设计成 `safe → assisted → invasive fallback` 三层：默认不动 WeChat，只有在前两条都不可行时才走 ad-hoc 重签，并先打出完整副作用清单让用户显式确认。在那之前，这是已知 trade-off。

---

## 七、App Store 沙盒版微信实测（macOS 15.0.1 / 微信 4.1.1）

> 2026-11-08 单机实测记录。与 §一、§二 的默认结论不同，本场景下**重签名必须换做法**，否则 App 无法启动。

### 环境

| 项 | 值 |
|------|------|
| macOS | 15.0.1 (24A348) |
| 微信 | 4.1.1，App Store 版本 |
| 签名 | `flags=0x10000(runtime)`，`TeamIdentifier=5A4RE8SF68`，含 `com.apple.security.app-sandbox=true` |
| wx-cli | 0.6.3（本仓库源码构建，与 `target/release/wx` 同 hash） |
| 其他 | 用户属 `_developer` 组，`DevToolsSecurity -status` 为 enabled |

### 四条路径的实测结果

| 路径 | 结果 | 关键错误 |
|------|------|---------|
| 官方硬化签名 + 本机 GUI Terminal + `sudo wx init` | ❌ | `task_for_pid 失败 (kr=5)` |
| 原地 `sudo codesign --force --deep --sign -` | ❌ | `Operation not permitted`（`In subcomponent: .../ilink2.framework`） |
| 复制 bundle 后 `--preserve-metadata=entitlements` 重签 | ❌ | 签名成功但启动失败：`RBSRequestErrorDomain Code=5` / `POSIX 153` |
| 复制 bundle 后**不带 entitlement** 重签 | ✅ | 可启动、数据完整、22/22 密钥 |

### 两个根的成因

**1. `Operation not permitted` 不是 TCC，是 `UF_COMPRESSED`**

App Store 安装的 bundle 内所有文件都带压缩标志，即使 root 也无法就地写入：

```bash
find /Applications/WeChat.app -type f -flags +compressed | wc -l   # 本机 118
ls -lO /Applications/WeChat.app/Contents/MacOS/WeChat             # 显示 compressed
```

`cp -R` 复制会解压（`find ... -flags +compressed` 归零），因此只能"复制 → 签副本 → 替换"。

**2. 保留 `app-sandbox` 会导致启动失败**

ad-hoc 签名没有 Team ID，但 entitlement 仍声称 `5A4RE8SF68.com.tencent.xinWeChat` 并要求沙盒容器；amfid 校验不到对应身份，直接拒绝 spawn（`POSIX 153`）。所以 App Store 沙盒版**不能**用 `--preserve-metadata=entitlements`，只能放弃沙盒 entitlement 才能启动。

### 可用路径（复制 + 去 entitlement 重签）

```bash
# 1. 退出微信并复制（cp -R 会解压 UF_COMPRESSED）
killall WeChat 2>/dev/null; sleep 2
cp -R /Applications/WeChat.app /Applications/WeChat-adhoc.app

# 2. 不带 entitlement 重签（去掉 app-sandbox，才能启动）
codesign --force --deep --sign - /Applications/WeChat-adhoc.app

# 3. 验证：flags 应为 0x2(adhoc)，无 runtime，entitlements 为空
codesign -dv --verbose=4 /Applications/WeChat-adhoc.app 2>&1 | grep flags
codesign -d --entitlements - /Applications/WeChat-adhoc.app 2>&1 | head -3

# 4. 交换（原包留作回滚，命名不要带 .app 后缀，避免被 LaunchServices 当成第二个 App）
mv /Applications/WeChat.app /Users/<user>/WeChat.official-backup
mv /Applications/WeChat-adhoc.app /Applications/WeChat.app

# 5. 启动并确认在读写原容器目录（数据未丢）
open /Applications/WeChat.app
lsof -p $(pgrep -f "WeChat.app/Contents/MacOS/WeChat" | head -1) | grep -c xwechat_files
```

> 以上需要 root 的步骤可用 `osascript -e 'do shell script "..." with administrator privileges'` 触发系统授权框，由用户在 GUI 输入密码。

### 去沙盒后的两个连带影响

| 影响 | 说明 |
|------|------|
| 失去沙盒隔离 | WeChat 以非沙盒身份运行，可访问全盘用户目录；这是安全性上的实际权衡 |
| 数据目录不变 | 实测仍使用绝对路径 `~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files`，聊天记录不丢 |

### 取密钥：root 枚举会被 TCC 拦住

- **用户态**（有 FDA 的终端）能枚举到库，但只能靠 LLDB hook 拿到部分 key（本机 45s 窗口只拿到 3/22）
- **root 态**（osascript / sudo）枚举返回 0：`ls` 该目录直接 0 行，属 TCC 完全磁盘访问问题，与签名无关

绕过方式：用**硬链接**把 DB 挂到不受保护目录（不复制数据），让 root 在那边扫描：

```bash
SRC=~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/<账号>/db_storage
DST=~/.wx-cli/probe-$(date +%H%M%S); mkdir -p "$DST"
cd "$SRC" && find . -name "*.db" | while read -r f; do
  mkdir -p "$DST/$(dirname "$f")"; ln "$f" "$DST/$f"
done
# 把 config.json 的 db_dir 临时指向 $DST，root 跑 wx key extract，完成后改回真实路径
```

本机结果：`找到 22 个加密数据库` → 读取约 2.6 GB 内存 → `22/22 全部匹配`。

### `config.json` 的 `db_dir` 粒度（易错）

| 写法 | 结果 |
|------|------|
| `.../xwechat_files`（根） | 枚举到 0 个库 |
| `.../xwechat_files/<账号>` | 枚举到 0 个库 |
| `.../xwechat_files/<账号>/db_storage` | ✅ 正确 |

另一个坑：密钥更新后若仍报 `无法解密 session.db`，是 daemon 缓存了旧状态，`wx daemon stop` 后重跑即可。

### 回滚

```bash
rm -rf /Applications/WeChat.app
mv /Users/<user>/WeChat.official-backup /Applications/WeChat.app
# 或直接官网重装微信
```

### 复发条件

微信自动更新会写回官方签名，上述步骤需全部重做。判断方法：

```bash
codesign -dv --verbose=4 /Applications/WeChat.app 2>&1 | grep flags
# 出现 runtime 即已被更新覆盖
```
