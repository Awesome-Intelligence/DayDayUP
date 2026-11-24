# DayDayUp 技术选型调研

> 调研时间：2026-11-24
> 原则：在自有 Flutter 底座上做加法，优先集成现成框架/库/开源项目，少造轮子；核心交互（标注）若无现成库，给出自研路径。
> 数据来源：pub.dev 包页面、GitHub 仓库页及下文标注的出处，均为 2026-11-24 前后核查。查不到或未核实的项明确标「待验证」，不编造。

## 0. 底座现状（先行核查结论）

已通读仓库 `Awesome-Intelligence/DayDayUP` 的 README、ARCHITECTURE.md 与根 pubspec.yaml，底座已经具备的东西不少，选型必须在此基础上做加法：

| 已有能力 | 底座实现 | 对 DayDayUp 的意义 |
|---|---|---|
| 跨端 UI 框架 | Flutter（Dart SDK ^3.14），pub workspace 多包：`kit` / `kit-agent` / `kit-membership` / `apps/*` | DayDayUp 写在 `apps/daydayup/`，`resolution: workspace`，**只新增、不改底座**（底座有 base-guard 门禁） |
| Agent 运行时 | `kit-agent`：`dart_agent_core` 本机运行，`AgentEvent.Text` 流式增量事件；或 `GatewayAgentRuntime` 走自带 TypeScript 网关（`backend/`，SSE 协议两端已对齐） | 对话的流式管线现成，UI 只消费事件流 |
| 聊天 UI 基础件 | `kit/lib/src/ui/agent/`：`ChatMessage` / `ChatMessageList`（气泡 + 工具过程折叠）/ `ToolApprovalCard` | 气泡列表不用从零写，也不用引第三方聊天框架（见 §1） |
| 主题系统 | token 驱动，5 套预设含 **Mono**（黑白灰） | 与已定的黑白灰极简设计语言天然吻合，加一套 DayDayUp tokens 即可 |
| 会员/额度 | `kit-membership`：档位表 + `consume()` 额度引擎 + 验签，纯 Dart 零依赖 | 订阅变现不用选型，播客生成/批问次数直接挂额度 |
| 数据层 | `kit` 的 `EntityStore`（JSON / Markdown codec，文件落盘） | 标注、卡片、文稿等业务数据直接落这层 |
| 推送 | ntfy（app 活着时）+ FCM（app 被杀时）双通道已打通 | 播客生成、文档解析这类异步任务完成通知现成 |
| 模型接入现状 | Anthropic API 直连或自建转发代理；key 走环境变量/`--dart-define`，不进仓库 | 网关层选型见 §8 |

**结论**：DayDayUp 真正需要新增选型的只有六块——Markdown/标注渲染、音频播放、播客生成管线、文档解析、向量检索、FSRS 调度。其余复用底座。

## 1. 总览表

