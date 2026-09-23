# 12 · 开放平台 TOP API 通道（订单 / 监控 / 客服 / 数据的官方通道）

> 来源：整合自 OpenClaw 生态技能 **taobao-integration** 的公开介绍（《每日热门skill：一个人管3家店？这个OpenClaw Skill让淘宝运营效率翻10倍》，CSDN 2026-09），按本仓库护栏改造。
> ⚠️ **状态标注**：本仓库其他章节全部来自本店实测，本篇不同——标注〔文章〕的内容是方法论整合，TOP API 通道**本店尚未开通实测**。开通后按此 SOP 走，走通回填实测记录。

## 定位：它补了浏览器通道的什么短板

现有通道（商品管家 / CDP / bsk / 浏览器 Agent，见 05/10）全部依赖"已登录千牛后台的浏览器"，擅长**商品编辑页内的写操作**，但三类活很吃力：

| 浏览器通道吃力的活 | 为什么吃力 | TOP API 对应 |
|---|---|---|
| 订单级筛查（待发货/退款/地址异常） | 卖家中心订单页翻页慢、无导出 API 级结构化 | `taobao.trades.sold.get` |
| 周期性监控（竞品价格/运营日报/库存预警） | 浏览器定时爬后台慢且脆，锁屏断连就断档 | 只读查询接口 + 定时任务，7×24 云上跑 |
| 7×24 智能客服应答（RAG 知识库） | 旺旺回复靠人工或半自动，夜里没人 | 商品/FAQ 数据进本地向量库，自动应答 |

TOP API（淘宝开放平台 open.taobao.com）就是官方给这三类活留的通道：**中文指令 → AI 解析成结构化参数 → 调官方接口 → 结构化结果**，全程不碰浏览器。

## 四模块映射 × 本店红线（先读这张表再动手）

| 模块 | TOP API 能做什么〔文章〕 | 本店口径（护栏优先于能力） |
|---|---|---|
| 商品管理 | 查商品、上下架、批量改标题/价格/库存、SEO 建议、类目匹配 | ✅ 只用**查**接口（`taobao.items.listing.get` / `taobao.item.get`）做台账与 SEO 分析；⛔ `taobao.item.add` 发新品、⛔ 改价/库存/运费——SKILL.md 护栏红线，**通道再官方也不开**（用户新品/改价一律人工决策） |
| 订单处理 | 查订单、异常单预警、批量发货、退款处理 | ✅ 只读查询 + 汇总表（地址不全/退款申请筛出来交人工）；⚠️ 发货属敏感操作，脚本只出**草稿清单**，人工确认后执行 |
| 客服自动化 | RAG 知识库 7×24 应答 | ✅ 知识库直接复用 templates.md 客服话术 + 详情页 FAQ 块当底料；回复口径同 04（**绝不留电话/微信/外链**），拿不准出草稿留人工 |
| 数据分析 | 运营日报、竞品价格监控（>5% 预警）、库存预警 | ✅ 全只读，最值得先做的一块；竞品监控与《获客率提升方案》"竞品监控只读化"决议一致 |

## 开通 SOP〔文章，个人 C 店以开放平台后台实际可勾选权限为准〕

1. open.taobao.com 注册开发者账号（店铺资质决定可申请的权限范围）
2. 创建应用，申请**商品 / 交易 / 物流**三大权限包——**审批约一周，提前办**
3. 拿到 `AppKey` / `AppSecret` / `SessionKey`（授权链路签发）
4. **现实核查**：个人 C 店的部分写接口（尤其 `item.add`、物流发货）可能不批，以后台实际可勾选为准；不批的能力照旧走浏览器通道，不批 ≠ 这条通道白开——订单查询、数据分析类只读接口通常可拿

## 技术要点〔文章 + TOP 通用规范〕

- **签名**：公共参数 + 业务参数按参数名 ASCII 升序拼 `name+value`，首尾接 AppSecret，按 `sign_method`（推荐 `hmac-sha256`）取大写 hex。调用量小，没必要引 SDK，几十行 Python 够用：

