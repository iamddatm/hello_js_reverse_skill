# 拼多多 mobile web anti-content — JSVMP(JVM2) + vm 沙箱执行 + 传输层风控

> 难度：★★★★☆
> 还原方案：B: 沙箱执行（Node vm 运行原版风控 SDK 模块）
> 实现语言：Node.js（签名生成）+ Python（请求）
> 最后验证日期：2026-09-01
> 平台类型：电商平台（移动端 Web）

---

## 技术指纹（供 Phase 0.5 自动匹配）

### JS 特征
- [x] 搜索业务包中出现 `getAntiContent(void 0, isDesktop)` 调用（如 1899/1246 chunk）
- [x] axios 请求拦截器将 `body.anti_content || body.antiContent` 复制为请求头 `anti-content`（react_pdd 主包）
- [x] 风控 SDK 动态加载：`Promise.all([serverTime, e.e(3636), e.e(4541)])` → 模块 43016 → `new SDK({serverTime})`
- [x] SDK 内部为独立 webpack 小包（`t.exports=function(t){var e={};function r(A){...`），入口 `r.s=5`
- [x] JSVMP 元数据：字节码 magic `JVM2`（74,86,77,50）、`Unsupported bytecode version`、`programCount` 校验
- [x] 常量表含 `"0at"` / `"0as"` 前缀与 64 字符自定义 base64url 字母表 `h1IUSN-zDTC...`

### 参数特征
- [x] `anti-content` 头与 `anti_content` body 字段同值（拦截器复制，必须一致）
- [x] 值：`0at` 前缀 + base64url（`-_` 无 `=`），约 600-750 字符，每次变化
- [x] 另有 `rc-res-conf: 1|<base64>` 网关加密头（实测纯协议可不带）

### 请求特征
- [x] `POST /proxy/api/search?pdduid=<uid>`，需登录态（无登录 403，重定向 login.html）
- [x] 前置接口 `GET /proxy/api/api/server/_stm?pdduid=` 返回 `{"server_time":毫秒}`，免鉴权
- [x] SSR 首屏不出 XHR，滚动触底才发第 2 页搜索请求（抓包需滚动）

### 反调试特征
- [x] 18+ 项反调试位（T() 函数）：`window.Buffer/emit/spawn`（Node 检测）、`navigator.webdriver`（要求原型链存在且非自有属性）、`plugins.length===0`、`document.getElementById.toString()` 含 `native code`、`cdc_adoQpoasnfa76pfcZLmcfl_*`、`Function.prototype.bind/toString` 完整性
- [x] 函数源码指纹：`l.pbc(T.toString()+ct.toString())` 打包进签名 → 原始代码一个字不能改
- [x] localStorage `anti_pre_sig` 历史签名尾 8 位链（PreviousAntiToken），每次打包后由 VM 写回

### 混淆类型
- [x] JSVMP（自研，JVM2 字节码，`createStub(n)` → `execute(program#n)`；program#3=getMessage 主打包）

### 指纹检测规则（Agent 执行）

```
快速检测命令（30秒内完成）：
  - search_code(keyword="anti-content") → react_pdd 主包命中拦截器复制逻辑
  - search_code(keyword="getAntiContent") → 业务 chunk 命中调用点
  - scripts list → 存在 3636_*.js（~145KB）+ RiskControl_*.js（<1KB 存根）
  匹配判定：3 项中命中 3 项 → 高置信度匹配
```

---

## 反爬类型判定

```
判定步骤：
  1. navigate(url) 不加 hook → redirect_chain 直接 200 加载脚本，无 412 循环
  2. 后续翻页 XHR 带 anti-content 签名参数
判定结果：行为型
判定依据：无 412 挑战循环，签名只作为参数附加
工具路径：search_code 静态定位（get_request_initiator 对本目标无效，页面拦截器先于 Hook 缓存引用）
```

---

## 加密方案

- **算法**：JSVMP 不可提取。采集器打包 → pako deflate → 自定义 base64url（"0at" 前缀）
- **密钥来源**：无传统密钥；打包内容 = 环境指纹 + 行为数据 + 服务器时间 + 源码指纹
- **打包内容**：屏幕尺寸、UA、referrer、location、cookie（pdd_user_id/api_uid/pdd_vds）、
  `_nano_fp` 指纹（cookie+localStorage 双来源）、反调试位掩码、鼠标/点击/滚动事件、
  距服务器时间偏移、历史签名链、`T.toString()+ct.toString()` 源码指纹