| 功能点 | 推荐方案 | 许可证 | 集成方式 |
|---|---|---|---|
| AI 对话 UI | 复用底座 `kit` 的 ChatMessageList + 在 app 内扩展气泡；不引第三方聊天框架 | 底座自有 | 直接复用 |
| Markdown 渲染 | **自研块级渲染**（`package:markdown` 解析 AST → 块模型 → TextSpan 树），与标注共用同一份块模型；`gpt_markdown` 仅作流式阶段的过渡备选 | markdown 为 BSD；gpt_markdown 为 BSD-3-Clause | 端侧 Dart 包 + 自研组件 |
| 划线标注（核心） | **自研**：W3C/Hypothesis 三选择器锚定（块ID + 块内偏移 + TextQuote 引文），SelectionArea/SelectableText 取选区 | 算法思路来自 Hypothesis client（BSD-2-Clause，可参考实现） | 端侧自研，落 `apps/daydayup` |
| 音频播放 | `just_audio` + `audio_service` + `audio_session` | MIT 系（pub.dev 包页） | 端侧 Flutter 插件 |
| 播客脚本+合成管线 | 服务端 Python 管线，编排参考 **Podcastfy**（可直接用其 client 或借鉴其分段/拼接结构） | Apache-2.0 | 服务端独立任务服务 |
| TTS（双音色+时间戳） | **MiniMax speech-2.8-hd**（词级时间戳自带字符偏移）；备选阿里云百炼 CosyVoice 云服务、火山豆包 TTS；降本远期选项为自部署 CosyVoice 2 | 云服务按量计费；CosyVoice 2 开源版 Apache-2.0 | 服务端调用云 API |
| 文档解析（重） | 服务端 **MinerU**（中文/公式/表格最强）；许可证敏感时用 **Docling** | MinerU：基于 Apache-2.0 的自定义许可证（有大规模商用门槛，见 §6）；Docling：MIT | 服务端 Python，异步任务 |
| 文档解析（轻，端侧） | PDF：`pdfrx_engine` 逐页抽文本；EPUB：zip+XML 自解或 `epubx`（待验证） | pdfrx：MIT | 端侧 Dart |
| 向量检索（端侧） | SQLite + **sqlite-vec**（vec0 KNN），经 `sqlite3` Dart 包集成，薄封装隔离 | MIT / Apache-2.0 双许可 | 端侧原生扩展 |
| Embedding 生成 | 经自有网关调云端 embedding API（国产云，OpenAI 兼容协议） | 按量计费 | 服务端代理 |
| 间隔重复 | pub.dev **`fsrs`**（dart-fsrs，FSRS-6） | MIT | 端侧纯 Dart 包 |
| AI 接入层 | 扩展底座自带 TS 网关做路由与密钥托管；需要聚合计费层时再引入 **LiteLLM** | LiteLLM 核心 MIT（企业目录除外） | 服务端 |

## 2. AI 对话 UI

| 候选 | 状态（出处：pub.dev / Flutter Gems，2026-11） | 评价 |
|---|---|---|
| 底座 `kit` ChatMessageList | 底座自带，气泡 + 过程折叠 + 流式追加的话术已处理（ARCHITECTURE.md 明确警告 Result.text 不能重复追加） | 与 agent 事件流、Mono 主题、契约体系严丝合缝 |
| flutter_chat_ui（Flyer Chat） | v2 大版本重构后活跃维护中，约 1.6K likes（Flutter Gems）；近期 changelog 在跟进 Flutter 的 material_ui 迁移 | 功能强，但它是通用 IM 模型（多用户、已读、附件），DayDayUp 是单用户 AI 学习对话，且标注要深度定制气泡内部，引入后大量能力用不上、气泡定制反而受其模型约束 |
| dash_chat_2 | v0.6.1，约 310 likes，Flutter Gems 标注维护状态 Average | 维护节奏一般，且内置 Markdown 走旧 flutter_markdown，不推荐 |

**推荐：不引入第三方聊天框架。** 用底座 ChatMessageList 做列表与输入区，DayDayUp 只在 app 内写自己的 AI 气泡组件（承载 §4 的标注渲染）。理由：底座 fork 规矩是「只新增不改底座」，而气泡内部正是标注交互的主战场，必须完全可控；聊天框架省下的只是列表壳子，底座已经给了。

## 3. Markdown 渲染

| 候选 | 状态 | 对标注的支持 |
|---|---|---|
| `gpt_markdown` | 专为 LLM 输出设计：流式增量渲染（只重建尾部）、内置 LaTeX 与代码高亮、处理半截语法；BSD-3-Clause，pub.dev 月下载约 19 万（出处：useval.io 2026-11-17 对比文） | 黑盒渲染，不暴露 span 级结构，**无法承载标注** |
| `flutter_markdown_plus` | Google 已停维的 flutter_markdown 的社区续作，活跃维护；`selectable: true` + `onSelectionChanged` + `contextMenuBuilder`（出处：pub.dev 包页/changelog） | 选区按内部块逐个 SelectableText 实现，能拿到块内纯文本选区，但拿不到统一的文档级偏移模型，跨块结构（表格/代码块）与标注锚定仍要自己再建一层映射 |
| `markdown_widget` | 活跃，自带 TOC、代码高亮（出处同上对比文） | 同上，渲染层黑盒 |
| **自研（基于 `package:markdown`）** | `markdown` 是 Dart 官方维护的解析器，输出 AST | 完全掌控：AST → 块模型（每块有稳定 blockId、纯文本、行内样式区间）→ 渲染与锚定共用同一份模型 |

