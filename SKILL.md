---
name: wecom-app-push
description: 企业微信自建应用的两个方向：① 给员工推送应用消息（审批待办、业务提醒、回执催办）；② 接收用户消息（设置 API 接收 / 回调 / 加解密 / 让同事在企微里直接问系统数据）。含可信域名校验文件部署、userid 的两种写法、消息加解密、Nginx 静态放行与 /v3/ 前缀隔离调试等非直觉步骤。当用户说"审批要能推到企业微信""企微里收不到消息""怎么给员工发应用消息""在企微里发消息系统没反应""API 接收怎么配"，或要新建/排查企业微信自建应用的收发通道时使用。
agent_created: true
---

# 企业微信自建应用 · 应用消息推送接入

适用：把自建的 Web 系统（ERP / 工单 / 审批流）的通知推到企业微信个人。
不适用：群机器人 webhook（只能发群，发不了个人）、企微原生审批流（需应用开通「审批接口」）。

## 三条必须知道的硬事实（违背会白折腾）

1. **`message/send` 不受「企业可信IP」限制，通讯录接口才受限制。**
   未配可信IP时：`user/list`、`department/list` 返回 `errcode 60020 not allow to access from your ip`，
   但 `message/send` 直接 `errcode 0` 成功。
   → **推送功能不要等 IP 白名单，可以立刻做。**

2. **🔴 同一个人在企微里有「两个 userid」，而且字符串不相等。**
   系统里的 `zhaohj` 这类自定账号在企微报 `errcode 81013 user & party & tag all invalid`。

   | 写法 | 长什么样 | 谁在用 |
   |---|---|---|
   | **成员账号**（明文） | `ZhangSan`、`LiSi`、`WangWu` | **`消息回调` 里 `FromUserName` 给的就是这个** |
   | **加密形式**（open_userid） | `wmFR5kPJzS9oC43h2Bvn7XqWx0eYdAgH`（示例值） | 某些接口 / 第三方视角返回的是这个 |

   ⚠️ **两种都可用于 `message/send`（实测都 `errcode 0`），所以"能发出消息"不能证明你存对了 ID。**
   一旦用错形式去做**入站匹配**（拿回调值去等值匹配 `bind`），症状是
   **"用户发了消息，系统毫无反应、也不报错"** —— 极难排查。

   → **取法与互转**：拿任意一种去调 `GET /cgi-bin/user/get?userid=<任意一种>`，
   返回里的 **`userid` 字段就是规范（明文）形式**，`name` 是姓名。
   ```bash
   curl -s "https://qyapi.weixin.qq.com/cgi-bin/user/get?access_token=$T&userid=ZhangSan"
   # -> {"errcode":0,"errmsg":"ok","userid":"ZhangSan","name":"张三",...}
   ```
   （`user/get` 单人可用；`user/list` 全量要通讯录权限，常见 `60011 no privilege`。）

   → **存哪一种？存明文形式。** 因为回调给的是它，入库后两边天然对齐。
   若担心其他路径给加密形式，**两种都登记成别名**（见下面「接收消息」一节）。
   → 拿到后**按 ERP 账号建映射表**（`bind`），不要指望两者同名。

3. **应用消息只能发给「可见范围」内的成员。**
   后台「应用详情 → 可见范围」没加的人，消息静默失败。
   这是最常被漏的一步。

## 前置：找用户要三样

| 要的东西 | 后台位置 |
|---|---|
| 企业ID（CorpID） | 我的企业 → 企业信息 → 页面最下方 |
| Secret | 应用详情页 → Secret → 点「查看」 |
| AgentId | 应用详情页（数字，如 1000009） |

顺带确认：**应用主页** 填 `https://你的域名`；**可信域名** 配你的域名（要校验文件，见下）。

## 可信域名校验：Nginx 反代站点的坑

企微会访问 `https://你的域名/WW_verify_xxxxx.txt` 校验，要求**能取到内容且不能跳转**。

但自建系统的 Nginx 通常把 `location /` 全量 proxy 给后端 → 校验文件永远 404。

**修法**：在 `location /` **之前**加一条静态放行（正则优先级高于前缀匹配）：

```nginx
location ~* ^/[A-Za-z0-9_-]+\.txt$ {
    root /www/wwwroot/你的站点目录;
    default_type text/plain;
}
```

然后把企微后台下载的 `WW_verify_*.txt` 放到该目录，`chown www:www` + `chmod 644`。

```bash
# 必须从外部视角验，且确认无跳转
curl -sI https://你的域名/WW_verify_xxxxx.txt | grep -Ei 'HTTP/|content-type|location'
# 期望：HTTP/2 200、content-type: text/plain、没有 location 头
```

