# 跨境吴老师转化率异动 MCP 路由

## 1. 总则

- 领星 MCP 只负责当前用户环境中的店铺、站点、自有 Seller ID、自有父子商品关系和历史子体范围。
- 卖家精灵 MCP 负责商品趋势、Coupon、竞品候选与评论。
- 卖家精灵首选命名空间 `mcp__sellersprite_mcp`；仅在不可用时使用 `mcp__sellersprite_mcp_2` 的同名工具、相同参数和相同字段。
- 卖家精灵站点统一使用 `US`、`JP`、`UK`、`DE`、`FR`、`IT`、`ES`、`CA`、`IN`、`MX`、`BR`、`AU`、`AE`。
- 月份使用 `yyyyMM`，时间戳使用毫秒。
- 每次卖家精灵调用都指定最小必要 `returnFields`。字段为空、核心值为 `null` 或时间覆盖不足时不得补值。
- 运行时先读取卖家精灵数据库 Skill：`$kuajing-wulaoshi-sellersprite-mcp-database`；若该 Skill 名称在当前环境不同，则读取同一跨境吴老师卖家精灵 MCP 数据库说明后再调用。

## 2. 固定调用顺序

```text
1. 领星：店铺/站点/Seller ID
2. 领星：父 ASIN、当前子体、两期历史子体
3. 卖家精灵：竞品候选池与最新完整月销量
4. 卖家精灵：自身与竞品 Keepa 聚合趋势
5. 卖家精灵：自身与竞品 Coupon
6. 卖家精灵：自身负面评论与竞品全部评论
7. 业务层：事件比较、数据复用、证据分组和输出
8. Prime：仅写人工核查，不调用
```

竞品集合必须在所有竞品因素之前生成一次，后续不能为不同因素重新换一组竞品。

## 3. 领星路由

### 3.1 店铺、站点和自有 Seller ID

工具：`LingXing-MCP.get_my_sids`

必取逻辑字段：

```text
sid
country
name / 店铺名
seller_id
mid / marketplace_id / marketplace
```

固定规则：

- 用用户指定站点匹配 `sid`。
- 用户指定店铺时，用店铺名和站点共同匹配。
- 同站点多店铺且用户未指定时停止并要求选择。
- `seller_id` 是 Buy Box 当前归属判断的自有 Seller ID。
- 所有 ID 每次运行动态取得，不复用测试环境或历史运行值。

### 3.2 父 ASIN和当前子体

工具：`LingXing-MCP.erp_listing`

用户输入按子 ASIN查询：

```json
{
  "sids": "<sid>",
  "mids": "<mid>",
  "search_field": "asin1",
  "search_value": ["<用户输入 ASIN>"],
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
  "search_value": ["<用户输入 ASIN>"],
  "offset": 0,
  "length": 200,
  "pvi_ids": "",
  "exact_search": "1"
}
```

确认父 ASIN后，再以 `search_field=parent_asin` 和该父 ASIN查询全部当前子体并分页。

必取逻辑字段：

```text
parent_asin
asin1
seller_sku
sid
marketplace
```

只接受同时命中目标 `sid`、目标 ASIN或目标父 ASIN的实际返回行。不能只看 `total` 判断精准筛选成功。

### 3.3 基准期和异常期子体

工具：`LingXing-MCP.query_product_performance_asin_lists`

两期分别调用，核心入参：

```json
{
  "sids": "<sid>",
  "start_date": "<期间开始 yyyy-MM-dd>",
  "end_date": "<期间结束 yyyy-MM-dd>",
  "date_range_type": 0,
  "date_type": "purchase",
  "currency_code": "CNY",
  "search_field": "parent_asin",
  "search_value": ["<父 ASIN>"],
  "summary_field": "asin",
  "turn_on_summary": 1,
  "offset": 0,
  "length": 500
}
```

必取逻辑字段：`asin`、`parent_asin`。如果上游同时需要核验转化率口径，可附带读取 `sessions_total` 和 `volume`，但不得在本 Skill 内自行发明异动阈值。

将当前 `asin1`、基准期 `asin`、异常期 `asin` 合并去重，生成 `child_asin_union`。任一历史期间只返回父商品汇总而没有子 ASIN时，完整并集相关结果标记为数据不足。

## 4. 竞品集合路由

### 4.1 候选池

工具：`mcp__sellersprite_mcp__traffic_listing`

核心入参：

```json
{
  "request": {
    "asinList": ["<单个异常子 ASIN>"],
    "marketplace": "<站点>",
    "relations": ["<当前工具 schema 支持的竞品或同类关系>"],
    "variations": true,
    "page": 1,
    "size": 40,
    "returnFields": "<只填写当前 schema 已确认的候选 ASIN、标题、关联关系字段>"
  }
}
```

`relations` 的合法值必须从当前工具 schema 取得，不能猜枚举。分页直到候选重复、返回空列表或已获得足够的严格相似候选。

### 4.2 补充与排序

工具：`mcp__sellersprite_mcp__competitor_lookup`

固定规则：

- `month` 使用运行日之前的最新完整自然月。
- 候选 ASIN最多 40 个一批，超过 40 个拆批。
- `variation="N"`，保留子 ASIN数据。
- `order.field="total_units"`，`order.desc=true`。
- 最小逻辑字段：`asin`、`parent`、`title`、月销量，以及当前工具可用的图片和卖点字段。

示意入参：