**推荐：渲染管线自研，这是全项目最值得自研的一块。** 标注是 DayDayUp 的核心资产，锚定数据（§4）必须建立在自己掌控的「块 + 偏移」模型上，任何黑盒渲染库都会让锚定变成逆向工程。具体做法：

- 解析：`package:markdown` 解析 AI 回答为 AST，抽取块级结构（段落/标题/列表项/引用/代码块/表格行）。
- 块模型：`AnswerBlock { blockId, type, plainText, inlineRuns[] }`，其中 inlineRuns 记录粗体/斜体/行内代码/链接在 plainText 上的区间——**渲染和标注都只认 plainText 坐标系**，Markdown 源码偏移永不进入标注数据。
- 渲染：每块构建一棵 TextSpan 树（样式区间 + 标注高亮区间求交叠切分后合并），放进 SelectableText（见 §4）。
- 流式阶段：生成中先以纯文本 + 光标追加显示（或临时用 gpt_markdown 渲染），**生成结束（Result 事件）后切换到结构化块渲染并开放标注**——流式中做标注本来也没有产品意义，这样切换最稳。

## 4. 划线标注（核心交互）自研方案

没有现成 Flutter 库能做「在 AI 回答上多处划线 + 每处挂问题 + 攒批提交 + 答案锚定回原文」。Web 生态的成熟参照是 Hypothesis：其 client（BSD-2-Clause）用**三选择器冗余**保存锚点、按级联策略回定位，是被生产验证过的方案（出处：web.hypothes.is《Fuzzy Anchoring》、hypothesis/client 仓库）。DayDayUp 的场景比开放网页简单得多——**AI 回答一旦生成即不可变、且原文在本地存档**，所以锚定可以做到近乎 100% 命中，模糊匹配只是保险丝。

### 4.1 数据结构

```text
Annotation {
  id            // uuid
  messageId     // 所属 AI 回答
  blockId       // 块定位（第一顺位）
  start, end    // 块 plainText 内的字符偏移（第二顺位，TextPositionSelector 等价物）
  quote         // 被标注的原文文本（exact）
  prefix        // quote 前最多 32 字符
  suffix        // quote 后最多 32 字符（以上为 TextQuoteSelector 等价物）
  question      // 用户挂的问题（可空 = 纯高亮）
  answerId      // 批问返回的答案 id（答案锚定回原文靠这条双向链接）
  state         // active / orphan
  fsrsCardId?   // 若已生成复习卡片
}
```

三选择器思路直接借 W3C Web Annotation Data Model 的 `TextPositionSelector` + `TextQuoteSelector`（出处：W3C Recommendation, 2017-02-23）。prefix/suffix 取 32 字符是 Hypothesis 的既有实践。注意 Hypothesis 在 2026 年有一个已知问题记录：当原文被编辑、且 32 字符上下文恰好与别处雷同时，模糊匹配可能静默锚到错误位置（出处：hypothesis/client issue #7571）——DayDayUp 原文不可变，此风险基本不存在，但回定位策略仍应「宁可 orphan，不可错锚」。

### 4.2 回定位级联（渲染时执行）

1. 按 blockId 找到块，校验该块 [start, end) 处文本 == quote → 命中（绝大多数情况走这条）。
2. 块内偏移失效时，在块内对 quote 做精确搜索（配合 prefix/suffix 消歧多处重复）。
3. 仍失败时，全文范围模糊搜索：归一化（空白/标点）后做近似匹配，相似度低于阈值则放弃。
4. 全部失败 → 标 `orphan`，UI 上把标注收进「未定位标注」列表展示原文与问题，不静默丢弃、不错锚。

