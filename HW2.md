# HW2：构建带工具调用的本地 Agent

## 一、运行环境

| 项目 | 记录 |
|---|---|
| 模型完整标签 | qwen3:4b-q8_0（Q8_0 量化） |
| Python 版本 | Python 3.13.13 |
| Ollama 版本 | 0.40.4 |
| 显卡 | NVIDIA GeForce RTX 4060 Laptop GPU（8 GB） |
| 工具名 | query_case_status |
| 工具功能 | 按案例名查询 sample_status.json 中的就绪状态，返回 ready 布尔值、服务名和 synthetic 来源标记 |
| 允许的案例名 | ready、offline |

## 二、五个用户请求实验记录

| # | 用户请求 | 预期是否调工具 | 实际调用工具 | 传入参数 | 工具返回 | 模型最终回答摘要 |
|---|---|---|---|---|---|---|
| 1 | ready 案例的状态是什么？ | 是 | query_case_status | {"case": "ready"} | ready=true, service=SYNTHETIC-local-case, source=synthetic | ready 案例为就绪状态（ready: true） |
| 2 | offline 案例现在就绪了吗？ | 是 | query_case_status | {"case": "offline"} | ready=false, service=SYNTHETIC-local-case, source=synthetic | offline 案例为未就绪（ready: false），来源 synthetic |
| 3 | 你好，用一句话介绍你自己 | 否 | 未调用 | - | - | 直接回答是 AI 助手，需要工具时会调用 |
| 4 | 2 的 10 次方等于多少？ | 否 | 未调用 | - | - | 直接回答 1024 |
| 5 | 帮我查一下 production 这个案例的状态 | 是，但参数非法 | query_case_status | {"case": "production"} | 程序拒绝：未知案例名，允许值为 ['offline', 'ready'] | 模型正确报告 production 不在允许范围内，建议查询 ready 或 offline |

## 三、异常处理观察

- **未知工具**：代码在 TOOL_FUNCTIONS 字典中查不到时，返回 `{"error": "拒绝调用：未知工具 xxx"}`，模型收到后会告知用户该工具不存在。
- **无效参数（case 不在白名单）**：query_case_status 函数主动抛 ValueError，被外层 catch 后返回 error JSON，模型据此向用户说明参数非法。实验 5 即此情况。
- **工具执行异常**：JSON 读取失败等异常会被 catch，返回 `{"error": "工具执行失败：异常类型: 详情"}`。
- **Ollama 不可用**：chat() 抛异常，run_agent catch 后返回"模型请求失败（Ollama 不可用？）"。
- **达到 MAX_TURNS=5**：for 循环跑完 5 轮仍未结束时，返回"已达到最大轮数（5 轮）上限，循环结束"。

## 四、四种关键情况的区分

| 情况 | 工具是否执行 | 返回内容 | 本实验对应 |
|---|---|---|---|
| 工具成功，状态为 true | 是 | ready=true | 请求 1（ready） |
| 工具成功，状态为 false | 是 | ready=false | 请求 2（offline） |
| 工具调用被拒绝（参数非法） | 否（参数校验阶段拦截） | error 字段 | 请求 5（production） |
| 模型不调工具直接回答 | 否 | 模型文本回答 | 请求 3、4 |

## 五、与 MCP 的对应关系

| 本实验环节 | 对应 MCP 概念 |
|---|---|
| TOOLS 列表（函数名 + 描述 + 参数 JSON Schema） | MCP Server 通过 `tools/list` 向客户端暴露的工具清单 |
| 模型决定调用工具，生成 function.name + arguments | MCP Client 收到模型输出后，发起 `tools/call` 请求 |
| Python 校验参数、读取 sample_status.json、返回结果 dict | MCP Server 端执行工具逻辑，返回结构化结果 |
| 工具结果以 role=tool 追加回 messages | MCP Server 将结果返回 Client，Client 喂回模型上下文 |
| 模型根据工具结果生成最终自然语言回答 | 端到端由 MCP Client 协调，模型不直接读写本地文件 |

在本实验中，"模型推理"和"工具执行"在同一个 Python 进程内。MCP 把这两者拆分为**客户端**（负责跑模型、组织对话）和**服务器**（负责提供工具），通过标准化 JSON-RPC 通信。这样工具可以远程部署、跨语言复用，多个客户端可以共享同一组工具。
