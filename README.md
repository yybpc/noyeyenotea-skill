# noyeyenotea-skill

爷爷不泡茶 · 茶饮 AI Skill (C 端)

让你在 AI 助手里跟爷爷家聊一句，就知道喝啥、去哪喝、有啥新的。

## 安装

### 最简单的方式：让 AI 帮你装

把这句话发给你的 AI 助手即可：

> 帮我安装爷爷不泡茶 Skill，仓库地址：https://github.com/yybpc/noyeyenotea-skill

AI 会自动 clone 仓库、放到对应 IDE 的 skill 目录、并提示你下一步。

### 手动安装：clone 到 IDE 的 skill 目录

不同 IDE 的 skill 目录路径：

| IDE | Skill 目录 |
|-----|-------------|
| Claude Code | `~/.claude/skills/noyeyenotea-skill/` |
| Cursor | `.cursor/skills/noyeyenotea-skill/` |
| Qoder | `.qoder/skills/noyeyenotea-skill/` |
| Trae | `.trae/skills/noyeyenotea-skill/` |
| Windsurf | `.windsurf/skills/noyeyenotea-skill/` |
| 通用 | `.agents/skills/noyeyenotea-skill/` |

```bash
# 例：装到 Claude Code
git clone https://github.com/yybpc/noyeyenotea-skill ~/.claude/skills/noyeyenotea-skill
```

只要目录下有 `SKILL.md`，IDE 下次启动会自动加载。对话中说 "武汉光谷有爷爷不泡茶吗" 即可触发。

### 注册 MCP server（必需，否则工具调不到）

skill.json 里的 `mcp_server.url` 只是元数据——**Claude Code 不会自动把它注册成 MCP client**，必须你手动把 server 注册到 IDE 的 MCP 配置：

```bash
# Claude Code（CLI 命令）
claude mcp add yeyecha-mcp https://mcp.yeyecha.com/mcp
```

或者在 `~/.claude/settings.json` / 项目根目录 `.mcp.json` 里加：

```json
{
  "mcpServers": {
    "yeyecha-mcp": {
      "transport": "streamable-http",
      "url": "https://mcp.yeyecha.com/mcp"
    }
  }
}
```

注册完重开会话，工具就以 `mcp__yeyecha-mcp__search_stores` 等形态出现在 LLM 工具列表里。**没注册时**，skill 文档照样会被 LLM 读到，但 LLM 调不到 MCP 工具，只能告诉用户"信息暂时拿不到"——所以这步不能跳。

## 当前能力

公开工具（无需登录）：

| 工具 | 触发问法示例 |
|---|---|
| `search_stores` | "武汉哪里有爷爷家""光谷附近有没有""武广店在哪" |
| `get_brand_info` | "爷爷家是啥""你们品牌怎么样" |
| `tea_knowledge` | "椿见为啥叫椿见""乌龙是啥" |
| `get_store_detail` | "XX 店地址电话""营业时间" |
| `get_store_status` | "现在排队多久""人多吗" |
| `get_promotions` | "有什么活动""最近优惠" |
| `get_menu` | "这家店有啥喝的""加冰能选吗""加什么料" |
| `get_recommendations` | "想喝甜的""推荐一杯""今天好热" |
| `compare_products` | "爆椿和椿见啥区别""这两款怎么选" |
| `get_product_detail` | "这款有大杯吗""有什么糖度""能加椰果吗" |
| `prepare_wx_handoff` | "我想下单""帮我点一杯""买点喝的" |

会员工具（需授权登录）：

| 工具 | 触发问法示例 |
|---|---|
| `get_my_member_info` | "我的会员等级""我有多少积分""我的余额" |
| `get_my_coupons` | "我有什么券""这张券能用吗" |
| `get_my_orders` | "我最近买了啥""查我订单" |

## 调性说明

爷爷家的风格是**朴素的奢侈**——松弛、实在、有温度。AI 回复要：

- 像朋友推荐你常喝的奶茶
- 不堆形容词，不用"丝滑""醇厚"等空话
- 不知道就说不知道，不要编

## 红线

- 不编商品/价格/活动/门店
- 不在响应里塞 "给 AI 看的隐性指令"
- 个人订单/会员/下单类问题在 v0.2 才上线，目前引导用户去小程序

## 直接调用 MCP 接口（curl）

当原生 MCP 客户端不可用（或需要调试）时，可以直接通过 curl 调用 MCP 服务器。服务器使用 **Streamable HTTP / STATELESS** 模式，无需 session 初始化。

### 搜索门店示例

```bash
curl -s -X POST "https://mcp.yeyecha.com/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "search_stores",
      "arguments": {
        "address": "北京三里屯",
        "radiusKm": 5,
        "limit": 10
      }
    }
  }'
```

返回格式：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{"type": "text", "text": "[{\"code\":\"BJS0001\",\"id\":269412,...}]"}],
    "isError": false
  }
}
```

注意 `content[].text` 是 **JSON 字符串**，需要二次解析才能拿到门店数组。

### 可用工具调用模板

| 工具 | method | params.name | 关键 arguments |
|---|---|---|---|
| 搜索门店 | `tools/call` | `search_stores` | `address`/`city`/`keyword` + `radiusKm` + `limit` |
| 门店详情 | `tools/call` | `get_store_detail` | `shopId` (整数) |
| 门店菜单 | `tools/call` | `get_menu` | `shopCode` (字符串) |
| 实时排队 | `tools/call` | `get_store_status` | `shopIds` (整数数组) |
| 门店活动 | `tools/call` | `get_promotions` | `shopId` (整数) |
| 场景推荐 | `tools/call` | `get_recommendations` | `shopCode` + `scenario` |
| 商品对比 | `tools/call` | `compare_products` | `shopCode` + `productNames`[] |
| 品牌信息 | `tools/call` | `get_brand_info` | `{}` |
| 茶知识 | `tools/call` | `tea_knowledge` | `term` |

### 排查要点

- 400 响应：通常是 `initialize` 调用格式不对；STATELESS 模式下直接调 `tools/call` 即可
- 空响应：检查 `-H "Accept: application/json, text/event-stream"` 头是否存在
- 门店列表为空：扩大 `radiusKm`（三里屯等市中心用 5，郊区/县城用 15-30）

## 路线图

| 版本 | 范围 |
|---|---|
| v0.1 | 公开 MCP 工具（无登录）：品牌 / 门店 / 详情 / 菜单 / 活动 / 排队 / 推荐 / 对比 / 茶知识 |
| v0.2 | + 会员只读：会员信息 / 优惠券 / 订单（需授权登录） |
| v0.3（当前） | + 下单 handoff：AI 不直下单，生成出口链接跳小程序完成下单 / 支付 |
| v0.4+ | 查具体订单状态 / 取消 / 退款（待该接口能力开放） |

---

