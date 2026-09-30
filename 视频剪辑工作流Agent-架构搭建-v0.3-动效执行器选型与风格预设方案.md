# AI 视频剪辑工作流 Agent · 架构搭建 v0.3 —— 动效执行器选型与风格预设方案

> 状态：架构迭代（2026-10-01）｜v0.2 → v0.3 变更见「修订记录」
> 一句话定位：在 v0.2（剪辑执行器操作层）基础上，完成**动效生成模块的选型收窄**——主线定为"代码即视频（确定性渲染）"，配套三层质量保障（组件库 + 审美指导 + design brief），并新增**制作前人工选风格预设**节点。

---

## 修订记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v0.1 | 2026-09-30 | 架构定稿：四层 + 内核 + 时间轴 JSON schema + 执行器接口 + 机检清单 + 人审工作台 |
| v0.2 | 2026-10-01 | 剪辑执行器操作层；词级时间戳 + 语义锚点；评审留痕/人审映射；社区调研吸收 |
| **v0.3** | **2026-10-01** | **动效生成模块调研与候选**：① 动效工具调研结论（**代码即视频 = 执行器渲染层候选之一，非方案主线**；方案主体 = 编排层语义判断；三层质量保障）；② Claude Opus 5.5 逐帧动画案例核实（技术原理 + 架构印证）；③ 真实采用与质感调研（Remotion 生产级证据 + "flat and generic" 教训）；④ 社区预设对标与采用建议（Remotion 官方 skill / HyperFrames / video-shotcraft 等，效果图证据）；⑤ 新增**风格预设选择节点**（制作前人选风格 → 注入 aesthetic-guide.json）；⑥ 五类动效事件 × 执行器映射更新（候选） |

---

## 0. 本次迭代解决的问题

v0.2 遗留缺口：**动效生成模块（五类执行器）只到选型级，模块内部未设计**；审美指导文件（aesthetic-guide.json）是它的第一个输入缺口——执行器首次选色/字体/特效/风格缺决策依据。

v0.3 回答三个问题：

1. **动效执行器走哪条技术路线？** → **渲染层候选**：代码即视频（确定性渲染）优先，diffusion 生成作 illustration/anim 候选——但这是执行器层的技术选项，**不是方案主线**；方案（怎么编排、语义判断用哪个工具）由编排层制定，等真实运行数据再锁定（见第 1/2/7 章）。
2. **质感从哪来？** → 三层质量保障：组件库（地板，不可能丑）+ aesthetic-guide.json（方向，不许跑偏）+ design brief/参考素材（目标，要长什么样）。不是玄学提示词（见第 3 章）。
3. **风格方向谁定？** → 人。新增**风格预设选择节点**：视频制作前，人从风格预设库（效果图对标后）选定基调，锁定后注入 aesthetic-guide.json（见第 5 章）。

---

## 1. 动效工具调研结论：代码即视频为主线

### 1.1 三条技术路线分法

| 路线 | 代表 | 特点 | 结论 |
|---|---|---|---|
| **A. 代码即视频** | Remotion v3 + Agent Skills / HyperFrames / Motion Canvas / 自建 SVG | 确定性渲染：帧 n = f(时间)，连贯性由函数保证；可控、可校验、可批量 | **渲染层首选候选**：overlay / card / diagram 优先（可替换） |
| B. 生成式（diffusion） | 文生图 / 文生视频 | 视觉上限高但不可控：逐帧采样漂移、随机错帧 | 仅 illustration / anim 中"没见过的东西"用 |
| C. 模板式 | 剪映 / CapCut | 无开放自动化 API，不能当执行器 | 已排除（仅人审台/交互范式参考） |

### 1.2 五类动效事件 × 执行器映射（更新 v0.2 §3.3）

| 执行器 | 吃的事件 | 产物 | v0.3 选型 |
|---|---|---|---|
| 剪辑执行器 | 整个任务 | 粗剪.mp4 + cut-list + timeline_base | **自建**（faster-whisper + 规则脚本 + ffmpeg，v0.2 §3.2，无变更） |
| overlay 渲染器 | `overlay` | 引导层（圈选/箭头/高亮/标注） | **Remotion 官方 skill（首选候选）** + HyperFrames 组件；自建 SVG 为替换候选 |
| card 渲染器 | `card` | 关键词卡图层 | Remotion（/remotion-captions 字幕、卡片组件） |
| diagram 生成器 | `diagram` | 结构图/思维导图/数据图表 | HyperFrames 模板（Decision Tree / NYT-Style Chart / Takram Organic）+ D3/Recharts（Remotion 内） |
| illustration 生成器 | `illustration` | 解释性图片 | vox-director（Vox 纸拼贴科普质感）；文生图服务为替换 |
| anim 生成器 | `anim` | 视频片段 | video-shotcraft（场景级方法论 + Remotion 模板）；文生视频 + motion-reel 为替换 |

