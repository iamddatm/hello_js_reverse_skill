# 阿里 mtop 网关 sign(MD5 令牌握手)+ x5sec 传输层风控

> 难度:★★☆☆☆(签名) / ★★★★☆(x5sec 风控边界)
> 还原方案:A: 纯算法还原(.NET 10 实现,本案例交付语言非 Node/Python)
> 实现语言:.NET 10(MD5/GBK/HTTP2 原生,零浏览器依赖)
> 最后验证日期:2026-09-22
> 平台类型:电商平台(阿里系 mtop 网关:1688 / taobao / tmall h5api 通用)

---

## 技术指纹(供 Phase 0.5 自动匹配)

### JS 特征
- [x] `g.alicdn.com/mtb/lib-mtop/x.y.z/mtop.js`(或 combo 中的 `@ali/lib-mtop.js`)
- [x] 搜索 `_m_h5_tk` 直接命中;变量特征:`var B="_m_h5_c",C="_m_h5_tk",D="_m_h5_tk_enc"`
- [x] 签名拼接源码模式:`i(d.token+"&"+j+"&"+h+"&"+c.data)`(i 为内嵌 md5)

### 参数特征
- [x] `sign`:32 位小写 hex,MD5
- [x] `t`:毫秒时间戳;`appKey`:1688 PC = 12574478(固定);`jsv`:SDK 版本号
- [x] `api`/`v`/`type`(originaljson|jsonp)/`dataType`/`callback`(JSONP 时)
- [x] Cookie `_m_h5_tk = <32hex>_<过期毫秒>` → token = 下划线前段

### 请求特征
- [x] GET `https://h5api.m.1688.com/h5/<api小写>/<版本>/?...`(taobao 为 h5api.m.taobao.com)
- [x] **令牌握手协议**:无 token 或 token 过期时响应 `FAIL_SYS_TOKEN_EMPTY::令牌为空` / `FAIL_SYS_TOKEN_EXOIRED`(注意网关历史拼写缺 P)+ Set-Cookie 新 `_m_h5_tk`;客户端用新 token 重算 sign 重试
- [x] 错误码分层:sign 错 → `FAIL_SYS_ILLEGAL_SIGN`;会话错 → `FAIL_SYS_SESSION_EXPIRED`;风控 → `RGV587_ERROR::SM` + `data.url` 指向 `_____tmd_____/punish`

### 反调试特征
- [x] 无(纯混淆的常规 JS,非 JSVMP)

### 混淆类型
- [x] 类型:webpack 压缩(lib-mtop 28KB,可读)

### 指纹检测规则(Agent 执行)

```
快速检测命令(30 秒内完成):
  - list_network_requests(url_filter="h5api") → 命中 sign= 且含 jsv=
  - search_code(keyword="_m_h5_tk") → 命中 lib-mtop
  - 响应体含 "FAIL_SYS_" 前缀错误码
  匹配判定:以上 3 项命中任意 2 项 → 高置信度匹配
```

---

## 反爬类型判定

```
判定步骤:navigate 首页,redirect_chain 全程 200 无 412 循环;XHR 带 sign 参数
判定结果:行为型(参数签名) + 独立的传输层风控层(x5sec)
判定依据:页面正常加载,mtop 请求带 sign;裸客户端请求触发 RGV587
工具路径:search_code 定位 lib-mtop → 读源码提公式 → 浏览器内 fetch 端到端验证
```

---

## 加密方案

- **算法**:MD5,无密钥无盐
- **密钥来源**:token 由服务端 Set-Cookie 下发(握手协议自取,**非登录态**)
- **加密流程**:
  1. 任意请求(sign 随便填)→ 服务端 Set-Cookie `_m_h5_tk=<token>_<ts>`
  2. `sign = MD5( token + "&" + t + "&" + appKey + "&" + data )`
     - t = 当前毫秒时间戳(与 URL 中的 t 一致)
     - data = JSON 字符串**原文**(未 URL 编码;传输层再整体 EscapeDataString)
  3. 重发正式请求
- **签名公式**:`sign = md5(f"{token}&{t}&{appKey}&{data}")`
- **业务编码坑**:1688 搜索 keywords 为 **GBK 百分号编码**(%CD%FA%D7%B0=童装),且嵌在 data JSON 内层再被 URL 编码一次(%25 前缀)

---

## 已验证定位路径(命中后直接执行)

