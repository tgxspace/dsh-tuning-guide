# billion-context（bili）优化指南 —— 授之以渔版

> 本文的目标不是给出"标准答案参数"，而是教你（及你的模型）**如何诊断自己的机器、如何从自己的窗口/线路/模型推导出自己的参数、以及如何验证改动有效**。参数是诊断的结果，不是起点。
>
> 本文档设计为可直接整份粘贴进你正在会话的编码 Agent（DSH / OpenCode / Pi 等），并在末尾附上要它对本人机器执行的指令（见 §8）。

---

## 1. 30 秒心智模型

billion-context（下称 bili）是嵌在宿主里的上下文压缩代理，模型流量经它转发。理解调参只需要四个概念：

1. **有效窗口（effective window）** = 模型原生窗口 − 输出预留。输出预留 = `min(客户端 max_tokens, outputHeadroomMaxPct × 窗口)`，默认 cap 25%。**所有压缩判定（nudge/preflight）都按有效窗口打分，不是原生窗口。** 客户端把 max_tokens 开得很大（很多宿主默认按窗口一半以上发），预留就会吃掉窗口的一整块——这通常是"压缩来得太早太密"的第一根因。
2. **四条阈值线**（全部按有效窗口的百分比派生，公式写死在 bili 内部，改窗口=改所有线）：
   - **增长门**（约 15% 处，按窗口派生，最低 22500）：可折叠内容相对基线增长达标才发"该压缩了"的提醒（nudge）。
   - **批次线**（约 34% 处）：攒够这么多可折叠 token 才发 T1 级提醒。
   - **压力带入口**（默认 75%，`compress.maxContextLimit`）：进入压力带后**绕过**上面两道节奏门，改用"最小可回收量"判定。
   - **紧急线**（默认 95%，`compress.emergencyThresholdPercent`，必须 ≥ maxContextLimit）：直接下 EMERGENCY 指令，并触发对超大工具输出的紧急截断。
   - 另有 **preflight 墙** = 有效窗口本身：请求的输入已到墙边就先压缩再转发（硬兜底）。
3. **nudge 是"模型驱动的"**：bili 不替你压，它注入提醒，由模型自己调 `compress` 工具决定压哪些区间、写什么摘要。所以**模型的性格直接决定压缩行为**——小模型/flash 模型被提醒后倾向只压最小的可压区间（每次 3–10K），大模型倾向整批折叠（30–50K）。同一条配置喂给不同的模型，产出完全不同的压缩频率。
4. **每次折叠都要付"前缀重暖"税**：折叠重写前缀 → KV cache 从折点起作废 → 折叠后第一次请求把整段前缀按新输入重付一次。压缩省不省，取决于"这次折掉的量 × 之后还要跑多少轮" 能不能盖过这次重暖。这就是为什么**同样省 10K，大窗口+长会话的深折赚、微折血亏**。

---

## 2. 先测量，后调参：本机数据采集清单

调参前让你的 Agent 先采齐以下事实（全部可程序化获取，不需要猜）：

### 2.1 基础参数
| 采什么 | 去哪采 |
|---|---|
| 模型原生窗口、输出上限 | 宿主模型配置（DSH: `cordis.patch.yml`；OpenCode: provider 配置）；bili 侧 `modelContextLimit` 覆盖见 §4 |
| 客户端 max_tokens 实发值 | 日志 `[headroom]` 行，格式：`effective window X -> Y (reserved R for max output M, cap 0.25 via compress.outputHeadroomMaxPct)` |
| 有效窗口与四条线的本机数值 | 同上 `[headroom]` 行 + §2.2 的 `nudge idle` 行分母 |
| bili 版本/端口/运行状态 | `GET http://127.0.0.1:<端口>/__bili/status`（端口从宿主代理配置找，netstat 辅助定位）；`/__bili/health` |
| 当前生效配置 | `GET /__bili/config`（返回解析后的合并配置 + raw 原文） |

