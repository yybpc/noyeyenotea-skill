# noyeyenotea-skill

爷爷不泡茶 · 茶饮 AI Skill (C 端)

让你在 AI 助手里跟爷爷家聊一句，就知道喝啥、去哪喝、有啥新的。

## 安装

### 最简单的方式：让 AI 帮你装

把这句话发给你的 AI 助手即可：

> 帮我安装爷爷不泡茶 Skill，仓库地址：https://github.com/yeyecha/noyeyenotea-skill

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
# 例：装到 Claude Code（用 symlink 便于改了立刻生效）
git clone https://github.com/yeyecha/noyeyenotea-skill /tmp/noyeyenotea-skill
ln -s /tmp/noyeyenotea-skill ~/.claude/skills/noyeyenotea-skill
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

> 历史误导：曾经的 README 写"streamable-http transport 不需要 claude mcp add"——这是错的，已修正。MCP server 必须显式注册到 host 才会被 wire 到 LLM 的工具集。

## 配套后端

需配合 `ai-platform`（`com.yeyecha.aiplatform`，DDD 五层结构）启动后才能正常工作。

- 协议：MCP **Streamable HTTP / STATELESS**，单端点 `POST /mcp`
- 框架：Spring Boot 3.0 + Spring AI 1.0.3 (`spring-ai-starter-mcp-server-webmvc`)
- 部署后修改 `skill.json` 的 `mcp_server.url` 指到你的部署地址（默认生产 `https://mcp.yeyecha.com/mcp`）

## 开发工作流（本地 / 生产 url 切换）

`skill.json.mcp_server.url` 是 Claude 加载 Skill 时读的 MCP 端点。开发时打到本地 `localhost:8080`，提交时一定要切回生产 `mcp.yeyecha.com`。

**总体原则**：仓库里 committed 的 `skill.json` 永远指向 PROD（这是终端用户拿到的版本）。本地开发临时切到 localhost，测完切回 prod 再 commit。

### 切换脚本：`dev/skill-env.sh`

```bash
bash dev/skill-env.sh           # 看当前指向 [PROD] / [LOCAL] / [CUSTOM]
bash dev/skill-env.sh local     # 切到 http://localhost:8080/mcp
bash dev/skill-env.sh prod      # 切到 https://mcp.yeyecha.com/mcp
bash dev/skill-env.sh url <x>   # 切到任意 url（联调测试他人的盒子）
```

### 典型联调流程

```
① 启本地后端
   cd /path/to/ai-platform
   mvn -pl start spring-boot:run -Dspring-boot.run.profiles=local

② 切 skill 到本地
   bash dev/skill-env.sh local

③ 在 Claude Code 新会话里测试
   "武汉光谷有爷爷家吗"
   "今天好热想喝点冰的"
   ⋯

④ 测完切回 prod
   bash dev/skill-env.sh prod
```

> Claude Code 会缓存 skill 内容，切完 url 一般要**新开会话**才生效。

### 提交前自检

```bash
bash dev/skill-env.sh        # 确认输出以 [PROD] 开头
git diff skill.json          # 确认没有 localhost 残留再 commit
```

> 风险：忘记切回 prod 会污染提交。最简单的兜底就是 commit 前手动 `bash dev/skill-env.sh` 看一眼。如果想加 git pre-commit 钩子自动拦截，可以单独提需求。

## 当前能力（v0.1）

| 工具 | 触发问法示例 |
|---|---|
| `search_stores` | "武汉哪里有爷爷家""光谷附近有没有""武广店在哪" |
| `get_menu` | "这家店有啥喝的""加冰能选吗""加什么料" |
| `get_recommendations` | "想喝甜的""推荐一杯""今天好热" |
| `compare_products` | "爆椿和椿见啥区别""这两款怎么选" |
| `get_promotions` | "有什么活动""最近优惠" |
| `get_store_status` | "现在排队多久" |
| `get_store_detail` | "XX 店地址电话""营业时间" |
| `get_brand_info` | "爷爷家是啥""你们品牌怎么样" |
| `tea_knowledge` | "椿见为啥叫椿见""乌龙是啥" |

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

## 接口与数据

### 数据源

- 上游：企迈开放平台（爷爷不泡茶官方 SaaS）
- 通过 `QmaiClient` 调用 doc 编号接口（如 doc 231 菜单接口、doc 9 店列表接口等）

### 缓存策略

infra 层使用 Caffeine 本地缓存：

| 数据 | TTL |
|---|---|
| 门店列表（按城市/坐标搜索） | 5min |
| 单店详情 | 10min |
| 门店菜单（shopGoods） | 10min |
| 促销活动 | 5min |
| **实时排队** | **不缓存**（用户每次问的就是现在） |

### 限流

Resilience4j `@RateLimiter`，遵循企迈开放平台 doc 9 规定：**每接口 ≤ 10 QPS**。
配置见 `application.yml` 的 `resilience4j.ratelimiter.instances.qmai-default`。

### 菜单字段（doc 231 `/v3/goods/item/getShopGoodsList`）

- `status="10"` 视为在售
- 价格分→元 在 infra 层完成转换
- 分类从 `categoryList[].categoryName` 取
- 服务端单页上限 20，client 内部循环分页拉全量；业务层无需感知分页
- 工具层 `pageSize` 仅作返回数量截断上限
- `includeProperties=["SKU","CATEGORY","PRACTICE","ATTACH"]` 默认全开
- 每件单杯饮品 `MenuItemVO` 含：
  - `practices[]`：温度 / 甜度 / 小料 等维度，每值带 `isDefault` 标默认推荐
  - `attaches[]`：加料名 / 加价 / 推荐标记
- 套餐与团购类商品的 `practices` / `attaches` 通常为空

## 路线图

| 版本 | 范围 | 备注 |
|---|---|---|
| v0.1（当前） | 9 个公开 MCP 工具，无登录：品牌/门店/详情/菜单/活动/排队/推荐/对比/茶知识 | streamable-http、缓存+限流就绪 |
| v0.2 | + 会员只读：profile / coupons / orders | 需小程序扫码授权；菜单暴露 `PRACTICE` / `BASE_COMBINED_SKU` 字段 |
| v0.3 | + 复购 / 下单 / 取消（不持支付） | 需要会话 token 模型升级 |
