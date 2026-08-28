---
name: okki-aireach-discovered-prospects-query
description: 用自然语言检索 AiReach 已挖掘的潜客，支持按时间、国家、分层、建档/触达状态、主营品类等条件定向查询。
---
## 0. 初始化

本技能依赖 OKKI AiReach 能力，执行前必须先确认 `aireach-cli` 可用且当前环境已授权。若缺少 `aireach-cli`，请先通过 npm 安装：`npm install -g @okki-aireach/aireach-cli`（或由宿主按插件约定注入），不要尝试绕过 CLI 直接调用内部接口。

执行前运行 `aireach-cli auth status` 检查登录状态，确认本地凭证与目标 gateway 环境匹配。若未登录或凭证失效，先按 `okki-aireach-auth-init` 的流程引导用户完成授权（授权 URL 需包含 `client_id=accio-work`），授权成功后再继续本技能任务。

不要要求用户粘贴或展示 PAT/token，也不要把授权登录路径、内部地址写进回复；认证是否可用以目标 gateway 的真实请求验证结果为准，而不是只看本地是否有 token。
# AiReach Discovered Prospects Query

## 能力边界
- 这是对 AiReach 已挖掘潜客池的只读查询，不创建新潜客，不搜索开放网络，不写回状态，不创建营销任务。
- 对外语义统一视为 AiReach 产品能力，不向用户暴露底层实现名。
- 数据语义是当前租户已有的 AiReach potential list / `AiSdrTaskDetail` 记录；后端会自动解析当前租户的 AiReach 自动挖掘任务，不向用户索要或展示 `task_id`。
- 概览统计和分布必须来自工具返回的完整匹配集，不能用最多 100 条 `records` 样本估算。
- 正常查询只返回数据、统计和 AiReach 查看链接；导出是独立动作，必须用户明确确认后才调用。
- 导出是独立确认动作：用户明确确认后调用一次 `potential_list_export`，只有工具返回真实 `download_url` 后才能给用户下载链接。
- 用户说全部价值范围、所有价值范围、不限价值、高中低未知都包含时，查询必须传 `quality=["all"]`；未指定质量时才走默认中高价值口径。

## 适用场景
- 用户问：`目前总共挖了多少潜客？`、`过去一周潜客情况怎么样？`
- 用户问：`列出过去一周新增的高价值潜客`、`查一下美国地区有海关数据的潜客`
- 用户问：`帮我找园艺类、OEM/ODM 类型的公司`、`把待建档的中价值潜客列出来`
- 用户在上一轮查询后说：`导出这些潜客`、`把这批结果导出`

## 不适用场景
- 查询贸易单据、HS 编码、进口商/出口商交易记录：使用 `aireach-import-export-query`。
- 新增潜客、开放网络找客户、语义模糊行业发现、图谱分支分析、按洲筛选、按行业领域大类筛选。
- 定时推送、创建营销任务、建档/归档/状态写回。
- 显式获取卖家画像、调用画像工具或把画像获取当作本 skill 的前置步骤。如果缺少业务上下文，只问一个具体澄清问题。

## 参数映射
- 时间 / 新增 / 最近 N 天：使用 `recent_days`，或 `start_date` + `end_date`。语义对齐 AiReach 潜客列表，按当前阶段使用对应阶段时间字段；不指定阶段时等同列表默认时间口径，不再固定理解为 `visible_time`。
- 国家 / 地区：使用 `country_codes`，只传 ISO 3166-1 alpha-2，例如 美国=US、巴西=BR。
- 高价值 / 中价值 / 低价值 / 未知 / 全部：映射到 `quality` 的 `high` / `medium` / `low` / `unknown` / `all`。
- 未指定质量时不要传 `quality`；后端默认按中高价值统计，返回 `quality_scope=default_medium_or_higher`。
- 未建档：`archive_status=["pending"]`。
- 建档成功：`archive_status=["archived"]`；合并建档：`archive_status=["combined_archived"]`。
- 已建档如果用户泛指已经完成建档，先按 AiReach 页面“建档成功”口径查 `archive_status=["archived"]`；只有用户明确要求包含合并建档时才同时传 `["archived","combined_archived"]`。
- 待建档如果用户指需要处理的建档失败项，传 `archive_status=["failed_duplicate","failed_switch_off","failed_customer_limit","failed_other"]`；不要把未建档 pending 混成待建档。
- 主营产品：传 `main_products` 精确关键词数组。不要扩展成语义行业。
- 公司类型：传 `company_types` 精确关键词数组，例如 `OEM/ODM`。
- 公司名 / 潜客名 / 产品数组中的安全关键词：传 `keyword`。
- 有海关数据 / 进出口贸易数据来源：传 `customs_source_only=true`，后端按 `dig_source=2` 过滤，并在回答中说“海关/进出口贸易数据来源潜客”。
- 列表样本上限：工具最多返回 100 条；对用户最多展示 10 行。

