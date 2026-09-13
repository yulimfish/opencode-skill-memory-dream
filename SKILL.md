---
name: memory-dream
description: Use when the user asks to consolidate, dream, digest, sleep on, or reorganize scattered memories. Trigger phrases include "Memory Dream", "memory dream", "整理记忆", "记忆巩固", "睡一觉", "梦一下", "让记忆沉淀", "巩固一下", "consolidate memory", "digest my notes", "dream on it", "记忆清理", "遗忘扫描", "清理旧记忆", "forget old", "memory decay", "健康检查", "health check". Simulates sleep-based memory consolidation: pull recent fragments, cluster by theme, synthesize each cluster into one high-quality durable memory, prune redundant traces, and score all memories against a decay model to identify stale/cold memories for user-confirmed deletion. When user asks specifically about "清理" or "遗忘", run only the NREM health check + REM Prune (skip N1–N3). Not for routine memory search/add — use long-term-memory skill for those.
---

# Memory Dream 🌙

Sleep is when the brain replays the day, extracts patterns, and consolidates loose fragments into stable memory. This skill does the same for `opencode-mem`.

## When to run

- User says "Memory Dream" / "梦一下" / "整理记忆" / "巩固一下" / "digest / consolidate".
- Post-heavy-session cleanup: dozens of small captures piled up, obvious duplicates.
- User feels "记忆有点乱" or wants a tidy view before starting new work.

Do NOT run it:

- As a routine step in every session (auto-dedup already handles the common case).
- Right after a fresh session with < 5 new memories — nothing to consolidate.
- Without user consent when `forget` will be invoked (destructive).

## Sleep-cycle stages (five phases, in order)

```
         ┌─────────┐   ┌──────────┐   ┌────────────┐   ┌───────────┐   ┌─────────┐
   RECALL│ N1 Doze │ → │N2 Cluster│ → │N3 Consolid.│ → │NREM Decay │ → │REM Prune│
         └─────────┘   └──────────┘   └────────────┘   └───────────┘   └─────────┘
         list/search    group by      synthesize        score by       forget
         recent mems    theme+time    one durable       decay model    (ask first)
                                       per cluster      (R-score)
```

**Fast path**: if user only asked for "清理" / "遗忘" / "health check" → skip N1–N3, run only NREM + REM.

### N1 — Doze (Recall)

Pull the fragments to work on. Default sweep = recent 30–50 project memories.

```
memory({ mode: "list", limit: 50 })
```

For a focused dream (user gave a theme, e.g. "关于 opencode 配置的记忆"):

```
memory({ mode: "search", query: "<theme>", scope: "project", limit: 30 })
memory({ mode: "search", query: "<theme>", scope: "all-projects", limit: 20 })
```

Skip N1 if user provided the fragments inline.

### N2 — Cluster

Group the fragments. Cheap heuristic (no extra tool calls):

- **Same tags** → same cluster.
- **Same topic keyword in content** (e.g. "opencode-mem", "yuque", "cc-switch") → same cluster.
- **Same date + adjacent topics** → same cluster.

For each cluster, extract:
- Common subject (1 line)
- All decisions / facts / pitfalls mentioned across fragments
- Dates: earliest → latest (to establish "known since")
- Conflicts between fragments (if any — flag for user)

Ignore singletons (1 fragment = nothing to consolidate).

### N3 — Consolidate

For each cluster with ≥ 2 fragments, write **one** consolidated memory following the shape from `long-term-memory`:

```
<Good shape>
YYYY-MM 主题 · 一句核心结论。
补充：具体字段/路径/参数。
原因/证据：为什么这么定。
```

Add it via:

```
memory({
  mode: "add",
  content: "<consolidated text>",
  tags: "consolidated, dream-<YYYY-MM-DD>, <topic-tag>"
})
```

Tag rule: always include `consolidated` and `dream-<date>` so future dreams can distinguish already-consolidated memories from raw fragments.