### 2.2 行为曲线（判断"压缩病"的核心证据）
日志（默认 `~/.local/state/billion-context/bili.log`，Windows 为 `C:\Users\<用户名>\.local\state\billion-context\bili.log`）里 grep 这几类行：

- `[headroom]` —— 有效窗口决议（每会话发布一次）
- `nudge idle` / `nudge INJECT` —— 每请求的决策状态行，含全部关键数：`usage=NN% (当前输入/有效窗口), growth=…/增长floor (ref=基线, interval=批次线), pendingT1=…/批次阈值`，尾部 `reason` 直接写明本轮为什么压/为什么不压
- `preflight` —— 触墙强压事件（含 k̂ 校准因子，见 §3.4）
- `output budget clamped` —— 输出预算钳制（保证输入+输出不炸原生窗口，正常安全机制）
- `[acp-usage] input=… cached=… (cache hit NN%)` —— 每请求 cache 命中

**健康的指纹**：cache 命中 95–97%；`pendingT1` 一路爬坡到批次线才出现一次 INJECT，一次注入后 usage 深跌（≥30% 窗口），之后长 plateau。
**亚健康的指纹**：
- usage 锯齿振荡（进带→小跌→爬回→再跌，跌距 3–10K），`nudge INJECT` 高频出现 → 高频微折（病 #3/#4）
- cache 命中长期 <85%，且 `[acp-usage]` 显示输入小命中也小（前缀刚被重写）→ 折太碎
- `preflight` 频繁 → 压得太晚，模型一直没听 nudge，全靠硬兜底
- `[acp-usage]` 出现"输入约 8 万、cache 命中 0% 且请求成功、反复如此"→ 该车道无 KV 缓存（sidecar 直连，属正常，见 §7 陷阱）

### 2.3 经济账
`acp_cache` 工具（宿主里）或 `GET /__bili/cache-report` 出报告：
- **GRAND LEDGER**：总输入/总命中/hit%；miss 拆成 新增内容 / 压缩重付 / 上游TTL
- **FOLD ECONOMICS**：每次折叠的"净省"与**是否回本**（回本 = 重暖成本 ≤ 省量 × 之后轮数）
- 判据：压缩自身成本应 ≤2%；出现"未回本"折叠 → 折叠事件数太多，走 §5 处方
- 经验基线：nudgeGrowthTokens 默认 50000（恒定与窗口无关）；**深折回本周期约 5 轮窗口，浅折约 2.5 轮**——会话平均长度（多少轮）比窗口绝对大小更能决定该压多深

### 2.4 历史尸检：分析一个指定会话的压缩过程

回答"那个会话当时压得怎么样/为什么亏/折叠几次"这类复盘问题，按以下取证链走（以 DSH 为例，其它宿主同理找对应存储）：

1. **骨架**：宿主会话工具（DSH: `session_trace({session_id})`）→ 轮数、每轮步数、事件数、时间范围、父/子会话树。会话 id 来源：会话搜索工具、客户端界面、宿主会话目录名。
2. **决策流（权威记录，覆盖任意历史会话）**：在 bili.log 里 grep `sess=session-<uuid>`——`[headroom]` 行给出该会话发布的有效窗口；`nudge idle`/`nudge INJECT` 逐请求给出阈值线与 reason；`preflight` 行给触墙强压与 k̂；`[acp-usage]` 行给每请求 cache 命中。会话的全部压缩决策都在这个流里。
3. **曲线重建**：每请求的 usage 存在宿主会话存储里（DSH: 会话目录下 `session.v4.jsonl.zstd`，取 `input_tokens+cache_read_input_tokens` 为 prompt 大小）。prompt 序列画出来，**每次大幅下跌 = 一次折叠**；跌点时间戳对回日志的 INJECT/preflight 行，即可逐折归因（哪条线触发、折掉多少）。
   - 无 zstd CLI 时的解压配方：Python `ctypes` 加载 `libzstd.dll`（DSH 的 `@dsh-ops/bash-win32-x64/ucrt64/bin/` 自带），`ZSTD_decompressBound` + `ZSTD_decompress` 两步即可；node 内建 zlib 无 zstd。
