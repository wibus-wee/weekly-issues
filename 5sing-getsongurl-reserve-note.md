# 5sing GetSongUrl API 逆向笔记

当你打开 5sing 具体的一个歌曲详情，它会请求一个 getsongurl 接口，这个接口返回的是歌曲的 URL

## 分析接口请求

接口示例：

```
https://5sservice.kugou.com/song/getsongurl
    ?appid=2918
    &clientver=1000
    &mid=8df3ec50950df4f8d9ef8ee2b292062e&uuid=8df3ec50950df4f8d9ef8ee2b292062e
    &dfid=0jHybf1yhbgA2JqmJT0ydTpE
    &songid=3173361
    &songtype=bz
    &version=6.6.72
    &clienttime=1721816628285
    &signature=170f24609f4fd5768f4c72eac2e437b8
```

其中 songid 是歌曲 ID，它可以从 Search 接口获取，这部分很简单稍微看一下网络请求分析一下就可以了，在接口里就是 `songId` field，在这里不做阐述，主要是分析其他的参数。

我们可以看到有一个 `signature`，我尝试过删除 `signature` 参数或者乱写一个值，返回的是签名错误或者 404 Not Found，所以这个接口是需要签名的。

而在后面的分析我们可以知道，`appid`, `clientver`, `mid`, `uuid`, `dfid` 这几个参数是可以调取出来的，所以我们只需要找到签名算法即可。

## 分析签名算法 `globalSign`

打开网络 Tab，找到这个请求，可以看到调用 Trace 里面有一个 `window.globalSign` 方法，这个方法直接内嵌在网页中，如下：

```js
var _appid = 2918;
var readyCbs = [];
var defaultSignOptions = {
  appkey: "5uytoxQewcvIc1gn1PlNF0T2jbbOzRl", // 注意一下这个 appkey，在很后面的签名算法分析中会用到
  useH5: true,
  isCDN: false,
  postType: "json",
};
window.globalSign = function (getData, postData, fn, options) {
  options = Object.assign({}, defaultSignOptions, options || {});
  options.callback = function (_get, _post) {
    typeof fn === "function" && fn(_get, _post);
  };
  window.getBaseInfo(_appid, function (baseInfo) {
    const signParams = {
      appid: baseInfo.appid,
      clientver: baseInfo.clientver,
      mid: baseInfo.mid,
      uuid: baseInfo.uuid,
      dfid: baseInfo.dfid,
    };
    window.infSign(
      Object.assign({}, signParams, getData),
      postData || null,
      options
    );
  });
};
```

其中，`_appid` 实际上只需要正则匹配即可：

```js
const reg = /var _appid = (\d+)/;
const _appid = reg.exec(html)[1]; // 2918
```

但是，我们可以发现这里有两个可疑的方法 `window.infSign` 和 `window.getBaseInfo`，这两个算法皆来自外部 JS 文件，我们可以找到这两个文件的地址，然后查看源码。

- `window.infSign` 来自 `https://5sstatic.kugou.com/public/common/inf_sign-2.0.0.min.js`
- `window.getBaseInfo` 来自 `https://staticssl.kugou.com/common/js/min/npm/getBaseInfo.min.js`

直接在网页上执行这两个脚本，会自动向 window 中注入这两个方法，而在 headless browser 中，就可以直接调用这两个方法了。

但是，我想知道这两个方法的逻辑是什么样的，所以我直接下载了这两个文件，然后查看源码，却发现这两个文件的代码有很多都很相似. 让我们来逐步分析他们吧。

### 分析 getBaseInfo

从代码中分析，出现了 Cookie 的 util 类：

```js
// ...
var d = {
  // d: object -> 猜测变量真实名字: utils
  Cookie: {
    // -> 用于操作cookie
    // ...
  },
};
// ...
```

于是，我猜测，获取信息的必定和 Cookie 相关，也是从 Cookie 中拿出来的（实际上，自己去请求 header 里搜索一下关键词也能知道）

#### 拿到 `mid`

我怀着好奇心，获取了一下 `d.Cookie.write` 的调用，发现有一个 `d.getKgMid` 方法：

```js
// var d = { ...
getKgMid: function() { // -> 用于获取 kg_mid
    var e = d.Cookie.read("kg_mid");
    // ...
}
// ...
```

里面我们只需要看 `navigator.cookieEnabled` 开启的情况下的代码，未开启的部分使用了 Canvas Fingerprint，这里不做分析。

所以对这个方法的简略版本是这样的：

