# WebHub 接入指南（SDK）

> 本文说明第三方插件接入 WebHub 所需的声明、页面、请求处理与能力配置。
> 本文接口适用于 LeviLamina 26.51 与 LSE QuickJS 0.22.3。

---

## 0. 平台提供的能力范围

| 插件需求 | 平台提供 | 免于实现 |
|---|---|---|
| 网页服务器 | HTTP 服务、静态站点、CORS/安全头 | 监听端口、编写路由框架 |
| 登录鉴权 | 账号、会话、角色、限流、审计；密码 / 本机网络 / 一次性登录码三种凭据 | 密码哈希、Cookie、防撞库 |
| 数据存储 | KV / 集合 / 指标 / 事件 / 小对象 | 建表、写 SQL、做备份 |
| 页面托管 | 沙箱 iframe + `postMessage` 桥 | 处理 Cookie、跨域 |
| 站内信 | 给指定玩家投递通知 | 做消息中心 |
| 管理动作 | 广播 / 踢人 / 发钱 / 改角色 | 写游戏侧接口 |
| 插件动作 | 声明 `meta.actions` 即得写通道（鉴权 / 限流 / 并发闸门 / 审计） | 做鉴权、做审计、做幂等 |
| 概览 banner | 声明 `meta.tiles` 即得一块概览页方块（取数 / 缓存 / 点击跳转 / 按角色过滤） | 写概览页、做卡片布局、做缓存 |

**平台只做转发。** 平台不解释插件的业务数据，不做统计聚合，不生成排行榜；这类处理由插件自行实现。

---

## 1. 三步接入

### 1.1 声明（注册命名空间与菜单）

在 `onServerStarted` 里调用（不要更早，那时平台可能还没加载）：

```js
const meta = {
  ns: "MyShop",                    // 命名空间，全局唯一，建议用插件英文名
  name: "商店",                     // 显示在顶部导航的名字
  version: "1.0.0",
  description: "服务器商店系统",     // 可选
  icon: "cart",                     // 可选，前端图标名
  scopes: ["kv", "rpc", "notify"],  // 声明要用哪些能力（见 §2）
  menus: [
    { path: "shop",    title: "商店",   icon: "cart", role: "player", sort: 10, page: "shop" },
    { path: "orders",  title: "我的订单", icon: "list", role: "player", sort: 20, page: "orders" }
  ]
};

const res = JSON.parse(ll.imports("WebHub", "register")(JSON.stringify(meta)));
if (!res.ok) logger.error("接入失败: " + res.error);
```

**每个插件在顶部导航栏占一个独立入口**，`menus` 里的每一项会在插件页内显示为二级标签页。

| 字段 | 说明 |
|---|---|
| `ns` | 1–48 位，字母/数字/`_`/`-`/`.`；全局唯一 |
| `menus[].path` | 插件内唯一，字母/数字/`_`/`-` |
| `menus[].role` | `player` / `helper` / `admin`，低于此角色看不到也调不通 |
| `menus[].sort` | 数字小的排前面 |
| `menus[].page` | 对应页面名；为空则用 `path` |

### 1.2 提供页面（可选）

页面 HTML 由插件推给平台托管，平台负责鉴权与沙箱。**平台不读取插件的文件系统**，页面须由插件主动推送：

```js
ll.imports("WebHub", "page")("MyShop", "shop", "商店", htmlString);
```

页面里**不能**用 `fetch` 直接调接口（沙箱禁了 `connect-src`），必须用平台注入的桥：

```html
<script>
  // 平台已注入 window.WEBHUB = { ns, call(path, opts), action(name, payload) }
  const items = await WEBHUB.call("items.list", { query: { page: 1 } });
  await WEBHUB.call("items.buy", { body: { id: 42 } });
</script>
```

`WEBHUB.call` 返回 `Promise`，超时 15 秒。失败时 reject 一个带 `code` / `message` 的错误对象。

