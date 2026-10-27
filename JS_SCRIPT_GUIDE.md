# mytv JS 脚本使用说明

内置 QuickJS 引擎，无外挂运行时

**先分清两个 JS 功能，它们互不相干：**

| 功能 | 输入框 | 文档 |
|---|---|---|
| **`js://` 脚本源** | 直播源地址 / EPG 地址 / 频道线路地址 | 本文 |
| **转换JS** | 添加直播源时的「转换JS」 | `TRANSFORM_JS_GUIDE.md` |

两者共用同一个引擎和全部 `mytv.*` 函数，所以本文的函数手册对两者都适用。

---

## 1. 五分钟跑起来

### 1.1 最小可运行脚本

```javascript
// 直播源脚本：返回一段 M3U 正文
export default function main(item) {
    return `#EXTM3U
#EXTINF:-1 tvg-id="cctv1" group-title="央视",CCTV-1
http://example.com/live/cctv1.m3u8
#EXTINF:-1 tvg-id="cctv2" group-title="央视",CCTV-2
http://example.com/live/cctv2.m3u8`;
}
```

保存成 `demo.js`，在「添加直播源」里填地址 → 保存 → 刷新即可看到 2 个频道。

### 1.2 地址怎么写

地址用 `js://` 开头，后面接脚本位置：

| 写法 | 含义 |
|---|---|
| `js://http://your.site/demo.js` | **网络脚本**：App 先下载再执行，地址上的 `?query` 会平铺进 `item` |
| `js://https://your.site/demo.js?id=ch1&token=abc` | 带参数：脚本里可直接用 `item.id`、`item.token` |
| `js://file:///storage/emulated/0/mytv/js/demo.js` | **本地脚本**：`file://` 后面必须是**裸绝对路径** |
| `js://PLTV/http://your.site/demo.js` | 回看/时移标记（`PLTV` 可换 `TVOD`/`TS`/`CATCHUP`/`SHIYI`，大小写不限），只是标记，不参与路径 |

- 本地脚本目录**由你自己在设备上创建**，App 只读、不建目录不写文件。推荐 `/storage/emulated/0/mytv/js/`。
- ⚠️ 本地脚本**只认 `js://file://` 绝对路径**，`js://demo.js` 这种相对路径写法已不支持（会提示「无法解析脚本源」），另外首次使用APP需要设置 → 权限 → 开「管理全部文件 / 读取外部存储」。
- 脚本是「列表脚本」还是「EPG 脚本」还是「播放脚本」，**由你把它填在哪个框决定**，不用写进地址。

### 1.3 入口约定

| 项 | 规定 |
|---|---|
| 入口函数 | `main(item)`，ESM 用 `export default` 导出 |
| `item` | JSON 对象：`item.url` = 完整地址，地址上每个 query 参数都是 `item` 的顶层键 |
| 返回值 | 必须是 JSON 可序列化的值（字符串 / 对象 / 数组），**不能返回函数或 undefined** |

```javascript
export default function main(item) {
    console.log(item.url);        // "js://https://a.com/p.js?id=ch1"
    console.log(item.id);         // "ch1"
    return '';
}
```

---

## 2. 两种写法：ESM（推荐）/ CJS（兼容）

引擎开工前会看源码：**行首**出现 `import` 或 `export` 就按 ESM 跑，否则按旧脚本（CJS）跑。两种都能用，**新脚本一律推荐 ESM**。

### 2.1 同一个脚本，两种写法

**ESM（推荐）**

