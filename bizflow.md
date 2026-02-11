目标：在当前目录一键初始化“飞书混合输入（聊天/纪要/语音转写/Markdown文档）→ 业务流程文档 → BRD”的工坊，并创建/覆盖所需文件与项目级 slash commands。

输入：$ARGUMENTS（主题，建议必填；例如：会员续费退款流程）

执行：

A) 初始化目录与文件（若不存在则创建）
- inputs/  research/  outputs/  .claude/commands/
- inputs/00_raw_feishu.md
- inputs/00_intake.md
- research/sources.md
- outputs/01_process.md
- outputs/02_brd.md
- decisions.md

B) 创建/覆盖写入 CLAUDE.md（只聚焦：零写作输入→流程→BRD；Superpowers 优先可降级）
写入内容如下（原样写入）：

# 业务流程梳理工坊（飞书混合输入：聊天/纪要/语音转写/Markdown文档）— 零写作闭环

## 目标
业务方不写文档。只基于原始材料产出：
- outputs/01_process.md（业务流程文档）
- outputs/02_brd.md（BRD，可对齐/可确认）
并在 decisions.md 保持可追溯的决策与未决项。

## 输入原则（强制）
- inputs/00_raw_feishu.md 是唯一事实源（允许混杂：聊天、纪要、转写、Markdown 文档）。
- 不允许脑补：无法从原文确定的信息必须标注【待确认】并进入确认清单。
- 同一结论冲突：必须标注冲突点 + 各自来源片段（以 EntryId 为依据）。

## 工具策略（Superpowers 优先，允许降级）
- 优先使用 /superpowers:brainstorm, /superpowers:write-plan, /superpowers:execute-plan
- 若命令不可用/失败：继续产出同等结构，并在文档顶部标注：
  ⚠️ Superpowers 降级模式：<原因>；同时写入 decisions.md。

## 输出门禁（强制）
- 确认点 <=10，按优先级，必须改写成“飞书可秒回”（是/否/A/B/C/短填空<=10字）。
- outputs/01_process.md 必须覆盖异常最低集合：
  权限、超时/重试、重复提交/幂等、人工介入、对账/纠错（按业务适配）
- 合规/规范/口径：必须写入 research/sources.md 并在文档里引用；否则标【待补引用】。

## 推荐命令
- /append：追加原始材料（支持混合：大段聊天+多篇Markdown；自动切块+索引；支持“Markdown内聊天二次拆分”）
- /capture：raw → MIS → 流程骨架 + 确认点<=10（默认 last=80）
- /confirm：确认点 → 飞书可直接发送的“秒回确认清单”（含依据 EntryId）
- /run：一键生成 01_process + 02_brd + QA 门禁

C) 写入项目级命令文件（创建/覆盖写入）
- .claude/commands/append.md
- .claude/commands/capture.md
- .claude/commands/confirm.md
- .claude/commands/run.md

把下面各文件内容按对应路径写入（原样写入）：

[FILE:.claude/commands/append.md]
目标：把 $ARGUMENTS（可混合：一大坨聊天 + 多篇 Markdown 文档 + 会议纪要 + 语音转写）追加到 inputs/00_raw_feishu.md，
自动识别并按“文档/章节/聊天发言”切块，生成多条 EntryId，并维护文件头部索引目录。
增强：支持“Markdown 内聊天二次拆分”（代码块外可拆，代码块内保持完整）。

关键约束：
- 不改写原文，只做切块与标注
- 保留 Markdown 代码块完整性（```...``` 不拆开）
- 混合输入：优先解析 Markdown 文档块，再解析剩余聊天/转写块

输入：$ARGUMENTS（原样粘贴；可一次粘贴多篇文档 + 大段聊天）