## 统计口径优先级
- 页面统计字段优先级高于原始枚举分布：用户问“已打开、已回复、已触达、已建档、待建档、未建档”等页面口径时，优先读取 `page_metrics`，不要从 `distributions` 自行相加或折叠。
- 总匹配数：使用当前查询的 `summary.total_count`；不要用 `records` 样本数估算。
- 海关/进出口贸易数据来源数：使用 `summary.customs_source_count`；如果用户要查看或导出这个子集，后续 list/export 必须继续传 `customs_source_only=true`。
- 已触达商家量：使用 `page_metrics.stage_cumulative` 中 `key="marketing"` 的 `count`，表述为“进入触达阶段的潜客/商家数”；不要把它当成建档数量。兼容旧结果时才使用 `summary.entered_marketing_stage_count`。
- 已打开：使用 `page_metrics.stage_cumulative` 中 `key="opened"` 的 `count`，该值包含当前处于已打开和已回复的潜客。
- 已回复：使用 `page_metrics.stage_cumulative` 中 `key="replied"` 的 `count`。
- 建档成功 / 已建档：使用 `page_metrics.archive_status_filters` 中 `key="archived"` 的 `count`。
- 已合并建档：使用 `page_metrics.archive_status_filters` 中 `key="combined_archived"` 的 `count`。
- 未建档：使用 `page_metrics.archive_status_filters` 中 `key="pending"` 的 `count`。
- 待建档：使用 `page_metrics.archive_status_filters` 中 `key="pending_archive"` 的 `count`；不要把未建档 pending 混成待建档。
- 缺少 `page_metrics` 时按页面口径分次调用并读取每次 `summary.total_count`：阶段累计传 `stage` + `stage_match="at_or_after"`；页面建档状态传对应 `archive_status` 组合。
- 用户明确问“当前阶段构成、原始阶段分布、建档状态构成”时，才使用 `distributions.stage` 或 `distributions.archive_status`。`distributions.stage` 是当前阶段枚举分布，已打开项不包含已回复；`distributions.archive_status` 逐项展示原始枚举状态，不要自行折叠成“成功建档 / 待建档”两类。

## 导出条件继承
- 导出必须继承上一轮查询的所有筛选条件，包括时间、国家、质量、阶段、建档状态、主营产品、公司类型、关键词和 `customs_source_only`。
- 如果上一轮回答的是海关/进出口贸易数据来源子集，导出必须继续传 `customs_source_only=true`，不要只沿用上一轮的宽条件。
- 如果上一轮回答的是某个页面口径统计，导出该口径时必须传对应筛选条件；例如导出已打开潜客传 `stage="opened", stage_match="at_or_after"`，导出待建档潜客传建档失败状态组合。
- 导出只导出完整匹配集，不是当前展示的 10 条或工具返回的 100 条样本。

## 执行流程
1. 判断请求是概览、列表还是导出确认。
2. 对概览或列表，调用 `potential_list_query`，不要传 `task_id`。如果用户问多个阶段/建档状态的页面统计，优先使用 `page_metrics.stage_cumulative` 和 `page_metrics.archive_status_filters`；只有“阶段构成/状态分布”类问题才读取 `distributions.stage` 或 `distributions.archive_status`。
3. 如果用户请求了不支持维度，先用支持维度查询或要求澄清，并明确说 unsupported 维度。
4. 如果是宽泛品类词（例如“园艺类”），只能作为安全 `keyword` 或明确产品词使用；不要承诺语义检索。
5. 输出时使用工具返回的 `summary`、`page_metrics`、`distributions`、`records`、`actions.view_link`。
6. 如果用户明确确认导出，调用 `potential_list_export`：
   - `confirmed=true`
   - 传入与上一轮查询一致的筛选条件
   - 遵守“导出条件继承”规则，确保导出总数对齐上一轮回答的页面口径或子集口径
