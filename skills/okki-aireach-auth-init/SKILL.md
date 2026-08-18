---
name: okki-aireach-auth-init
description: 检查并初始化 AiReach 运行所需的 OKKI 授权状态。
---
# AiReach Auth Init

检查本地授权状态，并确保当前可用凭证与**目标 gateway 环境**匹配后，再继续 AiReach 业务调用。

## 触发场景

- 用户提到“登录 AiReach”“配置 AiReach”“AiReach 授权”。
- 需要调用依赖 OKKI 授权的 AiReach 受保护能力。
- 业务 skill 在执行前发现本地凭证为空。
- 首次启动 `OKKI AiReach 助手`，需要做最小认证检查。
- 最近一次 AiReach 受保护调用返回了认证相关错误，例如：
  - `authorization_required`
  - `invalid_token`
  - `token_revoked`
  - `grant_revoked`
  - `token_expired`
  - `pat_revoked`
  - `pat_expired`

## 能力边界

- 本 skill 只处理 OKKI PAT 授权，不处理钉钉、多维表格、社媒账号或其他第三方登录。
- 授权入口统一使用 `aireach-cli`。
- Accio Work 插件发起授权时，OAuth `client_id` 必须使用 `accio-work`。
- 在 Accio Work 插件运行时，`agent.json -> cliToolIds` 绑定 `aireach-cli`，平台应按 `clis/clis.json` 从 npm 包 `@okki-aireach/aireach-cli` 安装并把同名 bin 注入到宿主 `PATH`。
- 如果 bash 输出 `shell-init: error retrieving current directory: getcwd`，说明宿主当前工作目录不可访问；不要继续依赖相对路径或当前目录，先切到 `$HOME` 或 `/tmp` 后再运行授权命令。
- 默认本地凭证位置为 `~/.okki-agent/social-media/settings.json -> oauth.pat_token`。
- 不在 skill 包里固化租户绑定、shared secret、内部部署地址、环境专属 URL 或宿主绝对路径。

## 工作流

### 1. 检查基础入口是否可用

```bash
command -v aireach-cli
aireach-cli --help
aireach-cli auth --help
```

如果 `aireach-cli` 不存在：

> ⚠️ 检测到 `aireach-cli` 未安装或未暴露给当前宿主。请确认 Accio Work 已按 `clis/clis.json` 安装 `@okki-aireach/aireach-cli`，且 `agents/aireach/agent.json -> cliToolIds` 包含 `aireach-cli`。
>
> 在 CLI 未注入前，AiReach 授权无法继续；不要要求用户手动安装或粘贴 PAT。

### 2. 检查当前本地授权状态

```bash
aireach-cli auth status
```

重点检查：

- `success`
- `data.logged_in`
- `data.client_id`
- `data.settings_path`

注意：`auth status` 不应输出 PAT 原文；不要要求用户粘贴或展示 token。

### 3. 判断目标 gateway 环境

在执行任何业务调用前，先明确当前目标 gateway 环境：

- 用户或宿主显式指定 `dev` / `prod` 时，以显式指定为准。
- 业务调用使用 `--jsonrpc-url` 或 `AIREACH_JSONRPC_URL` 时，以该目标 endpoint 推断环境。
- 未显式指定时，按 `aireach-cli` 默认 prod gateway 行为处理。

凭证必须与目标 gateway 环境匹配。

### 4. 判断是否需要重新授权

出现以下任一情况时，应视为需要重新授权：

- `aireach-cli auth status` 显示 `logged_in=false`。
- `oauth.pat_token` 为空或本地 settings 文件不存在。
- 最近一次受保护请求返回认证失败。
- 当前本地凭证无法通过目标 gateway 的真实请求验证。

注意：

- 本地存在凭证不等于当前环境可用。
- 凭证必须与当前目标 gateway 环境匹配。
- 不要假设任意历史 PAT 都能跨环境复用。

### 5. 执行环境匹配的登录流程

当确认需要重新授权时，运行：

```bash
aireach-cli auth login --gateway-env <prod|dev> --client-id accio-work --timeout 30m --detach
```

该命令生成的授权 URL 必须包含：

```text
client_id=accio-work
```

要求：

