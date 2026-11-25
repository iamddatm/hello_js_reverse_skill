# 案例:Cores(fengs)加载器 + "Compression v3.0" 双层自定义Base64字符串表 + MD5 请求签名

> 难度:★★☆☆☆
> 还原方案:A: 纯算法还原
> 实现语言:C# (.NET 10) 交付 + Node.js 验证脚本(算法还原与语言无关)
> 最后验证日期:2026-09-24
> 平台类型:电商 SaaS(抖店上货工具,Web AngularJS SPA)
> 样本站点:www.douyongxin.com(抖掌柜系,CDN 在 cdn.douzhanggui.com / cdn.doudianruanjian.com)

---

## 技术指纹(供 Phase 0.5 自动匹配)

### JS 特征
- [ ] 特征1:所有业务模块都包在 `Cores(function(e,t,n,...){var Ms=this.d,...},"<超长base64样字符串>")` 里,文件头注释 `/* Compression v3.0 YYYY-MM-DD */`
- [ ] 特征2:入口 HTML 有 `<meta name="page-config" content="cdn=...;appid=NN;...">`,全局有 `window.Cores` / `window.md5`(md5.js 无 var 赋值泄漏到全局)
- [ ] 特征3:模块经自研 AMD loader `core.js`(fengs,"FENGS_REMOTE_HOST") 动态加载,URL 形如 `cdn.../js/doudian/xxx.js?<版本号>`
- [ ] 特征4:localStorage 命名前缀 `DOU_`(DOU_USER / DOU_CERT / DOU_SUBID / DOU_OEM / DOU_APPID)

### 参数特征
- [ ] 参数名: sign | 长度: 32 | 字符集: hex 小写 | 编码: MD5(自定义urlencode(拼接串)),**放在 URL query 里**
- [ ] 参数名: timestamp/client/userid/subId | 特征: query 固定顺序 `timestamp=<秒>&client=11&userid=<手机号>&subId=0&sign=<md5>`
- [ ] 请求体: 自定义字母表 Base64(字符集 [A-Za-z0-9+/=],长度 4 倍数),`Content-Type: text/plain`,method POST(所有接口都是 POST 含查询类)

### 请求特征
- [ ] 同一接口重复出现两遍(XHR 层重发/双通道),Referer 为 /app/login 或 /app/regist
- [ ] 响应 Content-Type: `text/x-json;charset=UTF-8`,响应体为**明文 JSON** `{status,tips,data}`,status 是字符串("1"/"-9"),无响应加密

### 反调试特征
- [ ] 无。无 debugger 陷阱、无环境检测、无指纹采集——只是字符串表难读(纯混淆)

### 混淆类型
- 其他:"Compression v3.0" 自研混淆。非 OB、非 JSVMP。原理:业务代码的全部字符串字面量收进一张表,表放在文件尾一个超长字符串里,**双层自定义 Base64 编码**:外层 fr(big) → `#` 分隔的 token 列表 → 每个 token 再 fr() → 字面量。外层 factory 的参数按位置对应表项,代码里用 `this.d(token)`(常见 minifier 命名为 Ms/Gt/un 等)取字面量。

### 指纹检测规则(Agent 执行)

```
快速检测命令(30秒内完成):
  - search_code(keyword="Compression v3.0") 或入口 HTML grep "Cores(function" → 命中条件
  - search_code(keyword="this.d") + 超长引号字符串(≥1500字符,字符集 [0-9A-Za-z+/=]) → 命中
  - 请求 query 同时含 timestamp+client+userid+subId+sign 五件套 → 命中
  匹配判定:3 项中命中 2 项 → 高置信度匹配
```

---

## 反爬类型判定

```
判定步骤:navigate 不加 hook → 无重定向无 4xx,直接 200;XHR 带 sign 但输入不含环境指纹
判定结果:纯混淆(签名型反爬的"签名"只是 MD5 拼接,无环境依赖)
工具路径:任意组合;本案例实际= 下载 JS 离线解表 + 浏览器 Hook 取证,未用插桩
```

---

## 加密方案