7. 如果工具返回 `status=done` 且存在真实 `download_url`，给出下载链接；如果返回 `empty`、不支持或没有 URL，说明没有可导出的匹配结果或当前条件暂不支持导出，不要编造下载链接。


### 列表回答

列表回答的目标：在概览洞察之上，落到“看得见的具体潜客”。先说清查询口径，再给一张干净可读的明细表，然后用画像和优先级把表里的潜客讲透。信息要点必备，呈现形式可灵活。
**若商家给了固定格式或模版，一律以商家格式为准，覆盖以下默认结构。**

列表回答使用这些块：

- `查询概况结果`：一两句话说清“在查什么、查到多少”。
  - 核心数字：匹配总量（完整匹配集，不是展示的 10 行）、本次时间/国家/产品等筛选口径。
  - 质量与数据口径：高/中价值数量、海关/进出口贸易数据来源数、建档状态；只用工具返回值。

- `表格明细`：最多展示 10 行，按价值/匹配度等有意义的顺序排，不要随机截断。
  - **列必须严格固定为以下 7 列，顺序、数量、列名都不得改动，禁止删减或合并任何一列：公司名称｜网址｜国家｜主营产品｜公司类型｜联系人｜联系方式。**
  - **即使某列大面积为空，也必须保留该列表头**
  - 主营产品最多显示前 3 个，更多用“…”省略。
  - 只展示工具真实返回的字段；某行缺网址/联系人/联系方式时，对应单元格留空”，不得编造，也不得因此删列。
  - 表外用一句话点明“展示前 10 / 共 N 家”，并说明排序口径。

- `画像分析`：不是罗列维度计数，而是提炼这批潜客的共性与结构（结论基于工具返回的完整分布，不是展示的 10 行）。尽量覆盖：
  - 主结构：最集中的国家/品类特征，是集中还是分散。
  - 维度交叉：国家 × 主营产品 × 公司类型 × 质量的有意义组合。
  - 共性画像：这批公司的典型画像（例如“设计+品牌+零售一体的轻资产公司，依赖外部 OEM/ODM”），以及异常或长尾。
  - 业务含义：对触达/选品/合作方式意味着什么。
  - 不得用 10 行样本估算分布；没有的维度不硬编。

- `优先关注建议`：2-3 条具体可落地建议，避免空泛套话。每条尽量含：
  - 精准人群：用真实数字 + 维度交叉圈定细分人群。
  - 代表潜客：点名表里 2-3 家真实公司，**必须是高价值潜客，且主营产品、公司类型与商家高度匹配**，并给一句“为什么是它”。
  - 理由与时效：质量/海关背景/近期新增待建档等 + 时效提示。
  - 下一步抓手：落到先触达 / 先建档 / 先按某维度细分查询。

- `下一步动作引导确认`：引导用户确认下一步，支持的动作：
  - 导出完整明细（提示回复“导出这些潜客”，导出需确认且只导完整匹配集）。
  - 设置定时任务推送。
  - 按维度筛选细分查询（国家、地区、主营产品、公司类型、质量、建档状态等）。
  - 调整 / 放宽筛选条件（结果过多或过少时）。
  - 趋势对比、图表分析等增强视图（只在数据支持时提供，可按不同时间口径分别查询后对比，不承诺全量趋势）。

## 约束
- 不得编造 company、lead、联系人、download_url、统计总量或分布。
- 不得向用户展示 `task_id`、`detail_id`、`lead_id`、`company_id`、原始 `query` JSON、工具名或工具参数；这些只用于工具调用和证据检查。
- 不得从 `records` 样本估算质量/国家/产品分布。
- 不得把用户不支持的洲、图谱分支、行业领域、语义检索条件硬塞到 `keyword` 里并声称精准。
- 没有结果时，给 2-3 个具体放宽建议：扩大时间范围、去掉国家、放宽质量、去掉产品/公司类型关键词。
- 结果太大时，展示前 10 条并建议加时间、国家、质量、建档状态、产品或公司类型条件。
- 查看链接只作为简短的 AiReach 链接或动作给出，不要在正文里解释或展开其中的底层参数。
- 导出只导出完整匹配集，不是当前展示的 10 条或工具返回的 100 条样本。
- 导出没有返回真实下载链接时，不要声称已经生成文件；按工具返回说明没有结果、暂不支持或导出失败。

