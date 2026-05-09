# RoboOS Agent 工作说明

本文档记录 AI agent 在本仓库工作前需要了解的项目信息。内容来自当前仓库源码、README 和 `docs/` 文档，后续维护时请同步更新。

## 1. 项目定位

RoboOS 是一个面向具身智能的开源机器人操作系统，目标是用 **Brain-Cerebellum** 层级架构支持跨本体、多智能体协作。项目重点解决：

- 跨本体机器人协作与适配。
- 长时域任务的全局规划、拆分和调度。
- 多机器人状态同步与任务执行结果回传。
- 机器人技能工具的即插即用调用。

当前仓库主要是 RoboOS 1.0 的分布式实现：`master` 负责全局规划和调度，`slaver` 负责单个机器人执行任务。README 中提到 RoboOS 2.0 将走 SaaS + MCP / 单机轻量部署路线，但本仓库主干代码仍以 Master-Slaver 架构为主。

## 2. 技术栈

- 主要语言：Python 3.8+。
- Web/API：Flask，入口在 `master/run.py`，默认监听 `0.0.0.0:5000`。
- 用户界面：Gradio，入口在 `gradio_ui.py`，默认监听 `127.0.0.1:7861`。
- 消息与共享状态：Redis，通过 `flag_scale.flagscale.agent.communication.Communicator` 封装。
- 大模型调用：OpenAI 兼容接口或 Azure OpenAI，配置在 `master/config.yaml` 和 `slaver/config.yaml`。
- MCP：Slaver 侧通过 `mcp` 连接机器人工具服务器，示例工具服务器在 `slaver/profile/robot_tools_mcp.py`。
- 代码质量/测试依赖：`ruff`、`pytest`、`coverage` 已列在 `requirements.txt`，但当前仓库没有发现 tests 目录。

## 3. 目录结构

```text
.
├── AGENTS.md                     # agent 工作说明
├── README.md                     # 项目介绍、安装和快速启动
├── docs/
│   ├── PROJECT_DOCUMENTATION.md  # 中文技术文档
│   ├── ARCHITECTURE_DIAGRAMS.md  # 中文架构图和流程图
│   └── agents/                   # issue/triage/domain 工作约定
├── master/                       # 全局协调层
│   ├── run.py                    # Flask API 入口
│   ├── config.yaml               # Master 配置
│   ├── agents/
│   │   ├── agent.py              # GlobalAgent，注册、调度、结果处理
│   │   └── planner.py            # GlobalTaskPlanner，大模型任务拆分
│   ├── profile/                  # Master 侧机器人/场景初始 profile
│   └── prompt/                   # 全局规划 prompt
├── slaver/                       # 机器人执行层
│   ├── run.py                    # RobotManager 入口
│   ├── config.yaml               # Slaver 配置
│   ├── agents/                   # ReAct / ToolCalling agent
│   ├── profile/                  # 单机器人 profile 与 MCP 工具示例
│   ├── prompts/                  # Slaver agent prompt 模板
│   ├── robot/                    # 机器人核心抽象、错误定义与处理
│   └── tools/                    # 工具协议、默认工具、机器人技能工具
├── gradio_ui.py                  # Web 客户端
├── client_config.json            # Gradio 客户端连接配置
├── requirements.txt              # Python 依赖
├── assets/                       # README 和文档图片
└── third_party/                  # 第三方真实工具脚本占位
```

## 4. 核心运行流程

1. 用户通过 Gradio UI 或 HTTP API 提交全局任务。
2. `master/run.py` 的 `/publish_task` 接口调用 `GlobalAgent.publish_global_task`。
3. `GlobalTaskPlanner` 读取当前 `ROBOT_INFO_*` 和 `SCENE_INFO_*`，构造 prompt，并调用云端模型把全局任务拆成子任务。
4. `GlobalAgent` 按 `subtask_order` 分组，把任务发送到 Redis channel：`roboos_to_{robot_name}`。
5. 每个 Slaver 的 `RobotManager` 监听自己的任务通道，创建 `ToolCallingAgent` 执行任务。
6. Slaver 通过 MCP 发现/调用机器人工具，或使用 mock 工具模拟导航、抓取、放置、检测。
7. 执行结果通过 `{robot_name}_to_roboos` 回传 Master。
8. Master 收集结果，更新 Redis 中的机器人状态和场景状态。

关键 Redis channel / key：

- `robot_registration`：机器人注册。
- `ROBOT_REGISTER_{robot_name}`：机器人注册信息。
- `ROBOT_INFO_{robot_name}`：机器人状态、位置、工具等。
- `SCENE_INFO_{recep_name}`：场景/容器状态。
- `roboos_to_{robot_name}`：Master 下发给机器人的任务。
- `{robot_name}_to_roboos`：机器人回传执行结果。
- `ROBOT_SUBTASK_{robot_name}`：机器人最近子任务结果。

## 5. 主要模块说明

### Master

- `master/agents/agent.py`
  - `GlobalAgent` 是 Master 核心。
  - 初始化 Redis communicator、日志、全局规划器。
  - 监听机器人注册，动态监听机器人结果通道。
  - 从模型响应中提取 JSON 代码块，按拓扑顺序下发子任务。

- `master/agents/planner.py`
  - `GlobalTaskPlanner` 负责构造全局任务规划 prompt。
  - 只把 `robot_state == "idle"` 的机器人放进规划上下文。
  - 支持 `default` OpenAI 兼容接口和 `azure` 接口。