> 校验文件放在**独立目录**（如 `/opt/应用名/wecom-verify/`）备一份，防误删。
> 域名不必单独备案——子域已被主域备案覆盖。

## 凭证存放

放**应用目录外或 .gitignore 排除**的单文件，别写进代码：

```
/opt/你的应用/.wecom.json     chmod 600
{ "corpid": "ww…", "secret": "…", "agentid": 1000009,
  "base": "https://你的域名", "remind_hours": 4,
  "bind": { "系统账号": "企微userid" } }
```

环境变量优先级高于文件，便于容器化。**`.gitignore` 必须加 `.wecom.json`**。

## 代码骨架（Node.js）

`wecom.js` 的核心要点（完整实现见 `erp/wecom.js`）：

- **token 缓存**：`expires_in` 7200，本地存到 `expires_in - 300` 秒，避免频繁 gettoken（有日调用上限）。
- **token 过期自动重试**：`errcode` 为 `42001` / `40014` 时强制刷新后重发一次。
- **所有 notify 函数不抛异常**：返回 `{ok, msg}`，调用点用 `.catch()` 兜底，**绝不能让推送失败拖垮业务响应**。
- **消息用 `textcard`** 比纯文本好看得多：

```js
{ touser, msgtype: 'textcard', agentid,
  textcard: { title, description, url, btntxt: '去审批' } }
// description 支持 <div class="gray"> / <div class="normal"> / <div class="highlight">
// url 必须属于已配好的可信域名
```

- **挂载点要覆盖全部入口**：内网 UI 路由、对外 API 路由、以及"直接以已提审状态创建"的分支。
  只挂 `/submit` 一个口子，UI 里勾选"保存并提交"的单据就不会推。
- **防重复**：推送前查日志表 60 秒内是否已有同单据记录，挡双击。
- **超时催办**：定时器（如 30 分钟）扫超时未处理的单据，靠日志表做幂等。

## 多人可批：一个单据要通知多个审批人

审批人通常不是一个人。用**角色**而不是人名来定审批人，一条单据可以对应多个角色的所有人：

```js
const APPROVAL_RULES = {
  quote:          { roles: ['sales_approver', 'admin'], cc: [],        label: '报价单' },
  purchase_order: { roles: ['purchase_approver'],       cc: ['admin'], label: '采购订单' },
  outbound:       { roles: [],                          cc: [],        label: '出库单' },  // 登记即放行
};
// roles 为空数组 = 该单据无需审批，不推
```

推送时**逐个**发给全部审批人（不要只发第一个），文案里注明「谁先批谁生效」。
抄送按角色展开后，**要剔除已经在审批人名单里的人**，否则同一个人收两条。

三个配套的非直觉点：

1. **审批必须用条件更新挡并发**，否则两人同时点「通过」会双批：

```sql
UPDATE 表 SET status=?, approver=?, approve_time=? WHERE id=? AND status='待审批'
```
`changes === 0` 说明被别人抢先了，回 **409** 并明确告诉他是谁处理的
（「该单据已是「已批准」状态（张三 已处理）」）。只做 `if (doc.status !== '待审批')` 的
读-判-写挡不住并发。

2. **催办幂等键必须是「(单据, 审批人)」而不是「单据」**，
   否则第一个审批人收到催办后，其余审批人永远收不到。

3. **撤掉某个角色后要能收敛**：把离职同事的角色从 `roles` 里拿掉即可，
   不需要改代码逻辑。

## 🔴 催办扫描的致命坑：SQL 里写死了不是每张表都有的列

各单据表的往来单位字段名不同（销售侧 `customer`、采购侧 `supplier`）。
如果在扫描 SQL 里同时写这两列：

```sql
-- ✗ quotes 表没有 supplier 列 -> no such column: supplier
SELECT id, no, submitter, submit_time, total, customer, supplier FROM quotes ...
```

**后果不是报个错就算了**：它会让整段循环在第一张表就抛异常、`for` 中断，
于是「超时催办」这个功能**静默失效**，日志里只有一行不起眼的异常，
不专门去看根本发现不了。

正确做法三条一起上：

```js
for (const [type, t] of Object.entries(TABLE)) {
  let rows = [];
  try {
    rows = db.prepare(`SELECT * FROM ${t[0]} WHERE ...`).all();   // ① SELECT *
  } catch (e) {
    console.error(`[wecom] 扫描 ${type} 失败：`, e.message);
    continue;                                                     // ② 单表坏了不拖垮其它表
  }
  for (const d of rows) {
    const party = d.customer || d.supplier || '';                 // ③ 在 JS 里取
    ...
  }
}
```

