## 🧩 实测问题与修复记录

本案例在真实运行过程中发现并修复了以下问题（均已解决）：

### 1. AgentScope 版本不兼容（ImportError）

**现象**
```
ImportError: cannot import name 'ReActAgent' from 'agentscope.agent'
```

**原因**：本案例基于 AgentScope **1.x** API 编写（`ReActAgent`、`agentscope.pipeline.MsgHub`），但环境中默认安装了 **2.0.5**。2.x 是完全重写的新架构，删除了 `ReActAgent`，且 `agentscope.pipeline` 模块整体不存在。

**解决**：降级安装
```bash
pip install agentscope==1.0.2
```

### 2. 模型免费额度耗尽（AllocationQuota.FreeTierOnly）

**现象**
```
{"code": "AllocationQuota.FreeTierOnly", "message": "Free quota exhausted..."}
```

**原因**：默认模型 `qwen-max` 的免费额度已用完。

**解决**：更换为有免费额度的模型（实测 `qwen-plus` 可用）。

### 3. 模型名与端点不匹配（url error）

**现象**
```
{"code": "InvalidParameter", "message": "url error, please check url！"}
```

**原因**：`qwen3.7-flash-2026-07-15` / `qwen3.7-flash` 只在**百炼 MaaS compatible-mode 端点**有效，而 agentscope 走的是 **DashScope 原生端点**（`dashscope.aliyuncs.com/api/v1/...`），原生端点不认识这些模型名。

**解决**：使用 DashScope 原生端点可用的模型名（如 `qwen-plus`）。

### 4. thinking 模式与结构化输出冲突

**现象**
```
InternalError.Algo.InvalidParameter: The tool_choice parameter does not support
being set to required or object in thinking mode
```

**原因**：`structured_model=DiscussionModelCN` 会把 Pydantic schema 转成 `tools` + 强制 `tool_choice`，而 DashScope 在 **thinking 模式（`enable_thinking=True`）下不允许这种 tool_choice**，两者互斥。

**解决**：配置 `enable_thinking=False`。

### 5. 思考过程刷屏（thinking 内容被打印）

**现象**：控制台打印大量英文思考内容
```
关羽(thinking): As 关羽 (Guan Yu), I'm a wolf in this Three Kingdoms werewolf game...
```

**原因**：agentscope 默认 `print` 会把 `thinking` 类型块打印出来；DeepSeek 系列模型即使 `enable_thinking=False` 仍返回思考内容。

**解决**：自定义 `GameReActAgent` 重写 `print` 方法，只打印文本发言，隐藏 thinking（并做了流式增量打印，避免重复输出）。

### 6. 流中断导致崩溃（handle_interrupt TypeError）

**现象**
```
TypeError: ReActAgent.handle_interrupt() got an unexpected keyword argument 'structured_model'
```

**原因**：模型流式响应被中断（`CancelledError`）后，`AgentBase.__call__` 会把原始参数（含 `structured_model`）透传给 `handle_interrupt`，但 `ReActAgent.handle_interrupt` 不接收该参数，直接崩溃。

**解决**：`GameReActAgent` 重写 `handle_interrupt`，兼容任意参数，中断时优雅降级跳过。

### 7. 日志警告刷屏（Unsupported block type）

**现象**
```
WARNING | _dashscope_formatter:_format:206 - Unsupported block type thinking in the message, skipped.
```

**原因**：thinking 块被存入智能体记忆，格式化器拼接对话历史时不支持该块类型，反复打警告。

**解决**：为 agentscope 的 `as` logger 添加日志过滤器，仅屏蔽该条警告。

## 📺 运行效果示例

以下为修复后的预期输出格式（玩家发言为 AI 生成，内容随机）：

