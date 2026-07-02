# macOS Codex App 通过 Clash 代理启用 Remote Control

## 1. 背景

在 macOS 上使用 **Codex App Remote Control / 远程控制** 时，可能出现手机端无法连接、电脑端提示无法启用远程控制等问题。

经过排查，问题并不是 ChatGPT 手机端扫码本身，而是：

```text
Codex App 默认从 Dock / Launchpad / Finder 启动时，没有正确继承 Clash 代理环境。
```

手动在终端中给 Codex App 临时注入代理环境变量后，远程控制可以正常连接。因此可以确认，核心问题是：

```text
Codex App 需要显式吃到 Clash 的本机代理端口。
```

本文档记录两种处理方式：

1. **终端命令方式**：以后输入 `codexclash` 启动 Codex。
2. **LaunchAgent 持久化方式**：保留原来的 Codex 图标，重启后仍然自动设置代理环境。

---

## 2. 当前环境

本机环境如下：

```text
系统：macOS
代理软件：Clash / Clash Verge / mihomo 类客户端
本机代理端口：127.0.0.1:7890
Codex App 路径：/Applications/Codex.app
Codex 可执行文件：/Applications/Codex.app/Contents/MacOS/Codex
```

如果 Clash 的端口不是 `7890`，需要把下面所有命令中的：

```text
127.0.0.1:7890
```

替换成实际端口，例如：

```text
127.0.0.1:7897
```

---

## 3. 影响范围说明

本文方案只处理 **macOS 本机 Codex App** 的代理问题。

不会修改以下配置：

```text
~/.ssh/config
~/.bashrc
~/.profile
Ubuntu
VS Code Remote
RemoteForward
远程服务器上的 codex 配置
```

尤其不会影响之前用于远程 Ubuntu / VS Code Remote 的 SSH 代理链路。

需要注意的是，LaunchAgent 方案会设置 macOS 当前用户的 GUI 环境变量。因此从 Dock、Launchpad、Finder 打开的部分图形应用，也可能看到这些代理变量。

为避免本地回环地址被错误代理，配置中包含：

```text
NO_PROXY=localhost,127.0.0.1,::1
```

---

## 4. 代理连通性检查

先确认 Clash 本机代理端口可用。

在 macOS 本机终端运行：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

如果看到类似：

```text
HTTP/1.1 200 Connection established
HTTP/2 403
```

也可以继续。

这里的 `403` 通常是 Cloudflare challenge，不代表代理失败。关键是已经出现：

```text
HTTP/1.1 200 Connection established
```

如果出现下面这些，说明代理端口不可用：

```text
Connection refused
Could not connect
Operation timed out
```

---

# 方案一：添加 `codexclash` 终端命令

## 1. 适用场景

这个方案适合：

```text
不想影响 macOS 图形界面全局环境
只想在需要远程控制 Codex 时手动启动
希望不干扰之前的 SSH / Ubuntu / VS Code Remote 配置
```

缺点是以后不能直接点原来的 Codex 图标，而是需要在终端输入：

```bash
codexclash
```

---

## 2. 写入 `codexclash` 命令

在 macOS 本机终端运行：

```bash
cat >> ~/.zshrc <<'EOF'

# Start Codex App with Clash proxy for remote control
codexclash() {
  /usr/bin/osascript -e 'quit app "Codex"' >/dev/null 2>&1
  sleep 2

  nohup env \
  HTTP_PROXY=http://127.0.0.1:7890 \
  HTTPS_PROXY=http://127.0.0.1:7890 \
  ALL_PROXY=http://127.0.0.1:7890 \
  http_proxy=http://127.0.0.1:7890 \
  https_proxy=http://127.0.0.1:7890 \
  all_proxy=http://127.0.0.1:7890 \
  NO_PROXY=localhost,127.0.0.1,::1 \
  no_proxy=localhost,127.0.0.1,::1 \
  /Applications/Codex.app/Contents/MacOS/Codex \
  >/tmp/codex-clash.log 2>&1 &

  disown
  echo "Codex 已通过 Clash 代理启动。日志：/tmp/codex-clash.log"
}
EOF

source ~/.zshrc
```

---

## 3. 使用方法

以后需要启动 Codex Remote Control 时：

```bash
codexclash
```

然后在 Codex App 中启用 Remote Control，再用手机 ChatGPT 连接。

---

## 4. 日志位置

如果 Codex 没有正常打开，可以查看日志：

```bash
cat /tmp/codex-clash.log
```

---

# 方案二：保留原来的 Codex 图标，并在重启后仍然生效

## 1. 适用场景

这个方案适合：

```text
希望直接点击原来的 Codex.app 图标
希望重启 Mac 后仍然自动生效
不想每次都在终端输入 env 命令
```

核心思路是：

```text
使用 macOS LaunchAgent，在每次登录时自动执行 launchctl setenv。
```

这样从 Dock、Launchpad、Finder 打开的 Codex App，大概率可以继承这些代理环境变量。

---

## 2. 创建代理设置脚本

在 macOS 本机终端运行：

```bash
mkdir -p ~/bin

cat > ~/bin/set-gui-proxy-for-codex.sh <<'EOF'
#!/bin/zsh

launchctl setenv HTTP_PROXY http://127.0.0.1:7890
launchctl setenv HTTPS_PROXY http://127.0.0.1:7890
launchctl setenv ALL_PROXY http://127.0.0.1:7890

launchctl setenv http_proxy http://127.0.0.1:7890
launchctl setenv https_proxy http://127.0.0.1:7890
launchctl setenv all_proxy http://127.0.0.1:7890

launchctl setenv NO_PROXY "localhost,127.0.0.1,::1"
launchctl setenv no_proxy "localhost,127.0.0.1,::1"
EOF

chmod +x ~/bin/set-gui-proxy-for-codex.sh
```

