# RoboOS 项目技术文档

## 1. 项目概述

### 1.1 项目背景

RoboOS (Robotic Operating System) 是一个基于"**Brain-Cerebellum**"层级架构的开源具身操作系统，旨在解决当前机器人系统在以下关键问题上的局限：

- **跨本体适应性差** (Poor cross-embodiment adaptability)
- **任务调度效率低** (Inefficient task scheduling)
- **动态错误纠正能力不足** (Inadequate dynamic error correction)

当前端到端视觉-语言-动作(VLA)模型(如OpenVLA、RDT、Pi-0)在长时域规划和任务泛化方面表现较弱，而层级VLA模型(如Helix、Gemini-Robotics、GR00T-N1)缺乏跨本体兼容性和多智能体协作能力。

RoboOS 通过其创新的三层架构实现了从单智能体到蜂群智能的范式转变。

### 1.2 核心设计理念

```
┌─────────────────────────────────────────────────────────────────┐
│                        RoboOS 架构                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Embodied Cloud Model (Brain)                   │ │
│  │         多模态大语言模型 - 全局感知与高级决策                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓ ↑                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Cerebellum Skill Library                       │ │
│  │           模块化即插即用工具包 - 多技能执行                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓ ↑                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Real-Time Shared Memory                        │ │
│  │           时空同步机制 - 多智能体状态协调                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 技术架构版本

| 版本 | 架构特点 | 部署方式 |
|------|----------|----------|
| **RoboOS 1.0** | Master-Slaver 分布式架构 | 多机器部署 |
| **RoboOS 2.0** | SaaS + MCP (Model Context Protocol) | 单机轻量部署 |

---

## 2. 功能模块详解

### 2.1 Master 模块 (全局协调层)

#### 2.1.1 GlobalAgent (全局智能体)

**文件位置**: `master/agents/agent.py`

**核心职责**:
- 接收全局任务并发布给各个机器人
- 管理机器人注册和状态
- 收集和处理机器人执行结果
- 维护场景信息同步

**关键方法**:

| 方法名 | 功能描述 |
|--------|----------|
| `publish_global_task(task)` | 发布全局任务到各个Agent |
| `_handle_register(data)` | 处理机器人注册 |
| `_handle_result(data)` | 处理机器人执行结果 |
| `_group_tasks_by_order(tasks)` | 按拓扑顺序对任务分组 |

**通信通道**:

```
robot_registration     → 机器人注册通道
{robot_name}_to_roboos → 机器人结果汇报通道
roboos_to_{robot_name} → 下发任务到机器人通道
ROBOT_INFO_*           → Redis中机器人信息Key
SCENE_INFO_*           → Redis中场景信息Key
```

#### 2.1.2 GlobalTaskPlanner (全局任务规划器)

**文件位置**: `master/agents/planner.py`

**核心职责**:
- 加载机器人和场景配置
- 构建任务规划Prompt
- 调用云端大模型进行任务分解
- 支持多种模型后端 (Azure OpenAI, 标准OpenAI兼容接口)

**配置结构**:

```python
{
    "profile": {
        "ROBOT_PROFILE_PATH": str,    # 机器人配置路径
        "SCENE_PROFILE_PATH": str,    # 场景配置路径
    },
    "model": {
        "MODEL_SELECT": str,          # 选定的模型名称
        "MODEL_LIST": [
            {
                "CLOUD_MODEL": str,      # 模型名称
                "CLOUD_TYPE": str,       # "azure" | "default"
                "CLOUD_API_KEY": str,
                "CLOUD_SERVER": str,     # API服务器地址
            }
        ]
    }
}
```

### 2.2 Slaver 模块 (机器人执行层)

#### 2.2.1 RobotManager (机器人管理器)

**文件位置**: `slaver/run.py`

**核心职责**:
- 管理单个机器人的生命周期
- 与MCP服务器建立连接
- 处理来自Master的任务指令
- 维护机器人心跳机制

**架构特点**:
- 异步架构 (asyncio)
- 线程安全的任务处理
- MCP (Model Context Protocol) 协议支持

#### 2.2.2 ToolCallingAgent (工具调用智能体)

**文件位置**: `slaver/agents/slaver_agent.py`

**核心职责**:
- 基于 ReAct 框架的逐步任务执行
- 工具调用解析与执行
- 维护执行内存 (AgentMemory)
- 规划间隔 (Planning Interval) 支持

**ReAct 框架流程**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    Think    │ →  │    Act      │ →  │  Observe    │ →  │   Answer    │
│  (LLM思考)  │    │  (调用工具)  │    │  (观察结果)  │    │  (返回结果)  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
       ↓                  ↓                  ↓
       └──────────────────┴──────────────────┘
                    循环执行直到完成
```