**沙箱约束**（CSP `sandbox` + `default-src 'none'`）：
- 拿不到宿主 Cookie、localStorage
- 不能加载外部脚本/样式/字体/CDN
- 不能跳转顶层页面
- 只能用内联 `<script>` / `<style>` 和 `data:` 图片

> 若页面需要 WebSocket、外部 CDN 或宿主存储，应将相关逻辑放在插件侧，页面仅用于展示。

#### 页面样式基线

平台在返回页面时会往 `<head>` 里内联注入一套 CSS 变量（与宿主同名同值，跟随玩家当前选的配色）。**插件无需自行定义配色**，直接使用变量即可：

```css
body { background: var(--bg2); color: var(--fg); }
.card { background: var(--bg3); border: 1px solid var(--line); border-radius: var(--radius); }
.muted { color: var(--fg3); }
```

可用变量：`--bg --bg2 --bg3 --line --fg --fg2 --fg3 --accent --accent2 --accent-soft --ok --ok-soft --warn --err --radius --shadow`

其中 `--accent-soft` / `--ok-soft` 是「同色系的浅底」，适合做图标底、徽标底、选中态；
`--radius` 是圆角（14px），`--shadow` 是卡片阴影（浅色主题下更淡）。

**只有 `dark` 是深色主题**：`light / amber / green / blue` 四套都是浅底（白 / 暖黄 / 薄荷 / 天蓝），
`color-scheme` 也随之是 `light`。因此页面**不应**写死深色背景，
也不应假定「平台深色主题等同于玩家系统深色」——此处只依据玩家在顶栏选择的配色。

**务必带兜底值**：`background: var(--bg3, #1d242e)`。这样脱离平台单独打开页面时观感不变，在平台内则自动跟随主题，切换主题无需修改页面。

写死颜色的既有页面不受影响——注入的 `body` 默认值特异性最低，会被页面自身的规则覆盖。

#### 高度自适应（无需插件处理）

桥脚本会用 `ResizeObserver` + 轮询把页面内容高度报给宿主，宿主自动调整 iframe 高度。插件只需正常排版，不要给 `body` 设置 `height:100%` 或 `overflow:hidden`，否则量到的内容高度会失真、页面会被裁切。

### 1.3 处理请求（导出 dispatch）

```js
function dispatch(reqJson) {
  const req = JSON.parse(reqJson);
  // req = { path, method, query, body, account: { id, xuid, username, playerName, role } }
  // 注意：浏览器侧拿不到 xuid —— GET /api/v1/players 只返回玩家名。
  //      需要按玩家定位时，用 RPC 里下发的 account.xuid，或服务端调 getSnapshot()。

  if (req.path === "items.list") {
    const raw = ll.imports("WebHub", "kvGet")("MyShop", "items");
    return JSON.stringify({ items: raw ? JSON.parse(raw) : [] });
  }

  if (req.path === "items.buy") {
    // ... 业务逻辑
    return JSON.stringify({ ok: true, balance: 100 });
  }

  return JSON.stringify({ status: 404, body: { error: "未知路径" } });
}

ll.exports(dispatch, "MyShop", "webhub.dispatch");
```

**返回值约定**（必须是字符串化的合法 JSON）：

| 返回 | 浏览器收到 |
|---|---|
| `{"status":404,"body":{...}}` | HTTP 404 + `{...}` |
| `{"a":1}`（无 status） | HTTP 200 + `{"a":1}` |
| `"不是JSON"` | HTTP 502 `PLUGIN_BAD_RESPONSE` |
| 抛异常 | HTTP 500 `PLUGIN_ERROR` |

### 1.4 声明与处理动作

**查询是只读的，动作是能改游戏数据的。** 平台把两者分开走两条路：`WEBHUB.call` 走
`POST /plugins/{ns}/rpc`，`WEBHUB.action` 走 `POST /plugins/{ns}/actions/{name}`。
动作必须先在 `meta.actions` 里**声明**，没声明的名字平台直接 404，一个都不会转发。

#### 第一步：在 meta 里声明

