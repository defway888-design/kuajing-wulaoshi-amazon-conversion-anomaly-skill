# 跨境吴老师转化率异动 MCP 路由

## 目录

- 1. 数据源边界
- 2. 固定调用顺序
- 3. 领星：当前自有商品身份
- 4. 卖家精灵：竞品生成
- 5. 卖家精灵：预测近30天销量
- 6. 西柚：当前标题、标题修改与Prime价格
- 7. 卖家精灵：Keepa趋势
- 8. 卖家精灵：Coupon
- 9. 卖家精灵：评论
- 10. 缓存复用与错误处理

## 1. 数据源边界

同一种数据只能使用一个来源：

| 数据 | 固定来源 |
|---|---|
| 当前店铺、站点、自有 Seller ID | 领星 MCP |
| 当前自有父 ASIN和子 ASIN | 领星 MCP |
| 上游自有转化率、Session、订单 | 领星 MCP或上游已确认结果 |
| 叶子类目、BSR候选、关键词自然 Top 10 | 卖家精灵 MCP |
| 目标商品和候选商品当前标题 | 西柚 `get_asin_info` |
| 自身及最终竞品标题修改历史 | 西柚 `get_asin_info_change_trends` |
| 自身及最终竞品 Prime价格日趋势 | 西柚 `get_asin_info_trends` |
| 当前主图、五点、竞品父子关系 | 卖家精灵 MCP |
| 目标父 ASIN五点为空时的子 ASIN五点补抓 | 卖家精灵 MCP |
| 竞品预测销量 | 卖家精灵 `asin_prediction` |
| 自身及竞品基础价格、Deals、Buy Box历史、评分、页面评价数量 | 卖家精灵 `keepa_info` |
| 自身及竞品 Coupon | 卖家精灵 `asin_coupon_trend` |
| 自身及竞品新增评论 | 卖家精灵 `review` |

禁止用另一 MCP对同一指标补数、校验或替代。西柚标题缺失时不得使用卖家精灵标题补齐；西柚 Prime字段缺失时不得使用卖家精灵或 Keepa Prime数据补齐。卖家精灵返回的标题不参与当前标题、标题修改或标题相似性判断。西柚返回的其他价格字段不参与基础价格和 Deals正式判断。

运行时通过当前工具目录或工具搜索发现西柚和卖家精灵 MCP。确认其能够提供本文件规定的工具后，整次任务分别固定一个西柚命名空间和一个卖家精灵命名空间，不得因为字段缺失切换命名空间补数。

站点代码使用 `US`、`JP`、`UK`、`DE`、`FR`、`IT`、`ES`、`CA`、`IN`、`MX`、`BR`、`AU`、`AE`。时间戳使用毫秒。每次调用只请求必要字段。

## 2. 固定调用顺序

```text
1. 领星：目标店铺、站点、自有 Seller ID
2. 领星：当前父 ASIN和当前子 ASIN
3. 西柚取得目标当前标题；卖家精灵取得目标当前主图、五点、最细叶子类目和小类 BSR；高优先级目标父 ASIN五点为空时按 4.1.1 补抓子 ASIN五点
4. 卖家精灵：叶子类目 BSR 1—100候选
5. 西柚批量取得候选当前标题；卖家精灵取得候选主图、五点、父子关系和预测近30天销量
6. 若不足五个：放弃 BSR池，用主要自然流量词 Top 10重建候选池
7. 西柚：自身和最终竞品标题修改历史
8. 西柚：自身和最终竞品 Prime价格日趋势
9. 卖家精灵：自身和最终竞品 Keepa聚合趋势
10. 卖家精灵：自身和最终竞品 Coupon
11. 卖家精灵：自身负面评论和竞品全部评论
12. 业务层：时间切分、证据分组、方向判断与输出
```

竞品集合只生成一次并在本次任务中复用。标题修改和 Prime价格均只查询异常子 ASIN及最终竞品代表子 ASIN。