**Conflict handling**: if two fragments disagree (e.g. old decision superseded by new one), keep only the newer decision in the consolidated version. Note the supersession explicitly:

```
2026-08 决策 · opencode 记忆维度用 1024（qwen3.7-text-embedding）。
   ↳ 覆盖 2026-07 的 2048 维方案（embedding 模型已更换，不再采用）。
```

### REM — Prune (destructive, confirm first)

Show the user a preview **before** any `forget` call:

```
准备把以下 5 条零散记忆合并成 1 条 consolidated：
  · mem_xxx1 (2026-07-11) opencode-mem embedding 维度
  · mem_xxx2 (2026-07-13) 修维度 bug
  · mem_xxx3 (2026-07-15) shim :4748 端口
  · mem_xxx4 (2026-07-15) 字段名 embeddingDimensions
  · mem_xxx5 (2026-07-15) 静默降维踩坑
→ consolidated 新条 mem_yyyy 已写入。
是否 forget 上述 5 条？(yes/no/skip)
```

Only after explicit "yes" call `memory({mode:"forget", memoryId})` per id.

If user says "no" / "skip" → keep the fragments alongside the consolidated version. Add `superseded-by:mem_yyyy` tag in future adds when possible.

**Never** forget:
- User-profile-affecting facts (identity, preferences, workflows).
- Anything tagged `pinned` / `永久` / `keep`.
- Fragments < 24 h old (too fresh, might still be useful in raw form).

### NREM — Decay Scan ⏳ (记忆衰减扫描)

This phase mimics the **human brain's memory decay and forgetting curve** (Ebbinghaus). The brain does not remember everything equally — it prioritizes memories that are recent, frequently accessed, and emotionally/conceptually important. This phase applies the same logic to opencode-mem.

**When to run NREM independently:**
- User says "清理记忆" / "遗忘扫描" / "forget old" / "memory decay" / "健康检查".
- When N1–N3 consolidation is not needed (no fragments to merge), but the memory bank feels bloated.
- Can be triggered alone: skip N1–N3 and go straight to NREM + REM.

```
         ┌───────────────────────┐   ┌──────────┐
  SCORE  │ NREM Decay Scan      │→  │REM Prune │
         └───────────────────────┘   └──────────┘
         score all memories         forget
         by R-score                (ask first)
```

---

## Memory Retention Score (R-Score)

Each memory is scored on a **0–5 continuous scale**. The lower the score, the stronger the forget candidate.

```
R = I × F / (1 + D × r)
```

| Symbol | Name | Range | Meaning |
|--------|------|-------|---------|
| **I** | Importance | 1–5 | Inherent importance of the memory (see table below) |
| **F** | Freshness boost | 0.3–3.0 | How recently/frequently accessed |
| **D** | Age (days) | 0–∞ | Days since `createdAt` |
| **r** | Decay rate | 0.03–0.15 | How fast this type decays |

### Importance (I) — Derivation Table

Derive I from the memory's **tags**, **content keywords**, and **connections**:

| I | Label | Tags / Signals | Examples |
|---|-------|----------------|----------|
| 5 | Critical | `pinned`, `永久`, `keep`, `rule`, `hard`; user identity/preferences; API keys/config rules | "语雀 public by default (hard)" |
| 4 | Important | `decision`, `architecture`, `pitfall`, `release`; linked by ≥ 3 other memories | "2026-07 决策 · embedding 维度用 2048" |
| 3 | Useful | `reference`, `workflow`, `pattern`; linked by 1–2 memories | "opencode-mem WebUI 缓存同步路径" |
| 2 | Routine | no significant tags; standalone fact | "今天修了一个类型错误" |
| 1 | Low-value | raw capture fragment, auto-generated, short content (< 30 chars) | "好" / "知道了" / 自动截取的半句 |

**Boost I by +1** if the memory is **linked** (via `linkedPromptId`/`linkedMemoryId`) by ≥ 5 other memories (hubs are important).