```
步骤 1: search_code(keyword="_m_h5_tk") → 命中 g.alicdn.com/mtb/lib-mtop/x.y.z/mtop.js
步骤 2: scripts(save) 下载 → grep 上下文:grep -o '.\{0,280\}sign.\{0,200\}' | grep md5/token
        直接看到 i(d.token+"&"+j+"&"+h+"&"+c.data) 即公式确认
步骤 3: 浏览器里 evaluate_js 读 document.cookie 拿 token
步骤 4: 本地任意 MD5(pwsh/.NET/Node)算 sign → 浏览器 evaluate_js fetch(url,{credentials:'include'})
        → 响应 ret 含 SUCCESS 即端到端闭环
步骤 5: 裸协议(curl/.NET HttpClient)只测 sign:若返回 SESSION_EXPIRED/ILLEGAL_SIGN 说明 sign 已过
        网关校验;若返回 RGV587 见下节风控边界
```

---

## 还原代码模板

### .NET 10(本项目实际交付,见 MtopSearch/Program.cs)

```csharp
// sign 核心(输入 data 原文,UTF-8,小写 hex)
static string ComputeSign(string token, long t, string appKey, string data) =>
    Convert.ToHexStringLower(MD5.HashData(
        Encoding.UTF8.GetBytes($"{token}&{t}&{appKey}&{data}")));

// token 来自握手 Set-Cookie 的 _m_h5_tk,取下划线前段
var token = cookieContainer.GetCookies(apiUri)["_m_h5_tk"].Value.Split('_')[0];

// GBK 关键词百分号编码(童装 → %CD%FA%D7%B0)
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance); // .NET 10 已内置,无需 NuGet
var kw = string.Concat(Encoding.GetEncoding("GBK").GetBytes(keyword).Select(b => $"%{b:X2}"));
```

### Python 等价(参考)

```python
sign = hashlib.md5(f"{token}&{t}&{appKey}&{data}".encode()).hexdigest()
```

---

## 🚫 禁动清单

### ❌ 禁动对象 1:把"补 cookie"当作绕过 RGV587 的手段
- **理由**:信誉窗口被污染后,贴任何 cookie(含旧的 x5sec)都拦——但这不代表 cookie 无效。
  实测**自己滑动验证后拿到的新鲜 x5sec,粘贴到 .NET 裸 TLS 客户端立即放行**(5/5 SUCCESS,2026-09-22)。
  即 x5sec 通行证绑定的是「验证事实 + IP/UA/时效」,不强绑 TLS 指纹。
  区别在于:①新鲜 vs 过期(旧 x5sec 贴 curl 仍 RGV587 是假线索);②信誉窗口是否已被自己打脏。
- **正确做法**:x5sec 作为**用户自备的外挂凭证**(config 外置,过期人工重滑),不要写进代码硬编码;
  归因实验遵守坑 3 的窗口对照法。

### ❌ 禁动对象 2:对风控接口高频重试探测
- **理由**:同一 IP 连续十几次 RGV587 失败后,连真实浏览器指纹(Camoufox)从 SUCCESS 变为 RGV587,
  即**信誉降级会传染**;继续打只会越锁越死。
- **正确做法**:裸客户端实验失败 2~3 次即停手,把剩余差异归因交给"浏览器内 fetch 对照"。

---

## 踩坑记录

| # | 坑 | 现象 | 解决方法 |
|---|---|------|---------|
| 1 | 令牌握手用完整 data 被风控拦死 | 冷启动正式 data 请求直接 RGV587,拿不到 token | 握手请求 data 用 `%7B%7D`(空对象),实测稳定返回 TOKEN_EMPTY + Set-Cookie token |
| 2 | evaluate_js 注入 `<script>` JSONP 回调拿不到结果 | MCP evaluate_js 与页面主世界隔离,回调永不触发,表现为 timeout | 改用页面内 `fetch(url,{credentials:'include'})` 直读响应(mtop 网关对 .1688.com 开 CORS);或看 network_capture 的响应体 |
| 3 | 归因实验顺序颠倒导致误判 | 先测 curl_cffi 真 Chrome TLS 时 IP 已被前面几十次失败请求标记 → 误得"TLS 伪装无用"结论 | **信誉窗口对照法**:每组实验之间用浏览器 fetch 探一次路;浏览器也失败 = 窗口关闭,此前的裸客户端结论全部作废重测 |
| 4 | GBK 关键词双重编码 | 直接 UTF-8 关键词返回空结果/乱码 | keywords 必须 GBK 字节 → %XX 大写 → 放进 data JSON → 整个 data 再 URL 编码(% 变 %25) |
| 5 | `FAIL_SYS_TOKEN_EXOIRED` 搜不到文档 | EXPIRED 拼写不同 | 阿里网关历史拼写 EXOIRED(缺 P),重试条件要同时匹配 TOKEN_EMPTY / EXOIRED |
| 6 | PowerShell 函数参数 `$Md5` 与局部变量 `$md5` 冲突 | 布尔开关参数"吞掉"MD5 对象(MetadataError) | PowerShell 变量名大小写不敏感,开关改名/局部变量换名 |
| 7 | .NET SocketsHttpHandler 无 CookieUsePolicy | 编译失败 | 该属性在 HttpRequestMessage 上;.NET 10 默认 handler.CookieContainer + UseCookies=true 即可,直接自持 CookieContainer 引用 |