### 4.3 选区获取与渲染（Flutter 实现要点）

- **选区**：每个块用一个 `SelectableText.rich(spanTree)` 渲染，`onSelectionChanged` 给出**块内**的 TextSelection——正好落进数据模型的坐标系，零换算。长按选区后用自定义工具条（参考 flutter_markdown_plus 的 `contextMenuBuilder` 思路）提供「标注提问」按钮，弹出输入框写问题，确认后生成 Annotation 并重建该块 span 树。
- **高亮渲染**：把块的样式区间与所有标注区间做一维区间切分，逐段生成 TextSpan，标注段加 `backgroundColor`（黑白灰设计下用不同灰阶 + 下划线区分多处标注，与 v3 概念图一致）；序号徽章（①②③）用 WidgetSpan 或段首插入的带样式 TextSpan 实现，点按徽章打开对应问题/答案卡。
- **攒批提交**：同一回答内 state=active 且有 question 的标注攒成一批，一次请求发给 agent（systemPrompt 约定按序号逐条作答、答案 JSON 带 annotationId 回填），返回后按 answerId 锚定，在原文旁/答案页按序号呈现——即概念图 03→04 的动线。
- **V1 明确取舍**：选区**限制在单块内**（跨块选区先不支持）；表格、代码块内的标注 V1 先只支持代码块纯文本块、表格降级为整行标注。这两条是把工期压住的关键，不要在 V1 追求全结构标注。
- **播客文稿与导入文档复用同一套**：文稿是「句块」列表、文档是「段落块」列表，块模型与锚定逻辑完全一致——这是把块模型做成基础设施的最大回报。

## 5. 音频播放

| 候选 | 状态 | 评价 |
|---|---|---|
| **just_audio + audio_service + audio_session** | Ryan Heise 维护的事实标准组合：just_audio 管解码/播放（倍速、seek、播放列表、无缝拼接），audio_service 管后台播放/锁屏/通知/耳机按键，audio_session 管音频焦点（出处：pub.dev 两包页面，audio_service 明确列出 Podcast players 为适用场景） | 播客播放需求（后台、倍速、进度、锁屏控制）全覆盖，社区教程与踩坑资料最丰富 |
| audioplayers | 轻量播放器 | 后台/锁屏生态弱于 audio_service 组合 |
| media_kit | libmpv 内核，音视频通吃 | 为一门音频需求引入 mpv 过重，且包体积大 |

**推荐：just_audio + audio_service + audio_session。** 播客按「整期一个音频文件 + 句级时间戳表」交付，播放进度流（positionStream）驱动文稿当前句高亮与自动滚动；倍速用 just_audio 的 setSpeed。无悬念的一项。

## 6. 播客生成管线（服务端）

管线形态：话题/材料 → LLM 写双人脚本（结构化 JSON：逐句、说话人）→ 逐句 TTS 合成 → ffmpeg 拼接 → 产出「音频文件 + 句级时间戳文稿」。这条管线**放服务端异步执行**（底座推送通道负责完成通知），不要在端上做。

### 6.1 编排参考

| 候选 | 状态 | 用法 |
|---|---|---|
| **Podcastfy**（souzatharsis/podcastfy） | Apache-2.0，约 6.1K stars（出处：NousResearch/hermes-agent issue #484，2026-03）；支持多 LLM、多 TTS 后端（OpenAI/ElevenLabs/Edge 等）、PDF/网页/YouTube 输入、ffmpeg 拼接 | 可直接以其 Python client 为管线骨架，或只借鉴其脚本结构与分段拼接实现，自己接国产 TTS——后者更贴合国内渠道，推荐后者 |
| Open Notebook（lfnovo/open-notebook） | MIT，约 42.1K stars，v1.15.1（2026-11-04）；1–4 位说话人播客、20+ 模型提供商（出处：GitHub 仓库页/SelfHostMap） | 是完整的 NotebookLM 式产品（Next.js + Python + SurrealDB），**不直接搬全栈**；借鉴其播客生成的产品形态与脚本控制粒度即可 |