> 选型依据：全部候选**带效果图/用户产出证据**，按"效果图对标"逻辑采用（第 4 章）。

---

## 2. Claude Opus 5.5 逐帧动画案例：技术原理与架构印证

### 2.1 事实核实（多重信源交叉验证）

- 确有其事：2026-09-25 发布一周内 viral（Reddit / 抖音 / 微博 / YouTube 全线）。
- 一句话原理（微博"宝玉xp"）：**Claude 不是在生成每一帧，而是在写每一帧**——Opus 5.5 只输出文字，先写完整动画代码，浏览器/渲染器一帧一帧算出来，ffmpeg 合成 MP4。视频本质 = 每秒 24~60 张连续图片。

### 2.2 技术栈三路线（本质相同：确定性渲染）

| 路线 | 做法 | 案例 |
|---|---|---|
| ① 浏览器动画 + Playwright 逐帧截图 | JS + Playwright + ffmpeg，无视频生成模型 | Deedy 案例："The film is a web page, screenshotted" |
| ② Three.js / WebGL 逐帧计算 | 帧由计算产出 | 音乐卡点短片（节拍/鼓点/逐词时间轴写进代码） |
| ③ Remotion / headless Chromium 逐帧渲染 | React 组件渲染成帧 | 120s 1080p60 预告 one-shot（1M context） |

### 2.3 三个关键问题的答案（用户三连追问的结论）

1. **编排颗粒度**：场景级，不是逐帧也不是一口气。先写 design bible（全局风格/色板/字体约束）→ 拆场景（scene）→ 每场景独立代码 + 独立校验（`hyperframes check` 0 errors）→ 按分镜/节拍表合成。**对应我们的架构**：design bible ≈ aesthetic-guide.json；scene ≈ 语义组；独立校验 ≈ 机检；独立重做 ≈ 重做预算。
2. **如何知道自己写了什么**：代码即规格。Claude 推理代码语义（坐标/数值/缓动曲线都是写死的），验证靠中间产物：静态校验 + 抽帧（Playwright 关键帧）+ 单帧渲染（Remotion 预览）。错误是系统性的，抽几帧就够发现。
3. **连贯性如何保证**：**帧不是"一张张画对的"，是"同一个函数算出来的"**——帧 n = f(帧号/时间)，连贯性由函数连续性保证，不存在"随机错一张"。diffusion 才有逐帧采样漂移问题（脸崩/闪烁）。Claude 写错的风险是语义级（位置/风格/节奏理解错），不是像素级；修复方式是改代码参数重渲。

### 2.4 对架构的印证

| Claude 逐帧管线 | 我们的设计 | 结论 |
|---|---|---|
| design bible（全局风格约束） | aesthetic-guide.json | 同构，方向正确 |
| scene 拆分 + 独立校验 | 语义组 + 单事件重做预算 | 独立单元可重做、可并行 |
| `hyperframes check` 静态校验 | 机检层（抽帧/编码/锚点规则） | 校验靠中间产物，不靠逐帧预览 |
| 确定性程序动画 | overlay/card/diagram/anim 执行器渲染层候选 | **渲染层优先候选，非方案主线**；方案由编排层语义判断，执行器选型待运行数据锁定 |

---

## 3. 真实采用与质感调研：效果如何、用的人多不多

### 3.1 真实采用证据（Remotion 已是生产级）

| 使用者 | 干什么 | 结果 |
|---|---|---|
| Spotify Wrapped | 年度音乐回顾，数据驱动、数千万用户每人一条 | 标志性案例，顶级质感 |
| Submagic | AI 字幕工具，参数化模板 + 高度自动化 | 3 个月 $1M ARR、300 万用户 |
| AIVideo.com / Icon.me | AI 视频平台 / 病毒级广告 | 不到一年 $1M 收入 |
| SaaS 创业团队 | 12 条产品演示视频 | 动效工作室报价 $15,000/6 周 → Claude+Remotion 4 天零成本，转化 +340% |
| 电商 | 350 个产品批量生成视频变体 | 一夜完成，改价只重渲变化行 |
| Qubika（SaaS） | 5,000 条个性化 onboarding 视频 | 邮件互动 +78%、生产时间 -94% |
| 房地产 | 每套房视频 | 成本从 $1,800/套 → 输描述+审查 |

