# -B-SOP
只需要提供bilibili网课链接给你的AI，要求他按照这个流程走下去，你就能得到课程中所有的知识点而不需要一集一集的进行观看。
0. 总览
新视频链接
  │
  ├─ ① 下载音轨      download_audio.py            → audio/PXX.m4a + pages.json      （约10分钟）
  ├─ ② 本地转写      run_paced.py                 → transcript/PXX/segments.jsonl   （约12小时/24h音频）
  ├─ ③ 术语校正      glossary.py / mine_terms.py  → 干净逐字稿                       （人工2~3轮）
  ├─ ④ 存疑词初判    propose_terms.py → AI 初判    → 待确认清单 → 用户核定            （新增，见阶段④）
  ├─ ⑤ 逐字稿文档    build_md.py / build_index.py → P01~P35.md + README索引
  ├─ ⑥ 知识点提炼    AI 逐讲提炼（分篇写再合并）    → 知识点背诵手册.md + 三分册.md
  ├─ ⑦ 考纲删减版    联网核实考试说明 + 三道判定门   → _江西升本版.md（按需）
  └─ ⑧ 打印版定稿    md2print → convert_doc → 收尾 → 目录域刷新 → 校验  → .docx 交付
阶段
输入
输出
耗时参考
能否自动化
① 下载
视频链接
audio/ + pages.json
10 分钟
全自动
② 转写
音轨
分段 jsonl → 汇总
≈音频时长/2
全自动（断点续跑）
③ 术语
转写稿
校正后逐字稿
2~3 轮人工
半自动（机器挖掘+人工审定）
④ 初判
候选存疑词
带 AI 初判的待确认清单 → 用户确认
0.5~1 小时
AI 初判 + 人工拍板
⑤ 文档
segments
Markdown 逐字稿
分钟级
全自动
⑥ 提炼
逐字稿
知识点手册
4~6 小时
人工（AI 主力）
⑦ 删减
手册+考试说明
地区版手册
3~4 小时
人工（AI 主力）
⑧ 排版
手册 md
打印版 docx
1~2 小时
全自动+校验
1. 下次怎么用（新对话启动方式）
你只需要发这一句（把链接换掉）：
跑课程流水线：https://www.bilibili.com/video/BVxxxxxx/
课程名称：XXX | 用途：XXX
请先读 C:\Users\1\WorkBuddy AI\2026-10-20-09-43-51\流程SOP.md 并按其中阶段①开始执行
新对话的 AI 需要知道的事（SOP 已写全，但关键几条先说）：
- 代码在 bilibili_politics/，全流程脚本自包含（路径都相对脚本目录），复制整份文件夹即可开新项目；
- Python 环境、faster-whisper 模型、yt-dlp 都是机器级共享资源，不用重装；
- 新课程若同科目（政治），glossary.py 可沿用后按 mine_terms.py 结果增量补充；
换科目则必须重挖术语，并同步改 transcribe_one.py 里的 PROMPT（转写提示词，按新科目写关键术语）。
2. 阶段①：下载音轨
cd bilibili_politics
PY=/c/Users/1/.workbuddy-ai/binaries/python/envs/default/Scripts/python.exe
"$PY" download_audio.py https://www.bilibili.com/video/BVxxxxxx/   # 链接/BV号都行
- 脚本已参数化：支持命令行传 BV、环境变量 BV=、或不带参用旧默认值；
aid 从 view 接口返回值自动取，不写死。直接贴完整链接即可（含 ?p=3 也能解析）。
- yt-dlp 路径：.../envs/default/Scripts/yt-dlp.exe，格式固定 30280（约 84kbps 音频轨，够转写）。
- 断点续传：已存在且 >500KB 的音轨自动跳过，下载中断后重跑同一条命令即可。
- 音轨接口无需登录；B 站 AI 字幕接口未登录返回空，不要走字幕路线（已试过，不可行）。
验收：终端打印 完成：N/N 个音轨；pages.json 生成；任意抽一讲能播放。
产出：audio/PXX.m4a（转写完成后可删，本项目 808MB）+ pages.json（分P元数据）。
3. 阶段②：本地转写（faster-whisper）
模型与参数（12 核 AMD 实测，勿轻易改）：
项
值
理由
模型
faster-whisper medium int8
small 术语错太多（毛概→毛盖）；distil-large-v3 中文输出英文音译，弃用
beam_size
1
速度 1.9x→2.5x；副作用：输出繁体，交给 OpenCC 转简
分片
960 秒/片 + 前移 12 秒重叠
硬切会把一句切两半，140 个边界全是损伤
并发
4 进程 × 3 线程
实测 2.52x，优于单进程 12 线程的 1.69x（whisper 单进程吃不满全核）
启动命令（带作息节律：连续工作 1 小时、休息 10 分钟、自动循环，可随时 Ctrl+C）：
"$PY" run_paced.py                 # 默认 工作60分/休息10分，跑完自动 build_index.py
WORK_SEC=1800 REST_SEC=300 "$PY" run_paced.py   # 自定义作息
"$PY" run_all.py 3 5 11            # 只跑指定分P；不带参 = 全部剩余
WORKERS=6 "$PY" run_all.py         # 调并发
必须守住的两条：
1. 断点续跑按段数判定：done_chunk >= ceil(时长/960) 才算完成，绝不能只看
progress.json 存在（曾因这条把只跑了 1/5 段的章节判成已完成，静默丢失 80% 内容）。
2. 休息只落在章节之间：一个分片是原子 transcribe，中途 kill 会白算一整段。
环境坑（已修，复跑时若报错按此查）：
- Windows 下 huggingface_hub 建软链接静默失败 → snapshot_download 目录为空、
ctranslate2 报 Unable to open file 'model.bin'。解法见 fix_model.py（按 trees/*.json
把 blobs 复制成真实文件）。模型目录：C:\Users\1\.workbuddy-ai\binaries\models\faster-whisper-medium。
- faster-whisper 1.2.1 + PyAV 19：内部解码报 metadata_errors TypeError。
解法：audio_io.py 自己用 av 解码成 16k mono float32 再传给 transcribe()。
- venv：C:\Users\1\.workbuddy-ai\binaries\python\envs\default（Windows 用 Scripts/ 不是 bin/）。
验收：build_index.py 显示全部讲次「✅ 已完成」；每讲覆盖时长 ≈ 原时长。
4. 阶段③：术语校正与质量打磨（人工 2~3 轮）
这是质量的分水岭。medium 模型对政治标准提法基本准，但同音错字密集（实测约 258 条人工核定 + 10 条上下文正则）。
词表设计铁律（都是踩过的坑）：
1. 按词长降序替换，短词规则会误伤（三中全→三中全会 后面跟「会」会叠成「会会」）。
2. 禁止单字规则：「臣」有四种正确读法（蒋宋孔臣→陈、城中无农民→城…），一条单字规则必错。
3. 正确形式包含错误形式时，必须用 PATTERN_RULES 加否定断言：如 从严治党 ⊂ 全面从严治党，
直接加规则会重复叠加。glossary.py 导入时有自检，违规直接报错，别绕过。
4. 术语入表会不可逆覆盖原文：拿不准的一律先标「存疑，见原片 MM:SS」，最后请用户核定
（曾把「麻公臣」错判成毛概入表覆盖了原文，用户纠正为马克思，且定点修正很麻烦）。
机器辅助 + 人工审定闭环：
"$PY" mine_terms.py 3 60    # jieba HMM 挖候选错字（2231个噪声→5个真正存疑，终端速览）
"$PY" propose_terms.py 3 5  # 候选词的「AI初判→用户确认」结构化清单 → 见阶段④
"$PY" reapply.py transcript/P04   # 词表更新后重跑校正，无需重转写
"$PY" verify_terms.py        # 复扫叠加错字（「人民代表大会会」这类二次插入）
- mine_terms 局限：错字两部分各自都是合法词时（历史→真诚）抓不到，必须人工补。
注意 jieba 词典里也收过错字（「毛盖」），判断是否在词典无效，必须用词频阈值。
- verify_terms 的提法要用字符拼接构造（"".join(["人","民","代","表","大","会"])），
终端里肉眼分不出「人民代表大会大/会会」三个形态。
验收：用户抽查 3~5 讲无明显错字；年份、会议届次、人名、文件名回原视频核对（时间戳已在行内）。
5. 阶段④：存疑词初判（AI 先自己判断，再发用户拍板）
为什么单列一个阶段：机器挖出的候选词信噪比低（本项目实测 2231 个噪声 → 72 个候选），
且错字形态千奇百怪。原来直接抛给用户逐条问，用户要为每个词重新建立上下文，
负担重、还容易拍错。现在改成：AI 先逐条给出初判和依据，用户只做「确认/否决」。
5.1 机器汇总证据
"$PY" propose_terms.py 3 5     # 门槛：出现>=3次；每词列5处上下文
KEEP=1 "$PY" propose_terms.py 2 8   # 分批发：不清空旧清单，追加新批次
产出 待确认术语清单.md：每个候选词带全部出现位置（章节 + 时间戳 + 上下文），
时间戳可直接跳回原视频核对。与 mine_terms.py 的分工：后者只在终端速览，前者才产结构化清单。
5.2 AI 初判（本阶段的核心动作）
AI 逐条填写四样东西，不允许只写「不确定，请用户定」：
字段
要求
AI 初判
明说结论：是错字/是正常词/是老师口播简称/无法判断
判断依据
上下文语义、教材标准提法、出现位置是否成体系
置信度
高 / 中 / 低（低 = 这条用户必须重点看）
建议规则
若入表，写出具体规则 原词 → 改成什么（含是否需 PATTERN_RULES 否定断言）
判定抓手（按可靠性排序）：
1. 上下文有唯一合理解 → 直接判错字（例：「多相选的题材」→「多选的题材」）；
2. 成体系出现且语义通 → 判正常词或课程固定说法，标「不入表」；
3. 涉及年份/届次/人名/文件名 → 一律标低置信度，提示用户回原视频（时间戳已在清单里）；
4. 两种读法都讲得通 → 如实给出两种判断，请用户选，严禁猜一种入表。
填写示例（本项目实测数据）：
「相选」出现 27 处：P07 09:51「作为一个多相选的题材掌握」、P17 01:21「高质量发展单相选的题」…
- AI 初判：错字，应为「多选」/「单选」。「相」是「项」被 HMM 切错后的残留，
上下文全部是「多X选的题材/题」结构，无一例讲得通。
- 判断依据：27 处全部为「多相选 / 单相选 + 的题材/题」句式，教材无「相选」一词。
- 置信度：高
- 建议规则：相选 → 选（词长降序无冲突，无需否定断言）
- 用户确认：□ 按建议入表 □ 改为 ______ □ 不入表
5.3 用户确认与落地
1. AI 把填好初判的清单一次性发给用户（低置信度的排在前面），用户逐条勾选；
2. 只把用户确认过的规则写进 glossary.py——未确认的一律先不入表
（入表会不可逆覆盖原文，这是铁律）；
3. 每确认一批，立刻 "$PY" reapply.py transcript/PXX 重跑受影响了章节的校正，无需重转写；
4. 全部落地后 "$PY" verify_terms.py 复扫叠加错字，再进阶段⑤。
验收：清单中每个候选词都有「AI 初判 + 用户结论」两列填满；用户确认过的规则 100% 入表；
转写稿中不再出现清单里的未裁定词形。
6. 阶段⑤：逐字稿 Markdown + 索引
"$PY" build_md.py transcript/P04 4 "2027《毛概》第2章（下）"   # 单讲 → 带时间轴 md
"$PY" build_index.py            # 汇总索引（区分 已完成/不完整/待转写）
"$PY" build_all.py              # 收尾一键：重跑校正→生成md→生成索引
- 格式：**[MM:SS]** 文本，时间戳可直接跳回原视频，便于回查存疑处。
- 每讲开头自动标注覆盖率（「0~70:32 完整覆盖」或「仅转写至 …」）。
7. 阶段⑥：知识点提炼（AI 主战场，产出背诵手册）
逐讲读逐字稿，按固定结构提炼（原版五段式 / 删减版四段式）：
段
内容
说明
本章讲什么
一段话框架
删减版可去掉（ oral 复述价值低）
核心考点
3~6 条，含考纲定位
后面标注考纲条目号
必背标准提法
原文级表述，加粗
主观题答案主干，一条不许删
口诀助记
老师口播或自编
没有就写「本讲无口诀」，不要留「（本讲老师未给口诀）」这类授课口吻
易混淆易错
成对辨析
单选题高频陷阱
工程纪律（本会话多次 500 错误的根因）：
- 必须分篇写作再 cat 合并：35 讲一次生成必被网关截断。按「卷首 / 毛概上下 / 新思想上下 /
思修上下」分 7 篇分别写，每篇一个小文件，最后 cat 删减版_*.md > 知识点背诵手册.md。
- 每次 Write 后立即核对字数再继续，发现截断重写该篇。
- 口误/不确定表述统一标记「（存疑，见原片 MM:SS）」，提炼完成后统一修正，
修正要给出通说依据（如苏联解体 1991 年底），并把「存疑」字样清零再进成品。
产出四本：知识点背诵手册.md（总）+ 知识点_毛概/新思想/思修.md（三分册，内容同源拆分）。
8. 阶段⑦：考纲删减版（按需，以本项目的江西版为例）
先联网核实省级政策，这是删减的唯一硬依据（缺一项不删）：
考试说明全文（题型分值、能力层次比例）、各科目权重、时政窗口（精确到起止日）、
指定教材及版本年份、材料分析题是否要求本土案例。
三道判定门（按序过，任一不过即删）：
考纲门：该知识点在考试说明三级条目里吗？  不在 → 删
题型门：它能变成主观题答案吗？            纯客观碎点 → 删
本土门：材料题要「理论+本省案例」，需配省情素材卡 → 补
优先级原则：主观题 55 分＞客观题（江西实测：单选15×2＋多选5×3＋简答3×10＋材料2×12.5）。
- 一条都不删的：必背标准提法、口诀、易混淆易错（都是主观题弹药）。
- 优先删的：单纯识记型史实细节、考纲外的拓展、旧版教材已删内容。
- 新增三件套：卷首考试速览表（题型/分值/权重/时政窗口）、文末时政与会议精神专区
（按最新公报原文）、省情素材卡（12 个理论点 → 本省案例）。
- 瘦身幅度参考：本项目 56396 → 35125 字（-38%），主观题弹药零损失。
写作纪律同阶段⑥：分篇写、逐篇核对、cat 合并。
9. 阶段⑧：打印版 Word 定稿（全自动 + 机器校验）
9.1 md2print.py 生成 HTML
"$PY" md2print.py --md 知识点背诵手册_江西升本版.md --out formatted.html \
  --doc-title "专升本政治 · 知识点背诵手册（江西升本版）" \
  --cover-eyebrow "江西专升本 · 政治理论" --cover-title "知识点背诵手册" \
  --cover-sub "江西升本版 · 依据2026版考试说明删减" \
  --cover-meta-1 "题型：…" --cover-meta-2 "权重：…" --cover-meta-3 "共35讲 · 每讲独立起页"
# 分册加：--keep-first-h1 --first-h1-text "第一篇  毛泽东思想和中国特色社会主义理论体系概论（毛概）"
封面六件套从旧 HTML 反推还原即可；改版式时先重生成到 _new.html 做 token 级 diff，确认只有目标行变化再就位。
9.2 每讲独立分页（本版本核心版式，引擎铁律）
- html-to-doc 引擎只认 <section class="chapter"> 边界分页；
CSS page-break-before（类选择器/内联/h2 全试过）一律无效，实测页数反降（62→59）。
- md2print 的 render() 规则：遇到 h2 且当前 section 非空就 close_section()，
达到「每一讲独立起页」；篇标题（h1）开新节。规则收紧历史：
「节内含非标题块」→「节非空」（让篇标题后直接跟第一讲时也分页，两版观感一致）。
- 想恢复连续排版：把该条件改回永假即可，一句话的事。
- Markdown 表格必须走 <table class="md-table">（thead/tbody 结构 + 0.5pt 边框样式），
否则整表被拼成一个超长 <p>，触发 TQ-01 复检拒检。
9.3 转换链（顺序固定，预览锁必须绕）
CONV=.../workbuddy-docx/.../skills/html-to-doc/scripts/convert_doc.py
# ① 先写 staging，避开本地预览打开的 ~$ 锁（直接写目标会 save_file 失败）
"$PY" $CONV --html formatted.html --output staging/xxx.docx \
  --require-page-furniture --require-toc
# ② 收尾：去末尾空分节/空段落、「目 录」标题放大、补 pgNumType start=1
"$PY" docx_finish.py staging/xxx.docx
# ③ mv 覆盖目标
mv -f staging/xxx.docx 目标.docx
# ④ Word COM 刷新目录域（目录页码校准的唯一办法）
#    Fields.Update() → 逐个 TablesOfContents.Update() → 再 Fields.Update() → Save()
#    → ExportAsFixedFormat 出 PDF；用完必须 word.Quit()，否则残留 WINWORD 进程锁文件
# ⑤ 校验（见 9.4）
9.4 校验通过标准（未全过不交付）
"$PY" verify_paging2.py --docx 目标.docx --pdf check.pdf --lectures P01-P35 --toc-count 38
检查项
标准
讲次独立起页
35/35（每个 P 编号都出现在某页首行）
目录页码
42/42 与 PDF 真实页 100% 一致
空白页
无
加粗标记
全文 ** 零残留（标题也必须走 inline() 解析，曾漏转印出星号）
校验脚本的两个关键技巧：正文页靠页脚「第 N 页」识别（天然排除目录页干扰）；
<w:t[^>]*> 会误配 <w:tab/>，提取文本必须写 <w:t(?:\s[^>]*)?>。
9.5 交付
present_files 打开成品 docx，并附：最终页数、目录条目数、独立起页数、修正记录。
10. 已知坑速查表
#
坑
现象
解法
1
长文生成被截断
500/502 网关错误、Write 缺参数
分篇写（7 篇以内/篇），每篇写完核对字数
2
断点续跑误判完成
章节只跑 1/5 段却显示完成
按 done_chunk >= ceil(时长/960) 判定
3
分片硬切伤句
段尾截断句、段首半句重复
每片前移 12 秒重听，丢弃重叠区内片段
4
模型目录空
Unable to open file 'model.bin'
Windows 软链接失败，用 fix_model.py 复制真实文件
5
PyAV 19 不兼容
metadata_errors TypeError
audio_io.py 自解码再喂 transcribe()
6
术语表叠加错字
「会会」「济济」
词长降序 + 否定断言；verify_terms.py 复扫
7
单字规则误伤
「臣」被统一改错
禁用单字规则，改多字上下文规则
8
入表覆盖原文
错改后无法回溯
拿不准标存疑，用户核定后才入表；修正按行匹配不用浮点时间戳
9
CSS 分页无效
page-break-before 完全无效、页数反降
只用 <section class="chapter"> 边界
10
预览锁文件
save_file 报 ~$xxx.docx
先写 staging/ 子目录再 mv 覆盖
11
目录页码不准
目录页与真实页不符
Word COM 刷域后再导出 PDF 逐条核对
12
标题星号漏转
成品印出 **
标题也必须走 inline() 而非 esc()
13
Word 进程残留
下次打开文件被锁
COM 用完 Quit + ReleaseComObject
14
存疑词直接抛给用户
用户没上下文，拍板靠猜
走阶段④：propose_terms.py 出清单 → AI 填初判/依据/置信度 → 用户只做确认
11. 产物与脚本清单
项目根/
├── 流程SOP.md                    ← 本文（复跑入口）
├── bilibili_politics/            ← 阶段①~⑤的工程目录（复制即可开新项目）
│   ├── README.md                 逐字稿项目说明（含分P目录表与质量说明）
│   ├── audio/ transcript/ pages.json
│   ├── download_audio.py         ★新链接入口：python download_audio.py <链接或BV>
│   ├── transcribe_one.py / run_all.py / run_paced.py / audio_io.py / store.py
│   ├── polish.py / glossary.py / mine_terms.py / reapply.py / verify_terms.py
│   ├── propose_terms.py          ★阶段④：存疑词清单（AI 初判 → 用户确认）
│   ├── 待确认术语清单.md          阶段④产出（带时间戳的候选词全量上下文）
│   ├── build_md.py / build_index.py / build_all.py
│   ├── md2print.py / docx_finish.py / verify_paging.py / verify_paging2.py
│   ├── 术语对照表.md / 评审报告.md
│   ├── P01~P35 逐字稿 md、知识点_*.md、删减版_*.md、知识点背诵手册*.md
└── output/
    ├── 20261021-kbjx/            江西升本版（专升本政治·知识点背诵手册_江西升本版.docx，78页）
    └── 20261021-kbprint/
        ├── stage2/formatted-*.html    四本书的排版源（改版式只动这里）
        ├── stage3/                    四本成品 docx
        │   ├── 专升本政治·知识点背诵手册.docx（103页）
        │   ├── 《毛概》知识点提炼.docx（33页）
        │   ├── 《新思想》知识点提炼.docx（46页）
        │   └── 《思修》知识点提炼.docx（27页）
        └── pipeline-state.yaml        管线状态记录
12. 新对话可直接复用的两段话
给用户（启动）：
跑流水线：<新链接>｜读 流程SOP.md，从阶段①开始，遇到需要我拍板的（术语存疑、删减边界）先汇总问我。
给 AI（接单自查）：
1. 先读 流程SOP.md 与 bilibili_politics/README.md；2) 复制 bilibili_politics/ 到新目录或
清空 audio/transcript/ 后跑 download_audio.py 传新 BV；3) 转写用 run_paced.py，参数不改；
2. 同科目沿用 glossary.py 并用 mine_terms.py 增量补词，换科目重挖并改 PROMPT；
3. 存疑词必须走阶段④：propose_terms.py 出清单 → AI 逐条填初判/依据/置信度（低置信度排前）
→ 一次性发用户确认 → 只把确认过的写进 glossary.py → reapply.py 重跑校正；
4. 提炼/删减分篇写、逐篇核对字数再 cat 合并；7) 排版走 md2print→staging→docx_finish→mv→
Word COM 刷域→verify_pagin