- **关键**：签名不依赖请求体，可独立生成；`messagePackSync({antiVersion:"v2"})` 为唯一入口

---

## 已验证定位路径

```
步骤 1: search_code(keyword="anti-content") → 定位拦截器（只复制不生成）
步骤 2: 本地保存业务 chunk → grep anti_content 赋值点 → getAntiContent 调用链
步骤 3: 1246.js 模块 67312：getAntiContent = encodeURIComponent(sdk.messagePackSync(...))
步骤 4: 模块 11866：initRiskController = Promise.all([_stm 时间, chunk 3636+4541]) → new SDK
步骤 5: RiskControl_*.js 存根：模块 43016 → r(53636)
步骤 6: 3636.js 中括号配对提取 53636 工厂函数（145KB）→ module_53636_raw.js
步骤 7: vm 沙箱执行 + 真实浏览器基线环境 → 生成值与线上同格式（0at 前缀）
```

---

## 还原代码模板

完整可运行版在项目 `pdd_search/`：`anti_content.js`（vm 沙箱 + 环境仿真 + 合成行为事件）、
`main.py`（curl_cffi firefox 指纹 + 子进程调签名）。此处只列关键点：

```javascript
// 环境仿真要点（Node vm）：
// 1. navigator.webdriver 挂原型链 getter（in === true, hasOwnProperty === false）
// 2. Firefox UA：InstallTrigger 必须存在，window.chrome 必须不存在
// 3. plugins.length = 5，languages 非空
// 4. 所有仿真 DOM 函数 toString 伪装 "function X() { [native code] }"
// 5. 不能暴露 Buffer/emit/spawn/process（Node 检测位）
// 6. document.cookie 返回真实 Cookie（pdd_vds/api_uid/pdd_user_id/_nano_fp 会被打包）
// 7. localStorage 持久化 anti_pre_sig（签名链）
// 8. 工厂函数源码禁止任何修改（源码参与指纹）
const inst = sdkFactory({ serverTime: SERVER_TIME_MS });
const anti = encodeURIComponent(await inst.messagePackSync({ antiVersion: "v2" }));
```

```python
# 请求层必须：
# 1. curl_cffi impersonate="firefox147"（requests 裸指纹直接 403/40001）
# 2. anti-content 头与 body.anti_content 同值
# 3. UA 版本与 TLS 指纹档位自洽
```

---

## 🚫 禁动清单

### ❌ 禁动对象 1：模块 53636 工厂函数源码
- **理由**：`l.pbc(T.toString()+ct.toString())` 把函数源码打包进签名；
  VM 还有 checkNativeFunc 程序；任何格式化/重命名都会改变签名内容
- **正确做法**：原样读入 vm 执行；环境差异全部在沙箱全局对象上解决

### ❌ 禁动对象 2：Function.prototype.toString（不可包装）
- **理由**：T() 的 t[7] 位显式校验 `Function.prototype.bind/toString` 完整性，
  包装即置位
- **正确做法**：只做实例级 toString 覆写（fn.toString = ...）

---

## 🌐 UA 分支矩阵

### UA 分支 1：Firefox（本案例基线）
**必补**：InstallTrigger、plugins(5)、languages、DeviceOrientation/DeviceMotionEvent、
MediaSource、Element.prototype.attachShadow（toString 伪装）
**禁补**：window.chrome、userAgentData、opr/opera/safari

---

## 踩坑记录

| # | 坑 | 现象 | 解决方法 |
|---|---|------|---------|
| 1 | `get_request_initiator` 拿不到调用栈 | 页面拦截器先于 MCP Hook 缓存了 XHR 引用 | 放弃黄金路径，改静态源码追踪（save 脚本 + 本地 grep） |
| 2 | requests 裸 TLS 被拒 | 首请成功后连续 403 `error_code:40001` | 换 curl_cffi impersonate firefox |
| 3 | **传输层风控标记（未完全解决）** | 持续 54001 + verify_auth_token；重放浏览器签名同样被拒；换指纹档位/新 pdd_vds/10 分钟冷却均无效 | 结论：被标记后需真正完成人机验证或更长衰减；签名还原本身已验证正确（首请 200+125KB 数据） |
| 4 | Cookie 轮换 | `pdd_vds`/`PDDAccessToken` 会随浏览器导航轮换，脚本读旧值 | 请求前从浏览器同步最新 Cookie；长期方案需处理轮换 |
| 5 | 搜索页首屏无搜索 XHR | SSR 直出，翻页才发请求 | 抓包必须滚动触底触发第 2 页请求 |
| 6 | UBO 插件 451 | `python -m camoufox fetch` 装不全，启动报 InvalidAddonPath | 从 gorhill/uBlock GitHub release 手动解压到 `Cache/addons/UBO` |
| 7 | 风控信号语义 | 40001=指纹/签名拒绝（多为传输层）；54001=需人机验证（带 verify_auth_token） | 区分处理，54001 不要硬重试（会续标记） |