**关键特性**:
- `planning_interval`: 定期进行任务重规划
- `max_steps`: 最大执行步数限制
- `tool_parser`: 工具调用解析器
- `step_callbacks`: 步骤回调机制

#### 2.2.3 机器人技能工具 (Robotic Tools)

**文件位置**: `slaver/tools/robotic_tools.py`

| 工具名称 | 功能 | 类别 |
|----------|------|------|
| `navigate_to_where` | 导航到指定位置 | Chassis |
| `grasp_object` | 抓取目标物体 | Arm |
| `place_to_where` | 将物体放置到目标位置 | Arm |
| `detect_object` | 检测目标物体 | Camera |

#### 2.2.4 Robot 核心类

**文件位置**: `slaver/robot/core.py`

**核心职责**:
- 封装机器人底层操作
- 错误处理与恢复
- 状态同步到共享内存

**错误处理机制**:

```python
class ErrorHandler:
    # 错误分类
    - navigationErrors (E101-E102): 导航错误
    - visionErrors (E201-E202): 视觉错误
    - graspObjectErrors (E301-E303): 抓取错误
    - placeObjectErrors (E401-E403): 放置错误
```

### 2.3 通信层 (Communicator)

**底层依赖**: `flag_scale.flagscale.agent.communication.Communicator`

**通信模式**: Redis Pub/Sub

| 通道类型 | 用途 |
|----------|------|
| `robot_registration` | 机器人上线注册 |
| `ROBOT_INFO_{name}` | 机器人状态信息 |
| `SCENE_INFO_{name}` | 场景状态信息 |
| `{name}_to_roboos` | 机器人上报结果 |
| `roboos_to_{name}` | 下发任务指令 |

### 2.4 用户界面层

**文件位置**: `gradio_ui.py`

**核心功能**:
- 服务器连接测试
- 任务消息发送
- 响应结果展示
- 配置管理

---

## 3. 技术栈选型

### 3.1 核心技术栈

| 类别 | 技术选型 | 版本要求 | 用途说明 |
|------|----------|----------|----------|
| **编程语言** | Python | ≥ 3.8 | 主要开发语言 |
| **Web框架** | Flask | 3.1.1 | Master API服务 |
| **前端UI** | Gradio | 5.33.1 | 用户交互界面 |
| **消息中间件** | Redis | 6.2.0 | 实时通信与状态存储 |
| **日志系统** | Rich | 14.0.0 | 格式化终端输出 |
| **配置管理** | PyYAML | 6.0.2 | 配置文件解析 |
| **HTTP客户端** | httpx | 0.28.1 | 异步HTTP请求 |
| **云端模型** | OpenAI API兼容 | - | 任务规划大模型 |
| **MCP协议** | mcp | 1.9.3 | 机器人工具调用协议 |
| **数据序列化** | Pydantic | 2.11.5 | 数据验证与序列化 |

### 3.2 关键依赖分析

```
flag_scale @ git+https://github.com/FlagOpen/FlagScale  # FlagOpen大模型框架
├── agent
│   └── communication.py    # Redis通信封装
├── mcp                      # Model Context Protocol
└── 其他大模型支持组件
```

### 3.3 工具生态

| 工具类别 | 工具名称 | 功能 |
|----------|----------|------|
| **开发工具** | ruff | 代码检查与格式化 |
| **测试框架** | pytest | 单元测试与集成测试 |
| **代码覆盖** | coverage | 测试覆盖率统计 |

---

## 4. 核心业务流程

### 4.1 整体任务流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                         RoboOS 任务执行流程                            │
└──────────────────────────────────────────────────────────────────────┘