```json
{
  "request": {
    "marketplace": "<站点>",
    "month": "<最新完整月 yyyyMM>",
    "asins": ["<候选 ASIN，单批最多 40 个>"],
    "variation": "N",
    "order": {"field": "total_units", "desc": true},
    "page": 1,
    "size": 40,
    "returnFields": "<当前 schema 已确认的 asin,parent,title,totalUnits、图片和卖点字段>"
  }
}
```

字段字面名称以当前工具 schema 与实测返回为准。不得为了满足文档示例传入不存在的图片或卖点字段。若 `competitor_lookup` 无法给全图片、标题、卖点，可用同一 ASIN的 `keepa_info` 或商品详情工具补齐；仍不能补齐时按业务规则标记竞品集合数据不足。

排序值必须是子 ASIN最新完整月销量。销量缺失的候选不能进入销量前五名，也不能用 BSR 或评分替代销量。

## 5. Keepa 聚合调用

### 5.1 自身 ASIN

每个自身子 ASIN合并为一次 `keepa_info` 调用：

```json
{
  "asin": "<自身子 ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<覆盖基准期端点所需的最早毫秒时间戳>",
  "endTimestamp": "<异常期结束毫秒时间戳>",
  "dailyLatest": false,
  "returnFields": "asin,parentAsin,price,dealPrice,buyBox,buyBoxSellerIdHistory,rating,reviews"
}
```

### 5.2 竞品 ASIN

每个竞品 ASIN合并为一次 `keepa_info` 调用：

```json
{
  "asin": "<竞品 ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<覆盖基准期端点所需的最早毫秒时间戳>",
  "endTimestamp": "<异常期结束毫秒时间戳>",
  "dailyLatest": false,
  "returnFields": "asin,parentAsin,price,dealPrice,rating,reviews"
}
```

价格必须使用 `dailyLatest=false`，保留日内变化。为取得基准期开始前仍然有效的价格、评分或评价数量，需要让时间范围包含一个有效前置锚点；若限定窗口内没有锚点，可扩展一次时间范围，仍缺失则标记数据不足。

趋势数组统一结构：`timePoint` 为毫秒时间戳，`value` 为值。清洗后按时间升序排序，同一时间点和同值重复项去重。

Deals 只认 `dealPrice[]`；Coupon 不能从 `dealPrice[]` 判断。Prime 折扣也不能从本调用自动归因。

## 6. Coupon 调用

工具：`mcp__sellersprite_mcp__asin_coupon_trend`

每个自身分析子 ASIN和每个入选竞品各调用一次：

```json
{
  "asin": "<ASIN>",
  "marketplace": "<站点>",
  "returnFields": "marketplace,asin,date,type,asinPrice,couponPrice,finalPrice"
}
```

在返回历史记录中按站点当地时间切分基准期与异常期。成功返回空列表表示两期都没有可见 Coupon 记录；接口错误、日期缺失或无法确认时间覆盖则是数据不足。

不得用 Coupon 调用替代 Deals，也不得用 Deals 调用替代 Coupon。

## 7. 评论调用

工具：`mcp__sellersprite_mcp__review`

### 7.1 自身负面评论

基准期和异常期分别调用：

```json
{
  "asin": "<评论归属 ASIN>",
  "marketplace": "<站点>",
  "startTimestamp": "<期间开始毫秒>",
  "endTimestamp": "<期间结束毫秒>",
  "page": 1,
  "size": 10,
  "starList": [1, 2, 3],
  "returnFields": "date,star,title,content,verified,vine,skus"
}
```

### 7.2 竞品全部新增评论

基准期和异常期分别调用，`starList=[1,2,3,4,5]`，其余参数相同。

### 7.3 分页与去重

- `size` 最大使用 10。
- 从 `page=1` 递增，直到返回空列表、明确到达总页数，或评论日期已经早于窗口开始。
- 只有完整分页后的成功空列表才表示新增评论为零。
- API 失败、某页失败或日期无法解析时，该期间评论结果为数据不足。
- 若多个子 ASIN共享父商品评论，以卖家精灵父 ASIN和评论标识确认后只调用一次。
- 评论去重优先使用评论 ID；缺失时使用日期、星级、标题、内容摘要的组合键。

## 8. 缓存与复用

内部缓存键：

```text
MCP命名空间 | 站点 | ASIN或节点 | 时间窗口 | 粒度 | returnFields
```

规则：

- 已缓存结果的字段集合是新请求字段的超集时直接复用。
- 自身价格、Deals、Buy Box、评分和评价数量复用同一次自身 `keepa_info`。
- 竞品价格、Deals、评分和评价数量复用同一次竞品 `keepa_info`。
- 同一任务的流量异动模块已有相同 ASIN、站点、窗口的 `dealPrice[]` 时直接复用。
- 共享父商品评论只读取和统计一次。
- 成功空结果、调用失败和核心字段缺失必须以不同缓存状态保存，不能互相覆盖。

## 9. 错误处理

以下情况不得产生变化结论：

- MCP 未连接、无权限、超时或限流后仍失败。
- 站点、ASIN、Seller ID或父子商品关系无法确认。
- 时间戳无法转换到目标站点时区。
- 核心字段为 `null`、字段不存在或时间序列不覆盖比较端点。
- 竞品图片、标题、卖点或子 ASIN月销量不足以完成筛选。
- 评论分页未完整取得。

错误输出必须写明：失败的 MCP/工具、ASIN、期间、缺失字段或失败类型，以及受影响的因素。其他不依赖该数据的因素继续分析。

