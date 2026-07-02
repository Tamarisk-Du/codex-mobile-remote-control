# Fix ChatGPT Mobile → Codex App Remote Control on macOS

![Platform](https://img.shields.io/badge/platform-macOS-lightgrey)
![Type](https://img.shields.io/badge/type-troubleshooting%20guide-blue)
![Topic](https://img.shields.io/badge/topic-Codex%20Remote%20Control-purple)
![License](https://img.shields.io/badge/license-MIT-green)

中文：解决 macOS 上手机 ChatGPT 无法连接 Codex App Remote Control 的问题。

A bilingual macOS guide for fixing **ChatGPT mobile → Codex App Remote Control** connection issues when `Codex.app` needs a local proxy environment but does not inherit proxy variables from GUI launchers.

这是一份中英双语配置指南，用于解决 macOS 上从 **Dock、Launchpad、Finder 或 Applications** 启动 `Codex.app` 时，Codex App 没有正确继承本机代理环境变量，导致手机 ChatGPT 无法连接 Remote Control 的问题。

> This guide does not modify `Codex.app` itself.
> 本方案不会修改 `Codex.app` 本体。

## TL;DR / 一句话方案

If Codex.app works differently when launched from Terminal and from Dock / Finder, this guide helps you set proxy environment variables for macOS GUI apps using LaunchAgent.

如果终端里网络正常，但从 Dock / Finder 启动 Codex.app 后无法稳定连接，本方案通过 LaunchAgent 给 macOS 图形界面 App 设置代理环境变量。

---

## Common Error / 常见错误

## Screenshots / 截图

### Error on ChatGPT mobile / 手机端报错

![Codex Remote Control error on ChatGPT mobile](images/codex-remote-control-error-chatgpt-mobile.png)

When using Codex App Remote Control on macOS, ChatGPT mobile may show:

```text
Couldn't enable remote control. Try again.
```

中文界面可能显示：

```text
无法启用远程控制。请重试。
```

In many cases, the issue is not the phone itself. The real problem is that apps launched from the macOS GUI may not inherit the proxy environment variables that work in your terminal.

很多情况下，问题并不在手机本身，而是从 macOS 图形界面启动的 App 没有继承终端里的代理环境变量。

---

## Core Idea / 核心思路

Use a macOS **LaunchAgent** to set GUI proxy environment variables automatically after login.

通过 macOS **LaunchAgent**，在用户登录后自动写入图形界面会话的代理环境变量。

After setup, you can still open the original `Codex.app` normally from:

配置完成后，你仍然可以从下面这些位置正常打开原来的 `Codex.app`：

* Dock
* Launchpad
* Finder
* Applications folder / 应用程序文件夹

No modification to Codex App itself is required.

不需要修改 Codex App 本体。

---

## Who This Guide Is For / 适用人群

This guide is useful if you are using:

如果你符合以下情况，这份指南可能适合你：

* macOS
* ChatGPT mobile / 手机 ChatGPT
* Codex App Remote Control
* A local proxy client / 本机代理客户端
* Clash / Clash Verge / Mihomo or similar tools
* A local proxy address such as `127.0.0.1:7890`
* Codex works differently when launched from Terminal and from Dock / Finder

You may need this guide if:

你可能需要这份指南，如果你遇到：

- Terminal network works, but Codex.app launched from Dock / Finder cannot connect reliably.
- ChatGPT mobile shows `Couldn't enable remote control. Try again.`
- Your local proxy client is running, but Codex App still fails to enable Remote Control.
- Opening Codex from Terminal behaves differently from opening it from the app icon.

简单来说：终端正常，代理正常，但 Codex App 从图标启动后 Remote Control 仍然失败。

In short:

简单来说，本方案适用于：

```text
Terminal network works,
but Codex.app launched from Dock / Finder cannot connect reliably.
```

```text
终端里网络正常，
但从 Dock / Finder 启动 Codex.app 后连接不稳定或无法连接。
```

---

## Table of Contents / 目录

* [Common Error / 常见错误](#common-error--常见错误)
* [Core Idea / 核心思路](#core-idea--核心思路)
* [Who This Guide Is For / 适用人群](#who-this-guide-is-for--适用人群)
* [Tested Environment / 测试环境](#tested-environment--测试环境)
* [What This Fix Does / 本方案做了什么](#what-this-fix-does--本方案做了什么)
* [What This Fix Does Not Change / 本方案不会修改什么](#what-this-fix-does-not-change--本方案不会修改什么)
* [Quick Start / 快速开始](#quick-start--快速开始)
* [Step 1: Check Local Proxy / 检查本机代理](#step-1-check-local-proxy--检查本机代理)
* [Step 2: Create Proxy Script / 创建代理脚本](#step-2-create-proxy-script--创建代理脚本)
* [Step 3: Create LaunchAgent / 创建-launchagent](#step-3-create-launchagent--创建-launchagent)
* [Step 4: Load LaunchAgent / 加载 LaunchAgent](#step-4-load-launchagent--加载-launchagent)
* [Step 5: Verify Setup / 检查是否生效](#step-5-verify-setup--检查是否生效)
* [Step 6: Restart Codex App / 重启 Codex App](#step-6-restart-codex-app--重启-codex-app)
* [Step 7: Connect from ChatGPT Mobile / 使用手机 ChatGPT 连接](#step-7-connect-from-chatgpt-mobile--使用手机-chatgpt-连接)
* [Daily Usage / 以后怎么使用](#daily-usage--以后怎么使用)
* [Rollback / 撤销配置](#rollback--撤销配置)
* [FAQ / 常见问题](#faq--常见问题)
* [Disclaimer / 说明](#disclaimer--说明)

---

## Tested Environment / 测试环境

This guide was tested with the following setup:

本文使用以下环境进行测试：

```text
System: macOS
Proxy client: Clash / Clash Verge / Mihomo or similar local proxy clients
Local proxy address: 127.0.0.1:7890
Codex App path: /Applications/Codex.app
```

This guide assumes that your local proxy address is:

本文默认本机代理地址为：

```text
127.0.0.1:7890
```

If your proxy port is not `7890`, replace every `7890` in this guide with your actual local proxy port.

如果你的本机代理端口不是 `7890`，需要把本文所有命令中的 `7890` 替换成你的实际端口。

For example, if your proxy port is `7897`, replace:

例如，如果你的端口是 `7897`，就把：

```text
127.0.0.1:7890
```

with:

改成：

```text
127.0.0.1:7897
```

---

## What This Fix Does / 本方案做了什么

This guide creates a macOS LaunchAgent.

本方案会创建一个 macOS LaunchAgent。

The LaunchAgent runs automatically when the current macOS user logs in, and executes:

它会在当前 macOS 用户登录时自动执行：

```bash
launchctl setenv
```

This writes proxy environment variables into the current macOS GUI user session.

这样可以把代理环境变量写入当前 macOS 图形界面用户会话。

After setup, apps launched from these places can inherit the proxy environment:

配置完成后，从以下位置启动的 App 可以继承这些代理环境变量：

```text
Dock
Launchpad
Finder
Applications folder
```

This means you can keep launching `Codex.app` normally from the original icon.

也就是说，你可以继续直接点击原来的 `Codex.app` 图标启动 Codex。

---

## What This Fix Does Not Change / 本方案不会修改什么

This guide does **not** modify:

本方案不会修改以下内容：

```text
/Applications/Codex.app
~/.ssh/config
~/.bashrc
~/.zshrc
~/.profile
Ubuntu
VS Code Remote
SSH RemoteForward
Remote Linux Codex CLI
```

It does not modify Codex App itself, so it should not break the app signature or affect future app updates.

本方案不会修改 Codex App 本体，因此不会破坏 App 签名，也不会影响 Codex 后续更新。

---

## Quick Start / 快速开始

The full commands are provided below.

下面提供完整命令。

Basic flow:

基本流程：

```text
1. Check whether your local proxy works
2. Create a proxy setup script
3. Create a macOS LaunchAgent
4. Load the LaunchAgent
5. Verify environment variables
6. Restart Codex App
7. Connect from ChatGPT mobile
```

Before running the commands, make sure your local proxy client is already running.

运行命令前，请先确保你的本机代理客户端已经启动。

---

## Step 1: Check Local Proxy / 检查本机代理

Run the following command in macOS Terminal:

在 macOS 本机终端运行：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

If you see something like this:

如果看到类似输出：

```text
HTTP/1.1 200 Connection established
HTTP/2 403
```

It usually means the proxy tunnel has been established successfully.

这通常说明代理隧道已经成功建立。

The key line is:

关键是出现这一行：

```text
HTTP/1.1 200 Connection established
```

The later `HTTP/2 403` is usually caused by website-side protection or command-line request behavior. It does not necessarily mean that your local proxy failed.

后面的 `HTTP/2 403` 通常是网站侧保护机制或命令行请求行为导致的，不一定代表本机代理失败。

If you see errors such as:

如果出现以下错误：

```text
Connection refused
Could not connect
Operation timed out
```

Then your local proxy client may not be running, or your proxy port may not be `7890`.

说明你的本机代理客户端可能没有运行，或者代理端口不是 `7890`。

---

## Step 2: Create Proxy Script / 创建代理脚本

Run the following commands in macOS Terminal:

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

This script sets both uppercase and lowercase proxy variables:

这个脚本会同时设置大小写两组代理变量：

```text
HTTP_PROXY / http_proxy
HTTPS_PROXY / https_proxy
ALL_PROXY / all_proxy
NO_PROXY / no_proxy
```

`NO_PROXY` is important because local loopback addresses should not go through the proxy.

`NO_PROXY` 很重要，因为本机回环地址不应该走代理。

These addresses are usually used by local services, callbacks, or authentication flows:

这些地址通常用于本地服务、本机回调或认证流程：

```text
localhost
127.0.0.1
::1
```

---

## Step 3: Create LaunchAgent / 创建 LaunchAgent

Run the following commands:

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
      <string>\$HOME/bin/set-gui-proxy-for-codex.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
EOF
```

This LaunchAgent will run automatically when the current user logs in.

这个 LaunchAgent 会在当前用户登录 macOS 时自动运行。

---

## Step 4: Load LaunchAgent / 加载 LaunchAgent

You can load it immediately without rebooting:

不用重启，直接运行下面命令即可立即加载：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist 2>/dev/null

launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.codex.gui-proxy.plist

launchctl kickstart -k gui/$(id -u)/com.codex.gui-proxy
```

Explanation:

解释：

```text
bootout    unloads the old LaunchAgent if it already exists
bootstrap loads the LaunchAgent
kickstart  runs it immediately
```

```text
bootout    如果旧的 LaunchAgent 已存在，先卸载它
bootstrap 加载 LaunchAgent
kickstart  立即运行它
```

---

## Step 5: Verify Setup / 检查是否生效

Check `HTTPS_PROXY`:

检查 `HTTPS_PROXY`：

```bash
launchctl getenv HTTPS_PROXY
```

Expected output:

正常输出应该是：

```text
http://127.0.0.1:7890
```

Check `NO_PROXY`:

检查 `NO_PROXY`：

```bash
launchctl getenv NO_PROXY
```

Expected output:

正常输出应该是：

```text
localhost,127.0.0.1,::1
```

If both values are correct, the GUI proxy environment for the current macOS user session has been set successfully.

如果这两项输出正确，说明当前 macOS 用户图形界面的代理环境已经设置成功。

---

## Step 6: Restart Codex App / 重启 Codex App

First quit Codex:

先退出 Codex：

```bash
osascript -e 'quit app "Codex"'
```

Then open Codex again.

然后重新打开 Codex。

You can open it from:

可以从图形界面打开：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

Or from Terminal:

也可以从终端打开：

```bash
open -a Codex
```

Note:

注意：

```text
Dock
Launchpad
Finder → Applications → Codex.app
```

These are not terminal commands. They are normal macOS GUI launch methods.

这些不是终端命令，而是 macOS 图形界面的打开方式。

---

## Step 7: Connect from ChatGPT Mobile / 使用手机 ChatGPT 连接

After opening Codex App, enable Remote Control in Codex App.

打开 Codex App 后，在 Codex App 中启用 Remote Control。

Then connect from ChatGPT mobile.

然后在手机 ChatGPT App 中连接 Codex。

Please check the following:

需要注意：

```text
1. ChatGPT mobile and Codex App should use the same account
2. If you have multiple workspaces, select the same workspace
3. Your Mac should stay online and awake
4. Your local proxy client should keep running
```

```text
1. 手机 ChatGPT 和电脑 Codex App 应使用同一个账号
2. 如果有多个 workspace，需要选择同一个 workspace
3. Mac 需要保持在线和唤醒状态
4. 本机代理客户端需要保持运行
```

---

## Daily Usage / 以后怎么使用

After the setup is complete, the normal usage flow is:

配置完成后，以后的使用顺序是：

```text
1. Start your local proxy client
2. Make sure the local proxy port is still 127.0.0.1:7890
3. Open Codex from the original Codex.app icon
4. Enable Remote Control in Codex App
5. Connect from ChatGPT mobile
```

```text
1. 启动本机代理客户端
2. 确认本机代理端口仍然是 127.0.0.1:7890
3. 直接点击原来的 Codex.app 图标
4. 在 Codex App 中启用 Remote Control
5. 手机 ChatGPT 连接 Codex
```

The local proxy client still needs to be running.

本机代理客户端仍然需要先启动。

If your proxy client is not running, Codex may try to connect to:

如果代理客户端没有运行，Codex 可能会尝试连接：

```text
127.0.0.1:7890
```

but no proxy service is listening on that port, so Codex may still fail to connect.

但此时该端口没有代理服务，Codex 仍然可能无法正常连接。

---

## Rollback / 撤销配置

If you want to remove this setup later, run:

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

After rollback, newly opened GUI apps will no longer inherit these proxy environment variables.

撤销后，重新打开的图形界面 App 将不再继承这些代理环境变量。

---

## FAQ / 常见问题

### Why does `curl` return 403? / 为什么 `curl` 返回 403 也算正常？

Test command:

测试命令：

```bash
curl -I --proxy http://127.0.0.1:7890 https://chatgpt.com
```

If the output contains:

如果输出里有：

```text
HTTP/1.1 200 Connection established
```

it usually means your local proxy has successfully established the connection tunnel.

通常说明本机已经成功通过代理建立了连接隧道。

The later:

后面的：

```text
HTTP/2 403
```

is usually caused by website-side protection or command-line request behavior. It does not necessarily mean that your proxy port is broken.

通常是网站侧保护机制或命令行请求行为导致的，不一定代表代理端口不可用。

---

### Why set `NO_PROXY`? / 为什么要设置 `NO_PROXY`？

Because local loopback addresses should not go through the proxy.

因为本机回环地址不应该走代理。

These addresses include:

这些地址包括：

```text
localhost
127.0.0.1
::1
```

Codex, local services, OAuth callbacks, or other apps may use local addresses. Bypassing the proxy for these addresses can reduce local connection issues.

Codex、本地服务、OAuth 回调或其他应用可能会使用本机地址。让这些地址绕过代理，可以减少本地连接异常。

---

### Why not modify `Codex.app` directly? / 为什么不直接修改 `Codex.app`？

It is not recommended to directly modify:

不建议直接修改：

```text
/Applications/Codex.app
```

because it may:

原因是直接改 App 本体可能导致：

```text
Break app signature
Affect future updates
Be overwritten after app updates
Make troubleshooting harder
```

```text
破坏 App 签名
影响后续更新
更新后配置丢失
排查问题更困难
```

The LaunchAgent method does not modify Codex App itself.

LaunchAgent 方案不修改 Codex App 本体。

---

### Will this affect remote Ubuntu or VS Code Remote? / 这个方案会影响远程 Ubuntu 或 VS Code Remote 吗？

No.

不会。

This setup only sets environment variables for the current macOS GUI user session.

本方案只设置当前 macOS 用户图形界面会话的环境变量。

It does not modify:

它不会修改：

```text
~/.ssh/config
Ubuntu ~/.bashrc
Ubuntu ~/.profile
VS Code Remote
SSH RemoteForward
Remote Linux Codex CLI
```

So it should not affect your existing SSH remote development workflow.

因此它通常不会影响你原有的 SSH 远程开发流程。

---

### Do I need to run these commands after every reboot? / 每次重启后都要重新运行吗？

No.

不需要。

The LaunchAgent is configured with:

LaunchAgent 中设置了：

```xml
<key>RunAtLoad</key>
<true/>
```

So it will run automatically when the current macOS user logs in.

因此它会在当前 macOS 用户登录时自动运行。

---

### What if my proxy port is not `7890`? / 如果我的代理端口不是 `7890` 怎么办？

Replace every `7890` in this guide with your actual local proxy port.

把本文中所有 `7890` 替换成你的实际本机代理端口。

For example, if your local proxy port is `7897`, replace:

例如，如果你的本机代理端口是 `7897`，就把：

```text
127.0.0.1:7890
```

with:

改成：

```text
127.0.0.1:7897
```

---

### Does this make all apps use the proxy? / 这会让所有 App 都走代理吗？

This setup writes proxy environment variables into the current macOS GUI user session.

本方案会把代理环境变量写入当前 macOS 图形界面用户会话。

Apps that read these environment variables may use the proxy. Apps that ignore these variables may not be affected.

会读取这些环境变量的 App 可能会使用代理；不读取这些变量的 App 可能不会受影响。

---

### Does the proxy client still need to stay open? / 代理客户端还需要一直开着吗？

Yes.

需要。

This guide only tells GUI apps where the local proxy is. It does not create the proxy service itself.

本方案只是告诉图形界面 App 本机代理在哪里，并不会创建代理服务本身。

Your local proxy client still needs to keep running.

你的本机代理客户端仍然需要保持运行。

---

## Disclaimer / 说明

This is an unofficial troubleshooting guide based on a specific macOS local proxy environment.

这是一份基于特定 macOS 本机代理环境的非官方排查记录。

It is not affiliated with OpenAI, ChatGPT, Codex, Clash, Mihomo, or Apple.

本项目与 OpenAI、ChatGPT、Codex、Clash、Mihomo 或 Apple 官方无关。

Please adjust the proxy address and port according to your own environment.

请根据自己的实际环境调整代理地址和端口。