Step 0 — 初始化与索引准备
1) 若 inputs/00_raw_feishu.md 不存在则创建
2) 读取当前文件，定位当天已有 Entry 序号（用于递增）
3) 获取当前日期时间（用于 EntryId 的 YYYYMMDD-HHMM）
4) 确保文件顶部存在索引目录；若不存在则创建并置于最开头：
   # Raw Index
   | EntryId | CapturedAt | SourceType | DocTitle | SectionTitle | Keywords(3) | FlowHint |

Step 1 — 预处理：保护代码块
1) 将输入中 fenced code block（```...```）视为不可拆分单元
2) 记录每个 code block 的边界（用于后续“代码块外拆分”）

Step 2 — 粗分候选区（DocCandidate / ChatCandidate）
- DocCandidate：出现 Markdown 标题（# / ## / ###）或典型 Markdown 结构（列表/引用/表格），且不属于明显聊天格式的段落
- ChatCandidate：出现时间戳/发言人行/大量短句/@ 的段落
- Mixed：两者交错时，以“最近的 # 标题”到下一个同级 # 标题（或文本结束）视为 DocCandidate，其余归 ChatCandidate

Step 3 — Markdown 文档解析与切块（优先执行）
当存在 DocCandidate 时：
3.1 文档边界（多篇文档）
- 以一级标题 “# ” 作为新文档起点
- 若没有 #，但存在多个“文档抬头段”（如“标题：/背景：”）并以分隔线/空行隔开，也可作为边界
- 无法区分则视为单文档

3.2 文档内章节切块
- 优先按 “## ” 切；若无则按 “### ” 切
- 章节过长：在不破坏代码块的前提下，按空行/列表段落二次切分
- 每块建议 20~200 行；过短（<10行）可与相邻合并（同一 Section）
- 生成 DocTitle：取文档第一个 “# ” 标题；无则 UntitledDoc
- 生成 SectionTitle：取当前块的 “##/### ” 标题；无则 (NoSection)

3.3 【增强】MarkdownChunk 内聊天二次拆分（仅限代码块外）
对每个 MarkdownChunk，执行“聊天特征检测”（只在代码块外区域判定）：
- 若满足任一强特征，则触发二次拆分：
  A) 出现 >=5 行“<名字>: 或 <名字>：”发言人行
  B) 或出现 >=3 个时间戳（HH:MM / YYYY-MM-DD HH:MM）
  C) 或出现明显聊天导出格式（如多条短句+@+表情）且段落以换行分隔

二次拆分规则（只拆代码块外文本；代码块整体作为独立子块保留）：
- 先把 MarkdownChunk 按 code block 边界切成多个段：OutsideText / CodeBlock / OutsideText...
- 对 OutsideText 段按 Chat 规则拆分（见 Step 4.2）
- 将拆分后的每条聊天子块保留同一 DocTitle/SectionTitle（方便追溯它来自哪个文档章节）
- CodeBlock 段不拆，作为单独子块，SourceType 标记为 MarkdownDoc，FlowHint 标记为 ChatInCodeBlock（若内容像聊天）

Step 4 — 处理 ChatCandidate（以及未被 DocCandidate 覆盖的剩余文本）
4.1 判定 SourceType（Chat/Minutes/Transcript/Other）
- Minutes：包含“会议纪要/议题/结论/行动项/负责人/截止时间”等
- Transcript：口语化连续叙述/出现“转写/语音/嗯/然后/我们…”
- Chat：发言人行（<名字>: / <名字>：）或时间戳（HH:MM / YYYY-MM-DD HH:MM）或大量短句
否则 Other

4.2 Chat 拆分规则（用于 ChatCandidate 或 MarkdownChunk 的 OutsideText）
按优先级切块：
A) 时间戳行作为新块起点（YYYY-MM-DD HH:MM / MM-DD HH:MM / HH:MM）
B) 发言人行作为新块起点（<名字>: / <名字>：）
C) 空行分隔
目标粒度：每块 1~12 行；太长按空行二次切；太短可合并相邻同一发言人/同一时间段

