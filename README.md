# opencode-skill-memory-dream 🌙

> opencode 技能：睡眠隐喻的手动记忆巩固。触发之后，agent 走四个阶段 —— **N1 迷糊 → N2 聚类 → N3 巩固 → REM 修剪** —— 把零散的 `opencode-mem` 碎片压成一小组高信号、可持久的记忆。

隶属 [`opencode-codex-kit`](https://github.com/Yulimfish/opencode-codex-kit)。

## 四个阶段

```
 N1 迷糊 (Recall)     N2 聚类             N3 巩固               REM 修剪
 ─────────────────    ────────────────    ─────────────────     ─────────────
 拉最近 + 按 tag       按主题分组          每个 cluster 合成      预览列表 → 用户
 抓碎片；按话题/日期     → 3-8 个 cluster    1 条新记忆，带 tag     确认 → 忘掉原始
                       （语义，不是关键词）  consolidated /        （< 24h 跳过、
                                            dream-<YYYY-MM-DD>    pinned 跳过、
                                                                  画像事实保留）
```

## 触发词

`Memory Dream`、`memory dream`、`梦一下`、`睡一觉`、`整理记忆`、`记忆巩固`、`让记忆沉淀`、`巩固一下`、`consolidate memory`、`digest my notes`、`dream on it`。

## 硬约束

- REM 修剪**必须**给用户看完整预览列表，并且拿到明确的"确认"才能忘掉。
- 永远不修：< 24 小时的碎片、带 `pinned` / `永久` / `keep` 标签的、以及塑造用户画像的事实。
- 每条巩固后的新记忆必带 tag：`consolidated`、`dream-<YYYY-MM-DD>`、`<topic>`。被覆盖的旧记忆用 supersession 语句显式声明。

## 前置

需要 [`opencode-mem`](https://www.npmjs.com/package/opencode-mem)。搭配 [`opencode-skill-long-term-memory`](https://github.com/Yulimfish/opencode-skill-long-term-memory) 一起用最顺。

## 安装

```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/Yulimfish/opencode-skill-memory-dream.git \
  ~/.config/opencode/skills/memory-dream
```

## 许可

MIT © Yulimfish
