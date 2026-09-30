# issue #10 修复报告：反代正常但用量明细 502、token 为 0

- 对应 issue：[HUIdada1/AgentHub#10](https://github.com/HUIdada1/AgentHub/issues/10)
- 报障版本：v1.31.0（缺陷实际自 v1.30.0 引入，v1.31 / v1.32 均未修复）
- 修复版本：v1.32.0 之后（`electron/backend/proxy/server.cjs`）
- 状态：已修复并验证

---

## 一、问题现象

用户通过网关反代请求一切正常（客户端能拿到完整回答），但「用量统计 · 请求明细」里这些成功请求被记为：

- **状态 = 502**
- **请求 Tok = 0、响应 Tok = 0**
- **TTFT = `-`**

即：**业务成功、流水失败**。用户报障截图（issue #10 正文）中，明细表 10 行全部为 502 / 0 / 0，而反代请求本身可用。

---

## 二、根因（一句话）

`electron/backend/proxy/server.cjs` 成功收尾时，记账备注文案引用了**块级作用域外**的变量 `targetModel`，抛 `ReferenceError: targetModel is not defined`；该异常被同一 `try` 的 `catch` 捕获，于是把**已经成功返回给客户端的请求**重新记为 `502 + 0 token`。

### 2.1 缺陷代码

`targetModel` 在 `for (const chainModel of modelChain)` 循环体内用 `let` 声明（块级作用域），而成功收尾的 `record({...})` 在**循环之外**引用它：

```js
for (const chainModel of modelChain) {          // ← 循环开始
  ...
  let targetModel = chainModel;                 // ← 只在循环块内可见
  ...
}                                               // ← 循环结束，targetModel 出作用域
...
if (done) {
  ...
  record({
    status: 200, ...,
    error: usedModel !== actualModel
      ? "fallback→" + usedModel
      : targetModel !== actualModel             // ← ReferenceError：循环外取不到
        ? "rev→" + targetModel
        : ...,
  });
  return;
}
```

### 2.2 为什么表现为 502 + 0 token

`record()` 抛出的 `ReferenceError` 冒泡到 `handleChat` 外层的 `catch`：

```js
} catch (e) {
  ...
  record({ status: 502, error: msg.slice(0, 200) });   // ← 覆盖记账：502，且不带 token 字段
}
```

- 这条 `catch` 里的 `record` **只传 status 与 error，不传 promptTokens / completionTokens** → token 恒为 0、TTFT 恒为 0（`ttftMs` 未传）。
- 原本那条 200 的 `record` 调用**根本没执行完**（在 `error:` 表达式求值时就抛了），所以库里只有 502 这一条。
- 对**流式**请求，`res.end()` 在 `record` 之前已执行，客户端早已拿到完整成功响应 → 于是出现「反代能用，但明细 502」。

### 2.3 引入时间与证据链

| 证据 | 内容 |
|---|---|
| 代码 blame | 引入 `targetModel` 引用的提交是 **v1.30.0**（`12199ca`，"反代网关第五渠道 zcode"），循环内 `let` 声明自 v1.25.2 起就在，两处作用域错配由此形成 |
| 真实库统计 | 本机 `%APPDATA%\AgentHub\proxy\stats.db`：`200` 记录 569 条（12:41–18:44，v1.30 之前），`502` 记录 201 条（19:32 之后），**502 全部 error 字段为 `targetModel is not defined`** |
| 时间线 | 18 点前 0 条 502；19 点起 200 归零、502 接管 —— 与引入该行的时间点吻合 |
| 最小复现 | 用假上游跑真实代码路径：客户端 HTTP 200、SSE 含正文与 `total_tokens:12`，但流水记录 `status:502, promptTokens:0, completionTokens:0, error:"targetModel is not defined"` |

---

## 三、修复内容

修改文件：`electron/backend/proxy/server.cjs`（`tools/proxy-smoke.cjs` 补回归断言）。

### 3.1 主修：`targetModel` 提升到循环外

```js
let usedModel = actualModel;
let targetModel = actualModel;          // ← 新增：声明在循环外，收尾备注可引用
for (const chainModel of modelChain) {
  ...
  targetModel = chainModel;             // ← 原 `let targetModel = ...` 改为赋值
  ...
}
```

### 3.2 加固：记账绝不外抛

`record()` 内部写库包 `try/catch`。流水是**观测数据**，不是业务结果——写库失败（DB busy / 磁盘满 / 字段异常）只记日志，不得影响已发出的响应状态，更不能反过来把成功请求甩成 502。

```js
const record = (extra) => {
  usageRow.latencyMs = Date.now() - startedAt;
  Object.assign(usageRow, extra || {});
  try {
    store.insertUsage(usageRow);
    if (usageRow.accountId) store.bumpAccountUsage(...);
  } catch (e) {
    console.error("[proxy] 用量记账失败（不影响本次响应）：", e);
  }
  emitRequestThrottled();
};
```

### 3.3 加固：200 记账提前到副作用之前

原来「扣余额 / 清冷却」等 DB 写操作排在 200 记账**之前**，任一异常都会把成功请求拖进 `catch` 记成 502。现调整为：**响应发出 → 立即记 200 → 再做副作用（单独兜底）**。

### 3.4 加固：`catch` 尊重已结束的响应

若响应已结束（`res.writableEnded`）才抛错，说明客户端已拿到成功响应，不再补记 502，只打日志。

### 3.5 顺带修正：`fatalErr` 未被消费

循环靠 `if (done || fatalErr) break` 退出，但循环之后从未读取 `fatalErr`，最终状态一律回落 `lastErr`（可能是更早一次尝试遗留的码，或 null → 503），导致明细状态与真实退出原因脱节（如渠道级 WAF 503 被记成上一次的 502）。现改为 `const finalErr = fatalErr || lastErr;`。

---

## 四、验证

### 4.1 最小复现（修复前 → 修复后）

| 场景 | 修复前 | 修复后 |
|---|---|---|
| 流式 | 客户端 200，流水 **502 / 0 / 0** | 客户端 200，流水 **200 / 7 / 5** |
| 非流式 | 客户端 200，流水 **502 / 0 / 0** | 客户端 200，流水 **200 / 7 / 5** |

### 4.2 全量回归

`tools/proxy-smoke.cjs` 全绿（`SMOKE OK`），覆盖：换号 / 两种自动切换 / 流式双态 / 指纹头 / 模型回退禁用 / IDE 切换 / server 错误熔断 / 429 同号退避 / 模型级负缓存跨重启。

### 4.3 新增回归断言

在 `proxy-smoke.cjs` 的 10.2 段补充（锁定 issue #10，防复发）：

```js
const okRow = store.recentRequests(1)[0];
assert(okRow.status === 200 && okRow.promptTokens === 7 && okRow.completionTokens === 5,
  "成功请求流水为 200 且 token 正确（issue #10 回归）");
assert(!/is not defined/.test(String(okRow.error || "")), "成功流水不得带 ReferenceError 文案");
```

### 4.4 历史脏数据说明

修复只影响**新写入**的流水，**历史 502 行不会自动更正**（当时确实没写入 200 行，无法还原 token）。这些行可按 `error = 'targetModel is not defined'` 识别，如需清理可由用户自行决定是否删除。

---

## 五、影响面

- 受影响渠道：全部（该代码路径与渠道无关）。实测库中 trae 59 条、workbuddy 132 条、zcode 3 条 502 均由本缺陷产生。
- 受影响版本：**v1.30.0 起**（含 v1.31.0、v1.32.0）。v1.25.2 及更早不受影响。
- 修复后：成功请求恢复记为 200，请求 / 响应 token 与 TTFT 正常落库；余额扣减、冷却清理等副作用不再可能污染流水状态。

---

## 六、issue 中另一问题

issue #10 标题同时提到「4K 屏幕 200% 缩放下界面错位」，与本报告为**两个独立缺陷**。缩放问题不在本次修改范围，单独排查修复（涉及 `electron/main.cjs` 的 `applyViewportZoom` 与前端固定像素元素）。详见后续缩放修复说明。