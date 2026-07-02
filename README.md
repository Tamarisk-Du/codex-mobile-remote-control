# codex-mobile-remote-control
A macOS setup guide for connecting ChatGPT mobile to OpenAI Codex App Remote Control in Mainland network environments using Clash proxy./# 连接 Codex 手机远程控制
# 受限网络环境下连接 Codex 手机远程控制

本文记录在 **受限网络环境** 下，通过 **Clash / Mihomo 类本机代理** 让手机 ChatGPT 成功连接 macOS 上的 **OpenAI Codex App Remote Control** 的配置方法。

核心目标是：

```text
让原来的 Codex.app 图标也能继承本机代理环境，并且重启 Mac 后仍然生效。
```

---

## 1. 问题背景

在 macOS 上使用 OpenAI Codex App 的 Remote Control 功能时，手机 ChatGPT 可能无法成功连接电脑端 Codex。

常见现象是提示：

```text
无法启用远程控制。请重试。
```

英文界面可能显示：

```text
Couldn't enable remote control. Try again.
```

经过排查，问题并不是手机扫码本身，而是：

```text
从 Dock、Launchpad、Finder 直接启动的 Codex App 没有正确继承本机代理环境。
```

也就是说，本机代理本身是可用的，但 Codex App 直接从图形界面启动时，没有吃到代理环境变量，导致 Remote Control 无法正常启用或连接。

解决思路是：

```text
通过 macOS LaunchAgent，在用户登录时自动写入 GUI 代理环境变量。
```

这样以后直接点击原来的 `Codex.app` 图标，也可以让 Codex 走本机代理。

---

## 2. 测试环境

本文使用的环境如下：

```text
系统：macOS
代理软件：Clash / Clash Verge / Mihomo 类客户端
本机代理端口：127.0.0.1:7890
Codex App 路径：/Applications/Codex.app
```

本文默认本机代理端口是：

```text
127.0.0.1:7890
```

如果你的代理端口不是 `7890`，需要把本文所有命令里的 `7890` 替换成你的实际端口。

例如你的端口是 `7897`，就把：

```text
127.0.0.1:7890
```

改成：

```text
127.0.0.1:7897
```

---

## 3. 本方案做了什么

本方案会创建一个 macOS LaunchAgent。

它会在用户登录 macOS 时自动执行：

```bash
launchctl setenv
```

从而给当前 macOS 图形界面用户会话写入代理环境变量。

这样，从下面这些位置启动的 App 就可以继承代理环境：

```text
Dock
Launchpad
Finder
```

配置完成后，可以继续直接点击原来的 `Codex.app` 图标启动 Codex，并让 Codex 走本机代理。

---

## 4. 本方案不会修改什么

本方案不会修改以下内容：

```text
~/.ssh/config
~/.bashrc
~/.profile
Ubuntu
VS Code Remote
SSH RemoteForward
远程 Linux 上的 Codex CLI
/Applications/Codex.app 本体
```

也就是说，它不会影响原有的 SSH 远程开发、Ubuntu 代理转发、VS Code Remote 或远程服务器上的 Codex 配置。

本方案也不会修改 Codex App 本体，因此不会破坏 App 签名，也不会影响 Codex 后续更新。

---

## 5. 先检查本机代理是否可用

在 macOS 本机终端运行：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

如果看到类似输出：

```text
HTTP/1.1 200 Connection established
HTTP/2 403
```

说明代理连接基本可用。

这里的 `HTTP/2 403` 通常是 Cloudflare 对命令行请求的挑战或拦截，不代表代理失败。

关键是出现了：

```text
HTTP/1.1 200 Connection established
```

如果出现以下错误，说明代理端口可能没有正常工作：

```text
Connection refused
Could not connect
Operation timed out
```

此时需要先检查代理客户端是否正在运行，以及本机代理端口是否确实是 `7890`。

---

## 6. 创建代理设置脚本

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

这里同时设置了大小写两组代理变量：

```text
HTTP_PROXY / http_proxy
HTTPS_PROXY / https_proxy
ALL_PROXY / all_proxy
NO_PROXY / no_proxy
```

其中 `NO_PROXY` 很重要，因为本机回环地址不应该走代理：

```text
localhost
127.0.0.1
::1
```

这些地址通常用于本机回调、本地服务或 OAuth 相关流程。让它们绕过代理可以减少本地连接异常。

---

## 7. 创建 LaunchAgent

继续在 macOS 本机终端运行：