## 3. 领星：当前自有商品身份

### 3.1 店铺、站点和 Seller ID

工具：`LingXing-MCP.get_my_sids`

必取逻辑字段：

```text
sid
country
店铺名
seller_id
mid / marketplace_id / marketplace
```

规则：

- 用异常记录或用户指定的站点和店铺匹配。
- 同站点多店铺且无法确定目标店铺时，请用户选择。
- Seller ID每次动态取得，不使用历史运行值。

### 3.2 当前父 ASIN和子 ASIN

工具：`LingXing-MCP.erp_listing`

输入为子 ASIN时先精准查询：

```json
{
  "sids": "<sid>",
  "mids": "<mid>",
  "search_field": "asin1",
  "search_value": ["<ASIN>"],
  "offset": 0,
  "length": 50,
  "pvi_ids": "",
  "exact_search": "1"
}
```

未命中时按父 ASIN查询：

```json
{
  "sids": "<sid>",
  "mids": "<mid>",
  "search_field": "parent_asin",
  "search_value": ["<ASIN>"],
  "offset": 0,
  "length": 200,
  "pvi_ids": "",
  "exact_search": "1"
}
```

确认父 ASIN后，以 `search_field=parent_asin`分页取得全部当前子体。

必取逻辑字段：

```text
parent_asin
asin1
seller_sku
sid
marketplace
```

只接受实际返回行中同时匹配目标 `sid`和目标 ASIN/父 ASIN的记录，不能只看 `total`。不调用历史销售表现接口推断父子关系。

## 4. 卖家精灵：竞品生成

### 4.1 目标商品当前标题、主图、五点与叶子类目

固定使用西柚 `get_asin_info`读取目标商品当前标题，固定使用卖家精灵 `keepa_info`读取其他当前画像。

西柚最小逻辑字段：

```text
asin
title
```

卖家精灵最小逻辑字段：

```text
asin
parentAsin
features
主图 Amazon图片 ID或主图 URL
最细类目节点 ID及完整路径
小类 BSR
variationAsins
可靠上架日期
```

标题只接受西柚当前字段；主图、五点、类目和竞品父子关系只接受卖家精灵当前字段。主图、标题或五点任一缺失时，严格相似性判断数据不足；但高优先级目标父 ASIN五点为空时，必须先执行 4.1.1 子 ASIN补抓，补抓失败后记录证据缺口，不得仅因五点缺失阻断诊断报告生成或邮件发送。

### 4.1.1 高优先级父 ASIN五点为空时的补抓

触发条件必须同时满足：

- 本次任务是高优先级 ASIN深度诊断。
- 目标父 ASIN的卖家精灵五点字段为空、缺失或不可用。
- 本次诊断需要使用 Listing内容或严格相似竞品判断。

未同时满足时，不执行本节补抓，不对所有商品批量执行。

补抓顺序：

1. 先用卖家精灵 MCP 的已确认 ASIN详情或父子体关系能力确认目标父 ASIN下的子 ASIN列表。优先复用 4.1 已返回的 `variationAsins`；如当前卖家精灵工具契约确认 `asin_detail`、`competitor_lookup` 或同类父子体字段稳定，也可仅用于取得子 ASIN列表。
2. 如果能取得子 ASIN列表，按当前可售、价格有效、评论数较多、Buy Box状态正常、销量或流量更高的顺序选择候选子 ASIN。排序只能使用卖家精灵 MCP 已确认稳定字段或本次流程已取得的数据；某个排序字段不可用时跳过该排序维度，不得调用未确认工具补数。
3. 默认尝试前 3 个候选子 ASIN；如果前三个均无可用五点且还有更高质量候选，最多扩展到 5 个。不得超过 5 个。
4. 对每个候选子 ASIN读取 Listing五点字段。优先使用卖家精灵 `keepa_info` 或已确认 ASIN详情工具中稳定的 `features` 字段；如果当前工具契约没有稳定五点字段，只能记录证据缺口，不能把未确认字段当作正式证据。
5. 任一候选子 ASIN成功返回非空五点后，立即停止继续补抓，并把该子 ASIN五点作为本次父商品诊断的 Listing辅助证据。
6. 补抓成功时必须记录：五点来源类型为“子 ASIN补抓”，五点来源 ASIN为命中的子 ASIN，父 ASIN为目标父 ASIN，字段映射记录为“五点 <- 子ASIN {child_asin}”。
7. 3-5 个候选子 ASIN均无法返回可用五点时，记录为证据缺口，不得编造五点内容。