- `<prod|dev>` 必须替换为当前目标 gateway 环境。
- `--client-id accio-work` 是 Accio Work 插件授权的必填参数；如果由宿主统一注入，也可以通过 `AIREACH_OAUTH_CLIENT_ID=accio-work` 达成同一效果，但最终 `authorization_url` 中必须是 `client_id=accio-work`。
- 让用户在浏览器中完成登录与 PAT 创建。
- `aireach-cli auth login --detach` 会先启动后台 callback helper，并在 helper 确认监听本地回调端口后输出 `authorization_url`、`callback_url`、`helper_pid`、`log_path` 等 JSON 数据；agent 必须提取并展示 `authorization_url`，让用户打开完成授权，然后轮询 `aireach-cli auth status` 确认结果。
- 不要在非流式 bash 工具里使用阻塞式 `aireach-cli auth login`；只有确认当前宿主能实时展示命令输出并允许用户在命令运行期间打开 URL 时，才可省略 `--detach`。
- `aireach-cli auth login` 默认不自动唤起浏览器；只有在明确需要并确认宿主支持时，才使用 `--open` 显式请求 CLI 在 callback helper ready 后尝试打开浏览器。
- 默认 callback 地址是 `http://127.0.0.1:3000/callback`。这是 OAuth 端注册的回调地址；不要为了绕过端口冲突随意传 `--callback-addr`，否则可能被 OAuth 端以 `callback_uri mismatch` 拒绝。只有在明确目标 OAuth 客户端允许该回调地址时，才使用 `--callback-addr 127.0.0.1:<free-port>`。
- 提示用户打开授权 URL 时，只展示授权页面链接，不要求用户复制、粘贴或展示 PAT 原文。
- 浏览器授权成功后会显示 `Accio Work OKKI AiReach 已授权成功`，并提示用户关闭页面、回到 Accio Work 继续使用 OKKI AiReach；如果用户刷新已完成的 callback 页面，helper 可能已经退出，后续 connection refused 不代表授权失败，应以 `aireach-cli auth status` 为准。
- 如果页面提示当前客户端可保留的活跃 PAT 数量已达上限，先撤销不再使用的 PAT，再重新发起整轮登录。
- 如果本地 callback 登录已超时，必须重新运行 `aireach-cli auth login --gateway-env <prod|dev> --client-id accio-work --detach` 以建立新的回调监听，不能继续复用旧链接。
- 如果 callback 端口被占用，应先释放端口或让用户处理本机端口冲突后重新运行登录命令；不要擅自修改 callback 参数。CLI 的端口占用错误会包含可查看的 callback URL 与 `log_path` 诊断信息。
- 如果出现 `3000` 端口被之前的 `aireach-cli auth login` 命令占用可以尝试 kill 掉

### 6. 登录后重新校验

登录流程结束后，必须重新执行：

```bash
aireach-cli auth status
```

确认：

- `success=true`
- `data.logged_in=true`
- `data.client_id` 为 `accio-work`
- `data.settings_path` 使用 `~/.okki-agent/social-media/settings.json` 或用户显式配置的相对/可移植路径

### 7. 真实业务调用验证

如果本轮授权是为了修复某个实际调用失败，还必须重新执行刚才被阻塞的那条 AiReach 受保护调用。

如果只是首次初始化授权，则执行一个最小只读 smoke，优先使用业务 skill 原本要执行的最小查询；不要为了授权检查引入额外写操作。

只有目标 gateway 的真实请求通过，才能视为授权已就绪。

## 完成标准

当以下条件全部满足时，返回“AiReach 授权已就绪”：

- `aireach-cli auth status` 显示本地凭证存在，且 `data.client_id=accio-work`。
- 该凭证与当前目标 gateway 环境匹配。
- 刚才被阻塞的真实 AiReach 调用已重新验证通过，或首次初始化的最小只读 smoke 已通过。

## 注意事项

1. 本 skill 只处理 OKKI 授权，不处理钉钉、多维表格或社媒账号登录。
2. 任何 callback 登录一旦超时，都必须重新建立监听，不能继续使用旧链接。
3. 不要把环境专属登录路径、调试地址、内部脚本路径或宿主绝对路径暴露到最终上线版 skill 文案里。
4. 认证是否真正可用，以**目标 gateway 的真实请求验证结果**为准，而不是只看本地是否有 token。
5. 不要打印、记录或要求用户粘贴 PAT 原文。
6. Accio Work 授权必须使用 `client_id=accio-work`；不要让 auth-init 依赖 CLI 默认 client id。