[用户界面]                                                           [机器人执行]
     │                                                                    ▲
     ▼                                                                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│   Gradio    │ →  │   Master    │ →  │    Redis    │ →  │   Slaver    │ │
│   Web UI    │    │  (Global    │    │  (Message   │    │  (Robot     │ │
│             │    │   Agent)    │    │   Broker)   │    │   Manager)  │ │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘ │
                          │                   ▲                │        │
                          ▼                   │                ▼        │
                   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
                   │   Task      │    │   State     │    │   Tool      │ │
                   │ Decomposition│    │  Syncing    │    │  Execution  │ │
                   │   (LLM)     │    │             │    │             │ │
                   └─────────────┘    └─────────────┘    └─────────────┘ │
                                              │                             │
                                              ▼                             │
                                       ┌─────────────┐                      │
                                       │   Error     │──────────────────────┘
                                       │  Handling   │
                                       └─────────────┘
```

### 4.2 任务发布时序图

```
用户                    Master                    Redis               Slaver机器人
 │                       │                        │                      │
 │  发布任务请求          │                        │                      │
 │──────────────────────>│                        │                      │
 │                       │                        │                      │
 │                       │ 1. 任务规划 (LLM)      │                      │
 │                       │───────────────────────>│                      │
 │                       │                        │                      │
 │                       │ 2. 获取机器人/场景状态   │                      │
 │                       │<───────────────────────│                      │
 │                       │                        │                      │
 │                       │ 3. 任务分解结果         │                      │
 │                       │[subtask_list]          │                      │
 │                       │                        │                      │
 │                       │ 4. 按组发送子任务       │                      │
 │                       │                        │──────────────────────>│
 │                       │                        │                      │
 │                       │                        │              5. 执行任务
 │                       │                        │              ─────────
 │                       │                        │                      │
 │                       │                        │              6. 结果上报
 │                       │                        │<──────────────────────│
 │                       │                        │                      │
 │  7. 返回子任务列表     │                        │                      │
 │<──────────────────────│                        │                      │
 │                       │                        │                      │
```

### 4.3 任务执行流程 (Slaver侧)

```
┌─────────────────────────────────────────────────────────────┐
│                    Slaver 任务执行流程                       │
└─────────────────────────────────────────────────────────────┘

                    ┌─────────────────────┐
                    │   接收任务指令        │
                    │ {task, task_id}      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  初始化 ToolCalling  │
                    │      Agent          │
                    └──────────┬──────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │         ReAct 执行循环           │
              ├────────────────────────────────┤
              │  1. Think: LLM 生成工具调用     │
              │  2. Act: 执行工具               │
              │  3. Observe: 获取执行结果       │
              │  4. 更新记忆                    │
              │  5. 检查是否完成                │
              └───────────────┬────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
       ┌─────────────┐                 ┌─────────────┐
       │   继续循环   │                 │   结束循环   │
       │  (未完成)    │                 │  (完成/达到  │
       │             │                 │   最大步数)  │
       └──────┬──────┘                 └──────┬──────┘
              │                               │
              └───────────────┬───────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │    发送结果到Master   │
                    │   roboos_to_robot   │
                    └─────────────────────┘
```

### 4.4 错误处理与恢复流程

```
┌─────────────────────────────────────────────────────────────┐
│                    错误处理与恢复流程                        │
└─────────────────────────────────────────────────────────────┘

       工具执行                    错误检测                   错误恢复
         │                          │                         │
         ▼                          ▼                         ▼
┌─────────────────┐          ┌─────────────────┐     ┌─────────────────┐
│  perform_xxx()  │          │   ErrorHandler   │     │  Recovery       │
│                 │ ────────>│   .handle_error  │────>│  Actions        │
│  - navigate     │  触发错误  │                 │     │                 │
│  - grasp        │          │  - 查找错误定义   │     │  - 重试策略     │
│  - place        │          │  - 获取恢复列表   │     │  - 备选方案     │
│  - detect       │          │  - 依次尝试恢复   │     │  - 人工介入     │
└─────────────────┘          └─────────────────┘     └─────────────────┘

