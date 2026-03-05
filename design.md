# 可进化多 Agent 平台完整设计方案（design.md）

## 1. 文档信息
- 版本: v1.0
- 日期: 2026-03-05
- 目标: 设计一个可持续扩展的多 Agent 协作平台，可部署在用户本地环境，支持动态接入 Agent、Skills 商店、EvoMAP 经验包进化、多 Agent 群聊讨论，并由“小龙虾助手（OpenClaw 风格）”统一管理与调度。

## 2. 目标与范围

### 2.1 业务目标
1. 平台可持续新增 Agent（插件化接入，不影响在线任务）。
2. 对每个任务自动判断需要哪些 Agent，支持并行和串行协作。
3. 提供 Skills 商店，支持技能上架、版本管理、评分，以及 Skill 快速生成流水线。
4. 支持细粒度控制“某个 Agent 允许使用哪些 Skills”。
5. 基于 EvoMAP 思路沉淀经验包，支持上传、检索、评分增长与复用。
6. 支持“多 Agent 讨论群”，用户可拉多个 Agent 进入房间讨论同一问题，并输出统一结论。
7. 全链路实时可视化：看到当前有哪些 Agent 在做什么、做到了哪一步、耗时和状态。
8. 引入“小龙虾助手”作为平台总控 Agent，负责 Agent 管理、任务分配、评分考核、新 Agent 生成和群聊组织。

### 2.2 非目标（当前版本）
1. 不做跨组织的强一致协作编辑。
2. 不做任意第三方模型的无限制直连（需经过网关和策略控制）。
3. 不做完全自动化高危操作执行（保留 HITL 审批）。

## 3. 设计原则
1. Local-First: 任务执行与敏感数据处理优先在本地。
2. Cloud-Shared Wisdom: 脱敏后的 Skills 与经验包可同步云端共享。
3. Event-Driven: 所有关键动作标准化事件化，便于观测、追踪和回放。
4. Policy-by-Default: 权限、预算、风险策略前置，不依赖人工兜底。
5. Evolvable: 新 Agent、新 Skill、新经验包可以低成本接入和迭代。

## 4. 总体架构

### 4.1 本地运行时（Local Runtime）
1. API Gateway
2. 小龙虾助手（Lobster Assistant，OpenClaw 风格总控）
3. 任务接入与编排层（Meta-Agent Orchestrator + DAG Engine）
4. Agent Registry（注册中心 + 向量索引）
5. Agent Runtime Pool（Worker Agent 执行容器）
6. Skill Store Service（本地技能仓库 + 版本管理）
7. Skill Generator Pipeline（自然语言 -> Skill 草案 -> 沙盒测试 -> 发布）
8. Authorization Service（Agent-Skill 授权）
9. Experience Service（经验包生成、检索、打分、上传）
10. Discussion Service（多 Agent 群聊）
11. Observability Stack（Event Bus、Trace、Metrics、WebSocket）
12. Security & Governance（HITL、预算、断路器、脱敏）

### 4.2 云端 Hub（Cloud Hub）
1. Cloud Skills Hub（技能市场）
2. Cloud EvoMAP Experience Hub（经验包共享池）
3. Ranking & Reputation Service（评分与信誉）
4. Sync Service（上/下载同步、冲突处理）

## 5. 核心模块设计

### 5.1 动态 Agent 注册中心

#### 5.1.1 Agent Profile（标准模型）
- agent_id, name, version, owner
- capabilities（标签 + 文本描述 + embedding）
- supported_skill_ids（可用技能声明）
- model_config（默认模型、fallback 模型、温度、最大 token）
- runtime_limits（并发、预算、超时、重试）
- risk_level（low/medium/high）
- health_status（online/degraded/offline）

#### 5.1.2 生命周期
1. Register: 提交 Profile 并校验 schema。
2. Validate: 探测可用性（健康检查 + 冒烟任务）。
3. Index: 向量化能力描述并入库。
4. Activate: 加入可调度池。
5. Suspend/Retire: 出现异常或版本下线。