```js
const meta = {
  ns: "MyShop",
  name: "商店",
  scopes: ["rpc"],
  menus: [ /* ... */ ],
  actions: [
    {
      name: "delist",          // 只能字母/数字/下划线/连字符，1..48 位，同一插件内不能重名
      title: "下架商品",        // 审计日志里显示的名字
      role: "player",          // 最低角色：player / helper / admin，默认 helper
      params: {                // 纯说明，平台不校验，插件自己校验（前端也拿不到它）
        sid: "string",
        itemKey: "string"
      }
    }
  ]
};
```

> `role: "player"` 意味着**任何登录玩家**都能触发这个动作。注册时平台会打一条警告日志，
> 接入方须在 `dispatch` 中完成归属校验。动作数量上限 20 个。

#### 第二步：在 dispatch 里用 `kind` 区分

平台转发给插件的对象额外包含 `kind` 字段，动作请求格式如下：

```json
{ "path": "delist", "kind": "action", "body": { "sid": "s_1", "itemKey": "minecraft:stone" },
  "account": { "id": 7, "xuid": "...", "username": "steve", "playerName": "Steve", "role": "player" } }
```

```js
function dispatch(reqJson) {
  const req = JSON.parse(reqJson);
  const acc = req.account || {};

  if (req.kind === "action") {
    if (req.path === "delist") {
      const shop = getShop(req.body.sid);
      if (!shop) return JSON.stringify({ status: 404, body: { error: "店铺不存在" } });

      // 关键：平台只识别角色，不判断资源归属
      if (shop.xuid !== acc.xuid) {
        return JSON.stringify({ status: 403, body: { error: "无权操作该店铺" } });
      }

      return JSON.stringify(delistItem(shop, req.body.itemKey)); // 成功：原样作为 data 返回
    }
    return JSON.stringify({ status: 404, body: { error: "未声明的动作" } });
  }

  // 查询：和 1.3 一样
  if (req.path === "shop.list") return JSON.stringify(listShops(acc));
  return JSON.stringify({ status: 404, body: { error: "未知路径" } });
}
```

#### 三条红线

1. **资源归属必须由插件自行校验。** 平台只保证「该账号为 player 角色且已登录」。
   凡涉及「下架本人商品」「取消本人悬赏」「退还本人款项」的动作，第一行即应比对
   `account.xuid`（或业务数据中保存的 owner 字段），否则任何登录用户都可操作他人资源。
2. **动作的副作用必须可控。** 平台不提供事务、不回滚、不重试。重复点击与网络重发都可能
   使动作执行两次，应做成幂等（「已下架」直接返回成功），或在动作入口加入一次性令牌。
3. **审计日志里不要出现明文密码。** 平台会把动作参数写进 `audit_log`（超 512 字符截断，
   `password`/`token`/`secret` 等字段自动打码）。其余参数均会被记录，敏感数据不应放入。

#### 平台这边的限制（默认值，`config/webhub.json` 可改）

| 项 | 默认 | 配置项 |
|---|---|---|
| 单 IP 动作频率 | 30 次/分钟 | `limit.pluginActionPerMin` |
| 单插件并发动作 | 2 | `rpc.actionMaxConcurrent` |
| 服务端等待插件返回 | 5 秒 | `rpc.actionTimeoutMs` |
| 请求体大小 | 16 KB | 固定 |
| 必须登录 | 是 | 固定 |

浏览器侧 `WEBHUB.action(name, payload)` 的等待上限是 15 秒（和 `call` 一致）：
第 5 秒平台就会返回 504，插件即使之后再算完，结果也不会传回页面。

#### 页面里怎么用

```js
try {
  const r = await WEBHUB.action("delist", { sid: "s_1", itemKey: "minecraft:stone" });
  toast("已下架");        // r 就是 dispatch 返回的 JSON
  reload();
} catch (e) {
  toast(e.message || "操作失败");   // e.code 可能是 FORBIDDEN / UNKNOWN_ACTION / PLUGIN_TIMEOUT …
}
```

#### 示例

