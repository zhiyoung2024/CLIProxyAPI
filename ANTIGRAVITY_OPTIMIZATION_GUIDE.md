# CLIProxyAPI 生产级改造与 Antigravity 全阶梯思考优化全景复盘指南

本项目是基于官方上游 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) 进行企业级深度调优与扩展的高可用分支。本文档全面复盘并解析我们在生产环境中遇到的高并发断流、长推理超时、Google 上游思维链丢失以及多账号治理等核心痛点，并提供由浅入深的底层原理解析与端到端实战解决方案。

---

## 一、改造背景与四大核心痛点分析

在将 CLIProxyAPI 部署并接入 Antigravity 渠道（承载 `gemini-4-argon` 系列、`gemini-3.8-flash` 系列、`claude-opus-4-6`、`claude-sonnet-4-6` 等高阶大模型）后，高并发与深度思考场景下暴露出了四大底层缺陷：

```
                    ┌────────────────────────────────────────────────────────┐
                    │               客户端 (Web / IDE / SDK)                 │
                    └──────────────────────────┬─────────────────────────────┘
                                               │ HTTPS / SSE
                                               ▼
                    ┌────────────────────────────────────────────────────────┐
                    │                   Nginx 反向代理层                     │
                    │  [痛点 1] 默认 60s 超时 -> HTTP 499 / Connection error  │
                    │  [修复 1] proxy_read_timeout 600s + proxy_buffering off│
                    └──────────────────────────┬─────────────────────────────┘
                                               │ HTTP/1.1 (Keepalive)
                                               ▼
                    ┌────────────────────────────────────────────────────────┐
                    │                    CLIProxyAPI 核心                    │
                    │  [痛点 2] 思考长间隙空闲 -> 代理层断连               │
                    │  [修复 2] streaming.keepalive-seconds: 15 (SSE 心跳)  │
                    │  [痛点 4] payload 默认覆盖错误 (First-Write-Wins)      │
                    │  [修复 4] -low/-medium 精确优先阶梯式注入思维预算     │
                    └──────────────┬──────────────────────────┬──────────────┘
                                   │                          │
               [痛点 3: 物理端点屏蔽 thought]                 │
               [修复 3: Excluded + Alias 劫持]                │
                                   ▼                          ▼
                    ┌─────────────────────────┐  ┌───────────────────────────┐
                    │  OAuth 多账号负载均衡池 │  │ Google v1internal / API   │
                    │  [痛点 5] 脏文件 / 403  │  │ Argon 要求 Budget>=16384  │
                    │  [修复 5] 单凭据模型隔离│  │ 否则静默剥离思考链内容    │
                    └─────────────────────────┘  └───────────────────────────┘
```

### 1. 痛点一：Nginx 默认 60s 超时与 HTTP 499 连接重置
- **现象**：客户端在调用高思考深度的 Gemini 4 Argon 或 Claude Opus 时，常常在等待 60 秒整后突然弹出 `Error: Connection error.`，Nginx access.log 记录响应状态码为 `499` 或 `504`。
- **根因分析**：
  Nginx 作为反向代理，默认的 `proxy_read_timeout` 和 `proxy_send_timeout` 均为 60 秒。深度思考模型在生成最终答案前，前置思考耗时可能长达数十秒甚至上百秒；若在此期间没有 TCP 报文或未禁用缓冲，Nginx 会判定后端已无响应，主动切断连接并向客户端返回 499/504。
- **解决方案**：
  在 Nginx location 块中设置 `proxy_read_timeout 600s;`、`proxy_send_timeout 600s;`，并关闭代理缓冲区 `proxy_buffering off;`，配合 `chunked_transfer_encoding on;` 确保流式分块无延迟直通。

### 2. 痛点二：SSE 长连接静默超时与中间链路切断
- **现象**：即使调大了 Nginx 超时，部分经过云服务商 NAT 网关、Cloudflare 或移动宽带的客户端依然会遭遇不定时断流。
- **根因分析**：
  在模型生成首个 Token 或两段 Token 输出间隔较长时，传输链路处于完全空闲状态。许多中间代理与防火墙在 15~30 秒无任何流量时会强行断开闲置 TCP 会话。
- **解决方案**：
  在 `config.yaml` 中配置 `streaming.keepalive-seconds: 15` 与 `nonstream-keepalive-interval: 15`。CLIProxyAPI 在流式处于静默期时，每隔 15 秒向客户端下发标准的 SSE 注释行（`: keep-alive\n\n`），有效维持整条传输通路的活跃状态。

### 3. 痛点三：Google `v1internal` 上游对 `-low` / `-medium` 物理端点屏蔽 `thought` 回传
- **现象**：客户端请求 `gemini-4-argon-low` 或 `gemini-3.8-flash-low` 时，即使请求体携带了 `includeThoughts: true`，上游返回的数据包中依然缺失思考过程，思维链被完全抹除。
- **根因分析**：
  Google Antigravity 渠道上游的 `v1internal` 后端服务存在硬编码策略：对于 `-low`、`-medium` 物理命名的模型端点，服务端统一丢弃并剥离 `thought` 字段，只有对完整版物理端点（如 `gemini-4-argon-high`、`gemini-3.8-flash-high`）才放行完整的思维链回传。
- **解决方案（独创模型排除 + 同名别名接管机制）**：
  1. **物理模型排除**：在 `oauth-excluded-models.antigravity` 中排除 `gemini-4-argon-low`、`gemini-4-argon-medium`、`gemini-3.8-flash-low`，阻止系统直接向这些物理端点发包；
  2. **同名别名劫持**：在 `oauth-model-alias.antigravity` 中定义同名别名（`fork: true`），将客户端传来的 `-low`、`-medium` 模型名原样拦截，并在协议层映射到支持完整思维链的 `-high` 物理端点；
  3. **分档预算注入**：结合 `payload.default` 机制，给不同的子模型分别注入各自对应的阶梯预算。