### 6.2 TTS（双音色 + 时间戳是硬指标）

同步文稿要求 TTS 能给出时间戳。已核实的能力：

| 候选 | 时间戳能力（出处） | 评价 |
|---|---|---|
| **MiniMax speech-2.8-hd** | API 支持 `subtitle_type: word`，流式返回 `timestamped_words`：每词带毫秒级起止时间 **和在合成文本中的字符起止位置**（出处：MiniMax API 文档转载页，腾讯云 TokenHub 文档） | 字符偏移 + 时间双锚点，与文稿块模型天然对齐，**首选**；用户已有 MiniMax 渠道经验。2.8 为当前最新版，支持情绪与语气词标签，利于双人对话表现 |
| 阿里云百炼 CosyVoice 云服务 | SSE 流式带逐字时间戳（`word_timestamp_enabled`）（出处：brewreel 项目 v0.5.1 发布说明，2026-09，据官方文档实现） | 备选一；与自部署 CosyVoice 同源，迁移成本低 |
| 火山引擎豆包 TTS（seed-tts-2.0） | V3 流式接口带逐字时间戳（`enable_subtitle`）（出处同上） | 备选二；字节生态，中文音色丰富 |
| 自部署 CosyVoice 2（FunAudioLLM） | 开源 Apache-2.0（代码与 0.5B 权重均是），零样本克隆、流式、情感指令强（出处：GitHub 仓库及多个二手核查页） | 降本/自有音色时用；但**原生不直接输出词级时间戳**（据本次调研未见其 API 提供等价字段），同步文稿需另配 ASR 强制对齐（如 SenseVoice/Whisper 对合成音频回扫），列为待验证项；且需 GPU 服务器 |
| edge-tts | 免费、无需 key、带字幕输出 | **仅限内部原型**：非官方客户端、无 SLA，微软可随时限流/变更，商用有 ToS 风险，其包本身为 LGPLv3（出处：edge-tts 仓库说明及多方核查文）。绝不进生产链路 |

**推荐：生产用 MiniMax（speech-2.8-hd），按句调用、按句落时间戳。** 文稿同步以「句」为基本单位：每句存 {text, speaker, startMs, endMs}，播放时按 position 二分定位当前句；MiniMax 的词级时间戳可支撑二期的逐词高亮。脚本生成用强模型（见 §8），脚本 JSON 结构由 Podcastfy 的对话格式借鉴。

## 7. 文件导入学习

### 7.1 解析：分两档

**端侧轻解析（即时、离线、免费）**——覆盖「有文本层的常规文件」：

| 格式 | 方案 | 说明 |
|---|---|---|
| PDF | **pdfrx / pdfrx_engine**（MIT，PDFium 内核，pub.dev 当前 2.3.x） | 既做阅读器（自带文本选择、高亮绘制回调）又能逐页抽文本与字符矩形——导入文档的「读 + 标注」可与它共用页文本模型；注意它抽的是文本层，扫描件无文本层时抽不到 |
| EPUB | zip + XML 自解（`archive` + `xml` 包）或 `epubx` | EPUB 本质是打包的 XHTML，自解风险低；`epubx` 的当前维护状态本次未核实，列待验证，二选一在开发期定 |
| DOCX/TXT/MD | DOCX 同理 zip+XML 自解正文；TXT/MD 直读 | 足够学习场景 |

**服务端重解析（异步、高质量）**——扫描件、复杂版式、表格/公式、双栏论文走这条：