错误代码体系:
├── E101-E102: 导航错误 (ObstacleBlockade, UnreachableCoordinates)
├── E201-E202: 视觉错误 (CameraOcclusion, RecognitionEmptyObjects)
├── E301-E303: 抓取错误 (KinematicSingularity, ObjectDropping, InheritedVisionError)
└── E401-E403: 放置错误 (KinematicSingularity, ObjectDropping, InheritedVisionError)
```

---

## 5. 关键特性

### 5.1 多机器人协作

**特性说明**:
- 支持异构机器人本体 (单臂、双臂、人形、轮式)
- 基于拓扑排序的任务分组与调度
- 机器人间状态共享与同步

**任务顺序标记**:
```python
{
    "subtask_order": 0,  // 并行执行
    "subtask_order": 1,  // 等待 order=0 完成后执行
    "subtask_order": 2,  // 等待 order=1 完成后执行
}
```

### 5.2 跨本体适应

**本体抽象层**:
```
┌─────────────────────────────────────┐
│         统一工具接口                  │
├─────────────────────────────────────┤
│  navigate_to_where  (导航)           │
│  grasp_object      (抓取)           │
│  place_to_where    (放置)           │
│  detect_object     (检测)           │
└─────────────────────────────────────┘
           ↑
           │ 工具适配层
           ▼
┌─────────────────────────────────────┐
│         机器人特定实现                 │
├─────────────────────────────────────┤
│  Realman 单臂机器人                   │
│  Agilex 双臂机器人                    │
│  人形机器人                          │
│  轮式机器人                          │
└─────────────────────────────────────┘
```

### 5.3 实时状态同步

**共享内存机制**:
- Redis 作为分布式状态存储
- 机器人状态实时更新 (TTL 60秒)
- 心跳保活机制

### 5.4 错误恢复策略

| 错误类型 | 恢复策略优先级 |
|----------|---------------|
| 导航阻塞 | 尝试其他路径 → 请求其他机器人帮助 → 人工介入 |
| 视觉遮挡 | 移动到近处 → 请求帮助 → 重新规划 |
| 抓取失败 | 调整末端姿态 → 重新尝试 → 重新规划 |
| 放置失败 | 重新抓取 → 重新放置 → 重新规划 |

---

## 6. 部署架构

### 6.1 单机部署 (RoboOS 2.0 - Stand-alone)

```
┌─────────────────────────────────────────────────────────────┐
│                      单机部署架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│   │   Master    │    │   Slaver    │    │   Redis     │    │
│   │  (Flask)    │◄──►│  (Robot)    │◄──►│  (Pub/Sub)  │    │
│   │  localhost  │    │  localhost  │    │  localhost  │    │
│   │   :5000     │    │             │    │   :6379     │    │
│   └─────────────┘    └─────────────┘    └─────────────┘    │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Gradio    │                                              │
│   │   Web UI    │                                              │
│   │  localhost  │                                              │
│   │   :7861     │                                              │
│   └─────────────┘                                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 分布式部署 (RoboOS 1.0)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        分布式部署架构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                        Master 节点                           │   │
│   │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │   │
│   │  │   Master    │    │   Redis     │    │   Cloud     │     │   │
│   │  │  (Flask)    │◄──►│  (Broker)   │    │   LLM       │     │   │
│   │  │   :5000     │    │   :6379     │    │   API       │     │   │
│   │  └─────────────┘    └─────────────┘    └─────────────┘     │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                │                                     │
│              ┌─────────────────┼─────────────────┐                   │
│              │                 │                 │                   │
│              ▼                 ▼                 ▼                   │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │
│   │  Slaver 1   │    │  Slaver 2   │    │  Slaver N   │            │
│   │  (Robot 1)  │    │  (Robot 2)  │    │  (Robot N)  │            │
│   │  Raspberry  │    │  NVIDIA      │    │  不同机器人  │            │
│   │  Pi/Jetson  │    │  Jetson     │    │  本体        │            │
│   └─────────────┘    └─────────────┘    └─────────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 使用指南

### 7.1 环境准备

```bash
# 1. 系统要求
- Python 3.8+
- Redis server
- pip package manager

# 2. 安装依赖
git clone https://github.com/FlagOpen/RoboOS.git
cd RoboOS
pip install -r requirements.txt

# 3. 配置修改
# 编辑 master/config.yaml 和 slaver/config.yaml
# 设置 CLOUD_API_KEY, CLOUD_SERVER 等参数
```