#### 5.1.3 实时状态
- 心跳 + 任务槽位 + token 消耗 + 当前任务信息。
- WebSocket 推送给前端，支持按 Agent、任务、时间过滤。

### 5.2 智能任务路由与编排

#### 5.2.1 任务处理流程
1. Task Ingest: 任务入站并完成风险分类。
2. Task Parse: Meta-Agent 将任务拆分为 DAG 子任务。
3. Candidate Search: 从 Agent Registry 检索候选 Agent。
4. Capability Match: 语义相似度 + 技能匹配 + 历史表现评分。
5. Plan Build: 生成执行计划（并行/串行/依赖关系）。
6. Dispatch: 任务分配到 Agent Runtime Pool。
7. Monitor & Replan: 失败重试、降级路由、替换 Agent。

#### 5.2.2 路由评分函数（示例）
`RouteScore = 0.35*Capability + 0.20*SkillFit + 0.20*ExperienceFit + 0.10*Reliability + 0.10*CostScore + 0.05*LatencyScore - Penalty`

Penalty 包括：权限不足、预算超限、风险冲突、近期失败率过高。

### 5.3 Skills 商店与 Skill 快速生成

#### 5.3.1 Skill 包标准
- metadata: skill_id, name, category, semver, author
- contract: OpenAPI/JSON Schema、输入输出、错误码
- executor: 代码实现（Python/Node/HTTP wrapper）
- tests: 单元与集成测试
- policy: 权限需求、风险等级、审计标签

#### 5.3.2 Skill 快速生成流水线
1. 用户自然语言描述需求。
2. Tool-Creator Agent 产出 Skill 草案（接口 + 代码 + 测试）。
3. 沙盒自动测试（Docker/E2B）。
4. 失败时触发自我修复循环（读取日志 -> 修复 -> 重测）。
5. 通过后进入审核态（可选人工审批）。
6. 发布到本地 Skills 商店，可选同步云端。

#### 5.3.3 版本与回滚
- 采用 semver（major.minor.patch）。
- 运行时支持按 Agent 固定版本（防回归）。
- 出现问题支持一键回滚到稳定版本。

### 5.4 Agent-Skill 授权机制

#### 5.4.1 授权模型
- RBAC（角色级）+ ABAC（条件级）组合。
- 约束维度：Agent、Skill、环境、时间窗口、预算、数据域。

#### 5.4.2 执行时控制
1. 任务派发前检查 Skill 授权。
2. 注入仅允许的 Tool Schema 到 Agent 上下文。
3. 每次 Skill 调用写审计日志（谁在何时以何参数调用）。
4. 高风险 Skill 强制 HITL。

### 5.5 EvoMAP 经验包体系

#### 5.5.1 经验包结构
- package_id, source_agent_id, created_at
- Environment: 系统环境、依赖版本、资源约束
- Task: 任务描述、输入特征、领域标签
- Method: 关键步骤、调用工具链、参数策略、失败分支
- Outcome: 结果质量、耗时、成本、是否一次成功
- Evidence: 关键日志片段、指标摘要、可复现脚本引用
- Rating: 当前评分、使用次数、成功复用率

#### 5.5.2 经验包生成流程
1. 复杂任务完成后自动触发打包候选。
2. 系统做脱敏（PII/密钥/业务机密规则）。
3. 用户确认是否上传云端。
4. 云端入库并建立向量索引。

#### 5.5.3 经验检索与复用
1. 根据新任务生成 query embedding。
2. 检索相似经验包（本地优先 + 云端补充）。
3. 结合评分、时效、可复现性做重排。
4. 将 Top-K 经验摘要作为 Few-Shot 注入规划阶段。

#### 5.5.4 评分增长机制（示例）
- 初始分: 1.0
- 每次被复用成功: `+alpha*(1-failure_rate)`
- 被复用失败: `-beta`
- 长期未使用: 轻微衰减
- 最终分: 限制在 `[0, 5]`

### 5.6 多 Agent 群聊协作