**并且把扫描逻辑从推送逻辑里拆出来**（如 `overdueCandidates(hours)`，
返回纯数据、不发消息、不需要凭证）：

```js
module.exports = { ..., overdueCandidates, remindOverdue };
```

这样在没有企微凭证的开发机上也能直接测扫描是否命中，
不会再出现「功能坏了几个月没人知道」。

## 接收消息（回调）接入：让同事能在企微里直接问系统

推送是「出站」，这一节是「入站」。后台位置：**应用详情 → 设置 API 接收**（不是"用户消息"页）。

### 后台要填三样 + 勾一个

| 项 | 值 |
|---|---|
| URL | `https://你的域名/回调路径` |
| Token | 自己生成（16~32 位随机串） |
| EncodingAESKey | 43 位，**生成后必须重试到纯 `[A-Za-z0-9]`**（实测首次就带 `+` `/`，试了 10 次才拿到纯字母数字） |

⚠️ **「用户发送的普通消息」（或"接收消息"）必须勾选** —— 不勾，人在企微打的字根本推不出来。
后台默认还会勾一堆无关事件（审批状态通知 / 直播 / 外部联系人 / 会议 / 微信客服 / 支付退款 / 上下游），
**URL 从未配置过时全部去掉，无副作用**。

⚠️ 页面对域名有要求：**域名主体与企业主体相同或有关联关系**。
用同企业已备案的子域（如 `erp.example.cn`）即可满足；**这是现场唯一可能卡住的报错**。

⚠️ **点「保存」时企微会立刻发一次 GET 验证请求** ——
所以**顺序必须是：先把你的服务跑起来并自测外网可达，最后才让他去点保存**。

### 加解密四条，错一条就永远收不到

```js
// ① 明文结构：random(16B) + msgLen(4B 网络序大端) + msg + receiveId
const msgLen = plain.readUInt32BE(16);
const msg    = plain.slice(20, 20 + msgLen).toString('utf8');

// ② AES-256-CBC：密钥 = Base64(EncodingAESKey + "=")，IV = 密钥前 16 字节
const key = Buffer.from(encodingAesKey + '=', 'base64');   // 必须 32 字节
crypto.createDecipheriv('aes-256-cbc', key, key.slice(0, 16));

// ③ 🔴 PKCS#7 补齐块大小是 32，不是 16！（这条最坑）
const padLen = 32 - (buf.length % 32) || 32;

// ④ receiveId 是**企业 corpid**，不是 agentid
```

- **签名**：`SHA1(token/timestamp/nonce/encrypt 四值按字典序排序后直接拼接)`，
  比较用 `crypto.timingSafeEqual`。
- **URL 验证**（GET）：验签 → 解密 `echostr` → **把明文原样返回**（不加密、不加引号、`text/plain`）。
- 🔴 **POST 必须"立刻"回 `success`**：企微 5 秒内收不到响应就重推，
  而 AI 一次要 1~3 秒 → **先 `res.type('text/plain').send('success')`，再异步处理，回复走主动发送**。
  等 AI 出结果再回，迟早超时 + 消息重复。

### 入站身份识别：把两种 userid 都登记成别名

回调只给一个 `FromUserName`。**别赌它是明文还是加密形式** —— 建一张别名表，两种都认：

```js
const ALIAS = new Map();                       // 企微 userid（任意写法）-> 系统账号
async function buildAlias() {
  for (const [username, wid] of Object.entries(cfg().bind)) ALIAS.set(wid, username);
  for (const [username, wid] of Object.entries(cfg().bind)) {
    const g = await getUser(wid);              // /cgi-bin/user/get
    if (g && g.userid) ALIAS.set(g.userid, username);   // 规范形式也登记
  }
}
```

**查不到人时，调 `user/get` 反查姓名写进日志 —— 但绝不据此授权。**
否则日志里只有一句「(未绑定)」，下次还得从头查一遍。

### 隔离调试：不打扰正在用旧版的同事

线上并排起一个**完全独立的新实例**，靠 nginx 前缀分流：

```nginx
location ^~ /v3/ {                          # ^~ 优先级高于 location /，只截 /v3/
    proxy_pass http://127.0.0.1:3002/;      # ⚠️ 结尾斜杠会剥掉 /v3/ 前缀，漏了路径会双份
}
```