```
🎮 欢迎来到三国狼人杀！
=== 游戏初始化 ===
🎮 开始设置三国狼人杀游戏...
游戏主持人: 📢 【赵云】你在这场三国狼人杀中扮演狼人，你的角色是赵云。夜晚可以击杀一名玩家      
游戏主持人: 📢 【关羽】你在这场三国狼人杀中扮演狼人，你的角色是关羽。夜晚可以击杀一名玩家      
游戏主持人: 📢 【刘备】你在这场三国狼人杀中扮演预言家，你的角色是刘备。每晚可以查验一名玩家的身份
游戏主持人: 📢 【孙权】你在这场三国狼人杀中扮演女巫，你的角色是孙权。拥有解药和毒药各一瓶，可以救人或杀人
游戏主持人: 📢 【张飞】你在这场三国狼人杀中扮演村民，你的角色是张飞。无特殊技能，依靠推理和投票
游戏主持人: 📢 【司马懿】你在这场三国狼人杀中扮演村民，你的角色是司马懿。无特殊技能，依靠推理和投票
游戏主持人: 📢 三国狼人杀游戏开始！参与者：赵云、关羽、刘备、孙权、张飞、司马懿
✅ 游戏设置完成，共6名玩家

=== 第1轮游戏 ===
游戏主持人: 📢 🌙 第1夜降临，天黑请闭眼...
【狼人阶段】
游戏主持人: 📢 🐺 狼人请睁眼，选择今晚要击杀的目标...
游戏主持人: 📢 狼人们，请讨论今晚的击杀目标。存活玩家：赵云、关羽、刘备、孙权、张飞、司马懿    
赵云: 诸位狼兄，吾乃赵云。今夜之选，当以威胁最大者为先。关羽武艺超群，且为人刚正，恐是好人阵营 之领袖；司马懿谋略过人，若为好人，必是我狼族心腹大患。依吾之见，可先除关羽，以断其锋芒，诸位以 为如何？
关羽: 赵云将军此言差矣！吾关羽行事光明磊落，岂是狼人？若今夜误杀忠良，反让真正的狼人得利。依吾 之见，司马懿此人深藏不露，智谋过人，方是最可疑之人。若他真是好人阵营中的智者，必是狼族之心腹大 患，当先除之。而刘备素有仁德之名，但仁德之辈往往藏奸诈之心，亦不可不防。吾建议今夜先除司马懿， 以绝后患！
...
【预言家阶段】
游戏主持人: 📢 🔮 预言家请睁眼，选择要查验的玩家...
刘备: 作为蜀汉之主，我刘备深知天下大势。今夜初查，我选择查验司马懿。此人城府极深，善于伪装，在 三国乱世中惯于隐藏真实意图，极有可能是狼人阵营中的一员。若他查验为狼，吾当立即公布，引导诸位识 破其伪装。若为好人，亦可排除一个疑点，为来日布局。我刘备以仁德立世，必当如实禀报查验结果，为天 下苍生求一份清明。
游戏主持人: 📢 查验结果：司马懿是好人
【女巫阶段】
...
```

# AgentScope 三国狼人杀案例

本目录包含第六章 AgentScope 框架的完整实战案例，展示了如何使用 AgentScope 构建一个融合中国古典文化元素的多智能体在线游戏。

## 📁 文件说明

- `main_cn.py` - 主要游戏逻辑和控制器
- `prompt_cn.py` - 中文提示词管理
- `game_roles.py` - 游戏角色定义和配置
- `structured_output_cn.py` - 结构化输出模型定义
- `utils_cn.py` - 游戏工具函数和辅助方法
- `README.md` - 本说明文档

## 🎮 案例特点

- **消息驱动架构**：展示 AgentScope 的核心消息传递机制
- **并发协作**：演示多智能体同时在线的实时交互
- **角色扮演**：每个智能体具备双重身份（游戏角色+三国人物）
- **结构化输出**：通过 Pydantic 模型约束智能体行为
- **容错机制**：单个智能体异常不影响整体游戏流程

## 🛠️ 环境准备

### 1. 安装依赖

```bash
pip install agentscope==1.0.2
pip install dashscope
pip install pydantic
```

> ⚠️ **注意：agentscope 必须安装 1.x 版本**（本案例基于 1.x API 编写）。2.x 是完全重写的新架构，`ReActAgent`、`agentscope.pipeline.MsgHub` 等已被移除，会直接报 `ImportError`。

### 2. 配置环境变量

设置阿里云 DashScope API Key：

```bash
# Linux/Mac
export DASHSCOPE_API_KEY="your-api-key-here"

# Windows PowerShell
$env:DASHSCOPE_API_KEY="your-api-key-here"

# Windows CMD
set DASHSCOPE_API_KEY=your-api-key-here
```

获取 API Key：https://dashscope.console.aliyun.com/apiKey

### 3. 运行游戏

```bash
python main_cn.py
```

## 🎭 游戏角色说明

### 游戏角色
- **狼人**：夜晚击杀好人，白天隐藏身份
- **预言家**：每晚查验一名玩家身份
- **女巫**：拥有解药和毒药各一瓶
- **猎人**：被投票出局时可开枪带走一名玩家
- **村民**：通过推理和投票找出狼人

### 三国人物
- **刘备**：仁德宽厚，善于团结众人
- **关羽**：忠义刚烈，言辞直接
- **张飞**：性格豪爽，容易冲动
- **诸葛亮**：智慧超群，分析透彻
- **曹操**：雄才大略，善于权谋
- **司马懿**：深谋远虑，城府极深

## 🏗️ 架构设计