#### 5.6.1 目标
用户可创建讨论房间，拉入多个 Agent 围绕同一问题协作讨论，系统控制发言轮次、证据引用与冲突消解，输出最终结论。

#### 5.6.2 角色与机制
1. Moderator Agent: 控制节奏、总结阶段结论。
2. Planner Agent: 拆解问题与讨论议程。
3. Specialist Agents: 给出各自领域方案。
4. Critic/Reviewer Agent: 反驳、找漏洞、做一致性检查。
5. Synthesizer Agent: 最终整合答案与行动清单。

#### 5.6.3 讨论协议（Protocol）
1. Clarify: 明确问题边界与成功标准。
2. Propose: 各 Agent 提交方案与依据。
3. Challenge: 交叉质疑，指出风险与缺失。
4. Converge: 依据证据与评分合并方案。
5. Finalize: 形成统一结论 + 备选方案 + 风险说明。

#### 5.6.4 轮次控制与收敛
- 最大轮次（如 6 轮）+ 超时（如 10 分钟）。
- 若分歧持续，触发“投票 + 裁判模型”机制。
- 连续重复观点触发去重提醒，防止空转。

#### 5.6.5 群聊产物
- 结论摘要（主方案/备选方案/待确认事项）
- 决策依据列表（引用消息与工具结果）
- 自动生成经验包候选（便于复用）

### 5.7 实时观测与可视化
1. 任务视图: DAG 节点状态（queued/running/success/failed）。
2. Agent 视图: 当前任务、步骤、耗时、模型、成本。
3. 群聊视图: 发言流、轮次阶段、共识分数。
4. 追踪视图: LLM 调用链、Skill 调用链、错误链路。

### 5.8 安全与治理
1. HITL: 高危动作（外部写操作、生产变更、资金相关）需审批。
2. 预算控制: 任务/Agent/token/时间四维预算。
3. 断路器: 连续失败或重复错误触发熔断。
4. 脱敏: 云端同步前自动脱敏并支持规则扩展。
5. 审计: 所有关键动作可回放与追责。

### 5.9 小龙虾助手（OpenClaw 风格总控）

#### 5.9.1 角色定位
- 平台级 Control Tower Agent，不直接替代 Worker Agent，而是负责“组织、调度、评估、孵化”。  
- 通过统一指令面板让用户以自然语言进行平台管理，例如“给数据清洗任务组队并执行”。

#### 5.9.2 核心能力
1. Agent 管理: 新增/下线/启停 Agent，检查健康状态和负载。
2. 任务分配: 根据任务类型和实时资源挑选 Agent，并可一键重分配。
3. 评分考核: 从质量、时效、成本、稳定性、安全合规五个维度打分。
4. 新 Agent 生成: 根据需求自动产出 Agent Blueprint，完成脚手架、测试、注册。
5. 组织群聊: 自动拉起相关 Agent 讨论房间，指定 Moderator 和收敛策略。

#### 5.9.3 评分与考核机制
- 评分维度：`Quality(35%) + SLA(20%) + Cost(15%) + Reliability(20%) + Safety(10%)`
- 考核周期：任务级即时评分 + 周期性综合评分（日报/周报）。
- 处置策略：连续低分触发降权、限流、重新训练或下线建议。

#### 5.9.4 新 Agent 生成流水线
1. 需求解析：小龙虾助手将用户意图转为 Agent Blueprint（职责、输入输出、可用 Skills、风险等级）。
2. 代码与配置生成：调用 Agent-Generator 模板产出运行骨架与 Profile。
3. 自动测试：冒烟测试 + 工具权限测试 + 安全策略测试。
4. 注册发布：通过后写入 Agent Registry，默认灰度状态。
5. 试运行评估：完成最小基准任务后转为正式可调度。

#### 5.9.5 群聊组织策略
1. 自动选人：按任务域、历史表现、可用性自动挑选参与 Agent。
2. 自动定角：分配 Moderator、Specialist、Reviewer、Synthesizer。
3. 收敛守护：达到轮次上限后强制生成结论并标注分歧点。
4. 沉淀复用：群聊结论自动进入经验包候选池。

