# AI 制作动画视频 Studio · 开发记录

> NotebookLM 工作流驱动的动画视频制作工具：标题 → NBM Prompt → 解析大纲 → 旁白 / 语音 / 字幕 → 封面 / 封底 / 背景音乐
> 单文件 HTML（vanilla JS），无后端，所有处理在浏览器本地完成。
> 仓库：`vwen123/ai-director-studio`（main 分支）
> 线上：`https://vwen123.github.io/ai-director-studio/`

---

## 📦 架构

| 档案 | 用途 |
|---|---|
| `index.html` | 单文件应用（HTML + CSS + JS 全部在内）|

- **存储**：localStorage（key `KEY`），API Key 存 `gemini_key`，UI 语言存 `uiLang`
- **TTS**：Gemini API（`gemini-2.5-flash-preview-tts`）
- **语音转字幕**：transformers.js + Whisper-base（浏览器本地推理，首次约 150MB）
- **打包**：JSZip（CDN）
- **GitHub Pages**：`/Users/weiwen/ai-director-studio/index.html` push 到 main

---

## 🧭 核心工作流（NotebookLM 模式 · 唯一保留路径）

### 区块 1 · 项目标题 + 视觉风格 + NBM Prompt
1. 输入标题 + 选择视觉风格（中文标签 + 英文 prompt 关键字）
2. 自动生成 NotebookLM Prompt：
   ```
   请根据《<标题>》，给我 N 页简报插图建议大纲，每一页都是原文，用说故事的方式呈现，列出来。整体视觉风格：<风格中文>
   ```
3. 一键复制 → 贴到 NotebookLM → 让 NBM 根据用户上传的课文产出大纲

### 区块 2 · 贴入 NBM 回传大纲 → 旁白 / 语音 / 字幕
1. 贴入 NBM 回传内容（含「故事原文」段落）
2. `parseNbmAndBuildNarration()` 抽取每页原文 → 自动生成 `第1幕：... \n 第2幕：...` 旁白
3. 选声线 + 语气 → Gemini TTS → WAV
4. 「📝 将这段语音转字幕」 → Whisper 转录 + 旁白校正 → 可编辑字幕面板
5. ⬇️ 下载 SRT / ✂️ 按幕切割 + 打包 WAV ZIP

### 区块 3 · 特别素材
- 🖼️ 封面 / 封底剧照 Prompt（依视觉风格 + 故事原文 + 关键意象 + 情绪推断）
- 🎵 背景音乐 Prompt（依视觉风格 + 故事情绪 + 故事意象）
- ⬇️ 下载 .txt（封面 + 封底 + 背景音乐）—— 一份 txt 带作者署名

---

## 🔑 关键技术决策

### ① 从全功能 → NBM-only 简化
最初支持「仅标题 / 贴课文 / NotebookLM」三种模式，后来收敛到只剩 NotebookLM。`phase-chars`、`phase-scenes`、`export-area` 这些区块直接 `style="display:none"` 硬隐藏在 HTML 上，避免任何 JS 早期错误把它们闪出来。

### ② 单文件、无构建
所有 i18n 字典、CSS、JS 都内联到一个 HTML。`data-i18n` 翻译文字内容，`data-i18n-ph` 翻译 placeholder，`setUiLang(lang)` 在 init 和切换语言时跑一遍替换。

### ③ 「原文」解析容错
NotebookLM 回传格式不固定（有时 `**原文：** xxx`、有时 `**原文**` 换行下面才是内容、有时 `### 原文 "xxx"`）。`parseNbmAndBuildNarration()` 改成逐行扫描：
- 任何含「原文」二字的短标题行（≤15 字内）都识别为段落起点
- 同行有正文 → 取；同行只有标题 → 读下面几行直到遇到空行 / 下一个小标题
- 自动剥除 markdown 符号（`**` `#` `>`）和首尾引号（中英文都支持）

### ④ 音频按幕切割（核心算法）
**问题**：均分时间 → 长短不一的幕错位严重
**解法**：`computeSplitPointsByText(totalSec, sceneTexts, chunks)`
1. 按各幕**字数比例**算理想时间切点
2. 每个理想点**吸附到最近的 Whisper chunk 结束时间**（容许 ±35% 平均幕长）—— 利用语音停顿避免切断字
3. 强制单调递增防止吸附后逆序

