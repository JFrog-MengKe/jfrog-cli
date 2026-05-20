# AuthFollow302 本地构建与使用说明 / Local Build & Usage Guide

[中文](#中文) · [English](#english)

---

<a id="中文"></a>

## 简介

本文档说明如何从 GitHub 拉取 **AuthFollow302** 分支、本地编译 `jf`，以及 **302 跨域重定向时保留认证头** 功能的配置与代码位置。

该功能通过环境变量启用，并配合主机白名单，解决典型场景：nginx 将 `mengk.jfrogchina.com` 302 到 `demo.jfrogchina.com` 后，Go 默认重定向会丢弃 `Authorization`，导致 `jf rt curl` / `jf rt dl` / API 调用返回 401。

---

## 1. 涉及仓库与分支

| 仓库 | 默认主分支 | 功能分支 | 远端地址 |
|------|------------|----------|----------|
| `jfrog-client-go` | `master` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-client-go.git` |
| `jfrog-cli-artifactory` | `main` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-cli-artifactory.git` |
| `jfrog-cli` | `master` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-cli.git` |

**说明：** `jfrog-client-go` 的 `AuthFollow302` 基于 fork 的 `2813` 分支（含 lifecycle 等其它提交），重定向相关改动在提交 `b4371de1`。**编译 `jf` 时必须** 三个仓库目录同级，且 `jfrog-cli/go.mod` 使用 sibling `replace`（该分支已包含）。

与 **npm ETARGET / Curation 提示** 相关的开发在独立分支 `curation-audit-npm`，**不在** `AuthFollow302` 范围内。

---

## 2. 从远端 clone 代码

建议在同一父目录下克隆三个仓库（例如 `~/Documents/CLI`）：

```bash
export WORKDIR=~/Documents/CLI
mkdir -p "$WORKDIR"
cd "$WORKDIR"

git clone https://github.com/JFrog-MengKe/jfrog-client-go.git
git clone https://github.com/JFrog-MengKe/jfrog-cli-artifactory.git
git clone https://github.com/JFrog-MengKe/jfrog-cli.git

cd jfrog-client-go && git checkout AuthFollow302 && cd ..
cd jfrog-cli-artifactory && git checkout AuthFollow302 && cd ..
cd jfrog-cli && git checkout AuthFollow302 && cd ..
```

目录结构应为：

```text
CLI/
├── jfrog-client-go/          # AuthFollow302
├── jfrog-cli-artifactory/    # AuthFollow302
└── jfrog-cli/                # AuthFollow302（含 go.mod replace）
```

若已 clone，仅切换分支：

```bash
git -C jfrog-client-go fetch origin && git -C jfrog-client-go checkout AuthFollow302
git -C jfrog-cli-artifactory fetch origin && git -C jfrog-cli-artifactory checkout AuthFollow302
git -C jfrog-cli fetch origin && git -C jfrog-cli checkout AuthFollow302
```

确认 `jfrog-cli/go.mod` 末尾存在（`AuthFollow302` 分支已提交）：

```go
replace github.com/jfrog/jfrog-cli-artifactory => ../jfrog-cli-artifactory
replace github.com/jfrog/jfrog-client-go => ../jfrog-client-go
```

---

## 3. 环境要求

| 项 | 要求 |
|----|------|
| Go | `jfrog-cli` 需 **Go 1.26.3+**（见 `jfrog-cli/go.mod`）；子模块可能声明 1.25.x，以 CLI 为准 |
| 网络 | 能访问 `proxy.golang.org` 或配置企业 `GOPROXY` |
| 系统 curl | `jf rt curl` 会调用系统 `curl`；跨域重定向需 curl 支持 `-L` / `--location-trusted` |
| JFrog CLI 配置 | 已 `jf c add` 配置 Artifactory 实例（URL、用户、API Key / Access Token） |

可选：使用代理拉取 Go 模块（按本机环境调整）：

```bash
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
```

---

## 4. 本地构建命令

在 **`jfrog-cli` 根目录** 执行：

```bash
cd "$WORKDIR/jfrog-cli"

go mod tidy
go build -o jf .
```

成功后可执行：

```bash
./jf --version
```

将二进制加入 PATH（可选）：

```bash
cp ./jf /usr/local/bin/jf   # 或任意目录
```

**不要** 在未改 `go.mod` 的情况下单独只编译 `jfrog-cli-artifactory` 或 `jfrog-client-go` 来替代 `jf`；可执行文件入口仅在 `jfrog-cli`。

---

## 5. 参数与使用方法

### 5.1 环境变量（必须同时设置才生效）

| 变量 | 取值 | 含义 |
|------|------|------|
| `JFROG_CLI_REDIRECT_FORWARD_HEADER` | `true`（大小写不敏感） | 开启「在 allowlisted 目标主机上重定向时转发客户端头（含认证）」 |
| `JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS` | 逗号分隔主机列表 | 重定向**目标**主机白名单 |

**两者缺一不可：** 仅设 `true` 而无白名单时，行为与未开启相同。

推荐示例（JFrog China 演示环境）：

```bash
export JFROG_CLI_REDIRECT_FORWARD_HEADER=true
export JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS="*.jfrogchina.com,demo.jfrogchina.com"
```

白名单规则（实现在 `redirect_policy.go`）：

- **精确匹配**：`demo.jfrogchina.com`
- **子域通配**：`*.jfrogchina.com` 匹配 `foo.jfrogchina.com`，**不匹配** apex `jfrogchina.com`
- 最多跟随 **10** 次重定向

### 5.2 验证是否生效

**不要用** `ping`（匿名接口）。用需认证的 System API：

```bash
# 未开启环境变量：跨域 302 后可能 401
jf rt curl -XGET /api/system/version

# 开启环境变量 + 白名单后：应返回 200 与版本 JSON
export JFROG_CLI_REDIRECT_FORWARD_HEADER=true
export JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS="*.jfrogchina.com,demo.jfrogchina.com"
jf rt curl -XGET /api/system/version
```

其它常用命令（同样走 `jfrog-client-go` HTTP 客户端或 `rt curl`）：

```bash
# 下载（POST 重定向场景会走 client-go 自实现 redirect，避免丢 body）
jf rt dl <repo-path> <target>

# 显式 rt curl（未传 -L 时 CLI 会自动补 -L --location-trusted）
jf rt curl -XGET /api/storage/<repo>/<path>
```

### 5.3 行为摘要

| 场景 | 未开启 | 开启且目标在白名单 |
|------|--------|-------------------|
| GET 等跟随重定向 | 标准 Go：跨 host 去掉 `Authorization` | 重新附加 API Key / Token / User-Pass |
| POST 302（如部分 AQL/上传） | 可能丢认证或 body 处理不当 | 通过 `Send()` 重发 POST；非白名单目标则去掉凭证 |
| `jf rt curl` | 系统 curl 默认不保留跨 host 的 Authorization | 自动前置 `-L --location-trusted`（若参数里尚无 `-L`） |
| 重定向目标不在白名单 | — | 与标准 HTTP 一致，**不**转发认证头 |

调试时可看日志中的 `JFrog CLI redirect policy:` 行（Debug 级别）。

---

## 6. 代码改动位置与细节

### 6.1 `jfrog-client-go`（核心 HTTP 行为）

**提交：** `b4371de1` — `feat(http): forward auth headers on redirect for allowlisted hosts`

| 文件 | 改动要点 |
|------|----------|
| `http/httpclient/redirect_policy.go` | **新文件**。读取上述两个环境变量；`parseAllowedHosts` / `isHostInAllowlist`；`allowlistCheckRedirect` 在白名单 host 上调用 `setAuthentication` + `copyHeaders`；`httpClientsDetailsWithoutCredentials` 用于非白名单 POST 重定向 |
| `http/httpclient/redirect_policy_test.go` | **新文件**。白名单解析、通配、开关逻辑单测 |
| `http/httpclient/client.go` | `doRequest`：在 `followRedirect && forwardRedirectHeaders` 时克隆 client 并设置 `CheckRedirect = allowlistCheckRedirect`；**POST 302** 改为 `jc.Send(POST, ...)` 且按白名单决定是否 strip 凭证；`UploadFileFromReader` 同样挂载 allowlist redirect |

关键 API：

```go
func RedirectForwardHeaderEnabled() bool  // 供 artifactory 等模块查询是否开启
```

**分支基底说明：** 远端 `AuthFollow302` 还包含 `2813` 上的 `lifecycle` 等提交（如 `39069157`），与重定向无关；若只要最小重定向 diff，可对 `origin/master..b4371de1` 做 review。

### 6.2 `jfrog-cli-artifactory`（`jf rt curl`）

**提交：** `78fe4a9` — `feat(rt curl): follow cross-host redirects when redirect forwarding is enabled`

| 文件 | 改动要点 |
|------|----------|
| `artifactory/commands/curl/curl.go` | 新增 `PrependFollowRedirectFlagsForCurl`：当 `httpclient.RedirectForwardHeaderEnabled()` 为真且用户未传 `-L` / `--location` 时，在参数前插入 `-L`、`--location-trusted` |
| `artifactory/cli/cli.go` | `newRtCurlCommand` 中先 `PrependFollowRedirectFlagsForCurl(common.ExtractCommand(c))` 再构造 `CurlCommand` |

系统 `curl` 不经过 Go 的 `CheckRedirect`，因此需要单独加 curl 参数。

### 6.3 `jfrog-cli`（串联本地模块）

**提交：** `60055c45` — `build: point jfrog-cli-artifactory and jfrog-client-go at sibling modules`

| 文件 | 改动要点 |
|------|----------|
| `go.mod` | 增加 `replace` 指向 `../jfrog-cli-artifactory` 与 `../jfrog-client-go`，使 `go build` 使用本地 AuthFollow302 源码而非 registry 版本 |

**注意：** 提交 PR 到上游前通常应移除 sibling `replace`，改为依赖已发布的 module 版本或 `go get` 指定 commit。

---

## 7. 常见问题

**Q: 开了环境变量仍然 401？**  
- 确认重定向**落地 host** 在白名单内（可用 Debug 日志查看实际 host）。  
- `jf rt curl` 确认未手动去掉 `-L` 且系统 curl 支持 `--location-trusted`。  
- 用 `/api/system/version` 验证，不要用 `ping`。

**Q: 与 `curation-audit-npm` 分支关系？**  
- `curation-audit-npm`：`jf npm i` / `jf ca` 的 ETARGET 版本提示，独立分支。  
- `AuthFollow302`：仅 HTTP 302 认证转发 + `rt curl` 参数，二者可分别 checkout 编译；若同时要测 npm 提示，需在 `go.mod` 中自行增加对 `curation-audit-npm` 的 replace（本分支默认不包含）。

**Q: 如何更新到最新远端 AuthFollow302？**

```bash
for d in jfrog-client-go jfrog-cli-artifactory jfrog-cli; do
  git -C "$WORKDIR/$d" pull origin AuthFollow302
done
cd "$WORKDIR/jfrog-cli" && go mod tidy && go build -o jf .
```

---

## 8. 参考链接

- `jfrog-client-go` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-client-go/tree/AuthFollow302  
- `jfrog-cli-artifactory` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-cli-artifactory/tree/AuthFollow302  
- `jfrog-cli` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-cli/tree/AuthFollow302  

---

<a id="english"></a>

## Introduction

This guide explains how to clone the **AuthFollow302** branches from GitHub, build `jf` locally, and configure **preserving authentication headers on cross-host HTTP 302 redirects**, including where the code lives.

The feature is controlled by environment variables plus a host allowlist. It addresses cases such as nginx redirecting `mengk.jfrogchina.com` to `demo.jfrogchina.com`, where Go’s default redirect handling drops `Authorization`, causing `jf rt curl`, `jf rt dl`, and other API calls to return 401.

---

## 1. Repositories and branches

| Repository | Default branch | Feature branch | Remote |
|------------|----------------|----------------|--------|
| `jfrog-client-go` | `master` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-client-go.git` |
| `jfrog-cli-artifactory` | `main` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-cli-artifactory.git` |
| `jfrog-cli` | `master` | `AuthFollow302` | `https://github.com/JFrog-MengKe/jfrog-cli.git` |

**Note:** `jfrog-client-go`’s `AuthFollow302` is based on the fork’s `2813` branch (includes unrelated commits such as lifecycle). Redirect changes are in commit `b4371de1`. To build `jf`, all three repos must be **sibling directories**, and `jfrog-cli/go.mod` must use local `replace` directives (already on this branch).

**npm ETARGET / Curation hints** live on branch `curation-audit-npm`, **not** on `AuthFollow302`.

---

## 2. Clone from remote

Clone all three repos under one parent directory (e.g. `~/Documents/CLI`):

```bash
export WORKDIR=~/Documents/CLI
mkdir -p "$WORKDIR"
cd "$WORKDIR"

git clone https://github.com/JFrog-MengKe/jfrog-client-go.git
git clone https://github.com/JFrog-MengKe/jfrog-cli-artifactory.git
git clone https://github.com/JFrog-MengKe/jfrog-cli.git

cd jfrog-client-go && git checkout AuthFollow302 && cd ..
cd jfrog-cli-artifactory && git checkout AuthFollow302 && cd ..
cd jfrog-cli && git checkout AuthFollow302 && cd ..
```

Expected layout:

```text
CLI/
├── jfrog-client-go/          # AuthFollow302
├── jfrog-cli-artifactory/    # AuthFollow302
└── jfrog-cli/                # AuthFollow302 (go.mod replace)
```

If already cloned, switch branches only:

```bash
git -C jfrog-client-go fetch origin && git -C jfrog-client-go checkout AuthFollow302
git -C jfrog-cli-artifactory fetch origin && git -C jfrog-cli-artifactory checkout AuthFollow302
git -C jfrog-cli fetch origin && git -C jfrog-cli checkout AuthFollow302
```

Confirm the end of `jfrog-cli/go.mod` on `AuthFollow302`:

```go
replace github.com/jfrog/jfrog-cli-artifactory => ../jfrog-cli-artifactory
replace github.com/jfrog/jfrog-client-go => ../jfrog-client-go
```

---

## 3. Prerequisites

| Item | Requirement |
|------|-------------|
| Go | **Go 1.26.3+** for `jfrog-cli` (see `jfrog-cli/go.mod`); submodules may list 1.25.x—follow CLI |
| Network | Access to `proxy.golang.org` or a corporate `GOPROXY` |
| System curl | `jf rt curl` invokes system `curl`; cross-host redirects need `-L` / `--location-trusted` |
| JFrog CLI config | Artifactory configured via `jf c add` (URL, user, API Key / Access Token) |

Optional proxy for module download:

```bash
export https_proxy=http://127.0.0.1:7897
export http_proxy=http://127.0.0.1:7897
```

---

## 4. Local build

From the **`jfrog-cli` repository root**:

```bash
cd "$WORKDIR/jfrog-cli"

go mod tidy
go build -o jf .
```

Verify:

```bash
./jf --version
```

Optional install to PATH:

```bash
cp ./jf /usr/local/bin/jf
```

Do **not** build only `jfrog-cli-artifactory` or `jfrog-client-go` and expect a full `jf` binary—the entry point is `jfrog-cli` only.

---

## 5. Configuration and usage

### 5.1 Environment variables (both required)

| Variable | Value | Meaning |
|----------|-------|---------|
| `JFROG_CLI_REDIRECT_FORWARD_HEADER` | `true` (case-insensitive) | Enable forwarding client headers (including auth) on redirects to allowlisted hosts |
| `JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS` | Comma-separated hosts | Allowlist for redirect **target** hosts |

If only `true` is set without a non-empty allowlist, behavior matches the feature **disabled**.

Example (JFrog China demo):

```bash
export JFROG_CLI_REDIRECT_FORWARD_HEADER=true
export JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS="*.jfrogchina.com,demo.jfrogchina.com"
```

Allowlist rules (`redirect_policy.go`):

- **Exact:** `demo.jfrogchina.com`
- **Wildcard subdomain:** `*.jfrogchina.com` matches `foo.jfrogchina.com`, **not** apex `jfrogchina.com`
- Up to **10** redirect hops

### 5.2 Verify it works

Do **not** use `ping` (unauthenticated). Use an authenticated System API:

```bash
# Without env vars: cross-host 302 may yield 401
jf rt curl -XGET /api/system/version

# With env vars + allowlist: expect 200 and version JSON
export JFROG_CLI_REDIRECT_FORWARD_HEADER=true
export JFROG_CLI_REDIRECT_AUTH_ALLOWED_HOSTS="*.jfrogchina.com,demo.jfrogchina.com"
jf rt curl -XGET /api/system/version
```

Other commands:

```bash
jf rt dl <repo-path> <target>
jf rt curl -XGET /api/storage/<repo>/<path>
```

### 5.3 Behavior summary

| Scenario | Disabled | Enabled, target on allowlist |
|----------|----------|------------------------------|
| GET + redirects | Standard Go: strips `Authorization` on cross-host | Re-applies API Key / Token / basic auth |
| POST 302 (AQL, uploads, etc.) | May lose auth or mishandle body | Retries via `Send()`; strips creds if target not allowlisted |
| `jf rt curl` | curl drops cross-host `Authorization` | Prepends `-L --location-trusted` when `-L` not already present |
| Target not on allowlist | — | Standard HTTP: **no** auth forwarding |

Look for `JFrog CLI redirect policy:` in Debug logs.

---

## 6. Code changes

### 6.1 `jfrog-client-go` (core HTTP)

**Commit:** `b4371de1` — `feat(http): forward auth headers on redirect for allowlisted hosts`

| File | Change |
|------|--------|
| `http/httpclient/redirect_policy.go` | **New.** Env vars, allowlist parsing, `allowlistCheckRedirect`, credential stripping for non-allowlisted POST redirects |
| `http/httpclient/redirect_policy_test.go` | **New.** Unit tests |
| `http/httpclient/client.go` | `doRequest` + `UploadFileFromReader`: custom `CheckRedirect`; POST 302 via `jc.Send()` |

```go
func RedirectForwardHeaderEnabled() bool
```

Remote `AuthFollow302` may also include `2813` commits (e.g. `39069157` lifecycle)—unrelated to redirects. Minimal redirect diff: `origin/master..b4371de1`.

### 6.2 `jfrog-cli-artifactory` (`jf rt curl`)

**Commit:** `78fe4a9` — `feat(rt curl): follow cross-host redirects when redirect forwarding is enabled`

| File | Change |
|------|--------|
| `artifactory/commands/curl/curl.go` | `PrependFollowRedirectFlagsForCurl` adds `-L --location-trusted` |
| `artifactory/cli/cli.go` | Wires prepend into `newRtCurlCommand` |

System `curl` does not use Go’s `CheckRedirect`; extra flags are required.

### 6.3 `jfrog-cli` (wire local modules)

**Commit:** `60055c45` — `build: point jfrog-cli-artifactory and jfrog-client-go at sibling modules`

| File | Change |
|------|--------|
| `go.mod` | `replace` to sibling repos for local AuthFollow302 sources |

Remove sibling `replace` before upstream PR; use published modules or `go get` at a specific commit.

---

## 7. FAQ

**Q: Still getting 401 with env vars set?**  
- Confirm the redirect **landing host** is allowlisted (Debug logs show the host).  
- For `jf rt curl`, ensure `-L` is not removed and curl supports `--location-trusted`.  
- Validate with `/api/system/version`, not `ping`.

**Q: Relation to `curation-audit-npm`?**  
- `curation-audit-npm`: ETARGET hints for `jf npm i` / `jf ca`.  
- `AuthFollow302`: HTTP 302 auth forwarding only. Add a `replace` for security/artifactory on `curation-audit-npm` if you need both.

**Q: Update to latest remote `AuthFollow302`?**

```bash
for d in jfrog-client-go jfrog-cli-artifactory jfrog-cli; do
  git -C "$WORKDIR/$d" pull origin AuthFollow302
done
cd "$WORKDIR/jfrog-cli" && go mod tidy && go build -o jf .
```

---

## 8. Links

- `jfrog-client-go` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-client-go/tree/AuthFollow302  
- `jfrog-cli-artifactory` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-cli-artifactory/tree/AuthFollow302  
- `jfrog-cli` AuthFollow302: https://github.com/JFrog-MengKe/jfrog-cli/tree/AuthFollow302  