## 6. 关键数据模型（建议）

| 实体 | 关键字段 | 说明 |
|---|---|---|
| agents | agent_id, name, status, health, owner | Agent 主表 |
| agent_profiles | profile_id, agent_id, capabilities, embedding, model_config | Agent 能力与配置版本 |
| skills | skill_id, name, category, latest_version, rating | Skill 主表 |
| skill_versions | skill_id, version, schema, executor_ref, test_report, status | Skill 版本与测试信息 |
| agent_skill_grants | grant_id, agent_id, skill_id, constraints, expires_at | Agent-Skill 授权表 |
| tasks | task_id, user_id, intent, priority, budget, status | 任务主表 |
| task_nodes | node_id, task_id, dependency, assigned_agent_id, status | DAG 节点 |
| task_events | event_id, task_id, node_id, type, payload, ts | 任务事件流 |
| discussions | room_id, task_id, mode, max_rounds, status | 讨论房间 |
| discussion_messages | msg_id, room_id, round_no, speaker_agent_id, content, refs | 群聊消息 |
| experiences | package_id, source_agent_id, env, task_feat, method, rating | 经验包主表 |
| experience_usage | usage_id, package_id, task_id, result, delta_score | 经验复用记录 |
| agent_scores | score_id, agent_id, task_id, dimensions, total_score, judge_ref | Agent 评分记录 |
| agent_generation_jobs | job_id, request, blueprint, status, created_agent_id | 新 Agent 生成任务 |
| lobster_policies | policy_id, trigger, action, constraints, enabled | 小龙虾助手调度策略 |
| agent_lineups | lineup_id, task_id, selected_agents, rationale | 小龙虾助手选人记录 |
| approvals | approval_id, task_id, action, approver, status, ts | 审批记录 |

## 7. API 与事件契约（建议）

### 7.1 API（示例）
1. `POST /api/agents/register`
2. `POST /api/tasks`
3. `GET /api/tasks/{task_id}/live`
4. `POST /api/skills/generate`
5. `POST /api/skills/{skill_id}/publish`
6. `POST /api/agents/{agent_id}/skills/{skill_id}/grant`
7. `POST /api/discussions`
8. `POST /api/discussions/{room_id}/invite`
9. `POST /api/discussions/{room_id}/start`
10. `GET /api/discussions/{room_id}/result`
11. `POST /api/experiences/{package_id}/upload`
12. `POST /api/experiences/search`
13. `POST /api/lobster/dispatch`
14. `POST /api/lobster/agents/generate`
15. `GET /api/lobster/agents/{agent_id}/score`
16. `POST /api/lobster/discussions/auto-create`

### 7.2 事件主题（示例）
- `agent.registered`
- `agent.status.changed`
- `task.created`
- `task.node.assigned`
- `task.node.completed`
- `skill.generated`
- `skill.test.failed`
- `skill.published`
- `discussion.round.started`
- `discussion.message.created`
- `discussion.consensus.updated`
- `experience.created`
- `experience.reused`
- `agent.evaluated`
- `lobster.dispatch.created`
- `lobster.agent.generated`
- `lobster.discussion.assembled`
- `approval.required`

## 8. 核心业务流程

### 8.1 任务进入 -> 智能路由 -> 执行
1. 用户提交任务。
2. Meta-Agent 拆解 DAG。
3. 结合 Agent 能力、Skill 授权、经验匹配做路由评分。
4. 分配给多个 Agent 执行并实时回传事件。
5. 失败节点触发重试/换 Agent/请求人工。
6. 任务完成后输出结果与复盘数据。

### 8.2 Skill 快速生成
1. 用户提出工具需求。
2. Tool-Creator 自动生成 Skill + 测试。
3. 沙盒验证失败则自修复。
4. 验证通过后发布并可授权给指定 Agent。