```javascript
import CryptoJS from 'crypto';

const ku9 = globalThis.ku9;              // 宿主对象不从模块导入，取一次即可
const KEY = CryptoJS.enc.Utf8.parse('0123456789abcdef');

export default function main(item) {
    const body = ku9.get(`https://api.example.com/cfg?id=${item.id}`, {});
    const plain = CryptoJS.AES.decrypt(
        CryptoJS.lib.CipherParams.create({ ciphertext: CryptoJS.enc.Base64.parse(body) }),
        KEY,
        { mode: CryptoJS.mode.ECB, padding: CryptoJS.pad.Pkcs7 },
    ).toString(CryptoJS.enc.Utf8);
    return JSON.parse(plain).url;
}
```

**CJS / Ku9 风格（同样能跑，不用改）**

```javascript
function main(item) {
    // 旧写法：宿主对象本来就是全局的，ku9 / mytv 直接用，不需要声明
    var CryptoJS = require('crypto');                // require 引同一个库
    var KEY = CryptoJS.enc.Utf8.parse('0123456789abcdef');
    var body = ku9.get('https://api.example.com/cfg?id=' + item.id, {});
    var plain = CryptoJS.AES.decrypt(
        CryptoJS.lib.CipherParams.create({ ciphertext: CryptoJS.enc.Base64.parse(body) }),
        KEY,
        { mode: CryptoJS.mode.ECB, padding: CryptoJS.pad.Pkcs7 },
    ).toString(CryptoJS.enc.Utf8);
    return JSON.parse(plain).url;
}
```

### 2.2 写法差异速查

| | ESM（推荐） | CJS / 旧脚本（兼容） |
|---|---|---|
| 引库 | `import CryptoJS from 'crypto'` | `var CryptoJS = require('crypto')` |
| 入口 | `export default function main(item)` | `function main(item)` |
| 宿主对象 | `const ku9 = globalThis.ku9` | 直接用 `ku9` / `mytv` |
| 执行方式 | 按模块求值，严格模式 | 套垫片按**非严格模式**执行 |
| 顶层 `this` | `undefined` | 全局对象 |
| 未声明赋值 `x = 1` | ❌ `ReferenceError` | ✅ 隐式全局 |
| 被引模块 | 编译成字节码，**进程内只编译一次** | 每次 `require` 重新执行模块体 |

**Ku9 的 CJS 脚本能不能直接跑？** 能，原样放进来就行 —— 引擎会自动套一层垫片（`new Function` 执行、非严格模式、补齐 `module`/`exports`），Ku9 的 `function main(item)` + `require('crypto')` 写法零改动可用。唯一的差别是报错行号有时会显示成 `eval` 或匿名帧。

### 2.3 为什么推荐 ESM

1. **模块会编译成字节码并缓存**。被 `import` 的库（如 `crypto.js` 47 KB）在 App 生命周期内只编译一次，之后直接装载字节码。切频道反复跑脚本时，这个差别很明显。入口脚本本身不享受这层缓存，所以**入口里不写 `import` 就等于白拿不到这个收益**。
2. **依赖一目了然**。`import` 全在文件头；`require` 是运行时函数调用，藏在分支里也能跑，出问题不好定位。
3. **模块作用域干净**。ESM 顶层声明不会漏成全局，多个脚本互不污染。

### 2.4 从 CJS 迁移，三步

1. `var CryptoJS = require('crypto')` → 挪到文件最上面写成 `import CryptoJS from 'crypto'`；
2. `function main(item) {...}` → `export default function main(item) {...}`；
3. 顶层所有未声明的变量补 `const` / `let`（ESM 是严格模式，漏了会 `ReferenceError`）。

### 2.5 写 ESM 的三条硬规矩

```javascript
// ❌ 错：ESM 严格模式下未声明赋值直接报错
foo = 1;

// ✅ 对
const foo = 1;

// ✅ 确实要挂全局，显式写
globalThis.foo = 1;
```

- 顶层 `this` 是 `undefined`，不是全局对象 —— 依赖顶层 `this` 的 UMD 老库会炸，这种库请用内置的 `crypto` / `jsencrypt`（它们走垫片，内部不受影响）。
- 顶层 `await` 不支持（不是 async 上下文）。
- 入口文件里不要用 `with`、不要写八进制字面量 `0755`、不要重复声明同名参数 —— 这些都是严格模式禁止的。

---

## 3. 宿主函数手册

宿主对象同时挂在 **`mytv`** 和 **`ku9`** 两个名字上，指向同一个对象，写哪个都一样。下面统一用 `mytv.*`。

> 参数缺省口径：字符串参数缺失返回 `null`，数字参数缺失按 `0`。

### 3.1 网络请求

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.get` | `get(url, headers?)` | 响应体字符串，失败返回 `""` |
| `mytv.post` | `post(url, headers?, body?)` | 响应体字符串 |
| `mytv.request` | `request(url, method?, headers?, body?, redirect?)` | `{ furl, code, headers, body }` |
| `mytv.getHeaders` | `getHeaders(url, headers?, redirect?, method?, body?)` | 响应头的 **JSON 字符串** |

```javascript
// 最常用：拿正文
const body = mytv.get('https://api.example.com/list', { 'User-Agent': 'okhttp/3.12.11' });

// POST 表单：Content-Type 从 headers 里取
const ret = mytv.post('https://api.example.com/login',
    { 'Content-Type': 'application/x-www-form-urlencoded' },
    'user=abc&pass=123');

// 需要状态码 / 重定向后地址
const res = mytv.request('https://api.example.com/a', 'GET', {}, '', true);
if (res.code === 200) {
    console.log(res.furl);          // 重定向后的最终地址
    console.log(res.body);
    console.log(res.headers['content-type']);
}
// redirect 传 false 表示不跟随重定向（要拿 302 的 Location 时用）

// 只要响应头
const head = mytv.getHeaders('https://cdn.example.com/a.m3u8', {}, true, 'GET');
const h = JSON.parse(head);         // { "Content-Type": "...", "Server": "..." }
```

- `headers` 传 `null` 或 `{}` 都行；返回的是**完整响应体字符串**，不做 JSON 解析，需要自己 `JSON.parse`。
- 请求失败（DNS / 超时 / TLS）**不抛异常**，`get`/`post` 返回空串，`request` 返回 `code = 0`。

### 3.2 缓存

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.setCache` | `setCache(key, value, timeMs)` | `true` / `false` |
| `mytv.getCache` | `getCache(key)` | 字符串或 `null`（不存在 / 已过期） |

```javascript
// 缓存 10 分钟
mytv.setCache('cfg_v1', JSON.stringify(cfg), 10 * 60 * 1000);

// 0 = 永久有效
mytv.setCache('token', 'abc', 0);

// 读缓存
const cached = mytv.getCache('cfg_v1');
if (cached) {
    cfg = JSON.parse(cached);
}
```

- 缓存是**进程级**的，所有脚本共享同一张表，进程退出即清空。
- `value` 必须是字符串；存对象先 `JSON.stringify`。
- 缓存**跨脚本共享**，key 建议带脚本前缀（如 `ysptp_cfg_v1`）避免撞车。

### 3.3 URL 与参数

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.getQuery` | `getQuery(url, name)` | 参数值字符串或 `null` |
| `mytv.uri` | `uri(url)` | 9 个字段的解析对象 |

