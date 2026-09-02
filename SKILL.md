---
name: video-script-extractor
description: |
  This skill extracts a structured video script from a Douyin (or other platform)
  video link or a local video file, then produces a recreated (二创) script optimized
  for the user's own account or any industry. Use it when the user provides a
  Douyin/short-video link or a local video and asks to "提取视频脚本 / 提炼脚本 /
  生成视频脚本 / 二创脚本 / 把这条视频改成我的口播". The skill downloads the video
  (Douyin needs a cookie fetched via Playwright), extracts on-screen subtitles with
  OCR and voice with Whisper, then rewrites the script following fixed fidelity red
  lines plus an optional brand style guide (pluggable: user's own brand_spec.md,
  otherwise a built-in generic spec — works for any industry). Every 二创 output is
  MANDATORILY post-processed by (1) a Humanizer pass (references/humanizer_rules.md,
  or the humanizer-zh skill when available) so the script reads like a real person
  speaking, and (2) an altruistic call-to-action check: comment guidance must give
  the viewer real value, and is skipped entirely when no natural value angle exists.
agent_created: true
---

# Video Script Extractor · 视频脚本提取与二创（跨行业通用）

## Overview

把一条短视频（抖音链接或本地文件）自动转成**结构化脚本**，并按用户指定的**品牌/账号规范**二次创作成可直接拍摄/录制的口播脚本。

全流程无需用户额外操作：下载 → 抽帧 OCR → 语音转写 → 合并脚本 → 二创优化。

**适用任何行业**：本技能对"提取原脚本"完全通用（任何品类视频都能 OCR + 语音转写）；对"二创"部分提供一套**与行业无关的通用规范**，并支持用户用 `brand_spec.md` 覆盖为自家行业的专属规范（默认已激活汽车/匠欣工坊示例，仅作参考）。

## When To Use

- 用户给出 **抖音/短视频链接** 或 **本地视频文件**，并要求「提取/提炼/生成视频脚本」。
- 用户给出链接并要求「二创 / 改成我的口播 / 按我的账号调性重写」。
- 触发词示例：「根据视频内容提炼脚本」「把这个视频二创一下」「给我视频脚本」。

## 快速查 BGM（不用下载视频）

用户只问「这条视频的 BGM/背景音乐/配乐是什么」时，**无需走完整管线**，一条 yt-dlp 元数据命令即可秒查（抖音元数据自带 `track`/`artists` 字段，比 Shazam 音频指纹对翻唱/旋律版更准）：

```bash
python -m yt_dlp --dump-json --cookies "<工作根目录>/douyin_cookies.txt" --no-warnings "<视频链接>" \
  | python -c "import sys,json; d=json.loads(sys.stdin.read().splitlines()[0]); print('BGM:', d.get('track'), '| 歌手:', '/'.join(d.get('artists') or []), '| 发布者:', d.get('channel',''))"
```

- 拿到歌名后如不确定，用 WebSearch 搜「歌名 旋律版 抖音」核实真实性与原曲。
- 若元数据无 `track`（如本地视频/其他平台），再降级：下载视频 → ffmpeg 抽音频 → Shazam（shazamio 需 `pip install shazamio`）。

## Resources

- `scripts/video_script_pipeline.py` — 一键管线：下载(抖音用 Playwright 取 cookie + yt-dlp) → ffprobe 时长 → ffmpeg 抽帧 → RapidOCR 字幕 → faster-whisper 语音 → 生成 Markdown 脚本。
- `scripts/requirements.txt` — Python 依赖清单。
- `references/workflow.md` — 两阶段工作流（提取 + 二创）与冲突优先级。
- `references/humanizer_rules.md` — **强制后处理①：拟人化规则**（口播版 Humanizer），所有二创文本交付前必过；环境有 humanizer-zh 时可直接调用。
- `references/generic_brand_spec.md` — **通用二创规范（默认）**，与行业无关的标题/前2秒/5步模型/评论引导/时长/经营底线，跨行业直接可用。
- `references/jiangxin_brand.md` — **行业示例（汽车/匠欣工坊）**，展示如何写一份专属品牌规范；可复制改写为你自己的 `brand_spec.md`。
- `references/setup.md` — 环境安装（ffmpeg、venv、whisper 模型、playwright 浏览器）。

## 品牌规范加载机制（可插拔 · 跨行业关键）

二创时按以下优先级加载品牌规范：

1. 环境变量 `BRAND_SPEC_PATH` 指向的文件；
2. `工作根目录/brand_spec.md`（如 `D:\.workbuddy\video_scripts\brand_spec.md`）；
3. 以上皆无 → 使用内置 `references/generic_brand_spec.md`（通用默认）。

> 加载到用户规范后，通用规范仅作兜底（用户规范未覆盖的条目）。
> **分发友好**：分发包只含 `generic_brand_spec.md` 与 `jiangxin_brand.md`(示例)，不含任何特定行业的 `brand_spec.md`；其他行业用户收到后，把自己的规范放到 `工作根目录/brand_spec.md` 即可启用，无需改代码。

## Execution