4. **经济账**：当前会话用 `acp_cache`；其它会话看 Web UI 的逐会话缓存页（JSON 出口 `GET /__bili/sessions`），读 FOLD ECONOMICS 的净省与回本判定。
5. **授权边界**（DSH 实测）：会话事件级读取工具是 workspace 作用域的——跨工作区会话直接拒（"session target is outside the caller workspace"）；会话全文索引只覆盖**已持久化**的事件流，取不到时先确认持久化状态，再用日志路径重建。
6. 四个坑：
   - 同一宿主产品的桌面/网页实例各持独立会话库，同一对话的会话 UUID 不同——盘上找不到先确认是哪套库
   - 事件流未持久化的会话只能靠 bili.log + 摘要重建，缺什么要明说
   - 模型面看不到的工具调用（如压缩事件）在宿主日志里是 log-only 的——折叠次数以日志为准，别拿模型面可见数
   - 搜索工具默认只吃 UTF-8：GBK 编码的缓存/网页文件会被静默跳过（命中 0 ≠ 不存在），换 PowerShell `Select-String` 或先转码

---

## 3. 诊断决策树：四种病根，对号入座

按你的 `[headroom]` 行和 `nudge` 行为模式，逐条排查：

### 病 #1：预留吃窗口（最常见、收益最大）
症状：`[headroom]` 行显示有效窗口明显小于原生窗口（如 196608→147456，预留 49152）。
根因：宿主按默认发超大 max_tokens，25% cap 全咬进去。**压力线、preflight 墙全部按缩水后的窗口算**，于是"离顶还远"就开始密集报警。
处方（按侵入性从低到高）：
- 宿主侧把 max output 调到真实需要的量（编码会话 16–40K 足够）—— bili 侧参数不用动，预留自动跟随
- 或 bili 侧设 `compress.outputHeadroomMaxPct` = 期望预留 ÷ 原生窗口（如 40960/196608 = 0.20833333333333334）
注意：预留只是**判定基准**，出站的 max_tokens 钳制仍按原生窗口兜底，调小预留不会造成硬溢出。

### 病 #2：压力带高频微折
症状：§2.2 的锯齿振荡；5–10 次折叠/窗口周期；acp_cache 显示多笔"未回本"。
根因：进带后节奏门（增长/批次）被设计性绕过，进带触发只看 `compress.minPressureBenefitTokens`（默认 `max(5000, 窗口1%)`）——默认值就是为"高频小折"开门的。
处方：**调大 `minPressureBenefitTokens`，让折叠攒成大批量而非高频小折**。官方文档给的示例量级：~200K 窗口设 50000（≈有效窗口的 1/3）。设 0 = 任意 pending 都折（更糟，别设）。上限约束：不能超过你典型的"最大可压缩区间"大小，否则永不触发。

### 病 #3：模型被提醒后只肯压最小范围
症状：nudge 注入频繁但每轮只跌几 K；模型多为 flash/小尺寸。
处方组合：
- `compress.minPressureBenefitTokens` 调大（同上，从源头少发提醒）
- `compress.nudgeModelDecided: true`：tier-1 提醒触发时先跑一次走会话缓存前缀的短 side call 问模型"现在压划算吗"，答否本轮不注入。零 cache 代价（side call 吃的是已缓存前缀，输出仅 200 token，且**不在宿主的历史循环里**——不会作为历史被重发，也不消耗正式请求的 cache 命中）。注意它是**节流阀不是省钱开关**：治"提醒次数"，每次真折叠的重暖税一分不少
- 提醒词是程序生成的固定文本，**不要**试图注入自定义提示词改模型行为（固定提醒词是设计，非疏漏）