适用范围：TTS 自生 WAV、用户上传 mp3/wav（包含 Google AI Studio 生成的音档）—— 同一支 `handleAudioUpload` 处理。

### ⑤ 字幕面板即时同步
- 文字编辑（textarea oninput）→ 即时回写 `lastChunks[i].text` + `lastSRT`
- 时间编辑（input oninput soft + onchange hard）→ 即时回写 `lastChunks[i].timestamp`
- ✂ 切点 / ➕ 插入 / ➖ 删除 → 都直接改 `lastChunks` 并 re-render
- 下载 SRT 和打包 WAV 入口都先 `syncPanelEdits()` —— 即使没失焦也会捕捉到最新值

### ⑥ 封面 / 背景音乐 prompt 的故事感知
- `_storyKeywords(text)` —— 用 2-4 字中文 n-gram 频次提取关键意象（剔除常用虚词）
- `_inferStoryMood(text)` —— 关键词分类（悲/乐/紧张/温馨/神秘/冒险/励志/自然氛围）
- 封面：视觉风格 + opening/ending + 关键意象 + 情绪 + 作者署名（右下角）
- 背景音乐：视觉风格基底 mood + 故事情绪 mood（覆盖式调整）+ 故事关键意象

### ⑦ resetAll 真清空
`localStorage.removeItem(KEY); location.reload()` 不够（浏览器 form autofill 会留字）。`resetAll()` 显式清所有 input/textarea/prompt-out/隐藏面板/状态变量，再 reload。保留 API Key 和语言设置。

---

## 🛠️ 关键函数地图

| 函数 | 作用 |
|---|---|
| `generateNbmPrompt()` | 根据标题 + 视觉风格生成 NBM Prompt |
| `parseNbmAndBuildNarration()` | 从 NBM 大纲抽「原文」 → 旁白框 |
| `_storySource() / _storyOpening() / _storyEnding()` | 旁白主干 / 首幕 / 末幕 |
| `_storyKeywords(text, max)` | 故事关键意象（n-gram 频次）|
| `_inferStoryMood(text)` | 故事情绪标签 |
| `generateCover('front'/'back')` | 封面/封底 prompt 输出 |
| `generateMusic()` | 背景音乐 prompt 输出 |
| `downloadSpecialsTxt()` | 把封面 + 封底 + 背景音乐打包成 .txt |
| `parseNarrationScenes(text, n)` | 旁白 → N 幕字符串数组（实际幕数优先于 n）|
| `computeSplitPointsByText(totalSec, sceneTexts, chunks)` | 按字数比例算时间切点 + 吸附停顿 |
| `alignChunksToText(chunks, sceneTexts, totalSec)` | 把旁白文字按 chunk 时长比例填回 |
| `handleAudioUpload(e)` | 上传音档 → Whisper 转录 → 自动按旁白幕数切割 |
| `ttsAudioToSRT()` | 把生成的 TTS WAV 当作上传走同一流程 |
| `splitAudioByScenes()` | 用 sceneIdx 切 WAV + JSZip 打包 |
| `syncPanelEdits()` | 把面板上未失焦提交的文字 + 时间扫回 lastChunks |
| `updateChunkText / updateChunkTime / toggleCutPoint / addSrtRow / removeSrtRow` | 字幕面板编辑动作 |
| `resetAll()` | 全清空 + reload |
| `setUiLang(lang)` | 切换 zh/en + 重新替换所有 data-i18n |
| `onApiKeyChange(v)` | API Key 输入即时启用判断（≥20 字符）|

---

## 📝 迭代日志

### 第一阶段 · 全功能版本（已废弃）
- [x] 三模式：仅标题 / 贴课文 / NotebookLM
- [x] 角色锚定 + 分镜脚本 + 五维视频参数 + 首尾帧
- [x] 故事草稿确认流程

### 第二阶段 · 收敛到 NotebookLM-only
- [x] 移除「仅标题」「贴课文」按钮
- [x] 隐藏 phase-chars / phase-scenes / export-area
- [x] 标题改为「AI 制作动画视频 Studio」
- [x] 移除作者署名 / 故事主题 / 镜头规格 / 光影氛围 / 渲染质感等输入
- [x] 移除视觉风格搜索框、load demo