```js
function getKgMid() {
  var e = d.Cookie.read("kg_mid"); // 1. 从 cookie 中读取 kg_mid
  if (d.IsEmpty(e)) {
    // 2. 如果为空
    var n = d.Guid(), // 3. 生成一个 guid -> 用于生成 md5
      e = d.Md5(n); // 4. 生成 md5
    try {
      d.Cookie.write("kg_mid", d.Md5(n), 864e6, "/", "kugou.com"); // 5. 写入 cookie, 864e6 = 1 day, "/" = path, "kugou.com" = domain
    } catch (e) {}
  }

  return e;
}
```

其中，`d.Guid()` 就是一个生成随机 id 的一个方法，没有什么技术含量,`d.Md5` 是一个 MD5 加密方法，这里不做分析

```js
function Guid() {
  // -> 用于生成一个全局唯一标识符
  function e() {
    return ((65536 * (1 + Math.random())) | 0).toString(16).substring(1);
  }
  return e() + e() + "-" + e() + "-" + e() + "-" + e() + "-" + e() + e() + e();
}
```

通过查找 `d.getKgMid()` 我们可以看到在最后输出的 `mid` 调用了这个方法，所以我们可以直接调用这个方法获取 `mid` 了。搞定一个！

而这个 `var d` 后面的方法就不看了，我不太明白为什么要这么写，它使用了一个 depreciate 的 `window.external`，里面还有一个方法叫 `SuperCall`，但是实际上 External 中不存在这个东西，所以就不看了...

> 然后我们接下来到 `e` 和 `p` 变量，这里出现了前面一模一样的代码，一个是 `getKeysFromObject`，一个是 `mergeObject` 的作用，这里不做分析。

#### 分析其他参数

其他参数的设定关键在于最后的 `return function(e,i,n,r)` 当中，里面涉及了一堆三元运算符，看起来就碍眼，也很难去理解...

于是，我也就懒得理解了，丢给 AI 去处理这一坨东西，最后可以得到：

当然，下面是这个 `if-else` 条件逻辑的序号表，以便更好地理解代码流程：

1. MiniApp 检查
   - 条件: `if (window.MiniApp && window.MiniApp.listenGetUserInfo)`
   - 操作: 调用 `window.MiniApp.listenGetUserInfo` 获取用户信息，成功则执行 `callback(user)`，失败则执行 `callback(null)`。
2. Kugou 客户端（非 FXAPP）检查
   - 条件: `else if (isInClient() && !isFXAPP)`
   - 操作:
     - 调用 `mobileCall(CALL_TYPE_USER_INFO, null, callback)` 获取用户基本信息。
     - 如果状态为 1，设置用户详细信息；否则，将用户信息设为空。
     - 设置 `userInfo.appId` 后，调用 `mobileCall(CALL_TYPE_CLIENT_VERSION, null, callback)` 获取客户端版本。
     - 获取客户端版本后，调用 `mobileCall(CALL_TYPE_DEVICE_INFO, null, callback)` 获取设备信息。
     - 填充缺失字段后，执行 `callback(userInfo)`。
3. Kugou 客户端（FXAPP）检查
   - 条件: `else if (isInClient() && isFXAPP)`
   - 操作:
     - 调用 `mobileCall(625, null, callback)` 获取基本信息。
     - 设置 `userInfo` 的各个字段，包括 `platform`。
     - 调用 `mobileCall(410, null, callback)` 检查状态是否为 1。
     - 如果状态为 1，调用 `mobileCall(411, {}, callback)` 获取详细用户信息并执行 `callback(userInfo)`；否则，将用户信息设为空并执行 `callback(userInfo)`。
4. PC 客户端检查
   - 条件: `else if (isPCClient())`
   - 操作:
     - 调用 `GetPCClientUserInfo(callback)` 获取用户信息。
     - 设置 `userInfo` 的各个字段。
     - 调用 `getPCMID_APPID(callback)` 获取设备信息，填充缺失字段后执行 `callback(userInfo)`。
5. **默认情况（Cookie 检查）**
   - 条件: `else`
   - 操作:
     - 从 Cookie 中读取相关字段并设置到 `userInfo` 中。
     - 检查 `kugooAppId` 与 `userInfo.appId` 是否一致，不一致且符合特定条件时，将用户信息设为空。
     - 设置 `userInfo.clientVersion` 并执行 `callback(userInfo)`。

很明显，我们只需要默认情况下的 Cookie 检查即可，继续去解析默认情况的检查，分析后我们可以得到全部参数的来源：

- `appid` 来自传入的 `e` 参数，参考上面的分析我们就知道了，在 HTML 中使用 `var` 定义的 `_appid` 就是传来这里的
- `mid` 来自上文的 `getKgMid` 方法, 我们选择直接生成一个 `mid` 即可
- `uuid` = `mid`
- `dfid` 来自 `kg_dfid` Cookie 值，没有的情况下是 `-`，实际上，似乎是 `-` 也可以，所以我们可以直接使用 `-`
- `clientver` 是 headcode 的，默认 `1e3`，如果有变，使用正则匹配一下 `s.clientver = (\d+)` 即可

