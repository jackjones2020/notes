# Harness 的 10 个边界判断 — coding agent 骨架

- 作者：马东锡 NLP (@dongxi_nlp)
- 来源：https://x.com/i/status/2064098727867163124
- 摘录日期：2026-06-09
- 背景：马东锡用图解拆解 coding agent harness，核心论断「The harness is the product」

## 摘录

model 做出了一个 proposal。harness 接下来要回答一串问题：

1. src/config.py 是否在 workspace 内？
2. 这个 path 是否会通过 symlink 逃逸？
3. 这是新文件，还是 overwrite？
4. model 最近是否读过现有文件？
5. 已知的 file baseline 是否过期？
6. 这次 write 是否需要 human approval？
7. edit result 是否应该包含 diff summary？
8. 后面是否应该跑 validation command？
9. 有多少 output 应该进入下一轮 prompt？
10. 需要更新哪些 state，避免下一轮 stale？

这些判断不该交给 model 的随机处理。Prompt 可以描述期望行为。**Harness 负责落实边界。**

## 注解

对应 agent 骨架 10 根骨头：沙箱边界、安全检查、操作语义、上下文一致性、stale read 检测、权限门控、可审计性、质量门、上下文裁剪、记忆一致性。把这 10 个问题答好了，agent 就不会在编码时失控。
