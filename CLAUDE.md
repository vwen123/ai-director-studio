# AI Director Studio · 开发记录

> 单文件 HTML 工作坊工具：给华文老师用 NotebookLM + Gemini + Flow + Kling/Veo 做 AI 故事影片。
> 线上地址：https://vwen123.github.io/ai-director-studio.html
> 工作坊日期：2026-04-24

---

## 📦 架构

- **单文件**：`ai-director-studio.html`（HTML + CSS + JS 全在一个文件里，方便学员离线使用）
- **部署**：`/Users/weiwen/Downloads/ai_director_studio.html`（主编辑文件）→ `cp` 到 `/Users/weiwen/Projects/ai-director-studio/` → `git push` → GitHub Pages 自动更新
- **仓库**：`vwen123/vwen123.github.io`（main 分支）
- **持久化**：localStorage key `ai_director_v1`；API key 存 `gemini_key`

### 技术栈
| 用途 | 工具 |
|---|---|
| 文本生成 | Gemini 2.5 Flash |
| TTS 配音 | Gemini 2.5 Flash Preview TTS（30 种声线）|
| 语音转字幕 | Whisper base（transformers.js，浏览器端运行）|
| 打包下载 | JSZip |
| 音频编码 | PCM16 @ 24kHz → WAV Blob |

---

## 🧭 五阶段流程（Phase Cards）

用户整体工作流分 5 块，版面为深色底 + 5 个独立浅色卡片（phase-c1 ~ c4 + nb + cx）：

1. **Phase 1 · NotebookLM 简报大纲**
   起点：输入《标题》《主题》《幕数》《画风》→ 一键生成 NBM prompt → 用户复制到 NotebookLM 得到原文大纲
2. **Phase 2 · 场景拆解**
   把大纲变成 N 场的 start/end 图像 prompt（给 Flow / Nano Banana 画分镜）
3. **Phase 3 · 旁白 + 配音**
   旁白文本（原文 / 生动二选一）→ Gemini TTS 一键转语音
4. **Phase 4 · 字幕对齐**
   Whisper 转 SRT → 用旁白文本校正 → 可编辑面板（文字 / 时间 / 切点）
5. **Phase 5 · 打包**
   按幕切割 WAV + 完整 SRT 一起打 ZIP

---

## 🔑 核心技术决策

### ① 顶部字段全链路同步
幕数、画风、标题、主题四个字段改动 → `syncAllFromCore()` 触发：重建 NBM prompt + 每个场景 prompt + 旁白 + 封面。避免用户改了幕数底下却没跟着变。

### ② 旁白按幕对齐 Whisper 切片
- `parseNarrationScenes(text, n)`：按「第N幕」切开旁白原文
- `alignChunksToText(chunks, sceneTexts, totalSec)`：按时长比例把旁白原文分配进 Whisper 切片，同时打上 `sceneIdx` 标记
- 好处：Whisper 识别错的字，用原文校正后自动归位

### ③ 切点算法 · 两幕之间的静音中点
不是用 `first-chunk.start`，而是用 `(prev_scene_last.end + curr_scene_first.start) / 2`，切 WAV 时刚好切在静音，不会切到说话中间。

### ④ 字幕自动分行（标点优先 + 18 字兜底）
```
for ch of text:
  buf += ch
  if 标点 且 长度 >= 9：断行
  else if 长度 >= 18：断行
```

### ⑤ Gemini 429 节流 + 重试
```js
MIN_GAP_MS = { 'gemini-2.5-flash-preview-tts': 6500, 'gemini-2.5-flash': 3500 }
// 429 时解析 "retry in Xs" 自动等待重试 2 次
```

### ⑥ 三层同步保证下载一致
`syncPanelEdits()` → `maybeRealignFromNarration()` → 导出 SRT / WAV
- 面板改字 / 改时间 / 加减条 / 调切点 → 立即写回 `lastChunks`
- 旁白文本变了 → `__lastAlignedNarration` 快照检测 → 重新按比例分配文字（保留切点和时间）

---

## 🛠️ 关键函数地图

| 函数 | 作用 |
|---|---|
| `syncAllFromCore()` | 顶部字段改动时重建所有下游 |
| `generateNbmPrompt()` | 产生 NotebookLM 用的说故事大纲 prompt |
| `buildScenePrompt(s, 'start'\|'end')` | 单个场景的图像 prompt |
| `generateNarration()` | 生成旁白（原文 / 生动二选一）|
| `callGemini(model, body)` | 统一 API 调用，带节流 + 429 重试 |
| `testVoice()` | 试听 5 秒声线样本（带缓存）|
| `generateVoice()` | 一键把旁白全文转 WAV |
| `transcribeWhisper()` | 浏览器端 Whisper 转字幕 |
| `parseNarrationScenes(text, n)` | 按「第N幕」切旁白 |
| `alignChunksToText(chunks, sceneTexts, totalSec)` | 旁白文字校正 Whisper 切片 |
| `renderSrtPreviewGrouped(chunks, n)` | 可编辑字幕面板（CapCut 风格）|
| `updateChunkText / updateChunkTime` | 面板改字 / 改时间即时同步 |
| `addSrtRow(i) / removeSrtRow(i)` | 面板增删条目 + 时间顺延 |
| `toggleCutPoint(i)` | 面板切点开关 |
| `splitLongText(text, maxLen=18)` | 标点优先的字幕分行 |
| `splitAudioByScenes()` | 按幕切 WAV + 完整 SRT 打 ZIP |
| `secToSRT / secToSRTShort / parseTimeInput` | 时间格式工具 |

