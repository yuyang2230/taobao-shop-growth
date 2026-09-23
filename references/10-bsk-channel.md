# Phase 3 · bsk 扩展通道：千牛 React 后台全自动操作配方

> 来源：2026-09-22~23 某实验室仪器类淘宝C店实战验证——单日 13 件商品标题/字段修改全部提交成功、回读通过，全程零误操作（含一次会话失效自动拦截）。
> 定位：当 CDP 9222 不可用（Chrome 未带调试端口启动、且禁止重启浏览器保登录态）时，**bsk 浏览器扩展通道是 React 后台自动化的主力通道**。
> 前提：Chrome 已装 bsk 扩展（带 Agent Window 能力），`bsk.exe` 在 PATH（如 `C:\Users\Administrator\.local\bin\bsk.exe`）。

---

## 1. 通道架构与核心命令

```
bsk session start --no-focus          # 创建会话（返回 4 字母 sid，不抢焦点）
bsk navigate --session <sid> <url> --wait-until domcontentloaded --timeout 60s
bsk evaluate --session <sid> "<JS表达式>"        # 在页面执行 JS，返回结果
bsk fill --session <sid> --selector <css> --value <文本>   # 真实输入线（React 安全）
bsk press --session <sid> <键名> [--selector <css>]
bsk click --session <sid> --selector <css>
bsk screenshot --session <sid> --out <png路径>
bsk tab list --session <sid>           # 列标签页（agent/user 作用域）
bsk tab close --session <sid> <tabId>  # 关标签（最后一个标签关闭后会话自动结束）
bsk session list                       # 会话清单（"no active sessions"=全清）
```

**中文传参坑（Windows）**：PowerShell 5.1 给原生 exe 传参会吞/乱码中文 → 用 **Node 桥**：PS 写 spec JSON（UTF-8 无 BOM）→ `node bsk_node.js <spec.json>` spawnSync 调 bsk。JS 代码放 `.js` 文件读入（node 原生 UTF-8，无乱码）；evaluate 表达式作为 args 数组传递。

**会话生命周期坑**：
- 会话可能**静默超时失效**（报 "session not registered or already stopped"）→ 所有批量脚本必须**幂等**：每件操作前重读状态校验，失效就重建会话重来。
- `--no-focus` 创建的窗口可能出现在屏幕上 → 用 Win32 SetWindowPos 把 agent 窗口移到虚拟屏幕外（`vs.Right + 400`），**只移 title 精确匹配当前导航页的窗口**，绝不动用户自己的标签页/窗口。
- 关会话 = 关掉它的 tab；**tab 清理正则只匹配 agent 作用域的纯数字 tab id**，`tab list` 里 scope=user 的标签（用户自己的页面）绝不能碰。

## 2. 千牛 v2 编辑页字段定位（验证过的选择器配方）

页面 URL：`https://upload.taobao.com/auction/publish/edit.htm?item_num_id={ID}&auto=false&itemId={ID}`（自动跳 v2 publish.htm）
- **进入后必按 Esc**：关掉"商品发布提示"（SKU 升级公告）浮层。
- 字段 wrapper 统一结构：`.sell-component-info-wrapper-wrap`，label 在 `.sell-component-info-wrapper-label`。
- **两步标记法**：先用 evaluate 给目标控件打 `data-tbop` 标记，再用带 `--selector` 的 fill/click 操作标记：

| 字段 | 标记 JS（evaluate） | 操作选择器 |
|---|---|---|
| 宝贝标题 | 找 label 含"宝贝标题"的 wrapper 里 input/textarea → `setAttribute('data-tbop','btitle')` | `[data-tbop='btitle']` |
| 导购标题 | label 含"导购标题" → `'guide'` | `[data-tbop='guide']` |
| 型号 | label 含"型号" → `'model3'` | `[data-tbop='model3']` |
| 品牌 | label 精确"品牌"的 `.next-select` → `'brand'` | `[data-tbop='brand'] input` |
| 提交按钮 | `[...document.querySelectorAll("button")].filter(x => x.innerText.trim()==="提交宝贝信息")[0]` → `'submit'` | `[data-tbop='submit']` |

**品牌 auto-complete 特殊坑（实测 3 轮才通）**：
- 下拉是**虚拟化列表**（首屏仅 14 个热门品牌），fill 中文不触发过滤、逐键 press 会被 IME 上下文吞、fire input/change 事件也不刷新选项——都不用管。
- **有效配方**：`fill [data-tbop='brand'] input <品牌名>` → React 状态直接接受该值（field_read 回读 brand="<品牌名>" 即生效），失焦即选中。
- **绝不在品牌框按 Enter**——会触发表单默认提交（实测误提交过一次，好在字段当时已填好、基础分 100 无损）。填完按 **Escape** 收下拉。

