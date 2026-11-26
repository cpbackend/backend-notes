# AGENT-TEAM

四终端 Agent 团队的岗位分工、任务板 T0007–T0012 的职责归属、已装技能包的分配，以及统一的合法使用边界。

> **适用法律与授权声明（先读这一条）**
>
> 本文只约定"谁做什么、怎么交接、边界在哪"。所有涉及扫描、注入、漏洞验证的动作，**只在本地靶场（localhost / 自建隔离环境）或你有明确书面授权的项目**中进行。
> 对未授权系统执行上述动作，在大多数司法辖区属于违法行为。不确定是否在授权范围内时，默认视为**不在**范围内。
> 工具细节与复现步骤见 [SECURITY-LAB.md](SECURITY-LAB.md)，本文不重复 payload 与攻击细节。

## 1. 为什么要分工

`2026-11-18.md` 里的看板入口脚本已经把派活链路拆成四步：`board` / `context` / `dispatch` / `collect`，并区分 `owner=user`（发布、注册、付款、实名只能人做）与 `owner=agent`。但"派给哪个执行体"此前没有约定，导致上下文要人肉重拼、任务正文还会被模板分隔符静默截断（429 字符 → 41 字符）。

四终端模型解决的是**派给谁**：按能力形状分工，而不是四个终端抢同一类活。

## 2. 四终端分工

| 终端 | 定位 | 主要职责 | 典型交付物 | 不做的事 |
| --- | --- | --- | --- | --- |
| **OpenCode** | 主控 + 执行 | 任务拆解与派单、回收工单、维护看板与作战上下文、处理 `owner=user` 与 `owner=agent` 的分流；日常杂项执行。派单时按模板固定边界（`## 边界` / 回退 `## 结果`）切分正文 | 工单（`dispatch` / `collect`）、更新后的看板与 `context`、交接记录 | 不替人做发布/付款/实名；不把带真实凭据的目标派给执行体 |
| **Claude** | 深做 + 审查 | 复杂推理与设计决策、方案定稿、代码/文案审查、安全报告复核、漏洞挖掘技能包（Claude-BugHunter）的主使用方；对动态扫描结果做二次判定 | 设计结论、审查意见、脱敏后的报告、验收断言 | 不做机械批量；不靠改写措辞绕过模型侧安全策略 |
| **Codex** | 实现 + 批量 | 写代码、修 bug、批量脚本与重复任务（补样本、跑审计、格式迁移）、自动化测试；按断言给出可复验结果 | 补丁、批量脚本、测试输出、数据采集结果 | 不做最终合规裁决；不在未授权目标上跑批量扫描 |
| **Hermes** | 运维 + 长跑 | 定时链与巡检、数据管线/工具健康状态刷新、盯盘告警、CI 定时扫描、长跑进程与日志看护；发现失败先判瞬时故障还是代码问题 | 巡检记录、告警处置、定时任务状态、幂等重跑结论 | 不做需要深做判断的定稿；不把日志里的 `FAILED` 当结论，须定位到具体步骤 |

协作口径：**OpenCode 派单 → Claude/Codex/Hermes 执行 → Claude 审查 → OpenCode 回收并写回看板**。`owner=user` 的任务由 OpenCode 直接标阻塞，不进 `dispatch`。

## 3. 任务板 T0007–T0012 的职责

> 下表按仓库现有笔记（看板链路、安全工具、文案审计、罗盘、记忆链）与四终端能力归纳；**具体任务标题/状态以本地看板为准**，本文只定义职责归属与验收口径，不虚构完成状态。

| 任务 | 职责主题 | 主责 | 协作 / 复核 | 验收口径（断言式） |
| --- | --- | --- | --- | --- |
| **T0007** | 看板与派单链路：`board` / `context` / `dispatch` / `collect` 四命令、正文截断修复、`owner` 分流 | OpenCode | Codex 修解析、Claude 审查 | 源任务文件与解析结果逐字相同；`owner=user` 派单被拒 |
| **T0008** | 安全工具与技能包：Claude-BugHunter / Strix / XSStrike / PayloadsAllTheThings 的清单、安装、scope 校验与边界文档 | Claude | OpenCode 执行本地复现、Hermes 管定时扫描 | 工具和目标均在授权清单内；仓库内无凭据/真实目标 |
| **T0009** | 文案与素材红线审计：代码块逐个抽取、引用块与成品块区分、商品首图扫描 | Codex | Claude 审查红线口径 | 围栏→代码块数量一致；成品块红线命中 0 |
| **T0010** | 数据罗盘实现与修复：canvas DPR 重绘、DOM 交互与采样验证 | Codex | Claude 审查验收断言 | `paintedPx > 0`；DOM 数字与交互状态一致 |
| **T0011** | 记忆链与巡检：每日定时链、数据管线/工具健康状态刷新、告警处置、幂等重跑 | Hermes | OpenCode 回收结果 | 失败步骤单独重跑 `+0 new, 0 rewritten`；状态文件按时刷新 |
| **T0012** | 交付与合规复核：PR 文档、证据脱敏、披露流程、参考链接与边界声明 | OpenCode | Claude 审查为主责复核 | 证据已脱敏；无未授权目标信息；边界声明随文提交 |