禁止事项：

- 五点属于辅助证据，不是核心诊断字段。
- 不得因为五点缺失阻断诊断报告生成。
- 不得因为五点缺失阻断邮件发送或可发送状态。
- 不得编造五点内容。
- 不得使用标题、描述、评论内容、关键词或图片内容代替五点。
- 不得用西柚、领星或互联网内容补充卖家精灵五点。
- 补抓只针对高优先级目标父 ASIN执行，不对候选池、全部竞品或所有商品执行。

补抓成功时输出内部状态：

```text
核心诊断状态：完整完成
可发送状态：允许发送
证据缺口类型：空
证据缺口说明：空
字段映射记录：五点 <- 子ASIN {child_asin}
```

补抓失败时输出内部状态：

```text
核心诊断状态：可用但有证据缺口
可发送状态：允许发送
证据缺口类型：五点缺失 / 严格相似竞品审核未闭环
证据缺口说明：父 ASIN五点为空，已尝试最多 3-5 个候选子 ASIN，仍未获取到可用五点
字段映射记录：五点补抓失败：父ASIN {parent_asin}，候选子ASIN {child_asin_list}
```

需要校验类目路径时调用 `product_node`：

```json
{
  "request": {
    "marketplace": "<站点>",
    "nodeIdPath": "<目标商品类目路径>",
    "returnFields": "<节点ID、节点名称、完整路径>"
  }
}
```

选择路径中最深、且与目标商品小类 BSR对应的叶子节点。

### 4.2 叶子类目 BSR 1—100候选

工具：`product_research`

核心入参：

```json
{
  "request": {
    "marketplace": "<站点>",
    "nodeIdPath": "<最细叶子类目路径>",
    "nodeIdPathEqual": true,
    "minSubBsrRank": 1,
    "maxSubBsrRank": 100,
    "variation": "N",
    "page": 1,
    "size": 100,
    "returnFields": "<ASIN、父ASIN、小类BSR、上架日期>"
  }
}
```

业务层只保留小类 BSR 1—100并按小类 BSR升序建立原始优先级。不得换成大类 BSR或关联流量候选。

### 4.3 候选补充、父体分组与相似性

对候选 ASIN用西柚 `get_asin_info`批量取得当前标题；用同一卖家精灵详情口径补齐：

```text
parentAsin
variationAsins
features
主图 Amazon图片 ID或主图 URL
可靠上架日期
```

按 `references/business-rules.md`中的“按父商品分组并排除目标商品”“选择竞品代表子体”和“严格判断商品相似性”执行：

- 排除目标当前父商品的全部子体和明确异常 ASIN。
- 其他自有商品不排除。
- 按竞品父 ASIN分组。
- 当前主图、标题、五点任一缺失即不能默认相似。
- 卖家精灵即使返回标题也必须忽略，不能校验或替代西柚标题。

### 4.4 关键词候选池

只有 BSR池完成全部严格筛选后不足五个才执行。执行时丢弃 BSR池结果。

工具：`traffic_keyword`

```json
{
  "request": {
    "asin": "<异常子ASIN>",
    "marketplace": "<站点>",
    "badges": ["naturalSearching"],
    "trafficKeywordTypes": ["primary"],
    "includeTop10AsinData": true,
    "order": {
      "field": "trafficPercentage",
      "desc": true
    },
    "page": 1,
    "size": 50,
    "returnFields": "<关键词、trafficPercentage、gkDatas及自然排名>"
  }
}
```