### 病 #4：窗口声明与实际线路不符
症状：`nudge idle` 的 usage% 与你的体感/token 计数明显对不上；或宿主/注册表把窗口猜小了（日志 `[window] ... peek=...` 行显示来源）。
处方：
- 线路实际能力大于声明：用 `providers.<id>.compress.modelContextLimit` 按模型写真实窗口（nudge% 分母、preflight 墙、85/95 线全部按此重派生）
- 线路实际能力小于声明：**不要**用它当"防溢出阀"——它不约束出站的 max_tokens 钳制，只会造成无意义压缩；应修宿主声明
- k̂： bili 学到"上游真实 tokenizer 计数 ÷ 本地估算"的系数（见 preflight 行 `k̂=1.65` 这类）。k̂>1 = 计数真的更密（如中文密集+Qwen tokenizer），属正常，会自己学；`calibration` 钳制（`max:1` / `disabled`）只用于**虚报线**（上游报数系统性虚高），若 k̂ 稳定且请求正常则保持默认不设

---

## 4. 配置文件语法：纯覆盖层，算好派生线再写

- 文件位置：`~/.config/billion-context/billion-context.json`（XDG，Windows 落 `C:\Users\<用户名>\.config\...`，无则自动建模板）；覆盖途径：`BILI_CONFIG_FILE`、`XDG_CONFIG_HOME`、`--config`
- **纯覆盖层**：只写要改的键，未写的键走内置默认（默认值是内核设计，别全量抄进文件）
- 合并顺序：全局 `compress` → `providers.<id>.compress` → 模型级 `compress`，按字段就近取
- 语法错误 = **拒绝加载整份配置**（响亮报错，Web UI 返回 400）——写完必须 `GET /__bili/config` 看解析结果
- 热加载：nudge 阈值类改动 `GET /__bili/config` 确认解析后**即时生效于后续决策**；**有效窗口是每会话启动时发布的，headroom 类改动对新会话/重启 bili 后生效，不追溯当前会话**
- 写配置前先算派生线（以有效窗口 EW 为分母）：增长floor≈15%·EW（下限 22500）；批次≈34%·EW；压力入口=maxContextLimit×窗口；紧急=emergencyThresholdPercent（≥maxContextLimit 硬校验）

常用键速查（默认值 → 含义）：
```
compress.maxContextLimit            75%   → 压力带入口（字符串百分比可用）
compress.emergencyThresholdPercent  95%   → 紧急线（须 ≥ maxContextLimit）
compress.minPressureBenefitTokens   max(5000, 1%) → 进带后最小可回收量（#3211 主杠杆）
compress.nudgeGrowthTokens          50000(恒定)  → 增长门步长
compress.tierNudgeTokens            T1=50000, T2/T3=×1.5 → 分层触发
compress.nudgeModelDecided          off   → 模型自决压缩时机（#2228）
compress.outputHeadroomMaxPct       0.25  → 输出预留占窗口上限
compress.modelContextLimit          原生窗口 → 按模型改窗口口径（病#4）
compress.modelContextLimitSource    declared → 来源声明：dsh 上报等"声明值"会压制你的覆盖，须写 declared 才让覆盖生效
compress.tiers                      true  → T1/T2/T3 分层蒸馏摊平折叠成本
compress.ccr                        on    → 超大工具输出无损外置旁存（acp_retrieve 找回）——默认很好，别关
compress.reasoning.dropTokens       2048  → 超预算的历史思考丢弃
providers.<id>.calibration          未设  → 虚报线逃生阀（max:1 截半 / disabled 全关）
```

---

## 5. 给 Agent 的标准工作流（照着跑）

1. 采 §2 清单 → 产出本机参数表（原生窗口/预留/有效窗口/四条线实际值/模型名/宿主 max_tokens）
2. 按 §3 决策树定病根（多病共存时先治病 #1——它移动所有其它线）
3. **每轮只动 1–2 个键**，写进配置文件（纯覆盖层，保留模板其它部分原样）
4. 验证三连：
   - `GET /__bili/config` 解析正确（raw 与解析视图都看你写的键）
   - 下一批 `nudge idle` 行：分母/阈值是否变成期望值，`reason` 文案变化符合预期
   - 跑一段该用户的典型密集任务后 `acp_cache`：FOLD ECONOMICS 未回本笔数 → 0 或个位数；GRAND LEDGER hit% 向 95% 收敛