4.3 Minutes/Transcript/Other 拆分
- Minutes：按标题/空行/项目符号段落切为 1~5 块
- Transcript：按空行/明显换段切为 1~5 块
- Other：默认不拆或按空行粗拆 1~3 块

Step 5 — 为每个 Chunk 生成 Entry 并追加到 raw 文件末尾
对每个 Chunk（来自：MarkdownChunk/其聊天子块/ChatCandidate/Minutes/Transcript等）：
1) EntryId：YYYYMMDD-HHMM-XXX（XXX 当日递增 3 位序号）
2) Keywords(3)：抽取 3 个关键词（系统/角色/动作/指标/实体名优先）
3) FlowHint：一句话流程环节猜测（发起/审核/支付/履约/退款/对账/客服/配置/报表等；不确定写“待判定”）
4) SourceType：
   - 来自 Markdown 结构则 MarkdownDoc
   - 来自 ChatCandidate 拆分则 Chat
   - Minutes/Transcript/Other 同名
5) 追加到 inputs/00_raw_feishu.md 末尾（不改写原文）：

---
[EntryId]: <EntryId>
[CapturedAt]: <当前日期时间>
[SourceType]: <MarkdownDoc/Chat/Minutes/Transcript/Other>
[DocTitle]: <若有则填，否则留空>
[SectionTitle]: <若有则填，否则留空>
[Notes]: <可选：来源群/会议/文档名；无则留空>

<原文开始>
<Chunk 原文（原样）>
<原文结束>

Step 6 — 更新索引目录（必须）
为本次新增每个 Entry 追加索引行：
| EntryId | CapturedAt | SourceType | DocTitle | SectionTitle | Keywords(3) | FlowHint |

Step 7 — decisions.md 记录（必须）
追加：
- 本次 /append 输入概况：
  - MarkdownDoc chunks 数量
  - Chat chunks 数量（包含 Markdown 内二次拆分产生的）
  - Minutes/Transcript/Other chunks 数量
  - 总新增 Entry 数量
- 提示下一步建议运行 /capture（默认 last=80）

[FILE:.claude/commands/capture.md]
目标：从 inputs/00_raw_feishu.md（飞书混合材料）中抽取 MIS（最小事实集）并生成流程骨架与确认点<=10。
性能策略：默认只分析“最新 80 条 Entry”，必要时额外拉取与冲突相关的旧 EntryId 作为证据。

输入：可选参数 $ARGUMENTS
- 支持：last=80（默认），可改为 last=50/120 等
- 示例：/capture last=80

执行步骤：
Step 0 — 参数解析与取数
1) 从 $ARGUMENTS 解析 last=N；若未提供则 last=80
2) 读取 inputs/00_raw_feishu.md 顶部索引表（# Raw Index）
3) 取索引中最新 N 条 EntryId 作为本轮“主分析集”
4) 从文件中定位并读取这些 EntryId 对应的原文块（保持原样）

Step 1 — 材料解析（事实 vs 观点）
- 抽取事实陈述（流程步骤/规则/口径/异常处理/责任归属）
- 抽取观点/建议/猜测（仅作为待确认候选）
- 抽取实体要素：角色/系统/数据对象/动作/时间阈值条件（如有）
约束：不得脑补；无法确定则标【待确认】。

Step 2 — 冲突识别与旧证据拉取
- 识别冲突/歧义（口径/边界/阈值/异常策略等）
- 冲突必须记录：说法A（EntryId+短句<=25字） vs 说法B（EntryId+短句<=25字）
- 如可从索引定位更早相关 Entry：额外拉取最多 10 条 Old Evidence EntryId（仅用于引用）

Step 3 — 生成 inputs/00_intake.md（覆盖）
输出 MIS（不确定写【待确认】）：
1) 目标（一句话）+ 受影响对象
2) 成功标准（指标/口径/期望变化）
3) 流程边界（触发点/结束点）
4) 参与方（角色/系统/数据对象）
5) 关键规则（最多3条)
6) 最常见异常/卡点（1条）
7) 真实案例（优先抽取1个；若无要求补1个）
8) 冲突与歧义（必须：冲突点 + EntryId 证据）
9) 证据范围说明（主分析集 N=last + Old Evidence EntryId）

