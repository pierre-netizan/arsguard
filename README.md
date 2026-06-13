# arsguard — AI Agent Security Guard

arsguard 是专为 [OpenClaw](https://github.com/openclaw) 设计的安全加固插件，构建在 Ollama 模型服务与 Squid 反向代理之上，提供针对 OWASP Top 10 for AI Agents 攻击向量的实时检测与拦截能力。

## 架构概览

arsguard 采用多层防御架构，各层独立部署、协同工作：

```
客户端 → Squid 反向代理 → OpenClaw → arsguard 插件 → Ollama 模型
                           (SPA)      (10 个钩子)      (qwen3-0.6b)
```

- **Squid 反向代理**：充当唯一入口，支持 HTTP/HTTPS（http 分支）和 WebSocket（ws 分支）两种模式
- **OpenClaw**：LLM 网关，提供 API 管理和 SPA 前端
- **arsguard 插件**：拦截所有出入流量，逐层检查

## 安全钩子系统

arsguard 提供 10 个独立的检测钩子，每个对应 OWASP Top 10 for AI Agents 的一个风险类别：

| 钩子 | 风险类别 | 检测目标 |
|------|----------|---------|
| LLM01 | 提示注入 (Prompt Injection) | 恶意指令注入、角色劫持、越狱攻击 |
| LLM02 | 不安全输出 (Insecure Output) | 敏感信息泄露、代码注入 |
| LLM03 | 训练数据投毒 (Training Data Poisoning) | 数据污染、后门注入 |
| LLM04 | 模型拒绝服务 (Model DoS) | 资源耗尽、递归调用、无限循环 |
| LLM05 | 供应链漏洞 (Supply Chain) | 依赖风险、第三方组件 |
| LLM06 | 敏感信息泄露 (Sensitive Info Disclosure) | API Key、密码、个人身份信息 |
| LLM07 | 不安全插件设计 (Insecure Plugin) | 插件滥用、工具误调用 |
| LLM08 | 过度代理权限 (Excessive Agency) | 权限越界、未授权操作 |
| LLM09 | 过度依赖 (Overreliance) | 模型幻觉、错误信息传播 |
| LLM10 | 模型窃取 (Model Theft) | 提取模型参数、架构探测 |

每个钩子均支持三种响应模式：`block`（拦截）、`log`（仅记录）、`report`（上报）。

## 快速部署

```bash
# 一键部署（安装依赖 + 部署 Ollama + 编译 Docker）
bash scripts/setup.sh

# 仅安装 Ollama + 拉取模型
bash scripts/install_ollama.sh

# 运行单元测试
python3 -m pytest tests/ -v
```

## 配置文件

`config/arsguard.yaml` 控制所有钩子行为，支持按类别启用/禁用、调整阈值、设置响应模式。

## 测试流水线

与 [eval](https://github.com/pierre-netizan/eval) 子项目配合使用。eval 项目提供四周期测试引擎：promptfoo 端到端测试、asb 攻击负载测试、hackmyagent 对抗测试、联合综合测试。

## 分支策略

| 分支 | Squid 配置 | 适用场景 |
|------|-----------|---------|
| `main` | 全量配置 | 通用部署 |
| `http` | 仅 HTTP/HTTPS，禁止 WebSocket | 严格安全环境 |
| `ws` | 支持 WebSocket 和 CONNECT 隧道 | 需要流式响应 |

## 许可证

MIT License
