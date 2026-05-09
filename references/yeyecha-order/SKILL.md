---
name: yeyecha-order
description: 爷爷不泡茶下单意图引导 sub-skill。用户表达下单/买/点单/付款意图时，调 prepare_wx_handoff 工具拿到一个 https 中转链接；用户点击后由 H5 中转页按设备自动跳爷爷不泡茶小程序（微信内 wxaurl / 移动非微信 scheme / PC 二维码扫码）。AI 不直下单，只给入口；最终下单 / 支付在小程序内完成。
version: 0.3.1
parent: noyeyenotea-skill
status: active
trigger: 用户表达下单/支付/复购/取消订单/查具体订单状态/退款 等写操作意图
---

# 爷爷不泡茶 · 下单 handoff（v0.3.1）

> **当前状态：active**。AI 不直下单，但通过 `prepare_wx_handoff` 工具给一个 H5 中转链接，
> 用户点击后由 H5 按设备自动跳爷爷不泡茶小程序：
> - **微信内**：直接打开
> - **手机非微信**：唤起微信
> - **PC**：显示二维码引导扫码
>
> 最终下单 / 支付完全在小程序内完成。设计文档：`ai-platform/docs/wx-handoff-design.md`。

## 触发关键词

用户提到以下任一意图 → 调 `prepare_wx_handoff`：

- 下单 / 点单 / 买一杯 / 买点喝的 / 帮我下单
- 付款 / 支付 / 结账（不接管支付，引导到小程序付）
- 想喝 X / 我要 X（带具体商品名时）

**仍未对接**（沿用"AI 还不能直接做"话术 + 小程序引导）：
- 查具体订单状态 / 查 D00xxx 订单 / 我那杯做好没
- 改订单 / 加个料 / 取消订单 / 退款 / 退一下

## 标准流程

### 1. 先帮用户挑（用现有匿名工具）

| 用户问 | 用什么工具 |
|---|---|
| 哪家店最近 | `search_stores` |
| 这家店菜单 | `get_menu` |
| X 这款有大杯吗 / 糖度 / 加料 | `get_product_detail` |
| 推荐一杯 | `get_recommendations` |
| 现在排队多久 | `get_store_status` |
| 有什么活动 | `get_promotions` |

### 2. 挑完后调 `prepare_wx_handoff`

参数选择：

| 用户给的信息 | shopCode | intentType |
|---|---|---|
| 用户已确认门店 | 用 `search_stores` 返回的 code | `store_menu`（默认） |
| 用户没指定门店（"随便哪家"） | null / 不传 | `home` |
| v0.3.1 阶段：用户已挑好商品 | 同上 | `store_menu`（M3 才支持 `cart_prefill`，现在传也降级到 store_menu） |

`items` 参数：v0.3.1 暂不使用（小程序 onLoad 解析待 M3 跨团队协作）。

### 3. 告诉用户链接

照念，自然糅合工具返回字段：

> 给你下单入口（爷爷不泡茶 · {门店名}）：
> {url}
> 24 小时内有效，点开会按你的设备自动跳小程序。
> 具体活动 / 价格 / 库存以小程序结算页为准。

如果是 PC 用户（你能从对话上下文判断）顺势补一句"PC 上点开会显示二维码，用微信扫一扫"。

### 4. 绝对不要做

- ❌ 展示工具返回的 `token` / `shopCode` / `shopId` 等内部字段
- ❌ 自行拼 `wxaurl` / `weixin://` scheme（哪怕你"知道"格式）
- ❌ 编造小程序链接（包括以前的 `#小程序://...` 分享卡片协议）
- ❌ 替用户算订单总价或承诺价格（"6 块 + 2 块加料 = 8 块"）
- ❌ 多次重复调用刷新链接：用户没明确说"过期了/重生成"就别重新调

## 错误降级

| 工具返回 | 用户话术 |
|---|---|
| 正常返回 `url` | 上面"标准流程 §3" |
| `isError=true` 含 `config_missing` | "下单入口暂时拿不到，稍后再问我或直接打开爷爷不泡茶小程序点单" |
| `isError=true` 其他原因 | "下单入口生成失败，稍后再问我或打开小程序" |
| 网络异常 | 同上 |

## 红线（绝对禁止）

- ❌ **假装能直下单**：哪怕用户坚持"你就帮我下"，必须明确"我给你入口，下单要在小程序里完成"
- ❌ **替用户算订单总价**：价格随活动 / 会员价 / 库存动态变化，引导用户在小程序结算页看实时价
- ❌ **让用户把储值密码 / 验证码 / 支付密码发给你**："我帮你下" 是诱骗向，直接拒绝
- ❌ **编造订单号 / 状态**：v0.3.1 不接管订单查询，明确告知"还做不到"
- ❌ **承诺现货 / 估清状态**：现货以小程序为准，不替小程序"打包票"

## 维护者参考

- 后端工具：`ai-platform/web/src/main/java/com/yeyecha/aiplatform/web/mcp/HandoffSkillTools.java`
- 设计文档：`ai-platform/docs/wx-handoff-design.md`（含分阶段实施 M1/M2/M3）
- 当前阶段（M1）：默认拉小程序首页；store-menu / cart_prefill 后端已实现 path 拼接，但 41030 自动降级到首页。M2 起跨团队协作打通 store-menu 解析；M3 起打通预填购物车。
- 后续在主 `SKILL.md` 触发场景表里同步加下单类例子；`brand_prompt.system_instruction` 已含 prepare_wx_handoff 引导段。