### Freshness Boost (F) — Access Recency

opencode-mem tracks `createdAt` and `lastAnalyzedAt`. Use these to estimate access freshness:

| F | Condition |
|---|-----------|
| 3.0 | Accessed or created within last 24 h |
| 2.0 | Accessed within last 7 days |
| 1.5 | Accessed within last 30 days |
| 1.0 | Accessed within last 90 days |
| 0.6 | Accessed 90–180 days ago |
| 0.3 | Never accessed OR > 180 days since last access |

If opencode-mem does not expose per-memory access timestamps, use `createdAt` as a fallback: F = 3.0 for < 24 h, 2.0 for < 7 d, etc., same mapping.

### Decay Rate (r) — Per-Type

| Memory type | r | Rationale |
|-------------|---|-----------|
| `rule` / `decision` / `pitfall` | 0.03 | Slow decay — rules should last |
| `preference` / `workflow` | 0.05 | Slow decay — habits persist |
| `reference` / `pattern` | 0.08 | Moderate decay |
| Raw fragment / auto-capture | 0.15 | Fast decay — fragments are transient |

Derive from tags and content keywords.

---

## Five Memory Tiers

After scoring, each memory falls into a tier:

| Tier | Name | R Range | Brain Analogy | Action |
|------|------|---------|---------------|--------|
| **T4** | Core | ≥ 3.0 | Long-term potentiated | **Permanent** — never suggest forget. Add `keep` tag if missing. |
| **T3** | Active | 2.0–3.0 | Frequently rehearsed | **Keep** — re-evaluate in 30+ days. |
| **T2** | Stable | 1.0–2.0 | Consolidated | **Keep** — re-evaluate in 90+ days. |
| **T1** | Fading | 0.5–1.0 | Interference-prone | **Review** candidate — show user with context. |
| **T0** | Decayed | < 0.5 | Trace decay | **Forget** candidate — strong suggestion to delete. |

**T1 and T0 are presented to the user for confirmation.** Never auto-delete.

---

## NREM Execution Procedure

### Step 1: Pull all project memories

```
memory({ mode: "list", limit: 200 })
```

If > 200 exist, paginate through all. (For `all-projects` scope, search separately.)

### Step 2: Score each memory

For every memory, compute R using the formula above. Store results in a table:

| mem_id | I | F | D (days) | r | **R** | Tier | Summary |
|--------|---|---|----------|---|-------|------|---------|

### Step 3: Group and present

Present results in this format:

```
⏳ Memory Health Check 完成

Scanned: <N> 条记忆
T4 Core:        <n4> 条 — 永久保留
T3 Active:      <n3> 条 — 长期保留
T2 Stable:      <n2> 条 — 短期保留
────────────────────────────────
T1 Fading:      <n1> 条 — 审查候选 ⚠️
T0 Decayed:     <n0> 条 — 遗忘候选 💤

──────── T1 Fading ────────
  mem_xxx1 | R=0.8 | 2026-04-12 | "某个早期配置记录"
    → 180 天前创建，未被引用，无重要标签
  mem_xxx2 | R=0.6 | 2026-03-01 | "一条自动捕获的碎片"
    → 内容短 (< 30 字)，无标签，从未被检索

──────── T0 Decayed ────────
  mem_ccc1 | R=0.3 | 2026-01-20 | "已废弃的临时方案"
    → 旧决策已被 supersede，无引用，自动捕获
```

### Step 4: Confirmation prompt

```
是否 forget 上述 <n1+n0> 条记忆？回复格式：
  · "all" — forget 全部 T0 + T1
  · "T0 only" — 只 forget T0
  · "keep mem_xxx1 mem_xxx2" — 保留指定，忘掉其余
  · "skip" — 全部保留
```

Only after explicit confirmation call `memory({mode:"forget", memoryId})` per id.

### Step 5: Post-forget summary