---

## 3. 创建 LaunchAgent

继续运行：

```bash
mkdir -p ~/Library/LaunchAgents

cat > ~/Library/LaunchAgents/com.tamarisk.codex-gui-proxy.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.tamarisk.codex-gui-proxy</string>

    <key>ProgramArguments</key>
    <array>
      <string>/bin/zsh</string>
      <string>-lc</string>
      <string>$HOME/bin/set-gui-proxy-for-codex.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
EOF
```

---

## 4. 立即加载 LaunchAgent

不想等重启，可以立即加载：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.tamarisk.codex-gui-proxy.plist 2>/dev/null

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.tamarisk.codex-gui-proxy.plist

launchctl kickstart -k gui/$(id -u)/com.tamarisk.codex-gui-proxy
```

---

## 5. 检查是否生效

检查 `HTTPS_PROXY`：

```bash
launchctl getenv HTTPS_PROXY
```

正常输出应该是：

```text
http://127.0.0.1:7890
```

检查 `NO_PROXY`：

```bash
launchctl getenv NO_PROXY
```

正常输出应该是：

```text
localhost,127.0.0.1,::1
```

如果输出正确，说明当前用户的 GUI 环境变量已经设置成功。

---

## 6. 重新启动 Codex

先退出 Codex：

```bash
osascript -e 'quit app "Codex"'
```

然后可以用终端打开：

```bash
open -a Codex
```

也可以直接用鼠标点击原来的 Codex 图标：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

注意：上面这三行不是终端命令，只是图形界面的打开方式。

---

## 7. 使用顺序

以后重启 Mac 后，推荐顺序是：

```text
1. 启动 Clash
2. 确认 Clash 本机端口仍然是 7890
3. 直接点击原来的 Codex 图标
4. 在 Codex 中启用 Remote Control
5. 手机 ChatGPT 连接 Codex
```

如果 Clash 没有启动，Codex 会尝试连接：

```text
127.0.0.1:7890
```

但该端口没有代理服务，可能导致 Codex 网络失败。

---

# 方案对比

| 方案                | 优点                      | 缺点               | 是否推荐  |
| ----------------- | ----------------------- | ---------------- | ----- |
| `codexclash` 终端命令 | 影响范围最小，不影响其他 GUI App    | 每次要手动输入命令        | 稳妥推荐  |
| LaunchAgent 持久化   | 可以继续点原 Codex 图标，重启后自动生效 | 会影响当前用户 GUI 环境变量 | 方便推荐  |
| 直接修改 Codex.app 本体 | 看起来最彻底                  | 可能破坏 App 签名，影响更新 | 不推荐   |
| 全局修改系统代理配置        | 简单粗暴                    | 影响范围较大           | 不优先推荐 |

---

# 撤销 LaunchAgent 方案

如果以后想撤销持久化配置，运行：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.tamarisk.codex-gui-proxy.plist 2>/dev/null

rm -f ~/Library/LaunchAgents/com.tamarisk.codex-gui-proxy.plist
rm -f ~/bin/set-gui-proxy-for-codex.sh

launchctl unsetenv HTTP_PROXY
launchctl unsetenv HTTPS_PROXY
launchctl unsetenv ALL_PROXY

launchctl unsetenv http_proxy
launchctl unsetenv https_proxy
launchctl unsetenv all_proxy

launchctl unsetenv NO_PROXY
launchctl unsetenv no_proxy
```

撤销后，重新打开的 Codex App 将不再继承这些代理环境变量。

---

# 常见问题

## 1. 为什么 `Dock`、`Launchpad`、`Finder` 会提示 command not found？

因为下面这些内容不是终端命令：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

它们只是图形界面打开方式。

如果想在终端打开 Codex，应该使用：

```bash
open -a Codex
```

---

## 2. 为什么 `curl` 返回 403 也算代理通？

测试命令：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

如果返回中有：

```text
HTTP/1.1 200 Connection established
```

说明本机已经成功通过 Clash 代理建立了连接。

后面的：

```text
HTTP/2 403
```

通常是 Cloudflare 对命令行请求的挑战或拦截，不代表代理端口不可用。

---

## 3. 为什么要设置 `NO_PROXY`？

因为本机回环地址不应该走代理：

```text
localhost
127.0.0.1
::1
```

Codex、OAuth、本地服务或其他应用可能会用到本机回调。如果这些地址也被代理，可能导致本地连接异常。

所以设置：

```text
NO_PROXY=localhost,127.0.0.1,::1
```

---

## 4. 为什么不直接修改 Codex.app？

不建议修改：

```text
/Applications/Codex.app
```

因为这样可能导致：

```text
破坏 App 签名
影响自动更新
更新后配置丢失
排查问题更困难
```

LaunchAgent 方案不修改 Codex App 本体，因此更安全。

---

# 最终结论

如果目标是 **不影响原有 SSH / Ubuntu / VS Code Remote 配置**，并让 Codex Remote Control 正常走 Clash 代理：

```text
最稳方案：使用 codexclash 终端命令。
最方便方案：使用 LaunchAgent，让原来的 Codex 图标也继承代理环境。
```

当前已经验证：

```text
launchctl getenv HTTPS_PROXY
→ http://127.0.0.1:7890

launchctl getenv NO_PROXY
→ localhost,127.0.0.1,::1
```

说明 LaunchAgent 方案已经成功写入当前用户的 GUI 环境变量。