```python
# top_api_client.py —— TOP API 最小调用器（hmac-sha256 签名）
import hmac, hashlib, time, requests, os

APP_KEY    = os.environ["TOP_APP_KEY"]      # 凭证走环境变量，绝不入库
APP_SECRET = os.environ["TOP_APP_SECRET"]
GATEWAY    = "https://gw.api.taobao.com/router/rest"

def call(method, session=None, **biz):
    params = {
        "method": method, "app_key": APP_KEY, "format": "json", "v": "2.0",
        "sign_method": "hmac-sha256", "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
        **({"session": session} if session else {}), **biz,
    }
    plain = "".join(k + str(params[k]) for k in sorted(params))
    params["sign"] = hmac.new(APP_SECRET.encode(), plain.encode(),
                              hashlib.sha256).hexdigest().upper()
    r = requests.post(GATEWAY, data=params, timeout=15)
    time.sleep(0.5)                     # QPS 纪律：请求间隔 ≥500ms
    r.raise_for_status()
    return r.json()

# 用法：call("taobao.trades.sold.get", session=SESSION, fields="tid,status,payment", ...)
```

- **SessionKey 24 小时失效**：用 `refresh_token` 做自动刷新（credential-manager 或计划任务），否则每早手工授权一次很快就弃用
- **凭证安全**：AppKey/Secret/Session 只进环境变量或系统凭据库；`.gitignore` 兜底 + 提交前特征扫描（本仓库 06 节安全核对同款流程）
- **敏感操作二次确认**：发货/改价/删除类接口在脚本层加 `--confirm` 人工闸，不进自动队列
- **全程日志**：每次调用记 method/参数摘要/返回码，出问题可回放
- **7×24 任务**（监控/客服）部署到云服务器（30~50 元/月），本机只跑交互式任务

## 与浏览器通道的分工

| 任务类型 | 走哪条通道 |
|---|---|
| 编辑页写操作（标题/属性/SKU 文本/详情 FAQ） | 商品管家 > CDP/bsk > 浏览器 Agent（05/10）——TOP API 不覆盖编辑页细粒度字段 |
| 发新品、改价 | 人工。任何通道都不代做（红线） |
| 订单查询/异常筛查 | TOP API `trades.sold.get`；无权限时浏览器看板人工筛 |
| 竞品价格监控/日报/预警 | TOP API 只读接口 + 定时任务；无权限时降级打法见下 |
| 客服应答 | 知识库（templates + FAQ）+ TOP API 或旺旺半自动；口径同 04 |
| 逛逛/视频/问大家 | 只能浏览器/客户端通道（API 无此能力） |

## 无凭证期的降级打法（今天就能跑，不用等审批）

1. **竞品监控只读化**：定时任务打开买家端搜索页/竞品详情截图，只读比对价格，变动 >5% 记台账并提醒——不登录、不下单、不采集超出公开页面的数据
2. **客服知识库先行**：templates.md 话术 + 详情页 FAQ 块本身就是 RAG 底料，先在人工回复时引用，权限下来直接接自动应答
3. **订单筛查**：卖家中心订单页按"退款/待发货"筛选 + 台账记录，人工 10 分钟的活不值得为它开权限
4. **运营日报**：Phase 4 的生意参谋三指标人工看，攒成周报

## 避坑清单〔文章〕

- API 权限包审批约一周，别等急用了才申请
- SessionKey 24h 失效是"这条通道被弃用"的第一原因——先做自动刷新再谈别的
- 控制节奏：请求间隔 ≥500ms，别改内置请求队列，触发限流得不偿失
- 凭证泄露 = 店铺数据泄露，AppSecret 视同密码
- ⛔ 一切接口（尤其订单/物流类）**不得用于刷单、虚假发货**——平台风控直接封店，红线中的红线
- 多店铺/多应用场景做 Token 授权隔离，一个应用一个 Session，最小权限原则