```javascript
const id = mytv.getQuery(item.url, 'id');   // 等价于 item.id

const u = mytv.uri('http://site.com/a/b.m3u8?id=1&c=k');
u.Scheme;      // "http"
u.Host;        // "site.com"
u.Port;        // 80（没写端口为 -1）
u.Path;        // "/a/"（目录部分，以 / 结尾，不含文件名）
u.FullPath;    // "/a/b.m3u8"（完整路径，含文件名）
u.Query;       // "id=1&c=k"
u.Fragment;    // ""
u.UserInfo;    // ""（有账号密码时是 "user:pass"）
u.Params;      // { id: "1", c: "k" } —— 已经是对象，直接 u.Params.id

// 典型用法：把相对地址拼成绝对地址
function toAbs(base, rel) {
    if (/^https?:\/\//i.test(rel)) return rel;
    const b = mytv.uri(base);
    if (rel.charAt(0) === '/') return b.Scheme + '://' + b.Host + rel;
    return base.substring(0, base.lastIndexOf('/') + 1) + rel;
}
```

### 3.4 编解码

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.encodeBase64` | `encodeBase64(str)` | Base64 字符串 |
| `mytv.decodeBase64` | `decodeBase64(str)` | UTF-8 字符串，失败 `null` |
| `mytv.isBase64` | `isBase64(str)` | `boolean` |
| `mytv.isJsonObject` | `isJsonObject(str)` | `boolean` |
| `mytv.isJsonArray` | `isJsonArray(str)` | `boolean` |

```javascript
const b64 = mytv.encodeBase64('hello');   // "aGVsbG8=\n"
const txt = mytv.decodeBase64(b64);       // "hello"

// ⚠️ encodeBase64 用的是 Android DEFAULT 模式：每 76 字符换行、结尾补 \n
//    拼进 URL / Header 前必须自己清掉空白
const clean = b64.replace(/\s+/g, '');

mytv.isBase64('aGVsbG8=');        // true
mytv.isJsonObject('{"a":1}');     // true
mytv.isJsonArray('[1,2]');        // true
```

### 3.5 哈希

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.md5` | `md5(data)` | 32 位小写十六进制 |
| `mytv.sha1` | `sha1(data)` | 40 位小写十六进制 |
| `mytv.sha256` | `sha256(data)` | 64 位小写十六进制 |
| `mytv.sha512` | `sha512(data)` | 128 位小写十六进制 |

```javascript
const sign = mytv.md5('abc' + secret);          // "900150983cd24fb0d6963f7d28e17f72"
const h = mytv.sha256('abc');
```

### 3.6 加解密

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.opensslEncrypt` | `opensslEncrypt(data, algo, key, outType?, iv?)` | 密文字符串 |
| `mytv.opensslDecrypt` | `opensslDecrypt(data, algo, key, inType?, iv?)` | 明文，失败返回 `"opensslDecrypt Error：..."` |
| `mytv.rc4Encrypt` / `mytv.rc4Decrypt` | `(data, key, inType, outType, charset?)` | 字符串 |
| `mytv.encrypt3DES` / `mytv.decrypt3DES` | `(data, key, outType, inType)` | 字符串 |

- `algo` 写 openssl 风格串：`算法-位数-模式`，如 `AES-256-CBC`、`AES-128-ECB`、`DES-EDE3-CBC`。
- `outType` / `inType`：`0` = Base64（默认），`1` = HEX。
- `key` 直接用原始 UTF-8 字节：不足位数会返回错误，超长自动截断。

```javascript
const key = 'ilyB29ZdruuQjC45JhBBR7o2Z8WJ26Vg';
const iv  = 'JUMxvVMmszqUTeKn';

// AES-256-CBC，Base64 进 Base64 出
const plain = mytv.opensslDecrypt(cipherB64, 'AES-256-CBC', key, 0, iv);
const back  = mytv.opensslEncrypt(plain,     'AES-256-CBC', key, 0, iv);

// HEX 进出
const plain2 = mytv.opensslDecrypt(cipherHex, 'AES-128-ECB', key16, 1);

// RC4：inType / outType 含义同上，charset 缺省 UTF-8
const rc4out = mytv.rc4Encrypt('data', 'key', 0, 1);      // 输出 HEX
const rc4raw = mytv.rc4Decrypt(rc4out, 'key', 1, 0);      // HEX 进，字符串出

// 3DES：算法固定 DESede/ECB、无 IV
// ⚠️ 第 4 参语义不对称：
//    encrypt 时 4=1 表示「输入是 HEX」；decrypt 时 4=0 表示「输入是 HEX」
//    第 3 参 = 1 表示输出 HEX，否则按 charset 转字符串
const d3 = mytv.encrypt3DES(data, key24, 0, 1);
const p3 = mytv.decrypt3DES(d3,   key24, 0, 1);
```

> 原生 OpenSSL 实现，几百 KB 的正文也能瞬间解完。**大正文优先用它**，别用纯 JS 的 CryptoJS。

### 3.7 时间

| 函数 | 签名 | 返回 |
|---|---|---|
| `mytv.toTimestamp` | `toTimestamp(dateStr, inFormat, timezone?)` | 13 位毫秒时间戳，失败 `0` |
| `mytv.toDate` | `toDate(timestamp, outFormat, timezone?)` | 日期字符串，失败 `null` |
| `mytv.formatDateTime` | `formatDateTime(str, inFormat, outFormat, addDays?, inTz?, outTz?)` | 转换后的字符串 |

时区用 IANA 名称（`Asia/Shanghai`），缺省取设备时区。

```javascript
const ts = mytv.toTimestamp('2024-10-23', 'yyyy-MM-dd', 'Asia/Shanghai');
const ds = mytv.toDate(1698019200000, 'yyyy-MM-dd HH:mm:ss', 'Asia/Shanghai');