---

## 变体说明

| 变体 | 差异点 | 影响 |
|------|--------|------|
| 桌面 UA | getAntiContent 第 2 参 isDesktop 只控制服务器时间刷新，不进包 | 无 |
| 移动端内嵌 | UA 检测 `/iPhone|iPad|iPod|Android|harmony/`；native 平台走 JSBridge 分支 | 沙箱按 Firefox 桌面即可 |

---

## 浏览器指纹采集方法

```javascript
// evaluate_js 采集基线（对照沙箱仿真值）：
(() => ({
  ua: navigator.userAgent,
  webdriver_in: 'webdriver' in navigator,
  webdriver_own: navigator.hasOwnProperty('webdriver'),
  plugins_len: navigator.plugins.length,
  languages: navigator.languages,
  availWidth: screen.availWidth, availHeight: screen.availHeight,
  outerWidth: window.outerWidth, outerHeight: window.outerHeight,
  hasDOE: typeof DeviceOrientationEvent !== 'undefined',
  mp3: document.createElement('video').canPlayType('audio/mp3'),
  mp4: document.createElement('video').canPlayType('video/mp4; codecs="avc1.42E01E, mp4a.40.2"'),
  ls_nano_fp: localStorage.getItem('_nano_fp'),
  ls_anti_pre: localStorage.getItem('anti_pre_sig'),
  installTrigger: typeof InstallTrigger !== 'undefined',
}))()
```

---

## 关键经验总结

### 1. "复制型"请求头不是生成点
`anti-content` 头是拦截器从 body 复制的；见到头先找 `o.anti_content||o.antiContent`
这类读取模式，生成点在更上游的业务调用链。

### 2. 重放实验是隔离变量的终极手段
签名对不对，把浏览器刚生成的签名原样从脚本发一次即可二分：
本案例证明签名正确、被标记的是传输层——避免了在签名上浪费数天。

### 3. JSVMP 打包器源码指纹
`pbc(T.toString()+ct.toString())` 模式（源码入包）意味着"提取+重写"路线必死，
只能整段原样执行（vm/jsdom）。遇到 `toString()` 拼接进打包数据的迹象立即转路径 B。

### 4. 首次放行 ≠ 方案稳定
风控对首次接触放行（无评分），连续请求才上评分。验证期必须：单发、间隔、
遇 54001 立即停手（重试会续标记），不要用循环硬撞。

---

## 重大验证结论（2026-09-01 补充）

用本案例生成器（`0at` 格式）请求**免登录低风控接口**
`GET https://apiv2.pinduoduo.com/api/gindex/tf/query_tf_goods_info?tf_id=...&anti_content=<生成值>`
→ **HTTP 200 + error_code:1000000 + 50 条真实商品**。

三条推论：
1. 生成器还原**彻底正确**，被线上服务端接受。
2. Python `curl_cffi firefox147` 传输**未被全局封禁**（同 IP 同客户端，打 apiv2 通过）；
   `proxy/api/search` 的 54001 是**登录态搜索接口特有的额外门槛**。
3. 搜索接口与免登录接口的最大差异是浏览器恒带 `rc-res-conf`（网关加密头，
   react_pdd 的 `_gatewayEncryptionInterceptorPromise` 生成），本方案未复现，
   是突破 `proxy/api/search` 54001 的头号嫌疑与下一步方向。

**决定性反证（2026-09-01 补充2）**：真实浏览器请求同时携带 `anti-content` +
`rc-res-conf` + 真实 TLS，搜索接口**仍**返回 54001。
→ 证明 `rc-res-conf` **不是** 54001 的解锁条件；54001 是**账号级风控分**，
  密集请求会持续加分直至连真实浏览器也被拦截。