### 8.3 经验包复用闭环
1. 任务完成生成经验包候选。
2. 用户确认上传（脱敏后入云）。
3. 后续相似任务检索该经验包。
4. 使用效果回写评分并更新排序。

### 8.4 多 Agent 群聊协作
1. 用户创建讨论房间并邀请 Agent。
2. Moderator 宣布目标和议程。
3. 各 Agent 分轮发言、互相质疑和修正。
4. 共识引擎计算一致性与证据充分度。
5. Synthesizer 输出最终答案，附不确定项。
6. 讨论过程沉淀为经验包候选。

### 8.5 小龙虾助手总控流程
1. 用户向小龙虾助手提交目标（如“处理线上告警并给出修复方案”）。
2. 小龙虾助手解析任务并自动生成 Agent 组队方案。
3. 若现有 Agent 能力不足，小龙虾助手触发新 Agent 生成流水线。
4. 小龙虾助手下发任务并实时监控执行质量、耗时和成本。
5. 小龙虾助手对参与 Agent 打分考核，并更新调度权重。
6. 需要群体决策时，小龙虾助手自动创建讨论房间并组织收敛输出。

## 9. 部署与扩展建议

### 9.1 部署形态
1. 单机版（MVP）: API + Orchestrator + Runtime + DB + Redis + Web。
2. 团队版: Runtime、Orchestrator、WebSocket 分离部署。
3. 企业版: 本地私有集群 + 云端 Hub 同步服务。

### 9.2 扩展策略
1. Agent Runtime 水平扩容（按队列深度自动扩容）。
2. 向量检索分层（热数据本地、冷数据云端）。
3. 讨论服务隔离部署（防止长会话影响主任务吞吐）。
4. 小龙虾助手控制平面主备部署（避免单点故障）。

## 10. MVP 里程碑
1. M1（基础可用）
   - Agent 注册与任务路由
   - 小龙虾助手基础管理（Agent 启停、健康查看、任务派发）
   - 实时任务看板
   - 基础 Skill 商店与授权
2. M2（进化能力）
   - 经验包生成、检索、评分
   - Agent 评分考核体系（任务级 + 周期级）
   - 云端同步（脱敏上传/下载）
3. M3（高级协作）
   - 多 Agent 群聊
   - 小龙虾助手自动组队并组织群聊
   - 共识引擎与最终结论生成
4. M4（治理强化）
   - 新 Agent 自动生成与灰度发布
   - HITL、预算、断路器、完整审计

## 11. 风险与缓解
1. 路由误选 Agent
   - 缓解: 引入在线反馈学习与 A/B 路由评估。
2. Skill 自动生成质量不稳定
   - 缓解: 强化测试覆盖 + 分级发布 + 快速回滚。
3. 经验包污染（低质量或错误经验）
   - 缓解: 评分衰减、负反馈惩罚、人工举报与下架。
4. 群聊长时间不收敛
   - 缓解: 轮次上限、仲裁模型、强制收敛模板。
5. Agent 评分偏差导致误判
   - 缓解: 引入多裁判模型 + 人工抽检 + 指标可解释性。
6. 新 Agent 自动生成质量不达标
   - 缓解: 灰度发布、最小任务验收门槛、自动回滚。
7. 成本失控
   - 缓解: 多级预算、模型分层调度、缓存复用。

## 12. 验收标准（建议）
1. 新 Agent 从注册到可调度时间 <= 5 分钟。
2. 复杂任务自动路由命中率 >= 85%。
3. Skill 自动生成一次通过率 >= 70%，三轮内通过率 >= 90%。
4. 经验包复用后任务平均耗时下降 >= 20%。
5. 多 Agent 群聊在限定轮次内收敛率 >= 80%。
6. 平台可实时展示在线 Agent 状态和任务进度，端到端延迟 <= 2 秒。
7. 小龙虾助手自动任务分配成功率 >= 95%。
8. Agent 评分覆盖率 >= 99%，评分结果可追溯。
9. 新 Agent 自动生成后冒烟测试通过率 >= 85%。