// 日期格式转换 + 加 1 天
const out = mytv.formatDateTime('2024-10-23 21:23:00', 'yyyy-MM-dd HH:mm:ss',
                                'yyyy-MM-dd', 1, 'Asia/Shanghai', 'Asia/Shanghai');
// → "2024-10-24"

// 常用：算节目开始/结束（秒级时间戳给 EPG 用）
const startSec = Math.floor(mytv.toTimestamp(today, 'yyyy-MM-dd') / 1000);
```

### 3.8 WebSocket

`mytv.websocket()` 返回**管理器**（全局单例），连接在脚本销毁后依然存活。

| 管理器方法 | 说明 |
|---|---|
| `connect(url, headers?, name?)` | 建连接，返回连接对象；`name` 缺省用 url |
| `has(name)` | 是否已存在该连接，返回 `boolean` |
| `get(name)` | 取已有连接，不存在会抛错 |

| 连接对象方法 | 说明 |
|---|---|
| `send(text)` | 发文本 |
| `send(hexStr, true)` | 发二进制（hex 字符串） |
| `close(code, reason)` | 关闭 |
| `task(name, msg, intervalMs, hex?)` | 定时发送任务 |
| `removetask(name)` | 取消定时任务 |
| `message()` | 已收消息**数组快照**（最多最近 200 条） |
| `status()` | 任务状态码 |
| `log()` | 任务日志（最多最近 32 KB） |

```javascript
const ws = mytv.websocket();
const conn = ws.connect('wss://push.example.com/live', { 'User-Agent': 'okhttp' }, 'live');
conn.send('{"cmd":"sub"}');
conn.task('hb', 'ping', 30000);

// 没有入站回调，靠轮询取消息
const msgs = conn.message();
if (msgs.length) {
    handle(msgs[msgs.length - 1]);
}

conn.removetask('hb');
conn.close(1000, 'done');
```

> 引擎没有事件循环、没有 `setTimeout`，所以入站消息只能**主动轮询** `message()`，不会推给你。

### 3.9 模块加载

`import`（ESM）和 `require()`（CJS，也是全局函数）解析**同一套模块**，查找顺序：

1. 内置库 `crypto` / `jsencrypt`（与脚本位置无关，优先命中）；
2. 远程脚本：按**脚本 URL 所在目录**拼相对路径；
3. 本地脚本：按**脚本自身所在目录**拼相对路径，`../` 可用、缺 `.js` 自动补；
4. 同目录找不到，回退 `<脚本根>/js/<name>.js`；
5. `http(s)://` 开头的直接远程拉取。

```javascript
// ESM
import CryptoJS from 'crypto';        // 内置
import JSEncrypt from 'jsencrypt';    // 内置
import { helper } from './lib/helper'; // 同目录 lib/helper.js

// CJS
var CryptoJS  = require('crypto');
var JSEncrypt = require('jsencrypt');
var helper    = require('lib/helper');
```

- `import CryptoJS from 'crypto'` 和 `'crypto.js'` 等价（解析时先去掉 `.js`）。
- 内置的 `crypto.js` / `jsencrypt.js` 本身是 UMD/CJS，引擎会自动垫成 ESM 的 `default`，`import` 拿到的就是它导出的那个对象，不用写适配。
- ⚠️ **模块实例不缓存**：每次 `require` / 每次执行都会重新跑一遍模块体。模块里有网络请求或 `setCache` 副作用的话会重复触发 —— 这类逻辑请放到 `main()` 里自己控制。

---

## 4. 返回值怎么写

同一个 `main()` 返回值，按你把它填在哪个入口来解析。

### 4.1 直播源（列表脚本）

```javascript
// ① 直接返回正文（最常用）
return m3uOrTxtOrJsonText;

// ② 包一层 data
return { data: m3uText };

// ③ 返回地址，让 App 自己拉
return { url: 'http://site.com/list.m3u' };
```

### 4.2 EPG（EPG 脚本）

```javascript
return xmlText;                       // XML 正文
return { data: xmlText };
return { url: 'http://site.com/epg.xml' };
```

> `data` 必须是**字符串正文**。返回对象（如 Ku9 的 DIYP/SPTV JSON 形态）会被判「脚本返回格式不正确」。

### 4.3 播放地址（频道线路脚本）