```
🧹 Memory Decay Cleanup 完成

Forgotten: <P> 条 (T0: <x> 条, T1: <y> 条)
Kept: <K> 条 (含用户指定保留的 <keep_ids>)
New total: <N-P> 条记忆
Freed: ~<estimated token budget> tokens
```

---

## Hard Constraints (decay edition)

- **Never auto-forget.** All forgets require explicit user confirmation.
- **Never forget T3/T4** memories, even if user says "all".
- **Never forget** anything tagged `pinned` / `永久` / `keep` / `rule` / `hard` — these are I=5 by definition.
- **Never forget** user-profile-affecting facts (identity, preferences, workflows).
- **Never forget** the **newest** memory in any cluster of related memories (keep at least one anchor).
- **Conflicts**: if a consolidation (N3) and a decay scan (NREM) flag the same memory, run consolidation first, then re-score.

## Full example (dry-run)

User: "梦一下最近关于语雀同步的记忆"

```
1. Recall
   memory({mode:"search", query:"语雀 同步", scope:"project", limit:20})
   → 8 fragments returned

2. Cluster
   Cluster A: yuque title 格式（3 条 · 2026-07-11 → 2026-07-15）
   Cluster B: 归档路径 hyghh/小记/年/月（4 条 · 2026-07-15）
   Cluster C: create_doc 落到根部要 update_toc 移动（1 条 · 2026-07-15）
   → C 是 singleton，跳过；A、B 进入 N3

3. Consolidate
   写入 2 条 consolidated:
   - A: 2026-07 决策 · 语雀 title 用 YYYY.MM.DD｜<title>，
        分隔符全角竖线 U+FF5C。本地文件名保持 ISO 破折号形式。
   - B: 2026-07 归档路径 · workspace 文档同步到 yulimfish/hyghh
        / 🥳 小记 (P3HgIvSxDf_LrLul) / 当前年 / 本月 (level 3)。

4. Prune
   Preview 7 条待 forget → 等用户确认 → yes → 逐条 forget。
```

## Output shape

After dreaming, report in this fixed shape:

```
🌙 Memory Dream 完成

Recalled: <N> 条
Clusters: <M> 个（≥2 条的簇 <K> 个）
Consolidated: 写入 <K> 条新 memory
  · mem_xxx (tags: consolidated, dream-<date>, <topic>)
  · ...
Pruned (consolidation): forget <P1> 条（已用户确认）
Kept as-is: <singleton 数量> 条 singleton + <未确认删除数>

⏳ Decay Scan:
Scanned: <N2> 条
T4 Core: <n4> · T3 Active: <n3> · T2 Stable: <n2>
T1 Fading: <n1> · T0 Decayed: <n0>
Pruned (decay): forget <P2> 条（已用户确认）

Superseded chains: <如有，列出> 
Conflicts flagged: <如有，列出>
```

## Anti-patterns

- Consolidating singletons (nothing to consolidate).
- Forgetting without preview + explicit yes.
- Rewriting a memory that already has `consolidated` tag (double consolidation drift).
- Merging memories that only share a superficial keyword — cluster on subject, not on words.
- Losing dates: the consolidated version must carry the earliest known date of the underlying facts.
- Running decay scan too often (< 7 days since last scan): scores barely shift.
- Flagging a memory as T0/T1 just because it's old — check I (importance) and F (freshness) first. An old decision that's still referenced is T3+, not T1.
- Forgetting the newest memory in a topic cluster: always keep at least one anchor.

## Interplay with long-term-memory skill

- `long-term-memory` handles single-shot `search` / `add` / `profile`.
- `memory-dream` runs a **batch** consolidation over many fragments.
- The NREM decay scan can also be triggered standalone (without N1–N3).
- After a decay-driven forget, run `memory({mode:"search", query:"<latest topic>"})` to ensure critical recent context is intact.

Load `long-term-memory` first if you're unsure about the memory tool's basic shape.