---

## 变体说明

| 变体 | 差异点 | 影响 |
|------|--------|------|
| taobao/tmall h5api | 域名 h5api.m.taobao.com,appKey 不同(常见 12574478/21380790) | sign 公式完全一致 |
| type=jsonp vs originaljson | JSONP 有 callback 包裹,originaljson 纯 JSON | 协议端一律用 originaljson 省解析;实测风控行为一致 |
| 需登录接口(needLogin) | 返回 FAIL_SYS_SESSION_EXPIRED | 需 cookie2 等登录 cookie(用户提供,勿硬编码) |

---

## 关键经验总结

### 1. 信誉窗口对照法(最重要)
阿里 x5sec 风控的失败会**积累在 IP/会话上**。做传输层归因实验前,先用浏览器 fetch 打一次目标接口
确认窗口是干净的;实验中途浏览器从通过变拒绝,说明 IP 已被己方探测打脏,此后所有裸客户端实验
结果不可信。归因矩阵要**先测最可能失败的客户端、后测对照组**,或隔时间窗重测。

### 2. mtop 错误码分层是天然的诊断仪表盘
`TOKEN_EMPTY`(缺 cookie)→ `ILLEGAL_SIGN`(公式错)→ `SESSION_EXPIRED`(缺登录)→ `RGV587`(传输风控)
→ `SUCCESS`。看到哪一层就知道卡在哪一环,不需要任何附加探针。本次 sign 还原的"公式正确"证据链,
正是靠浏览器内请求直达 SUCCESS + 裸客户端到达 RGV587(即已越过 sign 校验)两侧夹逼确认的。

### 3. 握手 data 最小化绕风控
令牌握手对 data 内容无校验(网关层不解析 data),把握手 data 缩到 `{}` 可避开正式请求触发的
x5sec 决策点,实测这是纯协议路径能走通的第一道门。

---

## 可验证事实清单

- [ ] sign = MD5(token&t&appKey&data) UTF-8 输入、小写 hex 输出(2026-09 实测通过网关校验)
- [ ] token = cookie `_m_h5_tk` 下划线前 32 hex;`_m_h5_tk_enc` 不参与 sign
- [ ] 1688 PC appKey = 12574478(lib-mtop 源码默认值)
- [ ] 无 token 请求 → `FAIL_SYS_TOKEN_EMPTY::令牌为空` + Set-Cookie 新 token(握手协议)
- [ ] 网关过期令牌错误码历史拼写为 `FAIL_SYS_TOKEN_EXOIRED`
- [ ] data 参数 MD5 用**原文**,URL 传输再整体编码
- [ ] 搜索类接口 keywords 为 GBK 百分号编码(双层编码:data 内 %XX,传输层 %25XX)
- [ ] lib-mtop 路径 `g.alicdn.com/mtb/lib-mtop/<ver>/mtop.js`,combo 加载名 `@ali/lib-mtop.js`
- [ ] RGV587 响应的 `data.url` 即 punish 挑战页(x5step=2&action=captcha,AWSC noCaptcha 滑块)
- [ ] x5secdata cookie Max-Age=20 秒(挑战数据短时效)
- [ ] 匿名会话(无 cookie2)可正常返回目标接口业务数据(该接口无需登录)
- [ ] type=originaljson 与 type=jsonp 的风控/校验行为一致
- [ ] **新鲜 x5sec(人工滑动后获得)粘贴到 .NET/curl 裸客户端立即放行**——x5sec 不强制绑定 TLS 指纹;
      旧 x5sec 或浏览器会话专属 x5sec 复用到裸客户端会被拦(时效 + 归属校验)
- [ ] x5sec 有效期内连续 5 次不同时间戳请求全部 SUCCESS(2026-09-22 .NET 10 实测 5/5)