```javascript
// ① 纯地址
return 'https://cdn.example.com/live.m3u8';
return { url: 'https://cdn.example.com/live.m3u8' };

// ② 带请求头（对播放与所有切片生效）
return {
    url: 'https://cdn.example.com/live.m3u8',
    headers: { 'User-Agent': 'okhttp/3.12.11', 'Referer': 'https://example.com/' },
};

// ③ 域名重写：把播放地址的域名解析到指定 IP（仅 Media3 内核生效）
return { url, host: '127.0.0.1' };
return { url, host: { 'cdn.example.com': '10.0.0.1', 'img.example.com': '10.0.0.2' } };

// ④ 直接返回 m3u8 正文：本地路由会以 application/x-mpegURL 回给播放器
return { m3u8: '#EXTM3U\n#EXT-X-VERSION:3\n...', headers };

// ⑤ 内容暂时不可用：指定状态码，让播放器按 Retry-After 重试
return { status: 503, headers: { 'Retry-After': '5' } };

// ⑥ 直接报错：返回 error 时 App 按错误处理并提示
return { error: 'token 已过期' };
```

脚本返回的 `headers` 会被合并进线路请求头，并在每次 HLS 刷新 manifest 时自动重套最新值 —— 所以**签名类 header 每次现算就行**，不用自己缓存。

---

## 5. 内置 JS 库

两个库都随 App 打包，`import` 优先命中，与脚本放在哪无关。

### 5.1 `crypto` —— CryptoJS（全量构建）

纯 JS 实现。**短字符串（配置、签名、token、密钥）用它**；几百 KB 以上的正文请改用 `mytv.opensslDecrypt`（原生）。

#### 命名空间总览

| 命名空间 | 内容 |
|---|---|
| `CryptoJS.enc` | 编码器：`Hex` `Latin1` `Utf8` `Utf16` `Utf16BE` `Utf16LE` `Base64` |
| `CryptoJS.lib` | `WordArray` `CipherParams` `Base` `BufferedBlockAlgorithm` |
| `CryptoJS.algo` | 各算法实现（一般不直接用） |
| `CryptoJS.mode` | `CBC` `ECB` `CFB` `OFB` `CTR` `CTRGladman` |
| `CryptoJS.pad` | `Pkcs7` `ZeroPadding` `AnsiX923` `Iso10126` `Iso97971` `NoPadding` |
| `CryptoJS.kdf` | `OpenSSL`（EVP_BytesToKey） |
| `CryptoJS.x64` | `Word`（64 位字，SHA512/SHA3 内部用） |

#### 一、编码器 `CryptoJS.enc.*`

每个编码器都有两个方法：`parse(str)` → `WordArray`，`stringify(wordArray)` → `str`。

```javascript
const wa = CryptoJS.enc.Utf8.parse('abc');           // 字符串 → WordArray
const s1 = CryptoJS.enc.Base64.stringify(wa);        // "YWJj"
const s2 = CryptoJS.enc.Hex.stringify(wa);           // "616263"
const s3 = wa.toString(CryptoJS.enc.Base64);         // 等价写法

const wa2 = CryptoJS.enc.Base64.parse('YWJj');
const back = wa2.toString(CryptoJS.enc.Utf8);        // "abc"

// 各编码器对照
CryptoJS.enc.Utf8.parse('中文').toString(CryptoJS.enc.Hex);      // UTF-8 字节的 hex
CryptoJS.enc.Latin1.parse('\x61\x62');                           // 单字节编码
CryptoJS.enc.Utf16.parse('ab');                                  // UTF-16（含 BOM，小端）
CryptoJS.enc.Utf16LE.parse('ab');
CryptoJS.enc.Utf16BE.parse('ab');
```

#### 二、`WordArray`

```javascript
// 构造
const w = CryptoJS.lib.WordArray.create([0x61626300], 3);   // 直接给 words + 有效字节数
const r = CryptoJS.lib.WordArray.random(16);                // 16 字节随机（IV 用）

// 属性与方法
w.words;        // Uint32 数组
w.sigBytes;     // 有效字节数
w.toString(CryptoJS.enc.Hex);
w.clone();
w.concat(other);
w.clamp();      // 按 sigBytes 截断末尾多余字节
```

#### 三、摘要（8 个）

`MD5` `SHA1` `SHA224` `SHA256` `SHA384` `SHA512` `SHA3` `RIPEMD160`

```javascript
CryptoJS.MD5('abc').toString();        // "900150983cd24fb0d6963f7d28e17f72"
CryptoJS.SHA1('abc').toString();
CryptoJS.SHA256('abc').toString();     // 64 位 hex
CryptoJS.SHA512('abc').toString();
CryptoJS.SHA3('abc', { outputLength: 512 }).toString();   // outputLength: 224/256/384/512
CryptoJS.RIPEMD160('abc').toString();

// 也可以喂 WordArray（对二进制摘要）
CryptoJS.SHA256(CryptoJS.enc.Hex.parse('deadbeef')).toString();
```

#### 四、HMAC（8 个）

`HmacMD5` `HmacSHA1` `HmacSHA224` `HmacSHA256` `HmacSHA384` `HmacSHA512` `HmacSHA3` `HmacRIPEMD160`

```javascript
CryptoJS.HmacSHA256('message', 'secret').toString();       // hex
CryptoJS.HmacSHA1('message', CryptoJS.enc.Utf8.parse('secret')).toString();
// 第三个参数可指定输出长度（仅 SHA3 有意义）
CryptoJS.HmacSHA3('m', 'k', { outputLength: 256 });
```

#### 五、对称加解密（6 个）

`AES` `DES` `TripleDES` `RC4` `RC4Drop` `Rabbit`