**画像**：技术团队 / 自动化营销 / 数据驱动视频里已是主流；独立创作者 2026 年正在爆发（Claude/Codex+Remotion 教程潮）。普通大众仍在剪映/GUI，但门槛正被 AI 抹平。

### 3.2 质感分区（诚实边界）

- **成熟区（质感已顶级）**：数据动效/图表（D3/Recharts）、文字动画/字幕/关键词卡、剪纸拼贴科普（Vox 风格）、品牌动效模板/批量个性化、地图动画。案例：Spotify Wrapped、Vox 复制品。
- **可用区**：复杂转场/镜头运动、界面动效/产品演示、漫画分镜动画。
- **短板区（别指望代码）**：写实/摄影级、3D 复杂渲染、手绘艺术风、真人表演——这些仍靠 AE / 3D / diffusion。

> **关键：我们的需求（引导层 + 解释层 + 图表 + 科普质感）恰好全部落在成熟区。**

### 3.3 质感来源结论（回答"需不需要特殊提示词"）

- **不需要玄学提示词，需要结构化约束**。
- 渲染器零下限（裸代码没有审美）；LLM 默认输出 = "flat and generic"（平淡普通）——不丑但绝无质感（真实实践者原话）。
- 质感 = **组件库（下限）+ 风格锁定（方向）+ 参考素材（目标）+ 人审迭代（校准）**。Vox 复制品案例的第一原则就是"lock one shared background, font stack, and accent palette"（锁定共享背景/字体栈/强调色板）。

---

## 4. 社区预设对标与采用建议（效果图证据）

### 4.1 候选池

| 预设 | 形态 | 效果图证据 | 质感评估 | 采用建议 |
|---|---|---|---|---|
| **Remotion 官方 Agent Skill**（remotion-dev/skills） | Skill 包（/remotion-create、/remotion-captions、/remotion-interactivity 等） | 官方 demo 6M+ views（48 小时）；抖音博主成片约 2000 万播放；装机量 150K→44 万；用户实测：Polymarket 视频 30 分钟 4-5 提示词 | **最硬**：效果图最多、生态最大、字幕/标注/交互全覆盖 | **✅ 采用**：overlay/card/diagram 主执行器 |
| **HyperFrames 设计系统**（premade frames + FRAME.MD） | 设计主题库 + 模板库 | hyperframes.dev/design 有 12+ 主题视觉展示；OpenDesign 有 20+ 模板**每套带视频 demo**（Decision Tree / NYT-Style Chart / Takram Organic / Vignelli 等） | **风格上限最高**：Biennale Yellow / BlockFrame / Capsule / Cartesian / Creative Mode 等 | **✅ 采用**：aesthetic-guide.json 的风格候选库 + diagram 模板 |
| **video-shotcraft** | Skill 方法论 + 模板 + 风格库 | 自带 showcase：38 秒成片由 skill 自产（github.io 可看）；**157 镜头配方卡 · 214 风格 · 214 运动预览**；一键换主题（Ink Press / Modern Light / Midnight / Sage / Coral / Iris 等） | 完整管线、场景级方法论；运动语言提炼自 ClickUp/Perplexity/Slack/Notion/Figma 等官方产品影片 | **✅ 采用**：风格预设库（效果图对标载体）+ anim 方法论 + 音效库（149 SFX） |
| vox-director | Skill（Vox 纸拼贴动画） | 描述完整（剪纸拼贴+VO+音乐+字幕），单 API key+ffmpeg | Vox 科普质感 | **参考**：illustration 科普质感 |
| tutorial-video（ClaudSkills） | Skill（Fireship-quality 教程视频） | 直接对标 Fireship 频道（程序化视频标杆）；Code Hike 动画代码转换、auto-zoom 标注 | 教程/口播+操作演示质感成熟 | **参考**：引导层（高亮/聚焦/标注） |
| Claude Remotion Skill（claudskills） | Skill | 功能描述（springs/胶片颗粒/Ken Burns 等） | 与官方 skill 重叠，效果图证据弱 | 暂缓 |
| demo-producer | Skill | 管线描述（Content Detector→Remotion Composer） | 管线太重 | 暂缓 |

### 4.2 采用组合（按"效果图对标"逻辑，用户已认可 video-shotcraft 效果图程度）

1. **主执行器 = Remotion 官方 skill**：效果图最硬、生态最大，overlay/card/diagram/字幕全覆盖。
2. **风格预设库 = video-shotcraft（214 styles + 214 motion previews）+ HyperFrames premade frames**：制作前人选风格，效果图对标后拍板。
3. **方法论 = video-shotcraft（场景级）+ vox-director（科普质感）+ tutorial-video（引导标注）**。
4. **参考素材 = 上面所有效果图**，作为 design brief 的目标层输入。