### 分层架构
```
游戏控制层 (ThreeKingdomsWerewolfGame)
    ├── 游戏状态管理
    ├── 流程控制
    └── 胜负判定

智能体交互层 (MsgHub)
    ├── 消息路由
    ├── 并发处理
    └── 状态同步

角色建模层 (DialogAgent)
    ├── 角色提示词
    ├── 结构化输出
    └── 行为约束
```

### 核心组件

**1. 消息中心 (MsgHub)**
```python
async with MsgHub(
    participants=self.werewolves,
    enable_auto_broadcast=True
) as hub:
    # 狼人夜晚讨论
    for wolf in self.werewolves:
        await wolf(structured_model=DiscussionModelCN)
```

**2. 结构化输出**
```python
class VoteModelCN(BaseModel):
    vote: str = Field(description="投票目标玩家姓名")
    reason: str = Field(description="投票理由")
    confidence: int = Field(ge=1, le=10, description="信心程度")
```

**3. 并发管道**
```python
vote_msgs = await fanout_pipeline(
    self.alive_players,
    msg=vote_announcement,
    structured_model=get_vote_model_cn(self.alive_players),
    enable_gather=False,
)
```

## 🎯 游戏流程

### 夜晚阶段
1. **狼人讨论**：狼人通过 MsgHub 协商击杀目标
2. **预言家查验**：预言家选择查验对象
3. **女巫行动**：女巫决定是否使用解药/毒药

### 白天阶段
1. **死亡公布**：公布夜晚死亡玩家
2. **自由讨论**：所有存活玩家参与讨论
3. **投票淘汰**：投票选择淘汰对象
4. **猎人技能**：被淘汰的猎人可开枪

## 🔧 自定义配置

### 修改游戏人数
```python
# 在 main_cn.py 中修改
await game.setup_game(player_count=8)  # 支持 6-12 人
```

### 添加新角色
```python
# 在 game_roles.py 中添加
ROLES["守护者"] = {
    "description": "守护者",
    "ability": "每晚可以守护一名玩家",
    "team": "好人阵营"
}
```

### 自定义提示词
```python
# 在 prompt_cn.py 中修改
def get_role_prompt(role: str, character: str) -> str:
    # 自定义角色提示词逻辑
    pass
```

## ⚠️ 注意事项

1. **agentscope 必须是 1.x**（推荐 1.0.2），2.x 无法运行本案例
2. **模型名必须在 DashScope 原生端点有效**，`qwen3.7-flash-*` 系列只在 MaaS compatible-mode 端点可用
3. **`enable_thinking` 必须为 `False`**，否则与结构化输出冲突
4. **免费额度问题**：不同模型免费额度独立，耗尽后需换模型或充值（`qwen-flash`/`qwen-turbo`/`qwen-max` 实测额度均已耗尽，`qwen-plus` 可用）
5. **运行前必须设置环境变量** `DASHSCOPE_API_KEY`
6. **IDE 缓冲问题**：修改文件后若 IDE 有未保存的旧缓冲，可能覆盖最新改动（曾导致 `model_name` 被改回旧值），修改后请确认文件内容已保存

## 🐛 常见问题

### Q: 游戏无法启动？
A: 检查以下几点：
- 确认 DASHSCOPE_API_KEY 环境变量已设置
- 验证 API Key 是否有效
- 检查网络连接是否正常

### Q: 智能体输出格式错误？
A: 可能原因：
- 模型理解能力限制
- 提示词设计不够清晰
- 结构化输出约束过于复杂

### Q: 游戏流程卡住？
A: 建议：
- 检查 MsgHub 的消息传递
- 验证并发管道的执行状态
- 查看控制台错误日志

## 📚 技术亮点

### 1. 消息驱动架构
- 智能体间完全通过消息交互
- 支持异步并发处理
- 天然的分布式能力

### 2. 结构化输出约束
- 游戏规则转化为代码约束
- 提升系统稳定性和可预测性
- 便于调试和监控

### 3. 双重角色建模
- 游戏角色 + 三国人物的创新设计
- 展现不同人格的策略差异
- 增强游戏的趣味性和真实感

## 🚀 扩展方向

- **增加游戏模式**：支持更多狼人杀变体
- **优化 AI 策略**：提升智能体的游戏水平
- **可视化界面**：开发 Web 或桌面客户端
- **实时观战**：支持人类玩家观战和互动
- **数据分析**：统计游戏数据和智能体表现

## 🤝 贡献指南

欢迎提交 Issue 和 Pull Request：
- 报告游戏 Bug 或异常
- 提出新功能建议
- 优化代码实现
- 完善文档说明

---

*本案例是 Hello-Agents 教程第六章的核心实战项目，展示了 AgentScope 框架在构建复杂多智能体应用方面的强大能力。*