- 独立 **pm2 名 / 端口 / 目录 / 数据库文件**，四样都要分。
- 数据库用 `sqlite3 生产库 ".backup '副本路径'"` 做**一致性快照**，
  **绝不软链、绝不 `cp` 运行中的库**（WAL 会不一致）。验一句 `PRAGMA integrity_check;`。
- ⚠️ **快照会随时间变旧** —— 调试问答会答出"过去某一刻"的数据。
  出现"答案和系统里看到的不一样"时，先想这件事，刷新快照即可。

### 🔴 用户在企微发消息没反应 → 排错第一步

**先看 nginx 访问日志，确认请求到底有没有到你的服务器。** 一眼就能分成两类：

```bash
grep '你的回调路径' /www/wwwlogs/你的域名.log | tail -20
```

| 日志里看到什么 | 说明 | 往哪查 |
|---|---|---|
| **什么都没有** | 企微没打过来 | 后台 URL 没保存成功 / 域名主体不符 / 路径写错 |
| 有 GET 且 **200**，POST 且 **200**（body 7 字节 = `success`） | **入站链路是通的**，问题在你自己代码里 | 看应用日志 + 审计表 |
| GET 403 | 签名不符 | Token 填错、或参数顺序/拼接方式不对 |

**第二站：看应用日志里有没有 `收到消息 from=xxx`。**
有 → 说明解密成功，问题在「识别是谁」或「AI」环节；
没有 → 加解密或签名环节的问题。

**第三站：查审计表**（把每条消息的 `intent / ok / errmsg / ms` 都落库，
别只打日志 —— 日志会滚掉，表不会）：

```sql
SELECT id, from_user, from_name, content, intent, ok, reply, errmsg, ms, ts
FROM ai_chat_logs ORDER BY id DESC LIMIT 10;
```

`from_name = '(未绑定)'` 就是本文档开头那个「两种 userid」坑。

> **本项目的真实教训（2026-09-17）**：URL 保存成功、请求到达、回执 200，
> 但用户**一条回复都没收到**。原因是 `.wecom.json` 的 `bind` 里存的是加密形式
> `wmFR5kPJzS9oC43h2Bvn7XqWx0eYdAgH`，而回调给的是明文 `ZhangSan` → 等值匹配失败 →
> 静默判定"不是我们的人"。
> **推送一直正常，所以这个错一直没暴露。**

## 通知不到人时的自查顺序

1. 这个人有没有绑 `bind` 映射？`users.wecom_id` 是不是空？
2. 在不在应用「可见范围」里？
3. 他对应的角色有没有被写进 `APPROVAL_RULES[type].roles`？
4. 看 `wecom_logs` 里有没有 `ok=0` 的记录，`errcode` 是多少。

## 验证

登录后台用 `GET /cgi-bin/gettoken` 确认凭证；再直接发一条：

```bash
T=$(curl -s "https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=$CID&corpsecret=$SEC" \
    | sed -E 's/.*"access_token":"([^"]+)".*/\1/')
curl -s -X POST "https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=$T" \
  -H 'Content-Type: application/json' \
  -d "{\"touser\":\"$USERID\",\"msgtype\":\"text\",\"agentid\":$AID,\"text\":{\"content\":\"测试\"}}"
```

`errcode 0` 即通。常见错误码：
- `60020` → IP 未加白（只影响通讯录类接口）
- `81013` → userid 无效（用了系统账号而非企微 userid）
- `60011` → 无权限 / 不在可见范围
- `82001` → 缺少 agentid

### 🔴 做「真实推送」验证前，先算清楚会打扰谁

**这个项目的真实情况**：几乎**不存在只推给一个人的业务路径**。审批规则是按**角色数组**配的 ——
一条报价单会同时推给 `sales_approver` 和 `admin`，一条采购单会推给 `purchase_approver` 并抄送 `admin`。
对方说「先做真实的推送」，你想的可能是「只发给他验证一下」，但**只要走业务单据，就必然有同事一起收到**。

**动手前先查，不要猜**：

```bash
grep -n 'APPROVAL_RULES' -A 10 db.js                          # 每条业务线推给哪些角色
sqlite3 data/erp.db "SELECT username,name,role,active FROM users ORDER BY id;"
python3 -c "import json;d=json.load(open('.wecom.json'));print(d.get('bind'))"   # 用户名 -> userid
```

**最小打扰的真实推送 = 绕开业务单据，直接调发送函数**：

