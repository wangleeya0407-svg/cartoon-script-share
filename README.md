# 米亚老师 AI 课程自动化制作 Skill

把一段口播语音（或一段带声音的视频 / 录屏），自动做成一条 1920×1080 的知识动画课程成片。
输入只要声音，画面全部重新生成 —— 源视频只用来读懂内容结构，不进成片。

## 它不是一个动效模板

常见动效模板普遍有三个毛病：画面元素太多，看着眼花；没有视觉主体，观众不知道该看哪；缺少品牌 IP，做出来的东西跟谁都不像。

这个 Skill 的取舍正好相反：

- **画面克制**：不做花哨转场，主体始终是 IP 形象，久看不累。
- **结构成课**：章节卡、金句页、结尾卡的视觉贯穿全片，一套视觉本身就是一门课。
- **IP 埋进画面里**：形象分三层铺（背景底纹 / 右上角标 / 半身大图），裁切截图也去不掉，天然防搬运。
- **可以换成你自己**：形象素材和品牌名都能替换，输出即带个人标识。

## 两个版本

| 目录 | 版本 | 效果 | 适合 |
|---|---|---|---|
| `mia-ai-course-video` | 带真人头像版 | 右上角一个圆形真人小窗（彩色光环慢转），下挂「米亚老师 · 原创」名牌，卡通头像贴在小窗左下角 | 人格化出��[...] |
| `mia-ai-course-video-nopip` | 无真人头像版 | 不放真人小窗，输入是视频也只取声音，画面全部由卡通形象 + 动画构成 | 纯知识讲解、画面更干净、团队或机[...] |

两个版本共用同一套视觉结构与脚本，切换版本不影响内容。

## 规格

- 分辨率 1920×1080，30fps，H.264 + AAC
- 音量走独立音轨，字幕逐词对齐
- 首帧即完整封面：所有元素在 t=0 就在位，不从透明淡入（平台抽第一帧当封面也不会空）
- 全片 12–29 场，7 种版式：`chapter` / `quad` / `punch` / `list` / `vs` / `chain` / `end`

## 环境依赖

| 依赖 | 用途 |
|---|---|
| `hyperframes`（`npx hyperframes`） | 渲染内核，首次运行需联网拉包 |
| `ffmpeg` / `ffprobe` | 抽音轨、抽帧、压缩 |
| `faster-whisper` + `ctranslate2` | 本地转写，不联网 |
| `zhconv` | 繁简归一 |
| Node.js | 生成器与校订脚本 |
| `gsap.min.js` | 已内置在 `assets/`，无需下载 |

除首次拉包外，全程本地运行，断网也能出片。

## 五步流水线
```

语音/视频
  ↓ ① 转写      transcribe.py     → talk/<slug>.json
  ↓ ② 繁简归一   t2s.py            → talk/<slug>-cn.json
  ↓ ③ 校订      mkoverride.mjs + course-fix.mjs → <slug>/config.json
  ↓ ④ 生成      build-course.mjs  → <slug>/index.html
  ↓ ⑤ 渲染验收   hyperframes + ffmpeg

```

```bash
# 1. 建工程（输入是视频时，带真人版会自动生成真人小窗素材）
bash scripts/scaffold_course.sh <slug> <音频或视频路径> "第 05 讲 · 主题" "课程标题"

# 2. 转写
python3 scripts/transcribe.py <slug>/assets/voice.m4a --model small --out talk/<slug>

# 3. 校订（手写 override-src，逐行覆盖 ASR 错字）
node scripts/mkoverride.mjs talk/<slug>-cn.json pipeline/override-src/<slug>.json --out pipeline/override-<slug>.json
node scripts/course-fix.mjs talk/<slug>-cn.json --start 0 --end <时长> --out <slug>/config.json --out-dir <slug> --audio assets/voice.m4a --base false --chapter "第 05 讲 · 主题" --title "课��[...]

# 4. 生成 + 抽帧检查
npx hyperframes check <slug> --at 1.5,17,43,72 --snapshots

# 5. 渲染 + 压缩 + 抽帧验收
bash scripts/render.sh <slug> "课程名_成片.mp4"
bash scripts/verify.sh <slug> "<slug>/课程名_成片.mp4" <预期时长>
```