## 3. 提交四步与验证闭环（每件必走）

```
1. submit_guard（evaluate）：
   - 找"提交宝贝信息"按钮 → scrollIntoView → 标记 data-tbop='submit'
   - 若品牌下拉 overlay 残留（.options-content 且 aria-expanded=false）→ display:none 掉
   - activeElement.blur()
2. click [data-tbop='submit']
3. 等 11 秒
4. page_state 读 url：必须形如
   https://item.upload.taobao.com/sell/v2/success.htm?primaryId={ID}&catId=...&isEdit=true
   页面文本含"商品提交成功" + 基础分 + 扶优分
```

**bsk click 被 overlay 拦截时的兜底**（click 命中但页面不跳转）：JS 原生点击兜底——
`var b=[...document.querySelectorAll('button')].filter(x=>x.innerText.trim()==='提交宝贝信息')[0]; b.scrollIntoView({block:'center'}); b.click();`
（React 合成事件监听在真实 DOM click 上，此法实测触发成功跳转。）

**回读验证**：提交成功后**重新 navigate 编辑页 → 重读字段值**，与新值完全一致才算闭环（field_read 返回四字段实际值）。只看 success.htm 不算完。

## 4. 批量安全闸（实测拦下一次全量事故）

批量脚本每件必须走三态校验，任何一态不符就 SKIP 该件、绝不提交：

```
PRE   = 重新加载编辑页 → mark_title 读当前标题 → 必须等于预期的"旧标题"（防会话失效/防串商品）
FILL  = fill 新值 → mark_title 重读 → 必须等于"新标题"且 err=0、计数器 ≤60
POST  = 提交后 page_state.url 必须含 success.htm + primaryId 对应
```

实测案例：批量 7 件首次运行时会话已静默失效，PRE 读到空值 → 7 件全部自动 SKIP（0 误提交）→ 重建会话后幂等重跑 → 7/7 成功。**这就是为什么 PRE 校验不可省。**

## 5. 数据采集配方（选品与定位）

- **后台全量扫描**：`myseller.taobao.com/home.htm/SellManage/on_sale?current=1..4&pageSize=20`（pageSize=100 不支持会"没有数据"），evaluate 抓 body innerText，PS 按 `ID:(\d+)` 提取商品清单（价格/库存/累计销量/30日销量/质量分）。
- **搜索现状**：`https://s.taobao.com/search?q={encodeURIComponent(关键词)}`，evaluate 抓商品卡（id/标题/付款人数）；**翻页参数 s=44 会被平台回退第 1 页，未生效**——位次结论以首页为准。
- **千人千面意识**：卖家登录态的位次 ≠ 买家视角位次（画像/地域/设备都会重排）。**结论要么注明视角，要么综合排序+销量排序双采**；买家视角靠用户截图对照。
- **AI 面（2026-09 实测）**：网页端无独立 AI 搜索入口（ai.taobao.com 是旧"爱淘宝"）；**商品详情页有「问问 AI」**：`textarea[class*='textarea--']`（placeholder"关于这款商品，你还想知道什么？"），fill+Enter 可问答——这是检验商品结构化信息完整度的免费探针（标题/属性越全，AI 答得越准）。

## 6. 红线（每次都要灵守）

1. **绝不强杀/重启 Chrome**——淘宝登录态是 session 型，重启 = 全部重来。
2. 永不修改：价格、库存、SKU 规格表、运费模板（除非用户逐项明确授权）。
3. 台账驱动：每件三态（pre/fill/post）+ 回读全部落 JSON 存证据；中断后从台账续跑。
4. 同一操作连败 2 次停下汇报，不硬闯。
5. 用户标签页（scope=user）绝不动；agent 窗口操作完关干净（session list 验证）。

## 7. 已验证战绩（本配方）

- 2026-09-22 单日：13 件商品（标题×12 + 放料阀品牌/型号/导购标题字段补齐）全部提交成功 + 回读通过；Jev 决策 12 份原件存档。
- 安全闸拦截会话失效事故 1 次（7 件全 SKIP 零误操作）。
- 证据规模：219 文件（JSON 三态/截图/SERP 快照/Jev 决策）。