完整示例声明 5 个动作（`echo` / `reject` / `boom` / `slow` / `adminOnly`），
页面中提供可直接点击的按钮，覆盖 200 / 401 / 403 / 404 / 413 / 500 / 504 各条返回路径与审计、统计。
将其放入服务端 `plugins/ActionDemo` 即可使用，不依赖任何业务插件。

### 1.5 在概览页摆一块 banner

概览页上除了平台自己的数字卡，还会显示**插件自己摆的小方块**（和「在线玩家」同尺寸）。
它适合放「一眼就能看完的一个数」：今日成交、待处理订单、在线商店数、赛季剩余天数……

#### 第一步：在 meta 里声明

```js
const meta = {
  ns: "MyShop",
  name: "商店",
  scopes: ["rpc"],
  menus: [{ path: "shop", title: "商店", role: "player", page: "shop" }],
  tiles: [
    // id 在本插件内唯一，也是取数时 dispatch 收到的 path
    { id: "today", title: "今日成交", desc: "今天完成的订单数",
      page: "shop", role: "player", sort: 10 },
    // personal=true：每个账号看到的不一样（比如「我的订单」），平台不做缓存
    { id: "mine", title: "我的订单", page: "shop", role: "player", sort: 20, personal: true }
  ]
};
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `id` | 是 | 1–48 位，字母/数字/`_`/`-`；插件内唯一 |
| `title` | 是 | 方块标题（≤32 字） |
| `desc` | 否 | 鼠标悬停时显示的一行说明 |
| `page` | 否 | 点击后跳到哪个页面（`meta.menus` 里的 `page`）。留空则跳到插件首页 |
| `role` | 否 | 谁能看到，默认 `player` |
| `sort` | 否 | 排序，小的在前，默认 100 |
| `personal` | 否 | `true` = 每账号单独取数、平台不缓存（默认 `false`） |

上限默认 8 块/插件（`config.tiles.maxTiles`）。

#### 第二步：在 dispatch 里返回数据

平台取数时会发 `{ "path": "<banner id>", "kind": "tile", "account": {...} }`：

```js
function dispatch(reqJson) {
  const req = JSON.parse(reqJson);

  if (req.kind === "tile") {
    if (req.path === "today") {
      // 推荐写法：一个主数字 + 一行副标题
      return JSON.stringify({ value: shop.todayOrders, sub: "今日新增" });
    }
    if (req.path === "mine") {
      const who = req.account.xuid;         // personal=true 时按账号返回
      return JSON.stringify({ value: countMyOrders(who), sub: "未完成" });
    }
    return JSON.stringify({ status: 404, body: { error: "未知 banner" } });
  }

  // 查询 / 动作照旧
}
```

**渲染约定**（平台侧，按顺序匹配）：

| 插件返回 | 页面显示 |
|---|---|
| `{ "value": 12, "sub": "今日新增" }` | 主数字 `12` + 副标题「今日新增」 |
| `{ "lines": [{"k":"待发货","v":3}] }` | 第一项当主值，其余拼成副标题 |
| 任意含数字字段的对象 | 取第一个数字字段当主值，其余字段拼副标题 |
| 字符串 / 数字 | 直接当主值 |

> 返回结构不规范时不会出错，仅显示效果较差。建议统一使用 `{ value, sub }`。

#### 平台负责的处理

- **鉴权**：没登录不给取；`role` 不够返回 403；插件未声明这个 id 返回 404 `UNKNOWN_TILE`。
- **缓存**：`personal=false` 的结果按「ns + banner id + 角色」缓存 `config.tiles.ttlMs`（默认 15 秒）。
  概览页是登录后的落地页，十个人同时打开不该打十倍的请求到游戏线程。
- **离线不摆牌**：插件没 `register`（或已被管理员禁用）时，这块牌直接不出现，而不是显示一个红叉。
- **超时**：单块默认 3 秒（`config.tiles.timeoutMs`），超时只影响这一块，其它牌照常显示。

#### 几个容易踩的点

1. **取数在游戏线程**：和 RPC 一样，`kind: "tile"` 的 dispatch 会占用游戏线程，**必须快**。
   需要查询数据库的，应提前缓存；不应在此遍历全服实体。
2. **banner 取数不应包含耗时操作**：概览页每 20 秒刷新一次，全体在线用户均可能触发。
3. **`personal=true` 应谨慎使用**：它绕过缓存，每次刷新都会实际调用插件。
   仅在「不同用户必须看到不同数值」时使用。
4. **`page` 必须存在**：指向未推送的页面将返回 404。建议与 `menus` 中的 `page` 使用同一名称。

---

## 2. 能力（scopes）

**没声明就调不通**，这是硬性鉴权，不是建议。

| scope | 解锁 |
|---|---|
| `kv` | `kvSet` / `kvGet` / `kvDel` |
| `collections` | HTTP 数据面的集合接口 |
| `metrics` | HTTP 数据面的指标接口 |
| `events` | `publish` |
| `blobs` | HTTP 数据面小对象接口 |
| `rpc` | 推送页面 `page`、被平台 RPC 调用 |
| `notify` | `notify` |
| `account` | `getAccount` |
| `snapshot` | `getSnapshot` |

---

## 3. 平台导出函数（`ll.imports("WebHub", ...)`）

> 所有调用都发生在**游戏线程**，必须快。单次建议 < 5ms。

| 函数 | 签名 | 说明 |
|---|---|---|
| `register` | `(metaJson) -> string` | 返回 `{"ok":true}` 或 `{"ok":false,"error":"..."}` |
| `unregister` | `(ns) -> bool` | 注销（页面一并清除） |
| `page` | `(ns, name, title, html) -> bool` | 推送页面，单页 ≤ 512KB |
| `publish` | `(ns, channel, payloadJson) -> bool` | 发事件，负载 ≤ 8KB |
| `kvSet` | `(ns, key, valueJson) -> bool` | 覆盖写（无 TTL、无版本校验） |
| `kvGet` | `(ns, key) -> string` | 不存在返回空串 |
| `kvDel` | `(ns, key) -> bool` | |
| `notify` | `(ns, xuid, title, body) -> bool` | 站内信，标题 ≤ 120 字、正文 ≤ 4000 字 |
| `getAccount` | `(ns, xuid) -> string` | 脱敏账号（不含 salt/hash），需 `account` scope |
| `getSnapshot` | `() -> string` | 平台快照 `{rev,ts,online,players[]}`，每项含 `name/xuid/uuid` |
| `pushChat` | `(xuid, name, text) -> bool` | 把一条**真的发出去**的聊天转发给网页聊天栏。接管聊天流程的插件用，见 3.1 |
| `selfTest` | `() -> string` | 通道自检，有返回即说明接入成功 |
| `lastError` | `() -> string` | 最近一次失败原因（线程局部） |

**调试技巧**：任何调用失败后先调 `lastError()`，它会给出具体原因。

### 3.1 将插件消息同步到网页聊天栏：`pushChat`

**适用场景。** 若插件通过 `mc.listen("onChat", ...)` 接管聊天（称号、聊天格式、美化），
采用「cancel 原版事件 + 自行 `mc.broadcast`」的方式，平台默认**接收不到**聊天，原因在事件总线：

```cpp
// LeviLamina EventBus.cpp
for (auto& l : tempList) {
    l->call(event);
    if (event.mCancelled) { break; }   // 有人 cancel 就停，后面的监听器不再被调用
}
```

监听器保存于 `unordered_set` 中，**遍历顺序不确定**。若插件先行 cancel 事件，
平台的聊天监听器可能完全不会被调用——没有异常、没有日志，网页端也不会有任何消息。

**怎么接。** 在消息真正发出去之后，转发一份原文：

```js
mc.broadcast(finalMsg);
if (ll.hasExported("WebHub", "pushChat")) {
    ll.imports("WebHub", "pushChat")(pl.xuid, pl.realName, msg);   // msg = 玩家输入的原文
}
```

- 传**原文**就够。网页只展示「谁说了什么」，拼好的整行（时间 / 维度 / 前缀）在网页上是噪音。
- **被拦截的消息不应转发**（违禁词、禁言、刷屏）。这也是该方式较「由平台推测 cancel」更可靠的原因：
  插件明确知道哪些消息确实已发出。
- 平台会清洗换行与制表符、按 `chat.maxLength` 截断；名字和内容都当作不可信输入。
- 返回 `false` 时调 `lastError()`：内容为空，或与最近几秒内的同一条重复（平台正在去重）。

**要不要改配置。** 默认 `chat.mode = "both"`：事件与转发都收，重复的按 `chat.dedupSeconds`
（默认 5 秒）去重，因此不修改也不会出现重复。插件已接管聊天时，建议改为 `"forward"`：

```json
"chat": { "mode": "forward" }
```

重启后控制台会打印 `聊天：只听插件转发（mode=forward），事件监听未注册`。

**怎么确认通了。** 登录后看 `GET /api/v1/chat/stats`（`/api/v1/chat` 的 `data.stats` 里也有同一份）：

| 字段 | 含义 |
|---|---|
| `eventSeen` | 平台收到 `PlayerChatEvent` 的次数。装了接管聊天的插件后它应该一直是 0 |
| `forwarded` | `pushChat` 成功入列的条数 |
| `deduped` | 两条入口撞在一起被去重的条数（正常现象，不是错误） |
| `forwardRejected` | 内容清洗后为空被拒的次数 |
| `webSent` | 网页端发言的条数 |

`forwarded` 在增长、`lines` 有数值，即表示通道正常。若两者均为 0，说明插件未执行转发（未调用 `pushChat` 或调用失败）。

---

## 4. HTTP 数据面

前缀 `/api/v1/data/{ns}/...`，需要 **admin** 角色。

> 为什么是 admin：HTTP 数据面是给**运维、调试、外部系统集成**用的。
> 插件的读写走 RemoteCall（游戏线程，快且无需鉴权）。
> 玩家浏览器通过 RPC 间接读写，越权请求必须由插件的 `dispatch` 拦截。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/kv/{key}` | 读，含 `ver` / `expiresAt` |
| PUT | `/kv/{key}?ttl=&ver=` | 写；`ver=0` 必须不存在，`ver>0` 乐观锁 |
| DELETE | `/kv/{key}?ver=` | 删 |
| GET | `/kv?prefix=&limit=` | 前缀列举 |
| POST | `/kv/batch-get` | `{"keys":[...]}` 批量读，≤100 |
| GET | `/collections` | 集合列表 |
| POST | `/collections` | `{"name","meta"}` 建集合 |
| DELETE | `/collections/{coll}` | 删集合及其记录 |
| GET | `/collections/{coll}/records?limit=&offset=&order=&desc=` | 分页，limit ≤ 200 |
| POST | `/collections/{coll}/records` | `{"id?","data"}`，不传 id 则自动生成 |
| GET/PUT/DELETE | `/collections/{coll}/records/{id}` | 单条 |
| GET | `/metrics/{series}?from=&to=&step=&limit=` | `step` > 0 时按桶取平均 |
| POST | `/metrics/{series}` | `[{ts,value}]`，单次 ≤ 1000 点 |
| DELETE | `/metrics/{series}` | 删系列 |
| GET | `/events?since=&channels=&limit=` | 游标拉取，返回 `nextSince` |
| GET | `/blobs` / POST `/blobs` / GET `/blobs/{hash}` | 小对象 |