- **算法**:自定义字母表 Base64(请求体)+ 标准 MD5(签名与密码哈希)。**无 AES/DES/RSA,无环境指纹参与**
- **密钥来源**:全部硬编码在 req.js 字符串表里(用本 case 的解码法 30 秒可提取)
- **加密流程**:
  1. 业务对象 → `JSON.stringify`(字段顺序=对象字面量顺序,merge 的 oem/shopId 追加在尾部)→ UTF-8 字节 → 标准 base64 结构 → 64 字符表逐位替换为自定义表(填充位落在表第 64 字符 '=')
  2. `P = urlPath + "&" + userid + "&" + timestamp + "&" + bodyCipher[0:0x400]`
     (登录态且 mode>0 时,在 timestamp 与 body 之间插入 `&`+DOU_CERT 值;cert 长度>32 先剔除大写字符与 `admin` 子串;未登录接口无此段)
  3. `sign = MD5( d(P ).toLowerCase? 否 )` —— d() = encodeURIComponent,再对 `! ~ ( ) '` 替换为 `%XX`(**hex 大写**),再 `%20→+`;MD5 输出 32 位小写 hex
  4. 最终 URL:`path?timestamp=&client=11&userid=&subId=0&sign=`,各值 encodeURIComponent
- **关键常量**(该站实测值,改版需重新解表):
  - 请求体字母表(ckey):`deIHJLKNOMPQVfghijZ0213B4klmntuvwEFRSTopqxyzADrsGWCXY5b6c7a8U9+/=`
  - 密码哈希盐:`pass = md5(明文 + ",.p!~")`,`pay = md5(明文 + "~#@$*)!")`
  - loader 自身的字母表(core.js 由 64 个双字符 token 推导):`Mfg+kGHI4BCtz05AYQRS39U1lmn678uijDbcdeVWX2/JLaThsEFyKNOPopqrvwxZ=` —— **注意与请求体字母表是两张不同的表**,别搞混
  - client=11(web)/10(桌面客户端);oem/appid=46(来自 page-config meta)
  - timestamp = `(Date.now() + 服务端时钟偏移)/1000` 取整;用任意 HTTP 响应的 Date 头校时
  - 明文模板(找回密码流程):
    - msgCodeSend:`{"do":"login|reg|findpass","code":0,"orderid":"<ms>_<phone>","phone":"...","userid":"...","oem":46,"shopId":0}`
    - findPass:`{"code":"<验证码>","phone":"...","userid":"...","from":0,"pass":"<md5(新密码+盐)>","oem":46,"shopId":0}`

## 已验证定位路径(命中后直接执行)

