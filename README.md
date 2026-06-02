# notes

段落、短文的临时仓库存放处，作为 `k库 (ob-knowhow)` 的前置原料库。

## 定位

```
你收到一段文字/笔记
  → 扔进 notes/raw/（临时存放）
  → 后续整理、关联、扩充
  → 提炼为 wiki/ 下的结构化内容
  → 最终 ingest 进 k库 (ob-knowhow) 成为永久知识
```

**notes** 是**草稿层**和**加工层**，**k库** 是**成品层**。

| 层 | 仓库 | 内容状态 | 更新频率 |
|---|------|---------|---------|
| 草稿 | notes/raw/ | 零散段落、摘录、想法 | 随时 |
| 加工 | notes/wiki/ | 初步归纳、关联后的短文 | 按需 |
| 成品 | ob-knowhow | 结构化概念、术语、来源摘要 | 精炼后 |

## 架构

遵循 Karpathy-style LLM Wiki 四层模型：

```
raw/                    只读真相源，原始笔记/摘录
  └── *.md              → 一条笔记一个文件，标题即文件名

wiki/                  加工后的结构化知识
  ├── sources/          → 来源摘要（收集多个 raw 后的归纳）
  ├── concepts/         → 初步概念提炼（future ingest to k库）
  ├── syntheses/        → 跨来源综合
  ├── comparisons/      → 概念对比
  ├── maps/             → 知识地图
  ├── glossary.md       → 术语记录
  ├── index.md          → 仓库导航
  └── log.md            → 维护历史

meta/                  脚本自动生成的状态（只读参考）
scripts/               维护脚本
prompts/               LLM 提示词
templates/             页面模板
memory/                构建记忆（原则、错误、决策、批次复盘）
docs/                  设计稿
```

## 写入规则

- `raw/` 只新增，不删改已有文件
- 每条笔记一个文件，文件名简短达意
- 笔记尾部可标注 `# 待归纳` `# 待关联` 等标签
- 定期整理 raw → wiki，清理已加工的 raw