5. 效果不达标 → 回 §3 换病根假设，**不要**叠加更多键
6. 记录：最终配置 + 验证数据 + 推导过程写进用户自己的 README（这就是"渔"的存档）

---

## 6. 示例推演（示范数字，非特定机器实录）

环境特征：自建 vLLM 线路，原生窗口 196608，宿主发 max_tokens 131072，中文密集会话，flash 模型。

- 实测 `[headroom]`：196608 → 147456（预留 49152，25% cap 咬满）→ 病 #1
- 日志曲线：进 75% 带（110592）后 11 次微折，每跌 3.7–11.7K，usage 在 58–105K 间锯齿 → 病 #2 + #3
- k̂=1.65 稳定、请求正常 → 计数为真，calibration 不动
- 处方推导：minPressureBenefit 按"~200K 级窗口设 1/3 有效窗口"取 50000；开 nudgeModelDecided；maxContextLimit 提到 85%；预留经宿主+bili 侧压到 40960（pct=5/24）
- 结果线：EW 147456→155648，压力入口 110592→132301，微折预期 11 次 → 2–3 次批量折
- 关键教训（写在这里是因为它最容易坑别人）：headroom 行只在会话启动发布，**当前会话不追溯新 headroom**；minPressureBenefit 初版建议 15–20K，对照官方处方原文后修正为 50000——**引用处方要按窗口量级换算，不要抄数字**

---

## 7. 陷阱清单

1. **有效窗口 ≠ 原生窗口**：一切线按 EW 算；看到 usage% 先问分母是谁
2. **percent 语义**：maxContextLimit 是原生窗口的百分比（75%→85% 的例子），calibration 的 usage 是 EW 的百分比——文档写清口径，别混
3. **headroom 改动不追溯当前会话**（见 §6 教训）
4. **sidecar 直连车道**（标题生成等）无 KV 缓存：日志特征是"输入很大、cache 0% 且成功、反复出现"，属正常，不要为它调参
5. **别把 CCR 关了**：大输出无损旁存是默认红利，实测它只占 1% 输入
6. **压不动就检查宿主**：宿主把压缩工具摘了/模型不支持工具→ bili 自动降级为仅提醒，你的 bili 参数全白调（看 acp_status 的接管横幅/降级警告）
7. **max_tokens 钳制是原生窗口安全阀**：调小 bili 预留不会导致输入+输出溢出，钳制独立兜底
8. 紧急线必须 ≥ maxContextLimit，写反了整份配置被拒
9. 配置语法错误 = 全份拒载：写完必验 `GET /__bili/config`

---

## 8. 粘贴即用指令（用户把本文整份交给自己的 Agent 后，追加这一段）

> 请阅读上文《billion-context 优化指南》，然后对我这台机器执行 §5 的工作流：
> 1. 先采集 §2 全部事实（宿主模型配置、`[headroom]`/`nudge` 日志行、`/__bili/*` 端点、`acp_cache` 报告），产出一张本机参数表；
> 2. 按 §3 决策树判定我属于哪些病根（允许多病共存，注明证据行）；
> 3. 每轮只动 1–2 个键写进配置文件（纯覆盖层，保留原模板），按 §4 先算好派生线；
> 4. 按 §5 的验证三连做，直到 acp_cache 报告 FOLD ECONOMICS 基本回本、GRAND LEDGER 命中率收敛；
> 5. 把最终配置、推导过程、验证数据写成 `TUNING-RECORD.md` 存在仓库里，作为你自己的可复用结论。
> 约束：不得删除宿主配置；配置语法必须自验；所有结论必须引用日志原文行，不得凭记忆编造。

---

*版本基线：billion-context v0.1.209 实测 + 官方 README/CONFIGURATION 中文文档。bili 迭代快，动手前先看 `GET /__bili/status` 与对应版本文档。*