```javascript
// 统一签名：Algo.encrypt(message, key, cfg) → CipherParams
//           Algo.decrypt(cipherParams, key, cfg) → WordArray
```

**🔴 最重要的坑：key / iv 必须先 `enc.Utf8.parse` 成 WordArray。**
直接传字符串会被 CryptoJS 当成「口令」走 EVP_KDF 派生，和宿主 `mytv.opensslDecrypt`（拿原始字节当密钥）算出来的完全不是一回事，解密必然失败。

```javascript
const KEY = CryptoJS.enc.Utf8.parse('0123456789abcdef');   // 16 字节 = AES-128
const IV  = CryptoJS.enc.Utf8.parse('abcdef0123456789');

// 解密（Base64 密文 → 明文）
const plain = CryptoJS.AES.decrypt(
    CryptoJS.lib.CipherParams.create({ ciphertext: CryptoJS.enc.Base64.parse(cipherB64) }),
    KEY,
    { iv: IV, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.Pkcs7 },
).toString(CryptoJS.enc.Utf8);

// 加密（明文 → Base64 密文）
const encB64 = CryptoJS.AES.encrypt(CryptoJS.enc.Utf8.parse(plain), KEY,
    { iv: IV, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.Pkcs7 })
    .ciphertext.toString(CryptoJS.enc.Base64);

// ECB：不要传 iv
const ecb = CryptoJS.AES.encrypt('text', KEY, { mode: CryptoJS.mode.ECB }).toString();

// 密文对象：encrypt 的返回值就是 CipherParams
const cp = CryptoJS.AES.encrypt('t', KEY, { iv: IV });
cp.ciphertext;   // WordArray
cp.key; cp.iv; cp.algorithm; cp.mode; cp.padding; cp.formatter;

// CTR / CFB / OFB：与 CBC 同写法，注意 CTR 不需要 padding
CryptoJS.AES.decrypt(cp, KEY, { iv: IV, mode: CryptoJS.mode.CTR, padding: CryptoJS.pad.NoPadding });

// DES / 3DES：块长 8 字节，iv 也必须是 8 字节
const dk  = CryptoJS.enc.Utf8.parse('12345678');                 // DES 8 字节
const dIv = CryptoJS.enc.Utf8.parse('abcdefgh');
CryptoJS.DES.encrypt('t', dk, { mode: CryptoJS.mode.ECB }).toString();
const tk = CryptoJS.enc.Utf8.parse('123456781234567812345678');  // 3DES 24 字节
CryptoJS.TripleDES.encrypt('t', tk, { mode: CryptoJS.mode.CBC, iv: dIv }).toString();

// RC4（流加密，无 iv / 无 padding）
const rk = CryptoJS.enc.Utf8.parse('key');
CryptoJS.RC4.encrypt('t', rk).toString();
CryptoJS.RC4Drop.encrypt('t', rk, { drop: 768 }).toString();
```

> 解密失败不是返回 `null`，而是**抛异常**（`toString(Utf8)` 时校验填充失败）。务必 `try/catch`。

#### 六、KDF（2 个）

```javascript
// PBKDF2
const dk = CryptoJS.PBKDF2('password', CryptoJS.enc.Utf8.parse('salt'), {
    keySize: 256 / 32,      // 单位是 32 位字，256 bit = 8
    iterations: 1000,
    hasher: CryptoJS.algo.SHA256,   // 缺省 SHA1
});
dk.toString();              // hex

// EvpKDF（OpenSSL EVP_BytesToKey）
CryptoJS.EvpKDF('password', 'salt', { keySize: 256 / 32, iterations: 1 }).toString();

// OpenSSL 兼容的格式化/解析（带 Salted__ 头）
CryptoJS.kdf.OpenSSL.execute('password', 256 / 32, 128 / 32, salt);
```

#### 七、一个完整的 AES 解密 + 缓存示例

```javascript
import CryptoJS from 'crypto';

const ku9 = globalThis.ku9;
const KEY = CryptoJS.enc.Utf8.parse('0123456789abcdefg123456789abcdef');
const IV  = CryptoJS.enc.Utf8.parse('abcdef0123456789');

function decrypt(b64) {
    try {
        return CryptoJS.AES.decrypt(
            CryptoJS.lib.CipherParams.create({ ciphertext: CryptoJS.enc.Base64.parse(b64) }),
            KEY,
            { iv: IV, mode: CryptoJS.mode.CBC, padding: CryptoJS.pad.Pkcs7 },
        ).toString(CryptoJS.enc.Utf8);
    } catch (e) {
        return null;
    }
}

export default function main(item) {
    const cached = ku9.getCache('cfg_v1');
    if (typeof cached === 'string' && cached.length) {
        try { return JSON.parse(cached)[item.id].url; } catch (e) { /* 缓存坏了走远程 */ }
    }
    const cfg = JSON.parse(decrypt(ku9.get('https://api.example.com/cfg', {})));
    ku9.setCache('cfg_v1', JSON.stringify(cfg), 10 * 60 * 1000);
    return cfg[item.id].url;
}
```

### 5.2 `jsencrypt` —— JSEncrypt（RSA）

导出的是**构造函数**，必须 `new`。内部用纯 JS 做大数运算。

#### 构造

```javascript
const rsa = new JSEncrypt({
    default_key_size: 1024,          // 生成密钥对时的位数，缺省 1024（可选 512/1024/2048/4096）
    default_public_exponent: '010001', // 65537
    log: false,                      // 是否打印调试日志
});
```