```
步骤 1:入口 HTML 找 <script src="/assets/js/..."> → 定位 core.js(loader)+ 页面模块 URL
        (cdn.douzhanggui.com/js/doudian/...),curl 全部下载
步骤 2:在 core.js 底部 IIFE 实参里找到 64 个"双字符token"(252u2 v172z ...) →
        parseInt(p,36)→fromCharCode 拼出 loader 字母表 → 实现 fr()(解码)
步骤 3:curl req.js(= 请求层本体)→ 正则提取文件尾 `"([0-9A-Za-z+/=]{1500,})"` 大字符串 →
        fr() 一次 → split("#") → 每项再 fr() 一次 → 300 项字面量表
步骤 4:自动反混淆:正则抓外层 `Cores(function(参数列表){` 的参数名(必须 .trim()!),
        按序对应表项构建 map,替换 `Ms\(参数\)` 为 JSON.stringify(字面量) →
        明文源码直读 sign 拼接、字母表 ckey、两个盐
步骤 5:运行时取证(chrome-devtools MCP,camoufox 不可用时的降级路径):
        a) Hook XMLHttpRequest.prototype.open/send:记录最终 url+body 后【吞掉 send,不真发】
           —— 涉及短信/支付接口必须这样做,防真实下发
        b) Hook window.encodeURIComponent(它是 d() 的底层调用,属性查找在运行时,可 patch):
           直接截获取签名原文 P —— 一条请求就拿到拼接顺序铁证
        ⚠ 不要试图 patch window.md5:Cores 的依赖注入在模块加载时就把函数引用捕获进闭包,
          运行时 patch 全局名无效(实测)
步骤 6:离线重放:HAR/捕获样本逐条 md5(d(path&uid&ts&body))==sign 验证,全过再写交付代码
```

## 还原代码模板

### 核心函数(C# / .NET 10,完整实现见样本工作区 dotnet/DouyongxinApi/DouyongxinClient.cs)

```csharp
public static string EncodeBody(string utf8Json) // 标准base64→换表,与前端encoder逐位等价
{
    var std = Convert.ToBase64String(Encoding.UTF8.GetBytes(utf8Json));
    return new string(std.Select(c => c == '=' ? '=' : BodyAlphabet[StdAlphabet.IndexOf(c)]).ToArray());
}
public static string JsUrlEncode(string s)       // encodeURIComponent + !~()'→%XX大写 + %20→+
{
    var sb = new StringBuilder();
    foreach (var c in s)
        if (Uri.IsReservedUnreservedChar(c)) sb.Append(c);          // A-Za-z0-9-_.!~*'()
        else if (c == ' ') sb.Append('+');
        else foreach (var b in Encoding.UTF8.GetBytes(c.ToString())) sb.Append('%').Append(b.ToString("X2"));
    return "%21 %7E %28 %29 %27" 的对应替换;  // 见交付代码
}
// sign = Md5Hex(JsUrlEncode(path + "&" + userid + "&" + ts + "&" + body));
```

```javascript
// Node.js 解表器(提取该混淆方案的全部字面量):
const ALPHA = /* core.js 双字符token parseInt(,36) 推导 */;
function fr(e){ e=e.replace(/[^A-Za-z0-9+/=]/g,""); /* 4char→3byte,indexOf(ALPHA),padding=64 */ }
const table = fr(big).split("#").map(t => fr(t));
const params = src.match(/Cores\(function\(([^)]*)\)\{var/)[1].split(",").map(s=>s.trim());
const out = src.replace(new RegExp(resolver+"\\(([A-Za-z_$][\\w$]*)\\)","g"),
                        (_,p)=>JSON.stringify(table[params.indexOf(p)]));
```

## 🚫 禁动清单

### ❌ 禁动对象 1:把 loader 字母表(core.js 推导的那张)当成请求体字母表
- **理由**:两套自定义 Base64 并存——core.js/`this.d` 解字符串表用一张,请求体 encoder 用 req.js 字符串表里的 `ckey`。混用则解出乱码且毫无报错
- **正确做法**:解表用 core 表;验证请求体用 ckey 表,以 HAR 密文能解出合法 JSON 为准绳

### ❌ 禁动对象 2:对触发短信/支付/改密类接口做"未拦截"的真实点击
- **理由**:获取验证码按钮一点就是真发短信;用户抓的 HAR 用的是自己的号码,分析者不知道号码归属
- **正确做法**:先 Hook XHR.send 吞包,再点按钮(取证明即可);在线端到端验证用格式合法但未注册的号码,让服务端在发短信前走业务失败分支

### ❌ 禁动对象 3:运行时 patch window.md5 / 期望依赖注入是惰性的
- **理由**:Cores 依赖在模块加载瞬间注入并捕获函数引用,后续 patch 全局属性对闭包无效(实测无日志但签名照算)
- **正确做法**:Hook 链路要选"调用时才做属性查找"的点:`encodeURIComponent`、`JSON.stringify`、XHR 原型方法

## 踩坑记录

| # | 坑 | 现象 | 解决方法 |
|---|---|------|---------|
| 1 | 参数名 split(",") 后带前导空格 | `map` 键为 `" Gi"`,替换 0 命中 | `.split(",").map(s=>s.trim())` |
| 2 | 反混淆替换正则误伤 `function` 关键字 | `f"$locals"tion` —— 字面量 "function" 的括号内容被二次匹配 | 用精确 `\b` 边界或仅匹配解析器变量名;生成后 grep 乱码自检 |
| 3 | resolver 变量名每个文件不同 | 按 `Ms`/`Gt` 硬编码,某文件 0 替换 | 先 `src.match(/var (\w+)=this\.d/)[1]` 动态探测 |
| 4 | 大字符串正则取错对象 | 一个文件含多个超长字符串,首个是 core-js polyfill 的 | `{1500,}` 全部提取,逐个解、按可解性取正确的那个 |
| 5 | d() 的 hex 大小写 | 用 `%7e` 验签全错 | 前端 `.toString(16).toUpperCase()` 是**大写** `%7E`(仅 !~()' 五字符受影响) |
| 6 | timestamp 与本机时间对不上 | 本地时间算的 ts 与服务端接受值差 150+ 秒 | ts 用响应 Date 头校时(HAR 的 Date 头与 ts 完全一致),先 GET 一次再算 |
| 7 | HAR 无响应体 | 误以为响应加密 | 是抓包时未存 content.text;响应实际是明文 text/x-json |

## 变体说明

| 变体 | 差异点 | 影响 |
|------|--------|------|
| 登录态接口(/user/regOrLogin、/web/*) | mode>0 时 P 插入 `&DOU_CERT`;OAUTH 走 UID/TOKEN 组(g/w 表) | 客户端预留 key 段拼接分支即可 |
| 桌面客户端 | client=10,请求经 script_bridge/invokex | 与 Web 协议签名同构 |
| kuaishou 平台 | url 加 `/kuaishou` 前缀,shopId 归零逻辑不同 | 签名公式不变 |

## 关键经验总结

1. **"Compression v3.0" 类自研混淆的表是双层的**:fr 一次 → `#` 分隔 → 每项再 fr;单层解码得到的是 token 列表的假象(会误判"截断/乱码")
2. **字母表本身藏在 loader 自己的代码里**:core.js 尾部实参的双字符 token 数组(如 `"252u2","v172z"`)→ `parseInt(t,36)→fromCharCode` 就是那张 65 字符表(第 65 个固定 '=')。先还原 loader 才能还原一切
3. **取签名原文 P 的最短路径不是 hook 签名函数**——签名函数在闭包里,但它必然调用"运行时才查找属性"的全局工具(`encodeURIComponent`);hook 它一次拿到完整 P,比静态抠拼接代码快且铁证
4. **吞包 Hook(XHR.send 不调用原版)既取证又零副作用**,是有外发风险接口(短信/支付)的标准打法
5. **离线重放 21/21 + 在线业务错误响应** 双验证才交付:.NET 与 JS 实现存在 hex 大小写、Unicode 转义差异的风险只有真实样本能暴露
6. 自定义 Base64 站点先测"是否等价于标准表换表"(本例是):若 encoder 结构与标准 base64 逐位一致,直接 `Convert.ToBase64String` + 字符映射,少写一半易错代码
7. HAR 响应体缺失 ≠ 响应加密,先怀疑抓包配置,再去代码里找 decrypt

## 可验证事实清单

- [ ] 站点存在 `window.Cores` 且 `Cores.d` 是函数(自定义 base64 解码器)
- [ ] `Cores.d("deIHJL..." 大字符串)` 解出 `#` 分隔的 token,每 token `Cores.d` 解出可读字面量
- [ ] 请求体自定义字母表 = `deIHJLKNOMPQVfghijZ0213B4klmntuvwEFRSTopqxyzADrsGWCXY5b6c7a8U9+/=`(65 字符)
- [ ] 未登录时 `sign == md5( d(urlPath+"&"+userid+"&"+timestamp+"&"+bodyCipher) )`
- [ ] d() 中 `~`→`%7E`(大写 E),空格→`+`
- [ ] `pass` 哈希盐 `,​.p!~`,`pay` 盐 `~#@$*)!`
- [ ] query 顺序固定 timestamp,client,userid,subId,sign;client=11(web)
- [ ] msgCodeSend 明文 7 字段且 do∈{login,reg,findpass},orderid=`<ms>_<phone>`
- [ ] findPass 明文含 `from:0`,code 为字符串
- [ ] 响应 text/x-json 明文,status 字符串;"-9"=业务失败,"1"=成功
- [ ] timestamp 等于服务器 Date 秒值(±数秒),本机时钟不可信
