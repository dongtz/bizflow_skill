# Append 工作流：追加原始材料

将混合原始内容追加到 `inputs/00_raw.md`，自动识别类型、切块、生成 EntryId 并维护索引。

## 输入

用户粘贴的原始内容（可混合：IM 聊天 + 多篇 Markdown 文档 + 会议纪要 + 语音转写）。

## 关键约束

- 不改写原文，只做切块与标注
- 保留不可拆分单元的完整性（代码块、媒体块均不拆开，见 Step 1）
- 混合输入：优先解析 Markdown 文档块，再解析剩余聊天/转写块

---

## Step 0 — 初始化与索引准备

1. 若 `inputs/00_raw.md` 不存在则创建
2. 读取当前文件，定位当天已有 Entry 序号（用于递增）
3. 获取当前日期时间（用于 EntryId 的 YYYYMMDD-HHMM 部分）
4. 确保文件顶部存在 Raw Index 表；不存在则创建：

```markdown
# Raw Index
| EntryId | CapturedAt | SourceType | DocTitle | SectionTitle | Keywords(3) | FlowHint | Media |
|---------|------------|------------|----------|--------------|-------------|----------|-------|
```

## Step 1 — 预处理：保护不可拆分单元

以下内容视为**不可拆分单元**，切块时不得从中间断开：

### 1.1 代码块

fenced code block（`` ``` `` ... `` ``` ``）整体保留，记录边界。

### 1.2 媒体块（图片 / 视频）

**识别规则：**

| 媒体类型 | 识别模式 |
|----------|----------|
| Markdown 图片 | `![alt](path-or-url)` |
| HTML 图片 | `<img src="...">` |
| HTML 视频 | `<video ...>...</video>` |
| iframe 嵌入 | `<iframe ...>...</iframe>` |
| 纯视频链接行 | 独立一行且为视频 URL（.mp4/.mov/.webm 或含 bilibili/youtube/feishu 域名） |

**保护规则：**
- 媒体引用行 + 紧邻的上下文（前后各 ≤2 行非空行，如标题行、说明文字、标注）合并为一个"媒体块"
- 判定"紧邻"：媒体引用行与上/下文之间无空行，或仅有 1 个空行
- 多张连续图片（如截图序列）合并为一个媒体块
- 媒体块在后续步骤中作为不可拆分单元（与代码块同等对待）

**媒体标记：**
- 统计每个 Chunk 中的媒体数量，后续写入 Entry 的 `[Media]` 字段
- 格式：`img:N, video:M`（N/M 为数量；无媒体则留空）

## Step 2 — 粗分候选区

将输入分为两类候选区：

| 候选类型 | 判定特征 |
|----------|----------|
| **DocCandidate** | Markdown 标题（# / ## / ###）或典型 Markdown 结构（列表/引用/表格），且不属于明显聊天格式 |
| **ChatCandidate** | 时间戳行 / 发言人行 / 大量短句 / @ 提及 |
| **Mixed** | 两者交错时，以"最近的 # 标题"到下一个同级标题（或文本结束）为 DocCandidate，其余归 ChatCandidate |

## Step 3 — Markdown 文档解析与切块（优先执行）

当存在 DocCandidate 时：

### 3.1 文档边界识别（多篇文档）

- 以一级标题 `# ` 作为新文档起点
- 若无 `#`，但存在多个"文档抬头段"（如"标题：/背景："）且以分隔线/空行隔开，也可作为边界
- 无法区分则视为单文档

### 3.2 文档内章节切块

- 优先按 `## ` 切；若无则按 `### ` 切
- 章节过长时在不破坏代码块的前提下按空行/列表段落二次切分
- 每块建议 20~200 行；过短（<10 行）可与相邻合并（同一 Section）
- **DocTitle**：取文档第一个 `# ` 标题；无则 `UntitledDoc`
- **SectionTitle**：取当前块的 `## / ###` 标题；无则 `(NoSection)`

### 3.3 Markdown 内聊天二次拆分（仅限代码块外）

对每个 MarkdownChunk 执行"聊天特征检测"（只在代码块外区域判定）：

