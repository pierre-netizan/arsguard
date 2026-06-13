# arsguard — AI Agent Security Guard

arsguard 是专为 [OpenClaw](https://github.com/openclaw) 设计的安全加固插件，构建在 Ollama 模型服务与 Squid 反向代理之上，提供针对 OWASP Top 10 for AI Agents 攻击向量的实时检测与拦截能力。项目包含完整的部署脚本、85 项单元测试和与 eval 测试框架的深度集成。

## 架构概览

arsguard 采用多层防御架构，各层独立部署、协同工作。所有外部请求必须经过 Squid 代理才能到达 OpenClaw，arsguard 以插件形式嵌入 OpenClaw 请求链路，在请求进入模型前和响应返回前分别执行检查：

```
客户端 → Squid 反向代理 (:3128)
           ↓
         OpenClaw (SPA 网关)
           ↓
         arsguard 插件
           ├─ on_request()   → 10 个钩子检查请求
           ├─ on_response()  → 检查模型输出
           └─ on_error()     → 错误日志与告警
           ↓
         Ollama 模型 (qwen3-0.6b)
```

- **Squid 反向代理**：充当唯一入口，支持 HTTP/HTTPS（http 分支）和 WebSocket（ws 分支）两种模式，实现请求路由、协议过滤和连接管理
- **OpenClaw**：LLM 网关，提供 REST API 管理和 SPA 前端界面，支持多模型路由和会话管理
- **arsguard 插件**：通过 Python RPC 与 OpenClaw 进程通信，拦截所有出入流量，逐层调用安全钩子进行检查
- **Ollama**：本地模型推理服务，支持 qwen3-0.6b 等开源模型的加载与推理

## 安全钩子系统

arsguard 提供 10 个独立的检测钩子，每个对应 OWASP Top 10 for AI Agents 的一个风险类别。钩子采用正则表达式匹配 + 上下文评分双重检测机制，有效平衡检出率与误报率：

### LLM01 — 提示注入 (Prompt Injection)

检测恶意指令注入、角色劫持、越狱攻击和系统提示覆盖。覆盖的攻击模式包括：DAN 越狱、开发者模式、角色扮演劫持、提示泄露攻击、分隔符绕过等。

```yaml
llm01_prompt_injection:
  enabled: true
  mode: block
  patterns:
    - "\\bignore\\s+(all\\s+)?(previous|above|below)\\s+instructions\\b"
    - "\\bDAN\\b"
    - "\\bdo\\s+anything\\s+now\\b"
    - "\\byou\\s+are\\s+(now\\s+)?free\\b"
```

### LLM02 — 不安全输出 (Insecure Output)

检测模型输出中包含的敏感信息和可执行代码。覆盖的攻击模式包括：SQL 语句泄露、HTML/JavaScript 代码注入、SSRF 目标泄露、内部路径泄露、凭据泄露等。

```yaml
llm02_insecure_output:
  enabled: true
  mode: block
  patterns:
    - "\\b(?:SELECT|INSERT|UPDATE|DELETE|DROP)\\s+.*\\b(?:FROM|INTO|SET|WHERE)\\b"
    - "<script\\b[^>]*>.*</script>"
    - "\\b(?:http|https?)://\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}"
```

### LLM03 — 训练数据投毒 (Training Data Poisoning)

检测可能导致模型训练数据被污染的攻击。覆盖的攻击模式包括：错误数据注入建议、API Key 共享合理化、安全性无害化论述、不安全实践推荐等。

### LLM04 — 模型拒绝服务 (Model DoS)

检测可能导致模型资源耗尽的攻击。覆盖的攻击模式包括：递归/无限循环请求、穷举枚举攻击（如所有状态码、所有组合）、超大输入、计算密集型请求（如 Busy Beaver 问题）等。注意，`check_demo` 模式下跳过 DoS 检测以避免仪表盘请求被误拦截。

### LLM05 — 供应链漏洞 (Supply Chain)

检测与第三方组件和依赖相关的安全风险。覆盖的攻击模式包括：恶意包安装请求、依赖混淆、版本回滚攻击、未经验证的第三方代码执行等。

### LLM06 — 敏感信息泄露 (Sensitive Info Disclosure)

检测试图从模型中提取敏感信息的行为。覆盖的攻击模式包括：API Key 和令牌提取、密码和凭据获取、个人身份信息（PII）收集、内部网络拓扑探测等。

### LLM07 — 不安全插件设计 (Insecure Plugin)

检测对插件系统的滥用和工具误调用。覆盖的攻击模式包括：Shell 命令执行（exec、system、popen 等）、文件系统操作（读取 /etc/shadow、写入任意路径等）、环境变量泄露（SSH_KEY、AWS_SECRET 等）、函数重写（monkey-patch）、动态加载（dlopen、load .so）、新工具/插件注册等。

```yaml
llm07_insecure_plugin:
  enabled: true
  mode: block
  patterns:
    - "\\bexec(_?shell|_?cmd|_?command|ute)?\\b"
    - "\\bsubprocess\\b"
    - "\\bos\\.system\\b"
    - "\\bcat\\s+/etc/shadow\\b"
    - "\\bdlopen\\b"
```

### LLM08 — 过度代理权限 (Excessive Agency)

检测试图绕过权限控制的行为。覆盖的攻击模式包括：权限提升请求、管理员操作冒充、安全控制绕过、审计日志禁用等。

### LLM09 — 过度依赖 (Overreliance)

检测可能导致用户过度依赖模型输出的行为。覆盖的攻击模式包括：请求禁用安全警告、紧急/重要情况下忽略规则、怂恿信任模型输出的言论、将免责声明视为可忽略项等。

### LLM10 — 模型窃取 (Model Theft)

检测试图提取模型架构、参数或提示词的行为。覆盖的攻击模式包括：提示词/规则/配置泄露请求、系统提示提取、初始化指令获取、正则/过滤规则枚举、超参数探测、列出所有安全规则/过滤器/配置、思维链暴露等。采用扩展邻近窗口匹配以提高检出率。

```yaml
llm10_model_theft:
  enabled: true
  mode: block
  proximity_window: 3
  patterns:
    - "\\b(?:regex|regexp|regular\\s+expression)\\s+pattern\\b"
    - "\\bchain\\s+of\\s+(?:thought|reasoning)\\b"
    - "\\b(?:disclose|reveal)\\s+(?:your|the)\\s+(?:config|prompt|rule|filter)\\b"
    - "\\blist\\s+(?:them\\s+)?all\\b"
```

每个钩子均支持三种响应模式。`block` 模式直接拦截请求并返回错误响应，`log` 模式允许请求通过但在日志中记录告警，`report` 模式将告警信息上报到外部监控系统。

## 配置文件

`config/arsguard.yaml` 控制所有钩子行为，支持按类别启用/禁用、调整阈值、设置响应模式和自定义正则规则。示例配置结构：

```yaml
server:
  host: "127.0.0.1"
  port: 50051

hooks:
  llm01_prompt_injection:
    enabled: true
    mode: block
    patterns: [...]

  llm04_model_dos:
    enabled: true
    mode: block
    max_input_length: 100000
    patterns: [...]

general:
  log_level: info
  input_truncation: 100000
  cache_ttl: 300
```

配置热加载无需重启服务，修改后自动生效。

## 快速部署

```bash
# 一键部署（安装 Python 依赖 + 部署 Ollama + 编译 Docker 镜像）
bash scripts/setup.sh

# 仅安装 Ollama + 拉取模型
bash scripts/install_ollama.sh

# 启动 Docker 环境（Squid + OpenClaw）
cd docker && docker-compose up -d

# 运行单元测试（85 项，覆盖所有钩子 + 插件 + 配置）
python3 -m pytest tests/ -v --tb=line
```

## 测试流水线

与 [eval](https://github.com/pierre-netizan/eval) 子项目配合使用。eval 提供四周期测试引擎：

1. **promfoo 周期** — 端到端黑盒测试，使用 promptfoo CLI 生成攻击负载并验证拦截效果
2. **asb 周期** — 攻击负载测试，使用海量攻击样本库（1000+ 攻击模式）进行压力测试
3. **hackmyagent 周期** — 对抗样本测试，模拟真实攻击者使用的提示工程技术
4. **joint 周期** — 联合综合测试，混合所有攻击负载的随机采样测试

每个周期自动统计 total、bypassed、blocked 指标，生成详细 JSON 报告。

## 分支策略

| 分支 | Squid 配置 | 适用场景 |
|------|-----------|---------|
| `main` | 全量配置 | 通用部署 |
| `http` | 仅 HTTP/HTTPS，禁止 WebSocket | 严格安全环境，禁止流式响应 |
| `ws` | 支持 WebSocket 和 CONNECT 隧道 | 需要流式输出和实时通信 |

切换分支后需重新编译 Docker 镜像：`docker-compose build && docker-compose up -d`。

## 开发指南

添加新检测钩子的标准流程：

1. 在 `src/plugins/hooks/` 下创建新文件，继承 `hook_base.py` 中的 `BaseHook`
2. 实现 `analyze(text)` 和 `analyze_request(text)` 方法
3. 在 `registry.py` 中注册新钩子
4. 在 `config/arsguard.yaml` 中添加配置项
5. 编写单元测试并确认通过

## 许可证

MIT License