**先看画面再渲染**：`check --snapshots` 抽帧比渲染整片快几十倍，能拦住 90% 的返工。

## 目录结构

```
mia-ai-course-video/            # 带真人头像版
├── SKILL.md                    # 完整使用说明（先读这个）
├── scripts/
│   ├── scaffold_course.sh      建工程，输入视频时自动生成真人小窗素材
│   ├── make-pip.py             口播视频 → 头肩圆窗素材（仅带真人版有）
│   ├── align-script.py         有文字稿时按停顿对齐，比转写更准
│   ├── transcribe.py           本地转写，faster-whisper 词级时间戳
│   ├── t2s.py                  繁简归一
│   ├── build-course.mjs        数据驱动生成器（7 种版式 + IP 自动铺设）
│   ├── render.sh               渲染 + 压缩 + 参数自检
│   └── verify.sh               抽帧验收
├── assets/
│   ├── gsap.min.js
│   └── mascot/                 7 张 IP 形象（4 张半身 + 3 张头像，透明底）
└── references/
    ├── pipeline.md             五步详解：命令、参数、产物
    ├── scene-types.md          7 种版式的字段与时间规则
    ├── caption-rules.md        校订稿格式、视觉宽算法、断词判定、ASR 错词库
    ├── pitfalls.md             26 条踩坑：症状 → 根因 → 解法
    ├── checklist.md            交付验收清单
    └── cases.md                已交付片子的规模与节奏校准
```

## 怎么用

1. 把对应版本的文件夹放进 AI 客户端的 skills 目录，让它能读到 `SKILL.md`
2. 把口播视频（或音频）交给它，说一句「按米亚老师那套流程做成课程动画」
3. 它会自己跑完转写、校订、分镜、渲染，最后给你成片路径

没有转写模型、但你手上有文字稿时更推荐走 `align-script.py`：直接按停顿对时间，准确率比转写高。

## 常见问题

**Q：必须联网吗？**
不需要。除了首次 `npx` 拉包，转写、渲染、验收全部本地完成。

**Q：一定要用付费模型吗？**
转写用本地的 faster-whisper，不需要任何付费 API。

**Q：形象能换成我自己的吗？**
可以。替换 `assets/mascot/` 下的 7 张透明底 PNG，命名保持一致即可；品牌名在 config 里改。

**Q：字幕有错字怎么办？**
ASR 错字必须逐行清零，专有名词、繁简、同音字都在 `references/caption-rules.md` 里给了高频错词库。

## 关于

形象、脚本与视觉体系由米亚老师设计。

## 口播输入与成片效果

下图对比了两种成片样式：左侧为**带真人头像版**（右上角圆形真人小窗，搭配卡通头像和原创名牌）；右侧为**无真人头像版**（不显示真人小窗，保留卡通形象与课程动画）。两种版本都以口播声音驱动画面，课程画面由模板重新生成。

> 效果对比图待上传到仓库后插入此处。请将图片上传到本文件同目录，并命名为 `口播效果对比.png`，随后在这里添加：`![带真人头像版与无真人头像版效果对比](./口播效果对比.png)`。

### MP3 与 MP4 怎么选

- **MP3 音频**：提取口播声音，生成字幕、分镜和课程动画；不会从 MP3 中生成真人小窗。
- **MP4 视频**：提取视频中的声音，原始画面不进入成片。使用带真人头像版时，流程说明为可从视频素材生成右上角真人小窗；使用无真人头像版时只使用声音，不生成真人小窗。
- <img width="2176" height="810" alt="c9e3bb81948f03fc06d6182de33e45fa" src="https://github.com/user-attachments/assets/949b08c1-6517-45b0-8b71-455d22645d3b" />

- **如果想用 MP3 同时显示真人头像**：需要先确认该版本是否支持单独提供头像照片。当前流程说明只写明视频输入时会生成真人小窗，并未说明可仅凭一张照片自动生成该小窗。

选择版本时，想呈现真人小窗就选带真人头像版；想让画面更简洁、只使用卡通形象就选无真人头像版。