### Step 0 — 清理工作区（关键防复用）
> **务必先做**：管线下载会把视频固定存到 `工作根目录/url_video/video.mp4`。若上次处理残留该文件，本次会**复用旧视频而不重新下载**，导致脚本串片。
```bash
rm -rf "<工作根目录>/url_video/video.mp4" "<工作根目录>/url_video/audio.wav" \
       "<工作根目录>/url_video/frames" "<工作根目录>/url_video/脚本_video.md"
```
工作根目录解析见下方「本机适配」。

### Step 1 — 提取原脚本（自动化）
```bash
python scripts/video_script_pipeline.py "<视频链接或本地路径>"
```
脚本会打印 `OUTPUT:<路径>`（`工作根目录/url_video/脚本_video.md`）。读取该 Markdown，识别：选题方向、开头内容、核心观点、分镜结构。**原脚本必须随二创一并发给用户**（见 Step 3）。

### Step 2 — 二创（AI 创作，双重约束 + 两道强制后处理）
严格按 `references/workflow.md` 执行：
1. **保真红线**（通用、任何视频适用）：选题方向、开头实质、核心观点不变，只升级表达。
2. **品牌规范**（可插拔，见上文加载机制）：逐条核对加载到的规范（默认 `generic_brand_spec.md`；本机已激活匠欣规范）。
3. 用 WebSearch 检索同款高赞文案作表达参考（参考不抄袭）。
4. 初稿产出后，**强制后处理**（两道，缺一不可）：
   - **① Humanizer 拟人化**：按 `references/humanizer_rules.md`（环境有 humanizer-zh 时直接调用其流程）改写全稿——口语化、去 AI 味词汇、破模板/三段式、具体化、有对象感；**只改表达，不改事实与观点**。
   - **② 利他评论引导检查**：结尾评论引导必须给观众实际价值（见品牌规范"评论引导"一节）；若该条内容找不到自然、真实的利他切入点，**不强行植入评论引导**，以保证叙述流畅完整。
5. 输出二创脚本，必须含：保真声明、分镜描述、对应话术、标题×3、完整文案、适用标签、与原片差异&合规处理、拟人化与利他性说明。

### Step 3 — 交付（同时输出原脚本 + 二创）
**必须同时交付两份**：
- **原脚本**：`工作根目录/url_video/脚本_video.md`（管线自动生成，直接呈现或附文字版）。
- **二创脚本**：`工作根目录/url_video/脚本_二创_<主题>.md`（本次创作）。
两份都发给用户；同时把二创文字版直接贴给用户。

### 运行环境约定（可移植）
- 运行管线的 Python 须是**已装依赖的虚拟环境**（见 `references/setup.md`，含 yt-dlp / rapidocr / faster-whisper / playwright）。
- 工作根目录默认 `~/.workbuddy/video_scripts`，如需强制其他位置，设 `VIDEO_SCRIPT_ROOT`。
- Whisper 模型默认 `tiny`，首次运行自动从 HuggingFace 下载；如需指定本地模型，设 `WHISPER_MODEL_PATH`。
- **网络/沙箱**：抖音下载与 Playwright 取 cookie 需要外网访问，运行时需放开沙箱（如 WorkBuddy 中加 `dangerouslyDisableSandbox` 或等价权限）。
- 抖音链接首次会弹出一个 Edge 窗口（约 8 秒）由 Playwright 取 cookie，自动关闭，无需人工操作；cookie 缓存于 `工作根目录/douyin_cookies.txt`，过期会自动重取。
- 品牌规范放置于 `工作根目录/brand_spec.md`（或 `BRAND_SPEC_PATH` 指向的文件），二创默认按它输出；无此文件则回落到通用规范。

### 交付后日志
追加一条工作日志到项目记忆（`.workbuddy/memory/YYYY-MM-DD.md`）：链接、提取时长、选题、原脚本与二创文件名、品牌规范合规要点。

## Dependencies（一次性安装，见 references/setup.md）
- Python 3.10+ 虚拟环境
- `pip install -r scripts/requirements.txt`（yt-dlp、rapidocr-onnxruntime、faster-whisper、playwright）
- `playwright install msedge`（或 chromium）
- 系统安装 **ffmpeg** 并加入 PATH
- Whisper 模型默认 `tiny`，首次运行自动从 HuggingFace 下载；离线环境用 `WHISPER_MODEL_PATH` 指向本地模型目录

## Boundaries / 边界
- 仅处理用户提供的链接/文件，不主动爬取其他内容。
- OCR/Whisper 可能误识，整理二创以**画面字幕(OCR)为准**并统一简体。
- **所有二创交付文本必须经拟人化改写**（`humanizer_rules.md` / humanizer-zh），且评论引导遵守利他原则、无利他切入点时不植入——这两道后处理不可跳过。
- 二创不得违反经营底线（真实、不夸大、不诋毁、安全不诱导模仿）——无论使用哪份品牌规范。
- 与来源冲突时：保真红线 > 品牌表达；若来源观点违背经营底线，以底线为准并提示用户。
- 本技能为内容创作辅助，不替代专业判断；安全操作须由专业人员完成。
- **跨行业**：提取阶段完全通用；二创阶段默认通用规范，用户以 `brand_spec.md` 提供专属规范时优先采用，技能不绑定任何单一行业。