| 候选 | 状态 | 许可证要点 |
|---|---|---|
| **MinerU**（opendatalab） | 约 80K+ stars（出处：file2markdown 对比文约 80.5K、aislop 核查约 82.7K，2026-10）；原生支持 PDF/DOCX/PPTX/XLSX/图片，公式→LaTeX、表格→HTML，中文与学术文档是其主场 | **基于 Apache-2.0 的自定义许可证**：超过 1 亿 MAU 或月收入 2000 万美元需另购商业授权，在线服务有署名要求（出处：PyPI 页与对比文）。DayDayUp 现阶段远未触线，但署名要求要在产品里落地（如设置页标注），且要在规模化前复核 |
| Docling（IBM 发起，现 LF AI & Data 基金会） | 约 67.7K stars，MIT；CPU 可跑、部署轻（出处同上） | 许可证最干净；中文 OCR 需另配引擎、复杂中文版式精度弱于 MinerU，做许可证敏感时的备选 |
| LlamaParse 等云解析 | 付费云服务 | 数据出境 + 按页计费，不作默认路径，仅列备选 |

**推荐：默认服务端 MinerU，端侧 pdfrx 兜底快速导入。** 导入流程：文件先端侧轻解析立即能读能问；同时（或用户触发时）走服务端重解析替换为高质量结构化文本。两档产物都归一成 §4 的块模型（段落块带页码），分块、标注、RAG 全部复用。

### 7.2 分块与 RAG

- **分块**：不需要框架。按标题层级切分、段落聚合到约 512 token、重叠 10–15%，保留 {docId, blockId, page} 元数据，自研几十行。LangChain/LlamaIndex 的 splitter 思路可借鉴，但为这点逻辑引 Python 全家桶不值——解析服务本身是 Python 时顺手用其生态即可。
- **Embedding**：V1 走网关调云端 API（国产云的 OpenAI 兼容 embedding 接口，如各厂商 text-embedding 系列；具体型号与价格在接入时按当时文档定，本次未逐一核价，不写死）。理由：端侧 embedding 要下模型包、占体积且效果参差；用户文件是学习资料，云端生成可接受，密钥与计费由自有网关托管。
- **向量存储/检索（端侧）**：

| 候选 | 状态 | 评价 |
|---|---|---|
| **sqlite-vec** | asg017/sqlite-vec，MIT/Apache-2.0 双许可，约 8.3K stars；纯 C 零依赖，vec0 虚拟表做 KNN（出处：GitHub 仓库页） | 与底座的 SQLite/文件数据层同构；单本书千级 chunk 的规模下暴力 KNN 都够用。**风险：仍是 pre-v1（0.1.x），且 2026 年提交节奏放缓（出处：hysenlabs 项目页，仓库最近 push 为 2026-05）**——必须锁定版本并在自家代码里做薄封装（只暴露 upsert/search 两口），将来可平滑替换 |
| ObjectBox | HNSW 向量检索成熟、Dart SDK 完善（出处：多篇端侧 RAG 实践文） | 原生核心为闭源二进制，其免费商用条款需按官网复核（待验证）；与底座现有数据层并行再引一套 DB，收益不抵复杂度，列备选 |
| 纯 Dart 暴力余弦 | 零依赖 | 千级向量内完全可行，可作 V1 的兜底/过渡：先暴力检索上线，sqlite-vec 封装就绪后切换——检索层接口先行，實現可换 |

- **二期方向（仅记录）**：端侧离线 embedding 已有现成生态——Flutter Edge AI（原 Flutter Gemma）+ EmbeddingGemma，其 RAG 的 SQLite 存储恰好也是 sqlite-vec 实现（出处：pub.dev `flutter_edge_ai_sqlite` 页）。与本推荐路线同构，将来做离线版不用推翻存储层。

## 8. 学习模式：间隔重复与卡片

| 候选 | 状态 | 评价 |
|---|---|---|
| **pub.dev `fsrs`（open-spaced-repetition/dart-fsrs）** | MIT；py-fsrs 的 Dart 移植，FSRS-6 的 21 个参数，`Scheduler.reviewCard` API，Card/ReviewLog 均可 JSON 序列化（出处：pub.dev 包页，包元数据更新于 2025-06，当前 2.0.x） | 纯 Dart、零原生依赖，与底座 EntityStore 落盘方式天然契合；参数用官方默认值即可，个人化优化（optimizer）V1 不需要 |
| fsrs-rs-dart（官方 Rust 绑
...[truncated 5491 chars]