---

## 5. 错误码

| code | HTTP | 含义 | 处理建议 |
|---|---|---|---|
| `AUTH_REQUIRED` | 401 | 未登录/会话过期 | 跳登录 |
| `AUTH_FAILED` | 401 | 用户名或密码错 | 提示 |
| `NO_PASSWORD` | 401 | 该账号没有密码（注册时留空） | 引导用本机网络 / 登录码 |
| `METHOD_NOT_ALLOWED` | 403 | 该登录方式未开启，或免密只允许普通玩家 | 改密码登录 |
| `TRUST_MISS` | 401 | 本机 IP 不在信任记录内 / 登录码无效 | 改用密码 |
| `ACCOUNT_LOCKED` | 423 | 失败太多次，锁定中 | 显示倒计时 |
| `FORBIDDEN` | 403 | 角色不够 | 隐藏入口 |
| `RATE_LIMITED` | 429 | 触发限流 | 看 `Retry-After`，退避重试 |
| `QUOTA_EXCEEDED` | 507 | 超出配额 | 清理旧数据 |
| `VERSION_CONFLICT` | 409 | 乐观锁版本不匹配 | 重新读再写 |
| `PLUGIN_OFFLINE` | 503 | 插件没 register 或已禁用 | 检查加载顺序 |
| `PLUGIN_TIMEOUT` | 504 | 插件 3 秒未返回 | 优化 dispatch |
| `TOO_MANY_CONCURRENT` | 429 | 单插件并发 RPC 超限（默认 4） | 减少并发 |
| `PLUGIN_BAD_RESPONSE` | 502 | 返回值不是 JSON | 检查 `dispatch` |
| `UNKNOWN_ACTION` | 404 | 动作没在 `meta.actions` 里声明 | 补声明或改名字 |
| `PLUGIN_REJECTED` | 4xx | 插件用 `status` 拒绝了这次动作 | 看 `message` |
| `PLUGIN_ERROR` | 5xx | 插件自己出错了（状态码 500 或直接抛异常） | 查服务器日志 |
| `NOT_INTEGRATED` | 501 | 动作依赖的模组未安装 | 提示管理员 |