从返回词中选择最多三个符合产品类型、目标用户、结构和用途的通用核心词，排除品牌词和过宽词。按 `trafficPercentage`降序依次处理。

只使用每个词的 `gkDatas[]`自然搜索 Top 10：

- `gkDatas[]`顺序是自然排名顺序。
- 不把该顺序解释为销量、点击或评论排名。
- 不使用 `competitor_lookup.keyword`替代。
- 多个关键词结果取并集后按父 ASIN去重。

## 5. 卖家精灵：预测近30天销量

### 5.1 扩展竞品父商品子体

对每个候选父商品取得完整 `variationAsins`。只使用卖家精灵父子关系，不用领星或其他来源补竞品变体。

### 5.2 ASIN销量预测

工具：`asin_prediction`

对候选父商品的每个子 ASIN分别调用：

```json
{
  "asin": "<子ASIN>",
  "marketplace": "<站点>",
  "returnFields": "asin,dailyItemList"
}
```

必取：

```text
dailyItemList[].date
dailyItemList[].sales
```

规则：

- 目标窗口为站点当地运行日 `D`之前30个完整日 `[D-30, D)`。
- 同日重复记录保留最后一个有效值。
- `sales >= 0`有效；`-1`、`null`、非数字或缺日无效。
- 上架日期之前按0，仅限卖家精灵已返回可靠上架日期。
- 上架后任一天无效，或无法取得可靠上架日期且窗口有缺失，整个子 ASIN销量数据不足。
- 近30天和近7天都按日求和。

父商品代表子体和最终排序按 `references/business-rules.md`中的“选择竞品代表子体”“固定竞品销量来源”“统计最近30个完整日”“判断每日销量是否有效”和“处理销量并列”执行。输出字段名固定为“卖家精灵近30天预测销量”。

## 6. 西柚：当前标题、标题修改与Prime价格

每次西柚调用必须填写：

- `user_task`：保留本轮用户最原始任务的对象、范围和约束，只做必要脱敏；同一任务内所有西柚调用保持完全一致。
- `intent_summary`：只写本次调用的步骤目的，不扩写为完整分析计划。

### 6.1 当前标题

工具：`get_asin_info`

用于目标异常子 ASIN、BSR候选和关键词候选的当前标题。多个 ASIN优先合并到同一次列表调用；超过工具限制时按原候选顺序分批，禁止按另一业务指标重排。

```json
{
  "asins": ["<ASIN1>", "<ASIN2>"],
  "country": "<站点>",
  "user_task": "<本轮固定脱敏原始任务>",
  "intent_summary": "取得候选当前标题用于严格相似性判断。"
}
```

必取：

```text
entities[].asin
entities[].title
```

规则：

- `title`去除首尾空格后非空才有效。
- 返回列表缺少目标 ASIN、`title`缺失或为空时，该 ASIN标题数据不足。
- 卖家精灵标题不能补数或交叉验证。

### 6.2 标题修改历史

工具：`get_asin_info_change_trends`

每个异常子 ASIN和每个最终竞品代表子 ASIN各调用一次：

```json
{
  "asin": "<ASIN>",
  "country": "<站点>",
  "start_date": "<基准期开始当地日期 YYYY-MM-DD>",
  "end_date": "<异常期结束前最后一个当地日期 YYYY-MM-DD>",
  "user_task": "<本轮固定脱敏原始任务>",
  "intent_summary": "取得标题修改前后记录用于期间核查。"
}
```

必取：

```text
dateRangeNotice
trends[].date
trends[].previous.title
trends[].current.title
```

规则：

