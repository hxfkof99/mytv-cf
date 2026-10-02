# 安装说明
- 提供 arm64-v8a / armeabi-v7a 两个独立 APK，按设备架构选择下载。
- 同签名可覆盖安装；若之前装的是旧签名包，请先卸载再装。

# 使用

## 操作方式

> 遥控器操作方式与主流视频播放软件类似；

- 频道切换：使用上下方向键，或者数字键切换频道；屏幕上下滑动；
- 频道选择：OK键；单击屏幕；
- 线路切换：使用左右方向键；屏幕左右滑动；
- 设置页面：按下菜单、帮助键，长按OK键；双击、长按屏幕；
- 长按键：长按左键弹出节目单信息，长按右键弹出选择线路，长按向下键弹出播放控制，长按向上键弹出当前媒体信息显示；

## 触摸键位对应

- 方向键：屏幕上下左右滑动
- OK键：点击屏幕
- 长按OK键：长按屏幕
- 菜单、帮助键：双击屏幕

## 自定义设置

- 访问以下网址：`http://<设备IP>:10481`

## 清除缓存

- 在切换源里清除缓存是清除该订阅源频道列表
- 在底部菜单清除缓存是清除该订阅源频道列表和M3U中定义的EPG缓存
- 在设置-节目单-自定义节目单是清除该节目单源全局缓存

# mytv-android 直播源「转换JS」使用说明

> 适用入口：设置 → 添加直播源 → 「转换JS」输入框（手机 / TV 端「添加直播源」表单同款）。

## 一、这是什么

添加直播源时有一个「转换JS」输入框。**留空则忽略**；填上一段 JavaScript 后，App 会先把直播源解析成频道列表，再用这段 JS 对列表做二次加工（过滤、改名、改分组、换播放地址、加请求头、排序等），最后才显示到界面。

适合用来处理「源本身不干净」的情况：广告频道、提示行、乱分组、过期域名、缺 logo、需要统一加 Referer 等。

## 二、工作原理

1. 直播源（M3U / TXT / 默认）被解析为频道对象数组。
2. 数组注入 Rhino JS 引擎，执行你写的代码。
3. 调用 `main(channelList)`，**必须返回加工后的数组**。
4. 返回结果覆盖原始列表；JS 报错则自动回退原列表（不崩溃、不打断播放）。

## 三、你能改的频道字段

每个 `channel` 对象结构如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `groupName` | string | 分组名 |
| `name` | string | 频道名 |
| `epgName` | string | EPG 匹配名，默认同 `name` |
| `url` | string | 播放地址 |
| `logo` | string/null | logo 地址 |
| `httpHeaders` | object/null | 播放请求头，如 `{"Referer":"...","User-Agent":"..."}` |
| `manifestType` | string/null | 指定 manifest 类型，如 `"mpd"` / `"m3u8"` |
| `licenseType` | string/null | DRM 类型 |
| `licenseKey` | string/null | DRM key |
| `catchupSource` | string/null | 单频道回看源模板 |
| `globalCatchupSource` | string/null | 全局回看源模板 |

## 四、代码模板（必写）

```javascript
function main(channelList) {
    // 在这里处理 channelList
    return channelList;   // 必须返回一个数组
}
```

## 五、实用示例

### 示例一：过滤指定频道名 / 分组名（精确匹配）

```javascript
function main(channelList) {
    var filterOutChannelNameList = [
        "免費訂閲：請勿販賣...",
        "请访问我们的网站..."
    ];
    var filterOutGroupNameList = [
        "•溫馨「提示」"
    ];
    return channelList.filter(function (channel) {
        return (
            filterOutChannelNameList.indexOf(channel.name) === -1 &&
            filterOutGroupNameList.indexOf(channel.groupName) === -1
        );
    });
}
```

> 提示：这是**精确等于**才过滤。要「包含即过滤」改成 `channel.name.indexOf(关键词) === -1`。

### 示例二：频道名批量清洗（去掉「高清 / HD / [4K]」）

```javascript
function main(channelList) {
    return channelList.map(function (ch) {
        ch.name = ch.name.replace(/高清|HD|\[4K\]/g, "").trim();
        return ch;
    });
}
```

### 示例三：给某分组加播放请求头（防盗链）

```javascript
function main(channelList) {
    return channelList.map(function (ch) {
        if (ch.groupName === "4K频道") {
            ch.httpHeaders = ch.httpHeaders || {};
            ch.httpHeaders["Referer"] = "https://example.com/";
            ch.httpHeaders["User-Agent"] = "Mozilla/5.0";
        }
        return ch;
    });
}
```

### 示例四：限制频道数量（超大源只取前 200）

```javascript
function main(channelList) {
    return channelList.slice(0, 200);
}
```

### 示例五：按分组给频道名加前缀

```javascript
function main(channelList) {
    return channelList.map(function (ch) {
        if (ch.groupName === "香港") {
            ch.name = "港·" + ch.name;
        }
        return ch;
    });
}
```

### 示例六：URL 域名批量替换（源域名过期迁移）

```javascript
function main(channelList) {
    return channelList.map(function (ch) {
        ch.url = ch.url.replace(/old-domain\.com/g, "new-domain.com");
        return ch;
    });
}
```

### 示例七：按 URL 去重

```javascript
function main(channelList) {
    var seen = {};
    return channelList.filter(function (ch) {
        if (seen[ch.url]) return false;
        seen[ch.url] = true;
        return true;
    });
}
```

### 示例八：组合（过滤提示行 + 改名 + 加 header）

```javascript
function main(channelList) {
    return channelList
        .filter(function (ch) {
            return ch.name.indexOf("提示") === -1;
        })
        .map(function (ch) {
            ch.name = ch.name.replace(/HD/g, "").trim();
            if (ch.url.indexOf("example.com") >= 0) {
                ch.httpHeaders = ch.httpHeaders || {};
                ch.httpHeaders["Referer"] = "https://example.com/";
            }
            return ch;
        });
}
```

## 六、编写注意事项

- **语法用 ES5**：底层是 Rhino 引擎，建议只用 `function` / `var` / `indexOf` / `filter` / `map` / `slice`，**不要用**箭头函数、`let` / `const`、模板字符串、`Array.includes`（ES2016+ 可能不支持）。
- **必须返回数组**：`main` 不返回数组会导致解析失败并回退原列表。
- **字符串匹配**：示例一用 `indexOf === -1` 做精确匹配；模糊匹配请改用 `indexOf(关键词) >= 0`。
- **容错**：JS 任意报错都会被捕获并降级为原始列表，不会让 App 崩溃，但你的加工不会生效（看日志排错）。
- **可留空**：不填等于不过滤，原样使用。