### 第三阶段 · 双语 i18n
- [x] 右上角 zh/en 切换
- [x] 切换语言时 document.title + 所有 data-i18n + placeholder + NBM Prompt 全部同步
- [x] 语气描述加 6 个预设（温暖亲切 / 热血活力 / 神秘探险 / 专业知性 / 怀念感性 / 俏皮捣蛋），切换语言时跟着翻译

### 第四阶段 · 三段式布局
- [x] 区块 1：标题 + 视觉风格 + NBM Prompt（合并）
- [x] 区块 2：贴入大纲 + 旁白 + 语音 + 字幕（语音区被 mountVoiceIntoNbm IIFE 移入）
- [x] 区块 3：✨ 特别素材（封面 / 封底 / 背景音乐 + 下载 .txt）
- [x] 把「主题曲」全部改为「背景音乐」

### 第五阶段 · 故事感知 prompt
- [x] 封面/封底：视觉风格 + 故事原文（开场/结尾）+ 关键意象 + 情绪
- [x] 背景音乐：视觉风格 mood + 故事情绪 mood + 故事意象（30 秒 BGM，不写入完整剧情）
- [x] 作者署名「输入名字」placeholder，封面右下角显示

### 第六阶段 · 「原文」解析鲁棒
- [x] 容错多种 markdown 格式
- [x] 同行 / 跨行内容都能抓
- [x] 剥除引号和强调符号

### 第七阶段 · 字幕面板即时同步
- [x] 上传音档自动按旁白幕数切割（不需勾选）
- [x] 旁白幕数优先于 g-num-scenes（旁白有几幕就切几段）
- [x] 字数比例算切点 + 吸附停顿点
- [x] 文字 / 时间编辑即时回写到 SRT 和 WAV 切点

### 第八阶段 · 全清空 + 错误修复
- [x] resetAll 显式清所有字段（保留 API Key + 语言）
- [x] 修一个 `Identifier 'lines' has already been declared` 把整支 JS 击穿的 bug
- [x] 部署到 GitHub Pages（独立仓库）

---

## 🚀 部署流程

源文件在 `/Users/weiwen/Downloads/ai_director_studio.html`，每次改动后同步到 `~/ai-director-studio/index.html` 推上 GitHub Pages：

```bash
cp /Users/weiwen/Downloads/ai_director_studio.html ~/ai-director-studio/index.html
cd ~/ai-director-studio
git add index.html
git commit -m "描述改动"
git push origin main
# Pages 1-2 分钟后生效
```

每次 commit 推送前用 node 检查 JS 语法（避免再次出现整支击穿的 bug）：

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('/Users/weiwen/Downloads/ai_director_studio.html','utf8');
const m = html.match(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g) || [];
m.forEach((s,i) => {
  const code = s.replace(/^<script[^>]*>/,'').replace(/<\/script>$/,'');
  try { new Function(code); console.log('OK'); } catch(e) { console.log('error:', e.message); }
});
"
```

---

## ⚠️ 注意事项

1. **同名变量重复 const** —— JS 同作用域内禁止 `const x` 出现两次，否则整支 `<script>` 不执行。每次重写大函数前先 grep 确认。
2. **GitHub Token 不能贴聊天** —— GitHub 自动撤销，到 github.com/settings/tokens 重新生成。
3. **API Key 仅本地** —— `localStorage.gemini_key`，不上传任何服务器。
4. **Whisper 模型 150MB** —— 第一次「转字幕」会下载，需联网；后续浏览器缓存。
5. **JSZip 来自 CDN** —— 离线时打包 WAV 会失败，会有 toast 提醒。
6. **音频解码限制** —— `AudioContext({ sampleRate: 16000 })` 在某些 Safari 版本有限制；遇到失败建议改用 Chrome。
7. **旁白幕数 vs g-num-scenes** —— 一切按旁白里的「第X幕」实际数量为准，顶部输入框只是 NBM Prompt 用。
8. **bfcache / form autofill** —— 浏览器会保留 textarea 内容跨刷新；resetAll 显式清后再 reload。