---

## 📝 迭代日志

### 2026-04-19 · 第一阶段（整体联动）
- [x] 顶部 4 个字段 → 下游 prompt / 旁白 / 封面全链路同步
- [x] 旁白选项从「篇幅长短」改为「原文 / 生动」
- [x] NotebookLM prompt 简化成说故事大纲格式
- [x] 生成 WAV 时去掉「第N幕 / 第N页」前缀
- [x] SRT 面板支持按 1/2/3 分组 + 用旁白校正

### 2026-04-19 · 第二阶段（精简 + UX）
- [x] AI 功能瘦身，只留 Gemini TTS 一键转语音
- [x] 移除浏览器 TTS，改链到 Google AI Studio
- [x] 选 NotebookLM 模式时自动展开语音面板
- [x] 修 Phase 3/4 叠进 Phase 2 的版面 bug（HTML 标签缺 `>`）
- [x] 简化标签文字（去掉括号注释）
- [x] 一键模式自动填「语气描述」（关键词匹配）

### 2026-04-19 · 第三阶段（字幕面板升级）
- [x] 字幕面板全面可编辑（文字 + 切点）
- [x] 切点跟旁白幕数同步（7 幕 → 7 段）
- [x] WAV 切点改用「两幕之间静音中点」
- [x] 字幕按标点 / 18 字自动分行
- [x] 面板改动实时同步到下载 SRT / 打包 WAV
- [x] 加 ➕ ➖ 按钮：插入 / 删除条目自动分配时间
- [x] 旁白文本改了 → 自动按比例重新分配到切片（保留切点和时间戳）

### 2026-04-19 · 第四阶段（打包决策反复）
- [x] ZIP 初版含 N 段 WAV + N 段 SRT + 完整 SRT
- [x] 改为只留 WAV + 完整 SRT（用户反馈）
- [x] 一度加回每段 SRT（用户改主意）
- [x] 最终版：只留 WAV + 完整 SRT（完整 SRT 已自动分行，不需要再分段）

### 2026-04-19 · 第五阶段（CapCut 风格字幕面板 + 声线性别）
- [x] 字幕面板参照 CapCut 重新设计：
  - 顶部「＋ 新增字幕」胶囊按钮
  - 左：大字幕输入框
  - 中：上下两个独立时间输入框（开始 / 结束，格式 `h:mm:ss.ss`）
  - 右：竖排 ➕ ➖ ✂ 按钮
  - 时间输入支持 `h:mm:ss.ms` / `mm:ss` / 纯秒数
- [x] 修正声线性别错标（对照 Google 官方文档）：
  - `Achird` · 友善亲切：女 → **男**
  - `Gacrux` · 成熟智者：男 → **女（长者 / 奶奶）**

---

## 🚀 部署流程

```bash
# 1. 编辑主文件
edit /Users/weiwen/Downloads/ai_director_studio.html

# 2. 同步到仓库工作副本
cp /Users/weiwen/Downloads/ai_director_studio.html \
   /Users/weiwen/Projects/ai-director-studio/ai-director-studio.html

# 3. 提交推送（1-2 分钟后 GitHub Pages 生效）
cd /Users/weiwen/Projects/ai-director-studio
git add -A
git commit -m "描述改动"
git push
```

线上地址：https://vwen123.github.io/ai-director-studio.html

---

## ⚠️ 注意事项

1. **Gemini 免费层额度**：TTS 每天 10 次、Flash 每天 20 次。工作坊当天如果集体触发，建议学员各自用自己的 API key（面板右上角有输入框，存本地 localStorage）。
2. **Whisper 首次加载**：约 70MB 模型文件，首次打开要等 30 秒~1 分钟，之后走浏览器缓存。
3. **单文件限制**：不能用 npm 包、不能拆模块，所有依赖走 CDN（transformers.js / JSZip）。
4. **HTML 结构警惕**：改标签时留意 `<div ...></div>` 的闭合；曾经因为 `<div id="x"</div>` 少了 `>` 导致 phase 3/4 叠进 phase 2。

---

## 🎯 下一步（工作坊当天可能会补）

- [ ] 如果 API 额度不够：加「本地离线模式」开关
- [ ] 批量导入 NotebookLM 大纲自动切幕
- [ ] 导出 Flow 专用的 JSON 批次文件
- [ ] 试听 5 秒样本改成可选 2s / 5s / 10s