Step 4 — 生成 outputs/01_process.md（覆盖：可讨论版）
结构固定，且不确定项标【待确认】：
1) 业务目标与范围（In/Out：无法判断写【待确认】）
2) 角色/系统/数据对象
3) 主流程（编号步骤+输入/输出+决策点；不确定步骤标【待确认】并引用 EntryId）
4) 异常/边界（最低覆盖集合：权限、超时/重试、重复提交/幂等、人工介入、对账/纠错/撤销/回退）
5) 状态机/生命周期（如适用；否则“无/待确认”）
6) 关键规则（校验/计算/路由/审批/幂等）
7) Open Questions（<=10，优先级；可秒回；含为何重要/风险/建议确认人；引用 EntryId）
8) 引用与依据（research/sources.md；如需引用但缺失，标【待补引用】并列检索主题Top3）

Step 5 — decisions.md 记录（追加）
- /capture 参数 last=N
- 主分析集 EntryId 范围 + Old Evidence（如有）
- Top10 确认点摘要（与 Open Questions 对齐）

[FILE:.claude/commands/confirm.md]
目标：把 decisions.md 最新一轮“确认点<=10 / Open Questions<=10”转成业务方可在飞书秒回的确认清单，并输出“可直接发送的飞书消息稿”。

执行：
1) 读取 decisions.md 最新确认点列表（若没有则从 outputs/01_process.md 的 Open Questions 抽取）
2) 生成两份输出：
A) 飞书可直接发送消息稿（便于复制发送）
- 开头一句：已基于聊天/纪要/文档整理流程与BRD初稿，需要确认以下 N 点
- 每条用 #1~#N，必须可秒回：
  - 是/否，或 A/B/C，或 填空<=10字
  - 给出“回复示例”（如“#2 选B”/“#5 是”）
- 结尾一句：确认后我会更新流程与BRD并回发

B) 结构化确认清单（<=10条，留档）
- #N 问题：
- 回复方式：
- 若不确认的风险：
- 依据：EntryId（没有则写“无直接证据，需业务确认”）
- 建议确认人：业务/财务/法务/技术（可多选）

3) 将 A) 与 B) 追加写入 decisions.md（标注“Feishu Message Draft”“Confirmation Checklist”）

[FILE:.claude/commands/run.md]
目标：一键生成流程文档与BRD并做门禁检查（Superpowers 优先，失败允许降级）。

输入：$ARGUMENTS（主题提示，可选）

执行顺序：
1) 若 inputs/00_intake.md 不存在或明显缺失信息：提示建议先运行 /capture（仍可继续生成“可讨论版”）
2) 生成/更新 outputs/01_process.md：
   - 优先 /superpowers:brainstorm；失败则降级并标注原因
3) 若发现合规/规范/口径触发点：
   - 列出待检索主题Top3与关键词；能检索则写入 research/sources.md 并回填引用；否则标【待补引用】
4) 生成/更新 outputs/02_brd.md：
   - 优先 /superpowers:write-plan → /superpowers:execute-plan
   - 失败则降级并标注原因
5) QA 门禁：把 checklist 与“最小补齐清单（3-8条）+ 下一次对齐议程（<=5条）”追加写入 decisions.md

固定结构要求：
- 01_process 覆盖异常最低集合（权限/超时重试/幂等/人工介入/对账纠错）
- 02_brd 必含：背景目标、现状痛点、目标流程、规则例外、In/Out、风险合规（带引用或待补）、未决与行动项
- 若降级：必须在 decisions.md 记录原因与影响范围

D) 初始化提示（写入 decisions.md）
- 你现在可以直接开始用 /append 粘贴混合内容，然后 /capture（默认 last=80），再 /confirm 发业务确认；确认后再 /run 生成BRD。
