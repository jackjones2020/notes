# AGENTS — notes 仓库操作指南

## 仓库定位

这是 `k库 (ob-knowhow)` 的前置原料库。收到的段落、摘录、短想法先存这里，精炼后再进 k库。

## 生命周期

```
raw/ 笔记
  ↓ 整理归纳
wiki/sources/ 或 wiki/concepts/ 或 wiki/syntheses/
  ↓ 精炼完成，值得永久留存
k库 (ob-knowhow) 的 ingest 流程
```

## 写入流程

1. 新笔记 → 写入 `raw/<文件名>.md`
2. 内容较多/跨主题 → 拆为多个 raw 文件
3. 收集到一定量后 → 整理进 `wiki/sources/` 或 `wiki/concepts/`
4. 精炼稳定后 → ingest 进 `~/Work/wiki/ob-knowhow`

## 注意事项

- 不要一次性写过长内容到 raw（保持段落级、短文级粒度）
- 笔记格式：自由 Markdown，无强制 frontmatter
- 整理 wiki 内容时，可在文件头加 frontmatter 标记状态（draft/review/done）
- 不要跳过 notes 直接写 k库 —— 这是草稿缓冲层，保护 k库不被垃圾填满
