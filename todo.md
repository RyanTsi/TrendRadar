项目概览

- 数据流与职责分层清晰：抓取→存储→统计→AI分析→报告/通知
- 主流程入口负责模式选择、AI分析与推送整合，便于扩展 main .py
- 配置驱动强：平台源、显示区域、推送窗口、AI模型统一在一个 YAML 文件中管理 config.yaml
- AI 已用 LiteLLM 统一接入，提示词模板可自定义，输出结构可控 analyzer.py client.py
- 存储采用 SQLite 结构化表，含热榜与 RSS 两套模式与推送/分析状态记录 schema.sql rss_schema.sql
- 通知渠道完整（飞书/钉钉/企业微信/Telegram/Email/Slack/ntfy/Bark/通用Webhook）且支持分批与翻译 notification
- MCP 服务已内置搜索、趋势分析、对比等工具，适合做交互式投研助手 server.py
改造目标

- 定位：面向金融从业者（投研/交易/合规）提供“事件→市场影响→策略要点”的结构化洞察
- 能力：将新闻与RSS事件与实时/近实时市场数据关联，给出风险因素、影响方向、品种映射、时间维度与可操作性提示
- 交付：日报/盘前/盘后报告与盘中增量提醒，支持企业协同渠道与交互式MCP分析
修改计划

- 阶段 0｜建立基线与环境
  
  - 明确使用模式与窗口：日内用 incremental（有新增才推），盘前/盘后用 daily/current config.yaml:report
  - 启用 AI 分析与统一显示顺序，把 AI 放到顶部或末尾按使用偏好 config.yaml:display
  - 原因：保持推送与分析节奏稳定，让用户首先体验到金融化结构的报告框架
- 阶段 1｜金融数据源与关键词定制
  
  - 平台源聚焦财经与证券类（已包含财联社/华尔街见闻/格隆汇/雪球），可按需启用/禁用 config.yaml:platforms
  - 频率词新增板块/品种/宏观事件（如板块名、指数、票息、央行、经济数据、公司事件） frequency_words.txt
  - 使用组别名与正则实现同义词合并与歧义规避（如“美联储/联储/Fed”一组） frequency_words.txt
  - 原因：保证匹配到的内容与金融语境对齐，减少噪音并提升主题的聚合质量
- 阶段 2｜接入关键市场数据
  
  - 设计最小可用的行情抓取器：股指（沪深/恒生/标普/纳指）、外汇（主要对）、大宗（黄金/白银/原油）、利率与国债要点
  - 以 requests 或轻量库对兼容API拉取（如开放的Yahoo Finance或自有数据源），先用只读、限速与缓存，落地到新增的 timeseries 表
  - 扩展 SQLite Schema：新增 market_timeseries(系列标识、时间、价格/收益率/点位、来源)，与事件按时间窗口关联 schema.sql
  - 原因：新闻只给“方向”，金融需要“方向+价格行为”才能形成有效的投研结论或交易提示
- 阶段 3｜AI 提示词与输出结构金融化
  
  - 重写分析提示词模板，输出结构统一为 JSON，包含：市场影响评估、相关资产映射（股票/指数/FX/利率/商品）、时间维度（盘前/盘中/盘后/一周）、风险因子与不确定性、策略要点与观察指标 ai_analysis_prompt.txt
  - 若需要更强结构化，在 AIAnalysisResult 中补充 finance 字段：market_impact、asset_mapping、risk_drivers、timeframe、action_items analyzer.py
  - 在渲染中把金融区块与原有“核心趋势/弱信号/RSS洞察/策略”对齐展示 context.py:render_html
  - 原因：让AI输出直接可用于投研沟通与盘面观察，避免泛泛而谈
- 阶段 4｜报告与通知的金融视图
  
  - 报告模式增加“盘前/盘后”预设，结合 push_window 控制时段 config.yaml:notification.push_window
  - HTML 报告与渠道推送内容中插入“事件-市场联动摘要”和“重点观察列表”（如关键资产+触发阈值） main .py:generate_html
  - 通知分批策略保留，企业渠道优先（飞书/企业微信/Slack），必要时开启翻译（中英）供跨区交流 notification context.py:split_content
  - 原因：不同交易时段的信息关注点不同，结构化摘要能显著提升可操作性与团队协作效率
- 阶段 5｜MCP 工具扩展为投研助手
  
  - 在 MCP 工具新增“事件-资产关联检索”“时段对比（盘前vs盘后）”“共现网络（主题与公司/资产）”“事件后价格响应窗口统计” server.py
  - 与现有 analyze_topic_trend / compare_periods 保持接口风格一致，返回 {success, summary, data, error} 结构
  - 原因：让投研人员通过自然语言发起分析，快速得到结构化结果并可追问深挖
- 阶段 6｜验证、合规与治理
  
  - 验证链路：示范配置+样本日的完整跑通（热榜+RSS+行情→SQLite→分析→HTML/通知），检查数据对齐与时间窗
  - 加入速率与失败重试、错误提示统一规范；敏感键与Webhook仅走环境变量，不落盘 context.py:get_storage_manager config.yaml:notification
  - 文案加入“非投资建议”与风险披露；对外推送渠道启用严格过滤与白名单
  - 原因：面向金融场景必须保证稳定性、安全性与合规表达
落地建议

- 先不改代码，直接用现有配置把财经平台与频率词跑起来，打开 AI 分析，看实际输出是否接近金融语境
- 第二步最小接入 3—5 个核心品种的行情（指数/黄金/美元指数/10Y），先做时间对齐与粗关联
- 第三步再迭代 AI 提示词与输出结构，并把金融区块嵌入 HTML/通知
- 最后扩展 MCP，满足交互式投研的“问→查→比→看响应”的闭环