### 分析 infSign 签名算法

到现在，我们拿到了所有的参数，接下来就是签名算法了，这个算法在 `inf_sign-2.0.0.min.js` 中，我们可以直接查看源码。

映入眼帘的就是熟悉的代码，那些代码就真的不想管了..直接划拉到最后的 `return` 语句，我们可以看到这个签名算法的逻辑，其中 `function d(t)` 是整一个签名算法的核心，可以被称之为 `handleSignture` 的核心方法

经过人性化处理后：

```js
function handleSignature(data) {
  var keys = [],
    keyValuePairs = [],
    appKey = "NVPh5oo715z5DIWAeQlhMDsWXXQV4hwt";

  if (options && options.appkey) {
    delete data.srcappid;
    appKey = options.appkey;
  }

  for (var key in data) {
    keys.push(key);
  }

  keys.sort();
  keys.forEach(function (key) {
    keyValuePairs.push(key + "=" + data[key]);
  });

  if (bodyData) {
    if (Object.prototype.toString.call(bodyData) === "[object Object]") {
      if (postType === "json") {
        keyValuePairs.push(JSON.stringify(bodyData));
      } else {
        var bodyKeyValuePairs = [];
        for (var bodyKey in bodyData) {
          bodyKeyValuePairs.push(bodyKey + "=" + bodyData[bodyKey]);
        }
        keyValuePairs.push(bodyKeyValuePairs.join("&"));
      }
    } else {
      keyValuePairs.push(bodyData);
    }
  }

  keyValuePairs.unshift(appKey);
  keyValuePairs.push(appKey);
  data.signature = Md5(keyValuePairs.join(""));

  if (options.log) {
    console.log("H5签名前参数排序结果", keyValuePairs);
    console.log("H5签名前md5原字符串", keyValuePairs.join(""));
    console.log("H5签名后返回", data);
  }

  if (bodyData) {
    callback &&
      callback(
        data,
        Object.prototype.toString.call(bodyData) === "[object Object]" &&
          postType === "json"
          ? JSON.stringify(bodyData)
          : bodyData
      );
  } else {
    callback && callback(data);
  }
}
```

很明显，这个算法就是一个排序后的 MD5 签名算法，于是最后一个参数就是我们需要的 `signature` 了，这个算法的核心就是 `Md5` 方法...

但是但是但是！！！这里的 appkey 实际上并不是这里写的 key，而是在 HTML 里的那段脚本里的 `appKey`，它来自于 `defaultSignOptions` 如果使用这里文件里的 `appKey`，会导致签名错误。

## 参数模拟

为了防止远程出现改变，我们的 `appid`, `clientver` 都得从远程文件获取，还有一个 `appKey`（加密的时候需要），不过这是个 demo，所以就不管了。其他的参数我们可以直接模拟。

```js
const md5 = require("js-md5");

function Guid() {
  function e() {
    return ((65536 * (1 + Math.random())) | 0).toString(16).substring(1);
  }
  return e() + e() + "-" + e() + "-" + e() + "-" + e() + "-" + e() + e() + e();
}

// ======== FETCH FROM REMOTE START ========

const appKey = "5uytoxQewcvIc1gn1PlNF0T2jbbOzRl";
const appid = 2918;
const clientver = 1000;

// ======== FETCH FROM REMOTE END ========

// ======== CONFIG START ========

const songid = 2591126;
const songtype = "bz";

// ======== CONFIG END ========

// ======== START GENERATE DATA ========

const mid = md5(Guid());
const clienttime = Date.now();
const uuid = mid;
const dfid = '-'

// ======== END GENERATE DATA ========


const data = {
  appid,
  clientver,
  mid,
  uuid,
  dfid,
  songid,
  songtype,
  clienttime,
};

// ======== START SIGNATURE ========

let keyValuePairs = [];
for (let key in data) {
  keyValuePairs.push(key + "=" + data[key]);
}

keyValuePairs.sort();
keyValuePairs.unshift(appKey);
keyValuePairs.push(appKey);
const signature = md5(keyValuePairs.join(""));

// ======== END SIGNATURE ========

const url = new URL("https://5sservice.kugou.com/song/getsongurl");
for (let key in data) {
  url.searchParams.append(key, data[key]);
}
url.searchParams.append("signature", signature);

fetch(url, {
  method: "GET",
  headers: {
    "Content-Type": "application/json",
  },
})
  .then((res) => res.json())
  .then(console.log);
```

安装好 `js-md5` 后，直接运行这个脚本，我们可以发现，成功获取到数据了。