- 西柚日期参数按自然日闭区间请求；业务层再切分为统一 `[start, end)`。
- 检查 `dateRangeNotice`和实际首末日期，未完整覆盖基准期与异常期时为数据不足。
- 修改前后标题均为空的日记录是无事件占位，直接忽略。
- 只有一端为空、日期不可解析或字段结构不完整时，该 ASIN标题修改结果整体数据不足；其他有效事件仍需列出。
- 修改前后均非空且规范化后不同才是有效标题修改事件。
- 完整覆盖两期且没有有效事件时，输出“未检测到可获取的标题修改记录”。
- 主图字段即使返回也不使用；主图修改仍不属于本 Skill。

### 6.3 Prime价格日趋势

工具：`get_asin_info_trends`

每个异常子 ASIN和每个最终竞品代表子 ASIN各调用一次：

```json
{
  "asin": "<ASIN>",
  "country": "<站点>",
  "start_date": "<基准期开始当地日期 YYYY-MM-DD>",
  "end_date": "<异常期结束前最后一个当地日期 YYYY-MM-DD>",
  "user_task": "<本轮固定脱敏原始任务>",
  "intent_summary": "取得Prime价格日趋势用于期间核查。"
}
```

必取：

```text
dateRangeNotice
trends[].date
trends[].priceDistribution.prime
```

可展示但不得替代其他正式因素：

```text
trends[].priceDistribution.display
trends[].priceDistribution.strikethrough
trends[].priceDistribution.deal
```

规则：

- 检查 `dateRangeNotice`和实际日期覆盖；任一目标日缺失时，该 ASIN Prime结果数据不足。
- `prime`可解析为大于0的数值表示当日存在 Prime价格。
- 完整日记录中的空字符串表示当日没有可获取 Prime价格。
- `prime`为0、负数、非数字、`null`或字段缺失时为数据不足。
- 按日期升序识别开始、停止、价格上涨、价格下降、持续和短期出现后结束。
- 不持续抓取，不自行保存每日快照，不查询无关完整历史。
- `deal`不得用于 Deals正式判断；Deals仍只使用卖家精灵。

## 7. 卖家精灵：Keepa趋势

工具：`keepa_info`

### 7.1 自身异常子 ASIN

```json
{
  "asin": "<异常子ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<足以取得基准期前有效锚点的毫秒时间>",
  "endTimestamp": "<异常期结束毫秒>",
  "dailyLatest": false,
  "returnFields": "asin,parentAsin,price,dealPrice,buyBoxSellerIdHistory,rating,reviews"
}
```

### 7.2 竞品代表子 ASIN

```json
{
  "asin": "<竞品代表子ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<足以取得基准期前有效锚点的毫秒时间>",
  "endTimestamp": "<异常期结束毫秒>",
  "dailyLatest": false,
  "returnFields": "asin,parentAsin,price,dealPrice,rating,reviews"
}
```

必须使用 `dailyLatest=false`保留日内事件。趋势元素按 `timePoint`升序处理，同一时间点保留最后一个有效值。

字段用途：

- `price[]`：基础价格。
- `dealPrice[]`：Deals。
- `buyBoxSellerIdHistory[]`：当前 Buy Box Seller ID。
- `rating[]`：评分。
- `reviews[]`：页面评价数量。

基础价格、Coupon、Deals和Prime不得互相替代。价格、评分或页面评价数量缺少比较端点时为数据不足。

Deals空值：

- 字段存在且空数组：无 Deals。
- 只有 `value=-1`：无有效 Deals。
- 任一 `value>0`：对应时间存在 Deals。
- 字段缺失或 `null`：数据不足。

## 8. 卖家精灵：Coupon

工具：`asin_coupon_trend`

每个异常子 ASIN和每个竞品代表子 ASIN各调用一次：

```json
{
  "asin": "<ASIN>",
  "marketplace": "<站点>",
  "returnFields": "marketplace,asin,date,type,asinPrice,couponPrice,finalPrice"
}
```

按统一站点时区把历史记录切分到基准期和异常期：

- `couponPrice > 0`且`finalPrice > 0`才是有效记录。
- `type=M`为固定金额，`type=P`为百分比。
- 成功空结果表示无 Coupon。
- 调用失败、字段缺失或日期不可解析表示数据不足。