---

## 5. 新增节点：风格预设选择（制作前人工决策）

### 5.1 为什么加这个节点

用户拍板：**视频制作前，人去判断需要哪个风格的预设**。理由：风格方向不该由 Agent 第一次自由发挥（裸 LLM 默认 = "flat and generic"），而应人先定基调——"哪些人的效果图好，我们就采用他们的预设"。

### 5.2 节点设计（位于"输入 → 编排"之间）

```
输入（原始视频 + 需求 + 转录文本 + 参考素材）
  ↓ ① 风格预设库（只增不减）：video-shotcraft 214 styles / HyperFrames premade frames / 历次人审沉淀的自定义风格——每套带效果图
  ↓ ② 人挑选：Agent 按内容类型建议 2~3 套（附效果图），人看效果图拍板 1 套
  ↓ ③ 风格锁定 → 注入 aesthetic-guide.json：该套风格的色板/字体/动效语言成为本次制作的 allowed 基线
  ↓ 进入既有管线：策略路由 → 编排 → 剪辑 → 动效事件 → 机检 → 人审
```

### 5.3 三层价值

1. **风格方向人定**——不在"颜色/字体/特效第一次选择"上交给 Agent 自由发挥。
2. **驳回理由反哺风格库**——每次人审驳回的理由经确认后沉淀成该风格的约束或新风格，只增不减，库越用越贴合审美。
3. **对标即决策**——"效果图好就用谁的预设"从流程上实现：候选全部带效果图，人看了再拍板。

### 5.4 落地载体

- 风格预设库 = 配置文件（`style-presets.json`）：每套预设含 `id / name / source（video-shotcraft | hyperframes | custom）/ palette / typography / motion_language / preview（效果图链接）/ constraints（沉淀的驳回约束）`。
- aesthetic-guide.json 增加 `preset` 字段：记录本次制作选定的预设 id，其余字段由预设展开 + 人审沉淀覆盖。

---

## 6. 三层质量保障（动效执行器质量架构）

| 层 | 作用 | 来源 | 对应案例 |
|---|---|---|---|
| ① 目标层：design brief / 参考素材 | 要长什么样 | 风格预设效果图 + 每个事件的视觉 spec | Vox 案例："lock background/font/palette" |
| ② 方向层：aesthetic-guide.json | 不许跑偏 | 预设展开 + 人审驳回理由沉淀（只增不减） | Claude 案例 design bible |
| ③ 地板层：预制组件库/模板 | 不可能丑 | Remotion 官方组件 / HyperFrames 模板 / video-shotcraft 214 风格 | LottieFiles / Remotion 模板 |

> **结论**：质感 = 组件库（下限）+ 风格锁定（方向）+ 参考素材（目标）+ 人审迭代（校准）。不需要玄学提示词，需要结构化约束。

---

## 7. 待拍板决策（v0.3 更新）

| # | 决策 | 默认值（建议） | 影响 | 状态 |
|---|---|---|---|---|
| 1 | schema 语义锚点 | 语义锚点是主键、派生秒是缓存 | 重剪不崩 | ✅ v0.2 定稿 |
| 2 | 机检"内容一致"怎么验 | **已定（2026-10-01）：混合 + 视觉保底**——规则查硬指标 + LLM 摘要比对 + **视觉模型截图保底**（执行器产物直接抽帧截图 → 视觉模型做语义分析 vs 语音/字幕） | 机检实现成本 | ✅ 已定（C + 视觉保底） |
| 3 | 模块边界 | **已定（2026-10-01）：渐进式 + 外部优先**——初期手动为主（70-80% 可接受），执行器全部用外部 API / 外部执行器（含本地 CLI）测试选优，找到最合适的；工作流确认后才考虑自建（自建 = 定死，后置） | 先建什么 | ✅ 已定（C + 外部优先） |
| 4 | 重做预算 | 触发条件 = **机检失败基线**（基础内容没做好，见失败项清单）；人审驳回 ≠ 自动重做（人手动修改 overrides）；次数建议按类型分级，MVP 先统一后按失败率数据调 | 卡死上报频率 | ⏳ 待定（需用户确认失败基线定义） |
| 5 | 剪辑规则参数 | VAD 500ms / 删静音 ≥1.5s / 最短保留 0.5s | 剪辑质量 | ⏳ 待实测调参 |
| 6 | **风格预设库结构** | `style-presets.json`（v0.3 §5.4） | 风格选择节点落地 | ⏳ 待定（建议直接采用） |
| 7 | **动效执行器选型** | **渲染层候选**：Remotion 官方 skill（overlay/card/diagram）+ video-shotcraft（风格库+anim）+ HyperFrames（diagram 模板）；**方案主体 = 编排层语义判断，非渲染层** | 执行器落地 | ⏳ 候选待运行数据验证（用户已明确：渲染层是选项之一，非主线） |