#### 方法总表

| 方法 | 说明 | 返回 |
|---|---|---|
| `setKey(key)` | 自动识别公钥/私钥 | `void` |
| `setPublicKey(pemOrB64)` | 设公钥（可带或不带 PEM 头） | `void` |
| `setPrivateKey(pemOrB64)` | 设私钥 | `void` |
| `encrypt(str)` | 公钥加密 | Base64 密文或 `false` |
| `decrypt(b64)` | 私钥解密 | 明文字符串或 `false` |
| `sign(str, digest)` | 私钥签名，`digest` = `'md5'`/`'sha1'`/`'sha256'`/`'sha512'`/`'ripemd160'` | Base64 签名或 `false` |
| `verify(str, sig, digest)` | 公钥验签 | `boolean` |
| `getKey()` | 当前密钥的 `n` / `e` | 对象 |
| `getPublicKey()` | 公钥（hex） | 字符串 |
| `getPublicKeyB64()` | 公钥（PEM Base64） | 字符串 |
| `getPrivateKey()` | 私钥（hex） | 字符串 |
| `getPrivateKeyB64()` | 私钥（PEM Base64） | 字符串 |

#### 示例

```javascript
import JSEncrypt from 'jsencrypt';

const PUB = `-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC...（你的公钥）
-----END PUBLIC KEY-----`;

// ① 公钥加密（示例：登录密码加密）
const rsa = new JSEncrypt();
rsa.setPublicKey(PUB);
const cipher = rsa.encrypt('password123');     // Base64
if (cipher === false) throw new Error('加密失败：公钥无效或明文过长');

// ② 私钥解密
const pri = new JSEncrypt();
pri.setPrivateKey(PRIVATE_KEY_PEM);
const plain = pri.decrypt(cipher);             // false 表示解密失败

// ③ 签名 / 验签
const signer = new JSEncrypt();
signer.setPrivateKey(PRIVATE_KEY_PEM);
const sig = signer.sign('a=1&b=2', 'sha256');
const ok = signer.verify('a=1&b=2', sig, 'sha256');   // true

// ④ 现场生成一对密钥（1024 位，慢，只建议调试用）
const gen = new JSEncrypt({ default_key_size: 1024 });
console.log(gen.getPublicKeyB64());
console.log(gen.getPrivateKeyB64());
```

> ⚠️ RSA 能加密的明文长度受密钥位数限制（1024 位约 117 字节，2048 位约 245 字节）。**长正文请用 AES 加密，RSA 只用来加密 AES 密钥** —— 这就是常说的「混合加密」。
> ⚠️ 2048 / 4096 位运算在电视盒子上偏慢（几十到几百毫秒），能接受就别用 4096。

---

## 6. 脚本加密与混淆

先明确运行环境，选型才不会踩空：

| 环境能力 | mytv（QuickJS） |
|---|---|
| `eval` / `new Function` | ✅ 有 |
| `setTimeout` / `setInterval` | ❌ **没有**（引擎无事件循环） |
| `atob` / `btoa` | ❌ 没有（用 `CryptoJS.enc.Base64` 或 `mytv.decodeBase64`） |
| DOM / `window` / `document` / `XMLHttpRequest` | ❌ 没有 |
| `console.log` | ✅ 有 |
| 脚本大小上限 | **1 MB**（超过直接报「脚本过大」，不执行） |

### 6.1 三条路线

| 路线 | 做法 | 保密强度 | 适用 |
|---|---|---|---|
| **A. 代码混淆** | javascript-obfuscator 等工具变形 | 低-中（可逆，但人读不懂） | 日常首选，两种写法都支持 |
| **B. 整包 + eval** | 把源码打包成字符串，运行时 `eval` 还原 | 中 | **仅 CJS**（ESM 要用变体，见 6.3） |
| **C. 远程下发** | 脚本只做「拉密文 → 解密 → eval」 | 中-高（脚本本身可公开） | 需要远程改逻辑时用 |

### 6.2 CJS 脚本：三种方式都能用

因为 CJS 是走 `new Function(源码)` **非严格模式**执行的，函数内 `eval` 声明的 `main` 会落在同一个作用域里，宿主随后 `typeof main === 'function'` 就能取到 —— 所以整包 eval 完全可行。

```javascript
// 方式 B：整体打包（例如 packer / 自定义 base64 打包器）
eval(function (p, a, c, k, e, d) { /* ...打包器解包代码... */ }('...', 62, 379, '...'.split('|'), 0, {}));

// 方式 C：AES 密文 + 宿主桥解密
var src = mytv.opensslDecrypt(ENC, 'AES-256-CBC', KEY, 0, IV);
eval(src);      // src 里定义 function main(item){...}
```

```bash
# 方式 A：javascript-obfuscator（CJS 目标，不需要 ESM 处理）
npx javascript-obfuscator in.js --output out.js \
  --compact true \
  --string-array true --string-array-encoding base64 \
  --control-flow-flattening true --control-flow-flattening-threshold 0.75 \
  --dead-code-injection true --dead-code-injection-threshold 0.2 \
  --self-defending true
```

**CJS 混淆时必须关掉的选项：**

