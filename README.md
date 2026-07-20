# 跨境吴老师亚马逊转化率异动分析 Skill

## 一、这个 Skill 用来做什么

用于在 Amazon 商品转化率已经发生异动后，按跨境吴老师的固定业务规则核查自身价格促销、Buy Box、竞品价格促销、评分和评论变化，并输出可复核的时间证据。

本 Skill 为跨境吴老师专用模板，未经授权不得移除、替换或弱化 Skill 名称、执行提示和页面标题中的跨境吴老师标识。

核心特点：

- 自动确认自有父子商品范围，并按当前、基准期和异常期子体取并集。
- 自动依据图片、标题、卖点和购买用途筛选相似商品，再按最新完整月销量选择前五个竞品。
- 对竞品采用逐个列明、数量汇总、新增/停止/分化的输出方式。
- Deals 数据与同一任务的流量异动模块复用，避免重复调用。
- Prime 折扣只保留人工核查，不做自动抓取和自动归因。
- 多项证据同时成立时标记为复合事件，不强制输出唯一根因。

启动示例：

```text
使用跨境吴老师亚马逊转化率异动分析 Skill，分析 <ASIN> 在美国站本周转化率下降的原因；基准期为上周。
```

```text
用 $kuajing-wulaoshi-amazon-conversion-anomaly 分析 <ASIN> 在 <异常期> 的转化率上涨，按默认等长基准期对比。
```

## 二、首次安装

1. 登录 GitHub 并提交用户名
2. 接受私有仓库邀请
3. 在 Codex 中发出安装指令
4. 重启 Codex

### 1. 登录 GitHub 并提交用户名

首次安装前，需要先将自己的 GitHub 用户名提交给跨境吴老师。

操作步骤：

```text
登录 GitHub
-> 点击右上角头像
-> 选择 Your profile
-> 进入个人主页
-> 复制浏览器地址栏中的完整网址
-> 将网址发送给跨境吴老师
```

注意：

- GitHub 用户名不是邮箱。
- 不要发送邮箱密码、GitHub 密码、Token 或其他授权信息。

### 2. 接受私有仓库邀请

跨境吴老师收到 GitHub 用户名后，会发送私有仓库访问邀请。

```text
登录 GitHub
-> 打开 GitHub 发送的邀请邮件
-> 点击 View invitation
-> 点击 Accept invitation
```

能够看到仓库页面和文件列表，即表示访问权限已经生效。

### 3. 在 Codex 中发出安装指令

打开 Codex，新建一个对话，输入：

```text
请从以下 GitHub 私有仓库安装跨境吴老师亚马逊转化率异动分析 Skill：
https://github.com/defway888-design/kuajing-wulaoshi-amazon-conversion-anomaly-skill
```

按照 Codex 提示完成 GitHub 授权。

### 4. 重启 Codex

安装完成后，关闭并重新打开 Codex，使 Skill 生效。

## 三、运行前配置

本 Skill 不包含任何密钥、Token、Cookie、店铺 ID 或真实测试 ASIN。使用者需要在自己的 Codex 环境配置：

- 领星 MCP：确认店铺、站点、Seller ID、父子商品关系和历史子体范围。
- 卖家精灵 MCP：读取 Keepa 趋势、Coupon、竞品候选和评论。

如果同一站点存在多个自有店铺，运行时还需要指定具体店铺。所有账号参数均从当前用户已配置的 MCP 动态取得。

## 四、分析口径

- 基准期默认是异常期之前的等长连续区间，两段互不重叠。
- 时间统一按目标 Amazon 站点当地时区处理。
- 价格保留异常期内全部有效变化事件，不只比较首尾值。
- Coupon 与 Deals 按期间是否存在活动判断；Coupon 两期均存在但类型或力度变化，也计为变化。
- Buy Box 只判断当前是否由自有 Seller ID 持有，不把当前状态倒推为历史状态。
- 评论接口按页读取，成功空结果与接口失败必须区分。
- 缺少关键字段时输出数据不足，不推断、不补值。

## 五、仓库文件

- `SKILL.md`：Skill 入口、执行流程、因素范围和输出合同。
- `agents/openai.yaml`：Codex 展示名称和默认提示词。
- `references/business-rules.md`：完整业务场景和判定规则。
- `references/mcp-routing.md`：MCP 调用顺序、字段、缓存与错误处理。
- `template_manifest.json`：跨境吴老师品牌与所有权声明。