### 4. 痛点四：`gemini-4-argon-trial` 存在 16384 思维临界阈值与 `First-Write-Wins` 覆盖陷阱
- **现象**：
  - 测试发现，为 Argon 设置 8192 或更小的 thinkingBudget 时，思考内容依然无法输出；
  - 在配置多组规则时，子模型的配置常常失效，全部变成了默认模型的参数。
- **根因分析**：
  - **阈值约束**：Google 针对 `gemini-4-argon-trial` 设有强门槛约束：`thinkingBudget` 必须大于或等于 `16384` 才会回显思索过程，低于此阈值会被后端忽略；
  - **First-Write-Wins 规则**：CLIProxyAPI 的 `payload.default` 匹配引擎采用“先命中先写入”机制。如果将通配规则或默认规则置于前面，后续针对子模型（如 `-low`）的细化规则将永远无法生效！
- **解决方案**：
  在 `payload.default` 中，严格遵循**先精确后通用、先具体子模型后父模型**的书写顺序：
  - `gemini-4-argon-low`: 16384 (踩准触发临界线)
  - `gemini-4-argon-medium`: 24576
  - `gemini-4-argon-high` / `gemini-4-argon`: 32768 (满血思考)
  - `gemini-3.8-flash-low`: 1000
  - `gemini-3.8-flash-medium`: 4000
  - `gemini-3.8-flash-high` / `gemini-3.8-flash`: 16000

---

## 二、多账号池治理与单账号模型级隔离

在高负载企业部署环境下，管理数十个 Google Antigravity OAuth 账号时，常规轮询经常由于脏文件或个别账号受限导致全池崩溃。

### 1. macOS 隐藏元数据文件清理
在从 macOS 部署或传输文件至 Linux 宿主机时，会隐式生成类似 `._antigravity-user@example.com.json` 的 AppleDouble 隐藏文件。CLIProxyAPI 在加载凭据目录时若读取此类无效 JSON，会频繁在日志输出解析错误甚至导致凭证加载中断。
- **运维规范**：部署前统一执行 `find /root/.cli-proxy-api/ -name "._*" -delete`。

### 2. 账号安全风控快速剔除 (`disabled: true`)
当某账号在上游触发手机号校验或环境风控，返回 `403 VALIDATION_REQUIRED` 时，如果不断重试会累积冷却惩罚。
- **处置方案**：在对应 JSON 中添加 `"disabled": true`，无需物理删除凭据，代理层会自动跳过该账号，保留原有 Refresh Token 以便后续恢复。

### 3. 单账号模型级精准隔离 (`attributes.excluded_models`)
- **场景**：Google 账号的 `gemini-4-argon` 周配额相对有限，但 `gemini-3.8-flash` 配额极为宽裕。若某个账号的 Argon 配额已耗尽，盲目直接停用该账号会导致 Flash 算力的极大浪费！
- **实战方案**：
  在单个账号凭据的 `attributes` 字段中注入 `excluded_models`：
  ```json
  {
    "attributes": {
      "excluded_models": "gemini-4-argon*,gemini-3.8-flash-low,chat_20706,chat_23310,gemini-2.5-flash-lite,gemini-2.5-pro,gemini-2.5-flash"
    }
  }
  ```
  此时该账号仅承接 Flash 流量，Argon 流量自动导向其他充裕账号，实现单凭证细粒度算力最大化利用。

---

## 三、生产环境快速部署实战

### 1. 目录结构规划
```text
/opt/CLIProxyAPI/
├── cli-proxy-api             # 编译后的二进制可执行程序
├── config.yaml               # 生产环境运行配置
├── models_catalog.json       # 挂载的 Antigravity 35+ 全量模型目录
└── logs/                     # 运行日志目录

/root/.cli-proxy-api/         # OAuth 凭据目录 (chmod 700)
├── antigravity-user1.json
└── antigravity-user2.json
```

### 2. 快速启动与服务托管
使用本仓库提供的 `deploy/cliproxyapi.service`：
```bash
sudo cp deploy/cliproxyapi.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cliproxyapi
sudo systemctl status cliproxyapi
```

### 3. 配置 Nginx 反向代理
参考 `deploy/nginx-cliproxyapi.conf` 部署至 `/etc/nginx/sites-available/` 并建立软链接：
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 四、核心配置对照矩阵

| 模型名 (Model ID) | 客户端请求体验 | 内部路由映射端点 | 注入思考预算 (Budget) | includeThoughts |
| :--- | :--- | :--- | :--- | :--- |
| `gemini-4-argon-low` | 轻度思考 (快速响应) | `gemini-4-argon-high` | `16384` | `true` |
| `gemini-4-argon-medium` | 中度思考 (平衡型) | `gemini-4-argon-high` | `24576` | `true` |
| `gemini-4-argon-high` | 满血深度思考 | `gemini-4-argon-high` | `32768` | `true` |
| `gemini-4-argon` | 默认别名 -> 满血 | `gemini-4-argon-high` | `32768` | `true` |
| `gemini-3.8-flash-low` | 极速思考 | `gemini-3.8-flash-high` | `1000` | `true` |
| `gemini-3.8-flash-medium` | 适度思考 | `gemini-3.8-flash-high` | `4000` | `true` |
| `gemini-3.8-flash-high` | 全速思考 | `gemini-3.8-flash-high` | `16000` | `true` |
| `gemini-3.8-flash` | 默认别名 -> 全速 | `gemini-3.8-flash-high` | `16000` | `true` |