自身 Coupon力度按 `type + couponPrice`比较；竞品只比较期间存在性。

## 9. 卖家精灵：评论

工具：`review`

### 9.1 自身新增负面评价

每个目标父商品选择一个异常子 ASIN作为评论代表，基准期和异常期分别调用：

```json
{
  "asin": "<评论代表子ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<期间开始毫秒>",
  "endTimestamp": "<期间结束毫秒>",
  "page": 1,
  "size": 10,
  "starList": [1, 2, 3],
  "returnFields": "<评论ID、date、star、title、content、verified、vine、skus>"
}
```

### 9.2 竞品新增评价

每个竞品父商品只使用其代表子 ASIN，基准期和异常期分别调用。`starList=[1,2,3,4,5]`，其他参数一致。

### 9.3 分页、去重与分期

- `size`最大使用10。
- 从 `page=1`递增，直到明确到达总页数或返回空页。
- 不得因为某页评论日期早于窗口开始而提前停止。
- 任一页失败，该父商品评论结果整体数据不足。
- 两期结果合并后去重，再按发布日期分配期间。
- 去重优先评论 ID；没有 ID时使用日期、星级、标题、规范化内容。
- 只有完整分页后的成功空结果才表示0。

## 10. 缓存复用与错误处理

缓存键：

```text
MCP命名空间 | 工具 | 站点 | ASIN或节点 | 时间窗口 | 粒度 | returnFields
```

复用规则：

- 已有字段集合是新请求字段的超集，且时间窗口相同或为完整超集时才能复用。
- 西柚当前标题按“站点 + ASIN”缓存；批量结果拆分后仍按单 ASIN缓存。
- 西柚标题修改和 Prime趋势分别按“西柚命名空间 + 工具 + 站点 + ASIN + 日期窗口”缓存。
- 同一异常子 ASIN、同一竞品代表子 ASIN和同一日期窗口不得重复调用西柚相同工具。
- 多个异常子 ASIN复用同一最终竞品时，只有时间窗口相同或已有窗口为完整超集才复用历史结果。
- 自身基础价格、Deals、Buy Box、评分和页面评价数量复用同一次 `keepa_info`。
- 竞品基础价格、Deals、评分和页面评价数量复用同一次 `keepa_info`。
- 同一任务已有满足复用键的 `dealPrice[]`时直接复用。
- 同一父商品评分、页面评价数量和评论只分析一次。
- 成功空结果、字段缺失和调用失败分别缓存，不能相互覆盖。

以下情况标记数据不足：

- MCP未连接、无权限、超时或限流后失败。
- 站点、ASIN、自有 Seller ID或当前自有父子关系无法确认。
- 叶子类目、候选父子关系、卖家精灵主图/五点、西柚标题或预测销量不满足规则。
- 时间戳无法转换到上游报告时区。
- 核心字段缺失、为 `null`或时间序列不覆盖比较端点。
- 西柚 `dateRangeNotice`或实际日期未完整覆盖基准期与异常期。
- 西柚标题修改记录只有一端标题，或 Prime目标日缺失、字段无效。
- 评论分页不完整。

高优先级目标父 ASIN五点为空的例外处理：

- 已按 4.1.1 补抓成功时，不再把父 ASIN原始五点为空标记为数据不足，必须在字段映射记录中说明五点来自子 ASIN补抓。
- 已按 4.1.1 补抓失败时，记录为“五点缺失 / 严格相似竞品审核未闭环”证据缺口；其他不依赖五点的核心诊断继续执行，最终可发送状态仍为“允许发送”。
- 候选竞品或非高优先级商品缺少五点时，仍按严格相似性数据不足处理，不触发批量补抓。

错误输出必须写明 MCP、工具、ASIN或节点、期间、缺失字段或失败类型，以及受影响因素。其他不依赖该数据的因素继续分析，不调用其他 MCP补数。
