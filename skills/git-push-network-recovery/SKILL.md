---
name: git-push-network-recovery
description: Diagnose and recover GitHub HTTPS push timeouts, connection resets, HTTP/2 framing errors and Empty reply errors, especially when macOS system proxies are not explicitly configured for terminal Git. Preserve existing commits and remotes.
---

# Git push network recovery

恢复用户已授权的现有提交 push；不顺带推进项目 TODO、创建提交或修改历史。此 skill 不额外授权 push；当前请求已明确授权时直接沿用，不重复询问。

## 1. 固定操作对象

读适用 AGENTS.md，确认实际仓库、目标 branch/remote；nested repo 不要在父仓库操作。检查：

```bash
git status --short
git branch --show-current
git remote -v
git rev-parse HEAD
git rev-list --left-right --count origin/<branch>...HEAD
```

替换为实际目标；上面计数为 behind、ahead。记录 HEAD，检查 remote URL 时隐藏可能嵌入的凭据。若跟踪引用缺失先查明，不能猜测分支。dirty tree 先报告，不 stash、清理或覆盖；遵守该项目的 clean 要求。禁止 amend、rebase、reset、force push、改 remote 或创建“修复提交”。

## 2. 脱敏代理诊断

检查以下来源；输出先在内存中过滤，再展示。不要直接将原始结果写日志：

- shell：相当于 `env | grep -i proxy`，包括大小写 HTTPS_PROXY、ALL_PROXY、NO_PROXY。
- repo 中：`git config --show-origin --get-regexp 'http\..*proxy|https\..*proxy'`。
- global：`git config --global --get http.proxy` 和 `git config --global --get https.proxy`。无匹配的 exit 1 表示未设置。
- macOS：`scutil --proxy`，读取 HTTP/HTTPS/SOCKS 的 Enable、host、port，以及 PAC/auto-discovery。

隐藏 URL userinfo、token、密码、敏感 query、认证 header/cookie。不要打印完整环境、credential helper 输出或原始 Git trace。Git HTTPS 通常使用 `http.proxy`；不能仅检查 `https.proxy`。

只使用当前真实启用的地址/端口，不能猜 7890/7897。PAC URL 不是代理 endpoint；只有 PAC 时不能把 PAC URL 填入 HTTPS_PROXY，也不能随意执行下载的 PAC。没有可确认 endpoint 时报告缺口。

系统代理已启用且 env/Git 均未配置，只能说明“未显式配置”，还需连接证据；VPN/TUN 或透明路由可能影响实际路径。

## 3. 网络对照

在目标仓库有界运行：

```bash
curl -Iv --connect-timeout 10 --max-time 20 https://github.com
git ls-remote origin refs/heads/<branch>
```

Git 查询用进程级 timeout（例如 30 秒）；macOS 未必有 `timeout`，可用 Python subprocess timeout。只展示退出码、必要连接目标、HTTP 状态和目标 ref；过滤 verbose 中认证/cookie。curl 成功不等于 Git transport 稳定。

对照直接连接与显式系统代理的同样查询。检查 NO_PROXY 是否绕过 GitHub、URL-specific Git proxy 是否覆盖环境。若 Git 已有代理，不能声称本轮测量是直连。

## 4. 恢复顺序

已在当前会话失败过的步骤不必机械重做：

1. 正常 `git push origin <branch>`。
2. HTTP/2、reset 等临时错误，试单次 `git -c http.version=HTTP/1.1 push origin <branch>`。
3. 若系统启用代理而终端未使用，根据真实 endpoint 做**一次性**代理查询，`ls-remote` 成功后再 push。
   - 普通 HTTP CONNECT 代理承载 HTTPS 目标时：`HTTPS_PROXY=http://HOST:PORT`，必要时同时设置小写 `https_proxy`；不要因目标是 HTTPS 就误用 `https://` 代理协议。
   - 确认 SOCKS5 时可用 `ALL_PROXY=socks5h://HOST:PORT`，保留代理端 DNS。不要未经证据猜 SOCKS 版本。
   - 可用单次 `git -c http.proxy=PROXY_URL ...`；必要时组合一次性 HTTP/1.1。
   - host/port 从 `scutil` 解析后传进 subprocess 的 env/参数数组，避免 shell 拼接；IPv6 host 需要方括号。
4. 仅临时网络错误重试：建议退避 5/15/30 秒，总 push 尝试最多 4 次，每次进程超时 90 秒；用户另有明确预算则服从。耗尽后报告，不无限重试。

不永久写 global/local proxy，不改系统网络设置，不关闭 TLS 校验，不自动切 SSH。代理需要凭据时，不把凭据放进命令行、文件或报告。

出现 authentication failure、permission denied、repository not found、non-fast-forward、branch/remote 错误、明确 credential failure，立即停止该恢复流程并报告；这些不是可盲重试的网络错误。

## 5. 成功判据

push 返回成功后，通过同一有效连接方式查询目标远端 ref，再核对：

```bash
git rev-parse HEAD origin/<branch>
git rev-list --left-right --count origin/<branch>...HEAD
git status
```

要求服务器目标 ref、本地 HEAD、remote-tracking ref 一致；HEAD 与开始时记录一致；计数 0/0；working tree clean。若 push 响应超时/丢失，先 ls-remote 判断是否实际成功，不能仅凭旧 tracking ref 下结论。必要时同一路径 fetch 刷新跟踪引用，不 merge/rebase。

报告真实生效路径、commit、branch、计数和状态，以及是否改过永久配置。保留“直连偶尔成功、Git 直连超时、代理路径成功”等差异，不把一次成功夸大为永久网络稳定。