```js
// 服务器上的一次性脚本：require 项目自己的模块，复用它的凭证与 token 缓存逻辑
const w = require('/opt/app/wecom.js');
(async () => {
  console.log(JSON.stringify(w.status()));   // 自检 configured / agentid，没配就中止
  await w.token();                            // 取 access_token（失败就别往下走）
  const uid = w.wecomIdOf('zhaohj');          // 用户名 -> 企微 userid
  const r = await w.sendCard(uid,
    '【联调】推送链路测试',
    '如果你收到这条消息，说明通道正常。<br>本消息由系统测试发出，无需处理。',
    'https://<你的域名>', '打开工作台');
  console.log(JSON.stringify(r));             // errcode 0 = 已受理
})();
```

**零业务副作用**：不建单据、不动库存、只给指定的一个人发一条。

### ⚠️ 三个必踩的坑

1. **`logPush` 通常没有被导出。** 项目的 `module.exports` 一般只导出对外的
   （`send / sendText / sendCard / notifyPending / ...`），内部写日志的 `logPush` 不在里面 ——
   外部脚本调 `w.logPush(...)` 会 `TypeError: w.logPush is not a function`。
   补日志只能**直接 INSERT**：
   ```sql
   INSERT INTO wecom_logs (touser,kinds,ref_type,ref_id,title,ok,errcode,errmsg) VALUES (?,?,?,?,?,?,?,?)
   ```
   （字段顺序照 `wecom.js` 里 `logPush` 的实现抄。）

2. **🔴 消息发出后脚本报错，绝不能「修一下重跑」。** 重跑会**再发一条**给真人。
   正确做法：**单独**写一个只补日志／只补后续步骤的脚本，不要重放发送动作。
   动手前务必想清楚：**脚本里的发送动作是否幂等**。

3. **探针脚本自己也会初始化数据库。** `require('./db')` 会跑建表逻辑
   （`CREATE TABLE IF NOT EXISTS` 等），输出 `[ledger] 库存流水表已建立` 之类 —— 这是正常的、
   无副作用的，**别误以为独立进程把生产库改坏了**。

### 要跑完整业务链路前，先检查测试脚本能不能在线上跑

最常见的失败是**脚本硬编码了初始密码**（如 `zhaohj` / `123456`），
而线上真人**早就自己改过密码** —— 第一步登录就失败，还连带所有依赖登录会话
（没带 API Key）的后续步骤全红，把真实结论淹没。
修法：允许环境变量覆盖（`ERP_PWD_ZHAOHJ=xxx`），只是让它不再假红。

**⚠️ 但要注意：不是所有接口都有 API Key 版本。** 本项目的 `routes/wecom.js` 整个路由挂在
`router.use(auth)` + `router.use(requireAdmin)` 下，**没有任何 API Key 入口** ——
所以「把日志步改走 API Key」是行不通的。

**更干净的做法：验证脚本完全不依赖真人密码。** 只走 `X-API-Key` 的 `/api/v1/*` 路径
（建单 → 查待审 → 审批），日志用 `sqlite3` / `better-sqlite3` **直接查库**。

```js
// 服务器端跑，只靠 API Key，日志直连数据库 —— 这样永远不受"真人改过密码"影响
const r1 = await api('/api/v1/quotes', 'POST', {...submit: true});   // 触发「待审批」推送
const r3 = await api('/api/v1/approvals/approve', 'POST',
  { type:'quote', id: r1.d.id, action:'通过', operator:'zhaohj' });   // 触发「回执」推送
const logs = db.prepare(                                             // 直连库读日志
  'SELECT id,kinds,touser,title,ok,errcode,errmsg,ts FROM wecom_logs WHERE ref_id = ? ORDER BY id'
).all(r1.d.id);
```

**「多人审批」验证要点**：一条 `quote` 应产生**两条 `pending` 日志**（每个审批人一条）+ 一条 `decision`。
断言 `new Set(pending.map(x => x.touser)).size === 2`，就能证明「多人可批」这条逻辑真的生效，
而不只是「发出去了」。

## 排错顺序

1. `gettoken` 通不通 → 不通查 corpid / secret
2. `message/send` 报 `81013` → userid 错了，重新取
3. 报 `60011` → 可见范围没加这个人
4. 一直 `errcode 0` 但收不到 → 检查是否发给了自己以外的账号、或被免打扰收纳

## 部署陷阱

rsync 多文件时，**源文件跨目录必须拆成多条命令**分别指定目标目录：

```bash
# ✗ 会把根目录的 wecom.js 模块覆盖掉 routes/wecom.js 路由文件
rsync wecom.js routes/sales.js routes/purchase.js host:/opt/app/routes/
# ✓
rsync wecom.js        host:/opt/app/
rsync routes/*.js     host:/opt/app/routes/
```
