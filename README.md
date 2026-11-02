# 视频脚本提取与二创技能包（video-script-extractor）

## 快速了解

### 解决的问题

把短视频中的画面字幕与语音整理为结构化原脚本，并根据用户账号或行业规范改写为可拍摄的口播脚本，减少手动听写和重新组织内容的工作。

### 安装方式

下载或克隆完整仓库，放入兼容环境的技能目录并保留目录结构；其他能读取文件和运行命令的 Agent 可按 `SKILL.md` 执行。按 [环境安装说明](references/setup.md) 配置 Python 3.10+、依赖包、FFmpeg、Playwright 浏览器和 Whisper 模型。链接下载与首次模型获取可能需要网络。

### 使用方法

提供视频链接或本地文件，并说“使用 video-script-extractor 提取并二创这条视频”。技能先提取字幕和语音，再按照保真约束、品牌规范及拟人化规则整理二创稿。需要自定义风格时，可按原文说明配置 `brand_spec.md`；单独查询背景音乐时可使用对应快捷流程。

### 需要提供的输入

必须提供可访问的短视频链接或本地视频文件。可补充账号定位、目标受众、使用场景和品牌规范；没有自定义规范时使用内置通用规范。链接可用性和平台访问条件会影响提取结果。

### 最终交付

完整二创流程交付原视频结构化脚本和二创脚本两份 Markdown；二创稿包含分镜与话术、三个标题、完整文案、标签及差异说明。OCR 和语音转写可能有误，需结合原片复核。

---

把抖音/短视频链接或本地视频，自动提取结构化脚本，并按**用户自己的账号/行业规范**二次创作为可直接拍摄的口播脚本。

**跨行业通用**：提取阶段对任何品类视频都适用；二创阶段内置一份与行业无关的通用规范，并支持用 `brand_spec.md` 覆盖为任意行业的专属规范（默认已附汽车/匠欣工坊示例）。

## 包内含什么

```
video-script-extractor/
├── SKILL.md                      # 技能定义（触发条件 / 执行步骤 / 依赖 / 边界 / 品牌加载机制）
├── README.md                     # 本说明
├── scripts/
│   ├── video_script_pipeline.py  # 一键管线：下载→抽帧OCR→语音转写→生成脚本
│   └── requirements.txt          # Python 依赖
└── references/
    ├── setup.md                  # 环境安装（ffmpeg / venv / whisper / playwright）
    ├── workflow.md               # 两阶段工作流（提取 + 二创）与冲突优先级
    ├── generic_brand_spec.md     # 通用二创规范（默认，跨行业适用）
    └── jiangxin_brand.md         # 行业示例（汽车/匠欣工坊），可复制改写为你自己的 brand_spec.md
```

> 本包**不含**任何特定行业的 `brand_spec.md`，因此可直接分发给任意行业。

## 如何安装与启用

### A. 在 WorkBuddy / 兼容技能系统中
1. 解压本包到技能目录：
   - 用户级：`~/.workbuddy/skills/video-script-extractor/`
   - 项目级：`<项目>/.workbuddy/skills/video-script-extractor/`
2. 按 `references/setup.md` 安装依赖（ffmpeg、Python 包、Playwright 浏览器、Whisper 模型）。
3. 之后直接对 AI 说「提取这个视频脚本 / 把这个链接二创一下」即可触发。

### B. 在其他 AI 系统中（如 Codex、通用 Agent，无技能框架）
本技能的全部"可执行知识"都在纯文本里，任何能读文件、跑命令的 AI 都能直接照做：
1. 让 AI 读取 `SKILL.md` → 它知道何时用、怎么用、品牌规范如何加载。
2. 让 AI 读取 `references/workflow.md` → 它知道两阶段流程、保真红线、冲突优先级。
3. 让 AI 读取 `references/generic_brand_spec.md`（或你提供的 `brand_spec.md`）→ 它知道二创输出规范。
4. 把 `scripts/video_script_pipeline.py` 作为可执行工具运行（需先满足 `references/setup.md` 的依赖）。
5. AI 按文档复现工作流：跑管线得到原脚本 → 守保真红线 + 品牌规范写二创 → 同时交付原脚本与二创。
AI 照这几份文档即可复现相同流程，无需额外解释。

## 如何适配你自己的行业（品牌规范插拔）

二创规范按以下优先级加载，无需改代码：
1. 环境变量 `BRAND_SPEC_PATH` 指向的文件；
2. `工作根目录/brand_spec.md`（如 `D:\.workbuddy\video_scripts\brand_spec.md`）；
3. 以上皆无 → 内置 `references/generic_brand_spec.md`（通用默认）。

**定制步骤**：把 `references/jiangxin_brand.md`（汽车示例）复制改写为你行业的规范，保存为 `工作根目录/brand_spec.md`（或指向它的 `BRAND_SPEC_PATH`）。之后二创自动采用你的规范，通用规范仅作兜底。

## 最小可用命令

```bash
pip install -r scripts/requirements.txt
playwright install msedge
# 系统另装 ffmpeg 并加入 PATH
python scripts/video_script_pipeline.py "<视频链接或本地路径>"
```

## 关键约定（给接收方 AI 的提醒）

- 每次处理链接前，**先清理 `工作根目录/url_video/` 下的旧 video.mp4 / 脚本_video.md**，避免复用上一条视频。
- 交付时**同时输出两份**：① 原脚本 `脚本_video.md`（管线生成）；② 二创脚本 `脚本_二创_<主题>.md`。
- 二创须同时守两道约束：**保真红线**（选题/开头实质/核心观点不变）+ **品牌规范**（默认 generic，或用户 brand_spec）。
- OCR 取简体、Whisper 可能繁体，整理以画面字幕为准并统一简体。
- 不夸大风险、不诋毁同行、安全操作不诱导模仿（无论使用哪份规范，经营底线都不可违反）。

> 本包为内容创作辅助，不替代专业判断；安全操作须由专业人员完成。