**触发条件（满足任一）：**
- A) 出现 ≥5 行 `<名字>:` 或 `<名字>：` 发言人行
- B) 出现 ≥3 个时间戳（HH:MM / YYYY-MM-DD HH:MM）
- C) 出现明显聊天导出格式（多条短句 + @ + 表情，段落以换行分隔）

**二次拆分规则（只拆保护单元以外的文本）：**
1. 将 MarkdownChunk 按保护单元（代码块 + 媒体块）边界切成段：`OutsideText / ProtectedBlock / OutsideText ...`
2. 对 OutsideText 段按 Chat 规则拆分（见 Step 4.2）
3. 拆分后的每条聊天子块保留同一 DocTitle/SectionTitle（便于追溯来自哪个文档章节）
4. ProtectedBlock（代码块或媒体块）不拆，作为单独子块：SourceType = `MarkdownDoc`

## Step 4 — 处理 ChatCandidate

处理未被 DocCandidate 覆盖的剩余文本。

### 4.1 判定 SourceType

| SourceType | 判定特征 |
|------------|----------|
| **Minutes** | 包含"会议纪要/议题/结论/行动项/负责人/截止时间"等 |
| **Transcript** | 口语化连续叙述 / "转写/语音/嗯/然后/我们…" |
| **Chat** | 发言人行 / 时间戳 / 大量短句 |
| **Other** | 以上均不匹配 |

### 4.2 Chat 拆分规则

按优先级切块：
1. 时间戳行作为新块起点（YYYY-MM-DD HH:MM / MM-DD HH:MM / HH:MM）
2. 发言人行作为新块起点（`<名字>:` / `<名字>：`）
3. 空行分隔

目标粒度：每块 1~12 行。太长按空行二次切；太短可合并相邻同一发言人/同一时间段。

### 4.3 Minutes / Transcript / Other 拆分

| 类型 | 拆分策略 |
|------|----------|
| Minutes | 按标题/空行/项目符号段落切为 1~5 块 |
| Transcript | 按空行/明显换段切为 1~5 块 |
| Other | 默认不拆或按空行粗拆 1~3 块 |

## Step 5 — 为每个 Chunk 生成 Entry 并追加

对每个 Chunk 执行：

1. **EntryId**：`YYYYMMDD-HHMM-XXX`（XXX 当日递增 3 位序号）
2. **Keywords(3)**：抽取 3 个关键词（系统/角色/动作/指标/实体名优先）
3. **FlowHint**：一句话流程环节猜测（发起/审核/支付/履约/退款/对账/客服/配置/报表等；不确定写"待判定"）
4. **SourceType**：MarkdownDoc / Chat / Minutes / Transcript / Other
5. **Media**：该 Chunk 中的媒体统计（如 `img:2, video:1`；无媒体则留空）

追加到 `inputs/00_raw.md` 末尾（不改写已有内容）：

```markdown
---
[EntryId]: YYYYMMDD-HHMM-XXX
[CapturedAt]: <当前日期时间>
[SourceType]: <类型>
[DocTitle]: <若有则填，否则留空>
[SectionTitle]: <若有则填，否则留空>
[Media]: <img:N, video:M；无则留空>
[Notes]: <来源群/会议/文档名；无则留空>

<原文开始>
<Chunk 原文（原样保留，包括图片/视频引用）>
<原文结束>
```

## Step 6 — 更新索引目录

为本次新增的每个 Entry 追加索引行到 Raw Index 表。

## Step 7 — decisions.md 记录

追加本次操作记录：

```markdown
## Append - <日期时间>
- MarkdownDoc chunks：N 条
- Chat chunks：N 条（含 Markdown 内二次拆分）
- Minutes/Transcript/Other chunks：N 条
- 总新增 Entry：N 条
```

## 完成后提示（必须向用户输出）

```
---
✅ 追加材料完成
   新增 N 条 Entry（文档 X 条 / 聊天 Y 条 / 纪要 Z 条），当前共 M 条

👉 下一步：提炼流程
   说"提炼流程"，我会从材料中提取流程骨架和需确认的问题

💡 你也可以：
   - 继续粘贴更多材料（可多次追加，不着急一次给全）
   - 说"提炼流程 last=120"调整分析范围（默认最新 80 条）
---
```
