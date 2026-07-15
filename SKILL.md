---
name: memory-dream
description: Use when the user asks to consolidate, dream, digest, sleep on, or reorganize scattered memories. Trigger phrases include "Memory Dream", "memory dream", "整理记忆", "记忆巩固", "睡一觉", "梦一下", "让记忆沉淀", "巩固一下", "consolidate memory", "digest my notes", "dream on it". Simulates sleep-based memory consolidation: pull recent fragments, cluster by theme, synthesize each cluster into one high-quality durable memory, and prune redundant traces with user confirmation. Not for routine memory search/add — use long-term-memory skill for those.
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

## Sleep-cycle stages (four phases, in order)

```
        ┌─────────┐   ┌──────────┐   ┌────────────┐   ┌─────────┐
  RECALL│ N1 Doze │ → │N2 Cluster│ → │N3 Consolid.│ → │REM Prune│
        └─────────┘   └──────────┘   └────────────┘   └─────────┘
        list/search    group by      synthesize        forget
        recent mems    theme+time    one durable       duplicates
                                     per cluster       (ask first)
```

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
2026-07 决策 · opencode 记忆维度用 2048（doubao-embedding-vision）。
   ↳ 覆盖 2026-06 早期 768 维草案（不再采用）。
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
Pruned: forget <P> 条（已用户确认）
Kept as-is: <singleton 数量> 条 singleton + <未确认删除数>

Superseded chains: <如有，列出> 
Conflicts flagged: <如有，列出>
```

## Anti-patterns

- Consolidating singletons (nothing to consolidate).
- Forgetting without preview + explicit yes.
- Rewriting a memory that already has `consolidated` tag (double consolidation drift).
- Merging memories that only share a superficial keyword — cluster on subject, not on words.
- Losing dates: the consolidated version must carry the earliest known date of the underlying facts.

## Interplay with long-term-memory skill

- `long-term-memory` handles single-shot `search` / `add` / `profile`.
- `memory-dream` runs a **batch** consolidation over many fragments.

Load `long-term-memory` first if you're unsure about the memory tool's basic shape.