---

## 8. 落地路线影响（MVP 顺序）

| 阶段 | 做什么 | 验收标准 |
|---|---|---|
| ① MVP 闭环 | 剪辑执行器（自建管线）→ 手工填事件 → overlay 渲染器原型（Remotion）→ 机检 → 人审工作台 | 一段 15~60s 视频走完全流程，人审 30 秒校准完成 |
| ② 风格预设选择节点 | style-presets.json + aesthetic-guide 注入 + 人审沉淀 | 制作前人选风格 → 执行器首次选择不再自由发挥 |
| ③ 编排层 AI 化 | LLM 内容分析 + 信号→事件决策（锚语义锚点） | 事件决策与人工标注命中率 ≥ 阈值 |
| ④ 执行层替换 | diagram 接 HyperFrames 模板；anim 接 video-shotcraft；illustration 接 vox-director | 五类事件全部自动执行 |
| ⑤ 批量与稳定性 | 重做预算、并发、产物留痕完善 | 批量跑 N 条不卡死、不越界 |

> 注：②提前到 MVP 之后立即做——因为风格预设选择是动效执行器（③④）的前置输入。

---

## 附录 A：调研信源清单（v0.3 新增）

**Claude Opus 5.5 逐帧案例**
- 原理：https://m.weibo.cn/detail/5348999277839850 （宝玉xp："不是生成每一帧，而是写每一帧"）
- 音乐卡点机制：https://m.weibo.cn/detail/5348275771147718 （节拍/鼓点/逐词时间轴写进代码）
- Deedy 案例（浏览器动画+Playwright+ffmpeg）：https://traictory.com/news/2026-09-25-claude-opus-video-as-code
- 120s 1080p60 one-shot：https://m.youtube.com/watch?v=YXl_BnG2hzs

**真实采用与质感**
- Remotion 成功案例：https://www.remotion.dev/success-stories 、https://cloud.tencent.com/developer/article/2649052
- SaaS 12 条视频 $15k→4 天 +340% 转化：https://www.dplooy.com/blog/claude-code-video-with-remotion-best-motion-guide-2026
- 5000 条个性化 onboarding +78%：https://www.prompts.brightcoding.dev/blog/creating-viral-videos-with-react-components
- 电商 350 产品批量：https://terminalskills.io/use-cases/produce-automated-marketing-videos-at-scale
- Vox 风格 47s 成片 6 步法：https://moderncreator.app/2026-06-28-mosidd-ai-made-easy-i-made-vox-style-motion-graphics-using-only-claude-code-remotion
- "flat and generic" 教训（生产系统化）：https://ai-for-real-life.beehiiv.com/p/codex-gemini-omni-explainer-workflow-review
- 口播→全动效剪辑自动化（WhisperX+Remotion）：https://moderncreator.app/2026-07-27-ryan-ai-content-automations-how-i-fully-automated-video-editing-with-claude-code-and-remotion

**社区预设与效果图**
- Remotion 官方 skill（6M demo / 44 万装机 / 30 分钟实测）：https://www.ngram.com/blog/remotion-skills-sh-ai-video-creation 、https://aifor.dev/tools/remotion
- HyperFrames 设计系统：https://www.hyperframes.dev/design 、模板带视频 demo：https://open-design.ai/plugins/templates/hyperframes/
- video-shotcraft（214 styles / 38s showcase / 剪映导出）：https://raw.githubusercontent.com/Vincentwei1021/video-shotcraft/master/README.md 、https://vincentwei1021.github.io/video-shotcraft/
- vox-director：https://skillsllm.com/skill/vox-director
- tutorial-video：https://claudskills.com/skills/tutorial-video/SKILL.md

---

## 附录 B：变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-09-30 | v0.1 | 架构定稿：四层 + 内核 + schema v0.1 |
| 2026-10-01 | v0.2 | 剪辑执行器操作层；词级时间戳 + 语义锚点；评审留痕/人审映射；社区调研吸收 |
| 2026-10-01 | v0.3 | 动效执行器选型收窄（代码即视频主线 + 三层质量保障 + 社区预设对标 + 风格预设选择节点） |