→ 正确处置：立即停手等衰减 / 等真实滑块人工过；**不要**靠补 rc-res-conf 解锁。
→ rc-res-conf 仅适合作为"请求头补全"的完整性工作（网关加密，每请求变化，
  需同时实现 handleRequest 加密与 handleResponse 解密）。

**最终结论（2026-09-02 补充3，风控衰减一天后）**：账号解锁后**浏览器**搜索恢复
200+真实数据，但**所有**脚本客户端变体仍 54001，已穷尽：
firefox135/144/147、chrome116、UA↔TLS 同版本一致、0at/0aq 两版签名、
轮换 Cookie、新 pdd_vds、10 分钟/24 小时冷却。
→ 搜索接口（高风控）做了**账号-可信设备绑定**：只认建立信任的真实浏览器 TLS，
  拒绝一切脚本客户端，与 anti-content 正确性无关（apiv2 免登录接口证明签名正确）。
→ 这是**设计内的独立防线**，无法通过改进签名/补头突破。
→ 取搜索数据的现实路径：真实浏览器 / 未建立信任的干净账号+IP（首请宽限）/
  改用免登录 apiv2 接口。

**完整判定矩阵（2026-09-02 补充4，换账号实验后）**：

| 客户端 | 账号 | IP | 搜索接口结果 |
|---|---|---|---|
| 真实浏览器 | 旧(已衰减) | 本 IP | 200 |
| 真实浏览器 | **新** | 本 IP | 200 + 100KB |
| 脚本 curl_cffi | 旧 | 本 IP | 54001 |
| 脚本 curl_cffi | **新** | 本 IP | **54001(首请即拦,无宽限)** |
| 脚本 curl_cffi | 任意 | 本 IP | apiv2 免登录接口 200 |

→ 新账号+浏览器同 IP 正常、新账号+脚本同 IP 首请即拦：风控对
  "脚本 TLS 类别 × 有脚本流量历史的 IP" 零宽限；账号无全局黑名单。

**决定性 TLS 对照实验（2026-09-02 补充5，浏览器内重放）**：
把我方生成的 anti-content + body **原样**在浏览器里 `fetch` 发出
（真 Firefox TLS，同账号同 IP）→ **200 + 20 条商品**；
同样内容经 Python curl_cffi 发 → 54001。
→ 唯一变量=传输层：**搜索接口做 TLS/HTTP2 客户端指纹识别**，与 anti-content 无关。
→ 推翻"换干净 IP 可解锁脚本"的推断：判别因子是 TLS 指纹本身，换 IP 救不了 Python 栈。
→ 结论：anti-content 还原对新账号完全正确；搜索接口脚本化必须满足
  浏览器级 TLS（真实浏览器 / 精确复刻 Firefox152 指纹的栈），否则只能走 apiv2。

> 旁证（第三方）：
> - 2022 腾讯云文章（mms 后台，`messagePack()` 无 antiVersion）
> - gitbenxing/anti-content（2023，`0aq` 前缀，双层 webpack 提取 + execjs）
> - 2026-04 CSDN 文章（`0ar` 前缀，Proxy 拦截网测出"仅 24 个属性"，
>   踩坑：必须提供 `Element` 构造函数、`navigator.hasOwnProperty` 用
>   `Object.prototype.hasOwnProperty` 避免递归、`location.port` 必须 `''`）
> 前缀版本谱系：`0aq`(2023) → `0ar`(2026-04) → `0at`(2026-09 当前,带 `antiVersion:"v2"`)

---

## 可验证事实清单

- [ ] `anti-content` 以 `0at` 开头，自定义 base64url（无 `=`）
- [ ] **生成值可过 `apiv2.pinduoduo.com/api/gindex/tf/query_tf_goods_info`（免登录，200+数据）——生成器正确性的黄金验证口**
- [ ] 拦截器复制：`react_pdd` 中 `(a=o&&(o.anti_content||o.antiContent))&&(n["anti-content"]=a)`
- [ ] 签名入口：模块 53636 `messagePackSync({antiVersion:"v2"})`，SDK 单例导出
- [ ] JSVMP magic：字节码头 4 字节 `JVM2`，program#3 名 `getMessage`
- [ ] `_stm` 接口返回 `{"server_time":毫秒}` 免鉴权
- [ ] `pdd_vds`/`PDDAccessToken` 随浏览器导航轮换
- [ ] 风控码：40001（传输层拒绝）/ 54001（需验证+verify_auth_token）
- [ ] 反调试位要求 `navigator.webdriver` 原型链存在且非自有属性