| 选项 | 为什么必须关 |
|---|---|
| `debugProtection` / `debugProtectionInterval` | 内部用 `setInterval`，QuickJS 没有定时器 → 直接报错 |
| `domainLock` / `domainLockRedirectUrl` | 依赖 `document.location`，QuickJS 没有 DOM → 报错 |
| `sourceMap` | 白送明文，没必要 |
| `renameGlobals` | 会把 `main` 改名，宿主就找不到了 |

### 6.3 ESM 脚本：方式 A 与方式 C 变体

ESM 是**模块**，源码里必须有真正的 `import` / `export` 语句，不能把整段源码塞进 `eval`（`export` 在 eval 里是语法错误）。所以：

**方式 A（推荐）：混淆器保留 ESM 结构**

```bash
npx javascript-obfuscator in.js --output out.js \
  --compact false \
  --string-array true --string-array-encoding base64 \
  --control-flow-flattening true \
  --self-defending true
```

**方式 C 变体：自解密 + 真 `export` 语句**

把真实逻辑放进密文，`eval` 还原后挂到 `globalThis`，再用**独立的** `export` 语句导出（严格模式下 eval 里的声明不外泄，所以必须显式挂 `globalThis`）：

```javascript
// 解密后执行，密文里定义 globalThis.__main = function(item){...}
const src = mytv.opensslDecrypt(ENC, 'AES-256-CBC', KEY, 0, IV);
eval(src);

export default globalThis.__main;
```

### 6.4 🔴 ESM 混淆最容易踩的坑

引擎是按 **行首** 判断 ESM 的（正则 `^\s*(import\s|export\s|export\{)`）。混淆器默认 `--compact true` 会把整个文件压成**一行**，此时 `export` 出现在行中间 → **被判成 CJS** → 源码塞进 `new Function` → `export` 是非法语句 → `SyntaxError`，脚本一点都跑不起来，报错还看不出是混淆导致的。

两个解法，任选其一：

```javascript
// 解法 1（推荐）：混淆时关掉压缩
--compact false

// 解法 2：混淆后手动在 export 前打断行（保证 export 顶格）
var _0xabc = ...;
// ← 这里有个换行
export default _0xabc;
```

配套检查：混淆完用文本编辑器看一眼，**文件里必须存在一行以 `export` 开头**。

### 6.5 选型建议

| 场景 | 建议 |
|---|---|
| 自己用 / 逻辑不敏感 | 不加密，直接明文 ESM，好维护 |
| 想防别人直接抄 | **方式 A**：javascript-obfuscator + `--compact false`（ESM）/ 默认压缩（CJS） |
| 密钥硬编码在脚本里，不想明文暴露 | **方式 C**：密文 + 运行时解密。ESM 用 6.3 的变体 |
| 逻辑要能远程改 | 方式 C + 远程下发密文（脚本只保留解密入口） |
| 加解密逻辑本身很重（大量 AES） | 别放 JS 里做，改用 `mytv.opensslDecrypt`（原生 OpenSSL） |

> 加密只提高阅读/拷贝成本，密钥最终还是落在设备上。真要防逆向，核心逻辑应该放在服务端。

---

## 7. 注意事项

### 🔴 改了脚本没生效？重启 App

脚本源码、`import` / `require` 的模块、字节码缓存，全是**进程级缓存且永不失效**（没有按修改时间重读的机制）。你在设备上改了 `xxx.js`、甚至删掉重放一份，App 跑的还是第一次读到的那份。排查顺序永远是：**强制停止 App → 重新打开**。

远程脚本同理：只下载一次，服务端更新后也要重启 App。

### 其他约束

| 项 | 说明 |
|---|---|
| 脚本大小 | 单文件 / 单模块上限 **1 MB**，超了直接报错不执行 |
| 执行超时 | **没有超时**。脚本跑多久都行，但**绝不能写死循环** —— QuickJS 没有办法中断正在执行的脚本，卡住就挂到进程退出 |
| 报错展示 | 列表订阅：错误页直接显示原文；播放线路：弹一次提示（切走不再打扰）；EPG 订阅：只记日志不弹 |
| 错误前缀 | 脚本相关问题统一以 **`JS脚本: `** 开头，例如 `JS脚本: TypeError: xxx is not a function at main (play.js:12)` |
| 日志 | `console.log` 输出到 Logcat，不在界面显示 |
| 返回值 | 必须 JSON 可序列化；返回函数 / `undefined` 会被判「脚本返回格式不正确」 |
| 多脚本并存 | 每个脚本有独立的引擎实例和专属线程（LRU 上限 16），互不阻塞；但同一个脚本被并发调用会排队 |

### 排查清单

| 现象 | 先看这里 |
|---|---|
| 「无法解析脚本源」 | 地址是不是写成了 `js://demo.js` 相对路径？改成 `js://file:///绝对路径` |
| 「脚本未定义 main(input) 函数」 | 入口没导出：ESM 要有 `export default function main`，CJS 要有顶层 `function main` |
| 改了没反应 | 重启 App（见上） |
| 混淆后直接语法错 | 6.4：`export` 不在行首，被判成 CJS 了 |
| 解密结果不对 | CryptoJS 的 key 没转 WordArray（5.1 第五节）；或者 key 位数和算法不匹配 |
| 播放 403 | `headers` 没带上 / 带错了。`Referer`、`User-Agent`、`Origin` 都可能被校验 |
| 界面卡死 | 脚本里有死循环或超长同步网络请求 |