---

## 6. 配额（默认值，`config/webhub.json` 可改）

| 资源 | 上限 |
|---|---|
| KV 键数 | 10 000 / ns |
| 集合数 | 20 / ns |
| 单集合记录 | 50 000 |
| 指标系列 | 50 / ns |
| 事件条数 | 5 000 / ns（保留 7 天） |
| 指标保留 | 30 天 |
| 单个小对象 | 512 KB |
| 小对象总量 | 100 MB |
| 单页 HTML | 512 KB |
| 事件负载 | 8 KB |
| RPC 请求体 | 32 KB |
| 动作请求体 | 16 KB |

---

## 7. 性能红线

**HTTP 请求与游戏线程完全隔离**，但 **RPC 会占用游戏线程**。因此：

1. **`dispatch` 中禁止耗时操作**。遍历上万条记录、同步网络请求、大文件读写，均应异步化（写入队列，下一 tick 处理）。
2. **`publish` / `kvSet` 是单条 SQL**，可直接调用；但不应在循环中调用数千次。
3. **慢调用会被记日志**：超过 `rpc.slowWarnMs`（默认 50ms）平台会警告，说明该调用拖慢了服务器。
4. **RPC 有并发闸门**（默认每插件 4 个并发），超出后直接返回 429。应使单次 RPC 足够快，而不是依赖并发。
5. **动作另有闸门**（默认每插件 2 个并发、单 IP 30 次/分钟）。动作会写数据，耗时操作会使玩家在页面上等待——应将耗时工作拆分为「先记录、下一 tick 执行」，不应在单次动作中完成全部工作。

---

## 8. 完整示例

一个完整的最小接入需要：注册两个菜单、两个页面、KV 计数器、事件发布、站内信，下面各节的代码片段可以组合成它。

---

## 9. 常见问题

**Q：插件加载顺序不确定，WebHub 晚于本插件加载时应如何处理？**
A：`register` 返回失败时可用 `setInterval` 重试。示例代码中包含完整写法。

**Q：刷新服务器后菜单消失？**
A：`online` 状态只存在于内存中，重启后必须重新 `register`。数据库记录仅用于展示历史，不授予任何权限——用于防止其他插件冒用该命名空间。

**Q：能不能不写页面，只用数据接口？**
A：可以。`menus` 可以留空数组，但那样插件不会出现在导航里（纯后台插件）。

**Q：页面里怎么拿到当前登录玩家？**
A：`dispatch` 的 `req.account` 就是。页面侧不需要也不应该知道账号信息，由插件决定给什么。