```bash
mkdir -p ~/Library/LaunchAgents

cat > ~/Library/LaunchAgents/com.codex.gui-proxy.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.codex.gui-proxy</string>

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

这个 LaunchAgent 会在当前用户登录 macOS 时自动运行。

---

## 8. 立即加载 LaunchAgent

不用重启，直接运行下面命令即可立即加载：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist 2>/dev/null

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist

launchctl kickstart -k gui/$(id -u)/com.codex.gui-proxy
```

第一行用于卸载旧的同名 LaunchAgent，避免重复加载时报错。

第二行用于加载 LaunchAgent。

第三行用于立即启动它。

---

## 9. 检查是否生效

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

如果这两项输出正确，说明 macOS 当前用户的图形界面代理环境已经设置成功。

---

## 10. 重新启动 Codex App

先退出 Codex：

```bash
osascript -e 'quit app "Codex"'
```

然后正常打开 Codex。

可以从图形界面打开：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

也可以从终端打开：

```bash
open -a Codex
```

注意，下面这些不是终端命令：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

它们只是 macOS 图形界面的打开方式。

---

## 11. 连接手机 ChatGPT

打开 Codex App 后，进入 Remote Control 相关设置，启用远程控制。

然后在手机 ChatGPT App 中连接 Codex。

需要注意：

```text
1. 手机 ChatGPT 和电脑 Codex App 必须登录同一个账号
2. 如果有多个 workspace，也需要选择同一个 workspace
3. 电脑需要保持在线和唤醒状态
4. 本机代理客户端需要保持运行
```

---

## 12. 以后怎么使用

配置完成后，以后的使用顺序是：

```text
1. 启动本机代理客户端
2. 确认本机代理端口仍然是 127.0.0.1:7890
3. 直接点击原来的 Codex 图标
4. 在 Codex App 中启用 Remote Control
5. 手机 ChatGPT 连接 Codex
```

本机代理客户端仍然需要先启动。

如果代理客户端没有运行，Codex 会尝试连接：

```text
127.0.0.1:7890
```

但此时该端口没有代理服务，可能导致 Codex 无法联网。

---

## 13. 撤销配置

如果以后想撤销这个配置，可以运行：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist 2>/dev/null

rm -f ~/Library/LaunchAgents/com.codex.gui-proxy.plist
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

## 14. 常见问题

### 14.1 为什么 `curl` 返回 403 也算正常？

测试命令：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

如果输出里有：

```text
HTTP/1.1 200 Connection established
```

说明本机已经成功通过代理建立了连接。

后面的：

```text
HTTP/2 403
```

通常是 Cloudflare 对命令行请求的挑战或拦截，不代表代理端口不可用。

---

### 14.2 为什么要设置 `NO_PROXY`？

因为本机回环地址不应该走代理。

这些地址包括：

```text
localhost
127.0.0.1
::1
```

Codex、本地服务、OAuth 回调或其他应用可能会使用本机地址。让这些地址绕过代理，可以避免本地连接被错误转发。

---

### 14.3 为什么不直接修改 Codex.app？

不建议直接修改：

```text
/Applications/Codex.app
```

原因是直接改 App 本体可能导致：

```text
破坏 App 签名
影响后续更新
更新后配置丢失
排查问题更困难
```

LaunchAgent 方案不修改 Codex App 本体，因此更安全。

---

### 14.4 这个方案会影响远程 Ubuntu 或 VS Code Remote 吗？

不会。

本方案只设置 macOS 当前用户图形界面会话的环境变量，不会修改：

```text
~/.ssh/config
Ubuntu ~/.bashrc
Ubuntu ~/.profile
VS Code Remote
SSH RemoteForward
远程服务器上的 Codex CLI
```

因此不会影响原有远程开发代理链路。

---

## 15. 最终效果

配置成功后，检查结果应该类似：

```text
launchctl getenv HTTPS_PROXY
→ http://127.0.0.1:7890

launchctl getenv NO_PROXY
→ localhost,127.0.0.1,::1
```

之后就可以直接点击原来的 Codex App 图标，让 Codex 继承本机代理环境，并在受限网络环境下使用手机 ChatGPT 连接 Codex Remote Control。

---

## 16. 结论

本方案通过 macOS LaunchAgent 在用户登录时自动设置 GUI 代理环境变量，使 Codex App 在从 Dock、Launchpad 或 Finder 启动时也能使用本机代理。

这解决了受限网络环境下手机 ChatGPT 无法连接 macOS Codex App Remote Control 的问题，同时不会修改 Codex App 本体，也不会影响 SSH、Ubuntu、VS Code Remote 等原有远程开发配置。
