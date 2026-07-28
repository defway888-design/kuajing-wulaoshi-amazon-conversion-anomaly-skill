# 跨境吴老师转化率异动 MCP 路由

## 目录

- 1. 数据源边界
- 2. 固定调用顺序
- 3. 领星：当前自有商品身份
- 4. 卖家精灵：竞品生成
- 5. 卖家精灵：预测近30天销量
- 6. 卖家精灵：Keepa趋势
- 7. 卖家精灵：Coupon
- 8. 卖家精灵：评论
- 9. 缓存复用与错误处理

## 1. 数据源边界

同一种数据只能使用一个来源：

| 数据 | 固定来源 |
|---|---|
| 当前店铺、站点、自有 Seller ID | 领星 MCP |
| 当前自有父 ASIN和子 ASIN | 领星 MCP |
| 上游自有转化率、Session、订单 | 领星 MCP或上游已确认结果 |
| 叶子类目、BSR候选、关键词自然 Top 10 | 卖家精灵 MCP |
| 当前主图、标题、五点、竞品父子关系 | 卖家精灵 MCP |
| 竞品预测销量 | 卖家精灵 `asin_prediction` |
| 自身及竞品基础价格、Deals、Buy Box历史、评分、页面评价数量 | 卖家精灵 `keepa_info` |
| 自身及竞品 Coupon | 卖家精灵 `asin_coupon_trend` |
| 自身及竞品新增评论 | 卖家精灵 `review` |

禁止用另一 MCP对同一指标补数、校验或替代。卖家精灵不可用时标记数据不足。

运行时通过当前工具目录或工具搜索发现卖家精灵 MCP。确认其能够提供本文件规定的工具后，选择一个可用的卖家精灵 MCP命名空间，并在整次任务中固定使用该命名空间。不得因为某个字段缺失而切换到另一个 MCP补数。

站点代码使用 `US`、`JP`、`UK`、`DE`、`FR`、`IT`、`ES`、`CA`、`IN`、`MX`、`BR`、`AU`、`AE`。时间戳使用毫秒。每次调用只请求必要字段。

## 2. 固定调用顺序

```text
1. 领星：目标店铺、站点、自有 Seller ID
2. 领星：当前父 ASIN和当前子 ASIN
3. 卖家精灵：目标商品当前内容、最细叶子类目和小类 BSR
4. 卖家精灵：叶子类目 BSR 1—100候选
5. 卖家精灵：候选当前内容、竞品父子关系和预测近30天销量
6. 若不足五个：放弃 BSR池，用主要自然流量词 Top 10重建候选池
7. 卖家精灵：自身和最终竞品 Keepa聚合趋势
8. 卖家精灵：自身和最终竞品 Coupon
9. 卖家精灵：自身负面评论和竞品全部评论
10. 业务层：时间切分、证据分组、方向判断与输出
11. Prime：只输出人工核查，不调用
```

竞品集合只生成一次并在本次任务中复用。

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

### 4.1 目标商品当前内容与叶子类目

固定使用 `keepa_info`读取目标商品当前画像，最小逻辑字段：

```text
asin
parentAsin
title
features
主图 Amazon图片 ID或主图 URL
最细类目节点 ID及完整路径
小类 BSR
variationAsins
可靠上架日期
```

只接受卖家精灵当前字段。主图、标题或五点任一缺失时，严格相似性判断数据不足。

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
    "returnFields": "<ASIN、父ASIN、小类BSR、标题、上架日期>"
  }
}
```

业务层只保留小类 BSR 1—100并按小类 BSR升序建立原始优先级。不得换成大类 BSR或关联流量候选。

### 4.3 候选补充、父体分组与相似性

对候选 ASIN用同一卖家精灵详情口径补齐：

```text
parentAsin
variationAsins
title
features
主图 Amazon图片 ID或主图 URL
可靠上架日期
```

按 `references/business-rules.md`中的“按父商品分组并排除目标商品”“选择竞品代表子体”和“严格判断商品相似性”执行：

- 排除目标当前父商品的全部子体和明确异常 ASIN。
- 其他自有商品不排除。
- 按竞品父 ASIN分组。
- 当前主图、标题、五点任一缺失即不能默认相似。

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

## 6. 卖家精灵：Keepa趋势

工具：`keepa_info`

### 6.1 自身异常子 ASIN

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

### 6.2 竞品代表子 ASIN

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

## 7. 卖家精灵：Coupon

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

## 8. 卖家精灵：评论

工具：`review`

### 8.1 自身新增负面评价

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

### 8.2 竞品新增评价

每个竞品父商品只使用其代表子 ASIN，基准期和异常期分别调用。`starList=[1,2,3,4,5]`，其他参数一致。

### 8.3 分页、去重与分期

- `size`最大使用10。
- 从 `page=1`递增，直到明确到达总页数或返回空页。
- 不得因为某页评论日期早于窗口开始而提前停止。
- 任一页失败，该父商品评论结果整体数据不足。
- 两期结果合并后去重，再按发布日期分配期间。
- 去重优先评论 ID；没有 ID时使用日期、星级、标题、规范化内容。
- 只有完整分页后的成功空结果才表示0。

## 9. 缓存复用与错误处理

缓存键：

```text
MCP命名空间 | 工具 | 站点 | ASIN或节点 | 时间窗口 | 粒度 | returnFields
```

复用规则：

- 已有字段集合是新请求字段的超集，且时间窗口相同或为完整超集时才能复用。
- 自身基础价格、Deals、Buy Box、评分和页面评价数量复用同一次 `keepa_info`。
- 竞品基础价格、Deals、评分和页面评价数量复用同一次 `keepa_info`。
- 同一任务已有满足复用键的 `dealPrice[]`时直接复用。
- 同一父商品评分、页面评价数量和评论只分析一次。
- 成功空结果、字段缺失和调用失败分别缓存，不能相互覆盖。

以下情况标记数据不足：

- MCP未连接、无权限、超时或限流后失败。
- 站点、ASIN、自有 Seller ID或当前自有父子关系无法确认。
- 叶子类目、候选父子关系、主图、标题、五点或预测销量不满足规则。
- 时间戳无法转换到上游报告时区。
- 核心字段缺失、为 `null`或时间序列不覆盖比较端点。
- 评论分页不完整。

错误输出必须写明 MCP、工具、ASIN或节点、期间、缺失字段或失败类型，以及受影响因素。其他不依赖该数据的因素继续分析，不调用其他 MCP补数。
