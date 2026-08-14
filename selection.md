# 选题记录 · awaitscan

## 背景
- 时间：2026-08-14（Overlord 单项目深耕，每两小时一轮）
- 上一项目：a11ydoctor（已上架，family 四件套：devdoctor+testlite+debtlens+a11ydoctor）
- `next_action: select` → 触发 Phase A 开会选题

## 候选池（core-memory 落选萃取）
- `awaitscan`（perfscan 复活切口）· `nono` · `Book-to-Skill` · `formlite` · `Jay`

## 决策：复活 awaitscan（perfscan 落选萃取的单一切口）
- 来源：core-memory 第 190–193 条。`perfscan`（零依赖性能反模式扫描）整体被 oxlint（Rust/零配置/700+规则含 perf 类别）占领，不满足三维硬标准第③条；但其「仅检测循环内串行 await / 嵌套 O(n²) 循环」这一特定高信号切口，oxlint perf 未聚焦「串行 await 语义」，可复活为 `awaitscan` 单一切口。
- 符合策略线：core-memory 第 140/142 条裁定「零依赖切口枯竭 → 转质量深化 / 组合叙事」，family 已四件套，awaitscan 补第五切面「性能/可维护性」，延续「复活落选萃取单一切口」。

## 三维硬标准（马斯克终裁门槛）
1. **足够大受众量**：任何 Node/JS/TS 项目都有 async/await，循环内串行 await 是超高频性能 bug，受众极大。✓
2. **新机会实用性**：零依赖静态扫描循环内串行 await + map 内 await + 嵌套循环，确定性、可进 CI，实用性强。✓
3. **比竞品更好或市场无**：oxlint 需 Rust 工具链/ESLint 配置、axe 需浏览器；我们纯 JS 层零依赖单文件静态扫描「串行 await 语义」这一 oxlint 未聚焦的切口，差异化成立。✓

## 终裁结论
马斯克终裁：采纳 awaitscan。理由——它把「异步代码里最容易被忽略却最影响吞吐」的一类反模式（串行 await / map 内 await）做成确定性、零依赖、可进 CI 的一把尺子，且正好补齐 family「代码健康」五件套的最后一维（性能），抗切口枯竭能力强。落选的 nono/Book-to-Skill/formlite/Jay 仍偏离零依赖基线，暂不复活。