### 7.2 启动流程

```bash
# 终端 1: 启动 Redis
redis-server

# 终端 2: 启动 Master
python master/run.py

# 终端 3: 启动 Slaver (每台机器人)
python slaver/run.py

# 终端 4: 启动 Web 界面
python gradio_ui.py

# 访问 http://localhost:7861
```

### 7.3 配置说明

**Master 配置** (`master/config.yaml`):
```yaml
communicator:
  HOST: "127.0.0.1"
  PORT: 6379

model:
  MODEL_SELECT: "robobrain"
  MODEL_LIST:
    - CLOUD_MODEL: "robobrain"
      CLOUD_TYPE: "default"
      CLOUD_API_KEY: "YOUR-API-KEY"
      CLOUD_SERVER: "YOUR-CLOUD-SERVER-URL"
```

**Slaver 配置** (`slaver/config.yaml`):
```yaml
communicator:
  HOST: "127.0.0.1"
  PORT: 6379

profile:
  PATH: "./slaver/profile/robot_profile.yaml"
```

### 7.4 任务示例

```
输入任务: "Take basket to kitchenTable, and put apple and knife into basket, 
          and then take them back to customTable."

预期输出 (子任务分解):
{
    "reasoning_explanation": "任务分解推理...",
    "subtask_list": [
        {"robot_name": "robot_1", "subtask": "Take basket to kitchenTable", "subtask_order": "0"},
        {"robot_name": "robot_2", "subtask": "Put apple into basket", "subtask_order": "1"},
        {"robot_name": "robot_2", "subtask": "Put knife into basket", "subtask_order": "1"},
        {"robot_name": "robot_1", "subtask": "Take basket back to customTable", "subtask_order": "2"}
    ]
}
```

---

## 8. 目录结构

```
RoboOS/
├── master/                          # Master 模块 (全局协调层)
│   ├── agents/
│   │   ├── agent.py               # 全局智能体
│   │   └── planner.py             # 任务规划器
│   ├── profile/
│   │   ├── robot_profile.yaml     # 机器人配置
│   │   └── scene_profile.yaml     # 场景配置
│   ├── prompt/
│   │   ├── prompts.py             # Prompt 模板
│   │   └── utils.py               # Prompt 工具
│   ├── config.yaml                # Master 配置
│   └── run.py                     # Master 入口
│
├── slaver/                         # Slaver 模块 (机器人执行层)
│   ├── agents/
│   │   ├── slaver_agent.py        # 工具调用智能体
│   │   └── models.py              # 模型封装
│   ├── profile/
│   │   └── robot_profile.yaml      # 机器人配置
│   ├── prompts/
│   │   └── toolcalling_agent.yaml # Agent Prompt 模板
│   ├── robot/
│   │   ├── base.py                # 机器人基类
│   │   ├── core.py                # 机器人核心
│   │   ├── error_definitions.py   # 错误定义
│   │   └── error_handler.py       # 错误处理
│   ├── tools/
│   │   ├── robotic_tools.py       # 机器人工具
│   │   ├── memory.py              # Agent 记忆
│   │   ├── monitoring.py          # 监控与日志
│   │   └── ...
│   ├── config.yaml                # Slaver 配置
│   ├── run.py                     # Slaver 入口
│   └── utils.py                   # 工具函数
│
├── assets/                         # 资源文件
├── docs/                           # 文档
├── gradio_ui.py                   # Web 界面
├── requirements.txt               # 依赖清单
└── README.md                      # 项目说明
```

---

## 9. 总结

RoboOS 是一个创新性的具身智能操作系统，通过其独特的"**Brain-Cerebellum**"层级架构，成功实现了：

1. **跨本体兼容性**: 统一的工具接口抽象层支持多种机器人本体
2. **多智能体协作**: 基于 Redis 的实时状态同步和消息传递机制
3. **智能任务规划**: 云端大模型驱动的任务分解与调度
4. **鲁棒错误恢复**: 分层错误处理与多级恢复策略
5. **灵活部署模式**: 支持单机和分布式部署架构

该系统已在餐厅、家庭、超市等多种场景下验证了其多机器人协作能力，支持单臂、双臂、人形、轮式等多种异构本体。