## 4. 已装技能包分配

| 技能包 | 规模 / 定位 | 主使用终端 | 复核终端 | 使用方式 |
| --- | --- | --- | --- | --- |
| **Claude-BugHunter**（elementalsouls/Claude-BugHunter） | 85 skills / 16 commands，含 `hunt-xss`；按漏洞类别整理检测模式、payload、绕过表与链式利用模板，带 7-Question Gate scope 校验 | Claude | OpenCode | 深做与报告定稿主用；授权作业才启用，排除 0day 武器化、post-exploit、恶意软件开发、无差别目标 |
| **Strix**（usestrix/strix，Apache-2.0） | 9 skills 的 AI 渗透测试 agent；动态运行代码、多 agent 编排、可工作 PoC 验证、修复与报告，可接 GitHub Actions | Hermes | Claude | 定时/CI 扫描由 Hermes 跑，动态 PoC 结果交 Claude 复核；`strix view` 只绑 127.0.0.1 并带 token 链接 |
| **XSStrike**（s0md3v/XSStrike，GPLv3） | XSS 检测套件：反射与 DOM XSS、上下文分析、payload 生成、fuzz、爬虫、WAF 绕过 | Codex | Claude | 仅本地靶场批量验证；payload 只触发无害标记，不做数据外带 |
| **PayloadsAllTheThings** | 共享 payload 知识库，按漏洞类别索引，供检测/验证取值 | Claude 维护分类与白名单 | Codex 取用（批量场景） | 作为参考资料，不整库照搬进仓库；只取靶场所需的最小片段 |

分配原则：**谁最擅长谁主用，谁负责裁决谁复核**。动态工具（Strix、XSStrike）的结果一律走 Claude 二次判定后再写结论；知识库类（Claude-BugHunter、PayloadsAllTheThings）只在授权范围内取用。

## 5. 合法使用边界

开工前逐条确认，四条终端一视同仁：

1. 目标是本地靶场，或我有**书面授权**且目标在授权 scope 内；Docker 用 `-p 127.0.0.1:...`，不暴露到局域网/公网。
2. 测试窗口、允许的技术、禁止的动作都清楚（例如是否允许 DoS、是否允许社会工程）；不顺手扫相邻资产。
3. 工具会**实际发请求/动态测试**指向的目标；不在未授权系统上运行 Strix / XSStrike / Claude-BugHunter。
4. PoC 只用无害标记（`alert` / 固定字符串），拿到结果即停；不探测内网、不复用真实凭据、不做数据外带。
5. 证据先脱敏（Cookie、token、PII 一律打码）再保存或提交；LLM API key、`LLM_API_KEY`、HackerOne token 等走环境变量，不写进本仓库。
6. 发现真实漏洞走正规披露渠道，对方修复或同意前不公开细节。
7. 不做 0day 武器化、post-exploit、恶意软件开发；不通过改写措辞绕过模型侧实时安全策略——授权作业走正规验证渠道。
8. 有任何不确定，停下来问，而不是"先打了再说"。

违反以上任一条，任务立即停止并升级给人工处理。

## 6. 协作约定

- 工单正文标记与模板分隔符必须用不同字符串；派单只按固定边界切分，避免多小标题任务静默变空。
- 验收写成断言（输入/输出/逐字相同/命中数），不写"看着对"。
- 没做成的事单列一节，写清卡在哪；瞬时故障用幂等重跑确认，不直接改代码。
- 结论只写有证据的内容（命令 / 输出 / 文件路径），主观判断标为待补。

## 7. 参考

- 看板与验收口径来源：[2026-11-18.md](2026-11-18.md)
- 工具用途、payload 与完整合法边界：[SECURITY-LAB.md](SECURITY-LAB.md)
- Claude-BugHunter：<https://github.com/elementalsouls/Claude-BugHunter>
- Strix：<https://github.com/usestrix/strix>
- XSStrike：<https://github.com/s0md3v/XSStrike>
- PayloadsAllTheThings：<https://github.com/swisskyrepo/PayloadsAllTheThings>