- `master/config.yaml`
  - `communicator.CLEAR: true` 表示 Master 启动会清 Redis DB。
  - `profile.SCENE_PROFILE_ENABLE: true` 默认会加载场景初始状态。
  - `profile.ROBOT_PROFILE_ENABLE: false` 默认不从 Master profile mock 注册机器人。

### Slaver

- `slaver/run.py`
  - `RobotManager` 是 Slaver 核心。
  - 启动后读取 `slaver/profile/robot_profile.yaml`，根据 `robot_tools` 推导 MCP 脚本路径。
  - 用 stdio 启动 MCP server，列出 tools，再向 Master 注册机器人。
  - 通过心跳线程给 `ROBOT_INFO_{robot_name}` 续 TTL。

- `slaver/agents/slaver_agent.py`
  - 实现多步 ReAct 风格 agent 和 `ToolCallingAgent`。
  - 维护 `AgentMemory`、tool call 记录、规划间隔、执行日志。
  - 默认日志写到 `./.log/agent.log`。

- `slaver/robot/core.py`
  - `Robot` 封装底层动作：抓取、放置、导航、检测。
  - mock 模式下由 `DISABLE_ARM`、`DISABLE_CAMERA`、`DISABLE_CHASSIS` 控制是否绕过真实硬件。
  - 执行动作时会更新 Redis 中的机器人位置、抓取对象和场景对象列表。

- `slaver/tools/robotic_tools.py`
  - 机器人技能工具包括：
    - `navigate_to_where`
    - `detect_object`
    - `grasp_object`
    - `place_to_where`
  - 工具类通过 decorator 标记为 arm/chassis/camera 类别。

## 6. 配置和示例数据

- Master 默认场景在 `master/profile/scene_profile.yaml`：
  - `kitchenTable`：apple、pear、banana、knife。
  - `customTable`：basket、plate、cup。
  - `servingTable`：bowl、fork、spoon。
  - `basket`：egg。

- Master 示例机器人在 `master/profile/robot_profile.yaml`：
  - `robot_1`：realman_single，支持导航、检测、抓取、放置。
  - `robot_2`：songling_dual，支持检测、抓取、放置。

- Slaver 默认机器人在 `slaver/profile/robot_profile.yaml`：
  - `robot_name: robot_1`
  - `robot_type: realman_single`
  - `robot_tools: slaver/profile/robot_tools.py`
  - 初始位置：`initialPosition`
  - 可导航位置：`initialPosition`、`customTable`、`kitchenTable`、`servingTable`

- 云端模型配置中的 API key 和 server URL 都是占位符；实际运行前必须替换。

## 7. 本地运行

README 给出的 RoboOS 1.0 快速启动顺序：

```bash
pip install -r requirements.txt
redis-server
python master/run.py
python slaver/run.py
python gradio_ui.py
```

访问 UI：

```text
http://localhost:7861
```

也可以直接调用 Master API：

```bash
curl -X POST http://localhost:5000/publish_task \
  -H "Content-Type: application/json" \
  -d '{"task":"Take basket to kitchenTable, put apple into basket, then take it back to customTable."}'
```

注意：`master/run.py` 的 GET `/publish_task` 只用于连接测试，返回字段名当前是 `statis`，不是 `status`。

## 8. 开发注意事项

- 不要把真实 API key、云模型服务地址、机器人连接凭据提交进仓库；配置文件当前使用占位符。
- Master 启动时默认清 Redis，调试共享状态时留意 `master/config.yaml` 的 `communicator.CLEAR`。
- Slaver 默认不清 Redis，配置在 `slaver/config.yaml` 的 `communicator.CLEAR: false`。
- 真实硬件接入应优先放在 `third_party/` 或 Slaver profile 指向的工具脚本中，保持 `slaver/tools/robotic_tools.py` 的工具接口稳定。
- `RobotManager.connect_to_robot` 会把 `robot_tools` 路径从 `xxx.py` 推导为 `xxx_mcp.py`，新增工具脚本时要成对维护。
- `.log/` 和 `.logs/` 是运行时日志目录，代码会按需创建，通常不应提交生成的日志。
- 当前代码里有一些历史拼写（如 `slaver`、`chasis`、`_gat_model_info_from_config`），改名会影响导入路径和配置引用，除非专门做重构，否则不要随手改。
- 当前仓库没有测试目录；如果改动任务拆分、Redis 通信、工具调用或状态同步，建议补最小回归测试或至少用 mock Redis/本地 Redis 做端到端冒烟验证。

## 9. 文档入口

- 项目总览和快速启动：`README.md`
- 中文技术文档：`docs/PROJECT_DOCUMENTATION.md`
- 中文架构图：`docs/ARCHITECTURE_DIAGRAMS.md`
- Issue tracker 约定：`docs/agents/issue-tracker.md`
- Triage 标签约定：`docs/agents/triage-labels.md`
- Domain docs 约定：`docs/agents/domain.md`

## 10. Agent 工作约定

### Issue tracker

Issues 位于本仓库 GitHub Issues，使用 `gh` CLI。详见 `docs/agents/issue-tracker.md`。

### Triage labels

使用 5 个默认 triage 标签：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。详见 `docs/agents/triage-labels.md`。

### Domain docs

本仓库按单上下文文档布局组织：仓库根目录的 `CONTEXT.md` 加 `docs/adr/`。当前根目录尚未发现 `CONTEXT.md` 或 `docs/adr/`，如后续补齐，请按 `docs/agents/domain.md` 维护。
