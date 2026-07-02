# Fix Codex Remote Control on macOS with Clash / Mihomo

中文：解决 ChatGPT 手机端无法连接 macOS Codex App Remote Control 的问题。

This guide fixes a common issue where ChatGPT mobile cannot enable or connect to Codex App Remote Control on macOS, especially when Codex.app is launched from Dock, Launchpad, or Finder and does not inherit local proxy environment variables.

适用场景：受限网络环境下，macOS 上的 Codex App 需要通过 Clash / Clash Verge / Mihomo 类本机代理启用 Remote Control。

## Problem

When using Codex App Remote Control on macOS, ChatGPT mobile may show:

```text
Couldn't enable remote control. Try again.
```

or:

```text
无法启用远程控制。请重试。
```

The usual cause is that Codex.app launched from the macOS GUI does not inherit proxy environment variables such as `HTTPS_PROXY`, even though your local Clash / Mihomo proxy is working.

换句话说，本机代理端口本身可用，但从 Dock、Launchpad、Finder 直接启动的 Codex App 没有吃到代理环境变量，导致 Remote Control 无法正常启用或连接。

## Solution

This repository shows how to use a macOS LaunchAgent to set GUI proxy environment variables automatically at login, so the original Codex.app icon can keep working after reboot.

核心思路是：

```text
通过 macOS LaunchAgent，在用户登录时自动执行 launchctl setenv。
```

这样以后直接点击原来的 `Codex.app` 图标，也可以让 Codex 走本机代理。

## What This Guide Covers

- macOS GUI proxy environment setup
- Clash / Clash Verge / Mihomo local proxy port configuration
- `launchctl setenv`
- LaunchAgent auto-start
- Codex App Remote Control troubleshooting
- Safe rollback commands
- Optional terminal-only startup method

## Tested Environment

```text
System: macOS
Proxy client: Clash / Clash Verge / Mihomo
Default local proxy: 127.0.0.1:7890
Codex App path: /Applications/Codex.app
Codex executable: /Applications/Codex.app/Contents/MacOS/Codex
```

If your local proxy port is not `7890`, replace `127.0.0.1:7890` in the commands with your actual local proxy address, for example `127.0.0.1:7897`.

---

## Quick Start

1. Make sure your local proxy is running.
2. Replace `7890` with your actual proxy port if needed.
3. Run the setup commands below.
4. Restart Codex.app.
5. Enable Remote Control again in Codex App.

```bash
mkdir -p ~/bin ~/Library/LaunchAgents

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

launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist 2>/dev/null
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist
launchctl kickstart -k gui/$(id -u)/com.codex.gui-proxy

osascript -e 'quit app "Codex"' 2>/dev/null
open -a Codex
```

## Verify

Check that the macOS GUI environment has the proxy variables:

```bash
launchctl getenv HTTPS_PROXY
launchctl getenv NO_PROXY
```

Expected output:

```text
http://127.0.0.1:7890
localhost,127.0.0.1,::1
```

You can also test the local proxy port directly:

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

If the response includes:

```text
HTTP/1.1 200 Connection established
```

then the local proxy tunnel is working. A later `HTTP/2 403` from Cloudflare usually does not mean the proxy failed.

---

## 1. What This Setup Does

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

## 2. What This Setup Does Not Change

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

## 3. Check Local Proxy Connectivity First

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

## 4. Create the Proxy Setup Script

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

## 5. Create the LaunchAgent

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

## 6. Load the LaunchAgent Immediately

不用重启，直接运行下面命令即可立即加载：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist 2>/dev/null

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist

launchctl kickstart -k gui/$(id -u)/com.codex.gui-proxy
```

第一行用于卸载旧的同名 LaunchAgent，避免重复加载时报错。

第二行用于加载 LaunchAgent。

第三行用于立即启动它。

## 7. Restart Codex App

先退出 Codex：

```bash
osascript -e 'quit app "Codex"'
```

然后正常打开 Codex。

可以从图形界面打开：

```text
Dock
Launchpad
Finder -> Applications -> Codex.app
```

也可以从终端打开：

```bash
open -a Codex
```

注意，下面这些不是终端命令：

```text
Dock
Launchpad
Finder -> Applications -> Codex.app
```

它们只是 macOS 图形界面的打开方式。

## 8. Connect from ChatGPT Mobile

打开 Codex App 后，进入 Remote Control 相关设置，启用远程控制。

然后在手机 ChatGPT App 中连接 Codex。

需要注意：

```text
1. 手机 ChatGPT 和电脑 Codex App 必须登录同一个账号
2. 如果有多个 workspace，也需要选择同一个 workspace
3. 电脑需要保持在线和唤醒状态
4. 本机代理客户端需要保持运行
```

## 9. Daily Usage

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

## 10. Alternative: Launch Codex from Terminal Only

如果你不想设置当前 macOS GUI 用户会话的代理变量，可以只在需要 Remote Control 时从终端启动 Codex。

把下面函数加入 `~/.zshrc`：

```bash
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
  echo "Codex started with Clash / Mihomo proxy. Log: /tmp/codex-clash.log"
}
```

之后运行：

```bash
codexclash
```

这个方案影响范围更小，但以后需要用终端命令启动 Codex，而不是直接点原来的图标。

## 11. Rollback

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

## 12. FAQ

### Why is `HTTP/2 403` acceptable in the proxy test?

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

### Why set `NO_PROXY`?

因为本机回环地址不应该走代理。

这些地址包括：

```text
localhost
127.0.0.1
::1
```

Codex、本地服务、OAuth 回调或其他应用可能会使用本机地址。让这些地址绕过代理，可以避免本地连接被错误转发。

### Why not modify Codex.app directly?

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

### Will this affect remote Ubuntu or VS Code Remote?

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

## Screenshot Privacy Checklist

If you add screenshots, blur or crop:

- Account emails
- Avatars
- Workspace names
- Device names
- Real usernames in paths
- Proxy node names
- IP addresses

## Recommended GitHub Repository Settings

Suggested repository description:

```text
Fix ChatGPT mobile to Codex App Remote Control connection issues on macOS by setting GUI proxy environment variables with LaunchAgent for Clash/Mihomo.
```

Suggested GitHub Topics:

```text
codex
openai
openai-codex
chatgpt
chatgpt-mobile
codex-app
remote-control
macos
launchagent
launchctl
proxy
clash
mihomo
clash-verge
```

Suggested release title:

```text
v1.0.0 - macOS LaunchAgent proxy setup for Codex Remote Control
```

Suggested release notes:

```text
Initial stable guide for fixing ChatGPT mobile to Codex App Remote Control connection issues on macOS using LaunchAgent and local proxy environment variables.
```

## Conclusion

This setup uses macOS LaunchAgent to set GUI proxy environment variables at login, so Codex App can inherit local proxy settings even when launched from Dock, Launchpad, or Finder.

这解决了受限网络环境下手机 ChatGPT 无法连接 macOS Codex App Remote Control 的问题，同时不会修改 Codex App 本体，也不会影响 SSH、Ubuntu、VS Code Remote 等原有远程开发配置。
