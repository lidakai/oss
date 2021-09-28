EasyV 辅助工具之 chrome 扩展程序入门

# 前言

更加详细和专业的 chrome extension 开发教程网络上比比皆是，此文档仅是 EasyV 开发过程中内部实验性产物，用来提高个人和团队工作效率，若有瑕疵，欢迎评论交流，共同进步。


# 项目结构
首先要写个配置文件 `manifest.json`

```
{
    "name": "Easyv开发助手", // 插件的名称
    "description": "Easyv开发工具", // 插件的描述
    "version": "1.0", // 插件的版本
    "manifest_version": 3, // 清单文件的版本，这个必须写，很多教程是2版本，但是我就喜欢尝鲜，咱就上3，
    "background": { // 是一个常驻的页面，它的生命周期是插件中所有类型页面中最长的，它随着浏览器的打开而打开，随着浏览器的关闭而关闭，所以通常把需要一直运行的、启动就运行的、全局的代码放在background里面（看你需求是否需要此类常驻功能）
        "service_worker": "background.js"
    },
    "permissions": [ // 权限申请 - 类似于客户端同学开发app 需要声明你这个应用都需要哪些系统权限
        "storage", // 存储
        "cookies", // cookie
        "activeTab", // 当用户调用扩展时，该权限允许暂时访问当前激活的选项卡
        "tabs",
        "webRequest"
    ],
    "action": {
        "default_popup": "/src/options/index.html", // 类似于index.html（不允许内联JavaScript。）
         "default_title":"Easyv开发助手", // 图标悬停时的标题，可选
        "default_icon": { 
        // 图标，一般偷懒全部用一个尺寸的也没问题，chrome 右上角工具栏的图标
        // 至少建议使用16x16和32x32尺寸，方形
            "16": "/src/assets/images/app_logo.png",
            "32": "/src/assets/images/app_logo.png",
            "48": "/src/assets/images/app_logo.png",
            "128": "/src/assets/images/app_logo.png"
        }
    },
    "host_permissions": [ // 当前扩展应用匹配域名规则
        "*://*.easyv.cloud/"
    ],
    "optional_permissions": [ // 扩展权限
        "alarms",
        "background",
        "clipboardWrite"
    ],
    "content_scripts": [ // 需要直接注入页面的JS
        {
            "matches": [
                "https://*.easyv.cloud/*" // 注入的域名
            ],
            "run_at": "document_end", // 注入的时机
            "js": [
                "src/assets/scripts/getIndexdDB.js",
                "src/assets/scripts/screenExport.js"
            ]
        }
    ]
}
```

# 主体内容

可能会有同学有疑问，哎呦 ~ 

1、`optional_permissions` 和 `permissions` 都属于权限相关，那？有什么区别呢？

OK，那么ta来了。

`permissions ：in advance permissions（预先许可） ->`
你可以这么理解，你在安装APP的时候，有些应用在启动时就开始给你要权限比如这样：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9f160f084c243689576f54ea12a94f8~tplv-k3u1fbpfcp-watermark.image?)

↑ 这个就属于 `permissions`

2、`optional_permissions ：are granted by the extension's user at runtime（由扩展的用户在运行时授予） ->`
而当你在使用APP内某个功能的时候，APP才开始申请

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1367d7029dff45d6acef755b4cb5ecda~tplv-k3u1fbpfcp-watermark.image?)

> 扩展的运行过程中，而不是在安装的时候请求权限。这能帮助用户清楚为什需要此权限，且仅在必要的时候才运行扩展使用此权限。

这样一看是不是清晰很多。OK ~ 概念性问题聊完，开始看我们主要实验什么功能，用来做什么。

---

首先是 `action -> default_popup -> index.html`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fae81b19941c42549fc3a410c6ed7abe~tplv-k3u1fbpfcp-watermark.image?)

应用打开首屏即如此

目前主要有两个功能`导出本地存储大屏Layers`和`清除大屏404组件`

EasyV 平台会在用户浏览器做大屏数据本地存储功能，防止由于某些莫名原因客户数据发生丢失（概率很小，有备无患）做的一种容灾措施。

数据存放在indexdDB，主要是看中ta存储量大的优点，读写速度也相对不错，总之我们只存储图层数据，这些小小数据量，足矣。
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16ca368bd2044647a831fa748a8e0fc7~tplv-k3u1fbpfcp-watermark.image?)

需求既然已经了解了，那么开始干活。

我们`action`打开的`default_popup`页面是单独线程，和我们期望的`https://easyv.cloud/workspace/create/xxxx`并不是一个，所以也不存在说直接操作目标网页，因此就用到了我们前面配置项中提到的`content_scripts`;

这部分脚本，简单来说是插入到网页中的脚本。它具有独立而富有包容性。

所谓**独立**，指它的工作空间，命名空间，域等是独立的，不会说跟插入到的页面的某些函数和变量发生冲突；

所谓**包容性**，指插件把自己的一些脚本（content script）插入到**符合条件**的页面里，作为页面的脚本，因此与插入的页面共享dom的，即用dom操作是针对插入的网页的，在这些脚本里使用的window对象跟插入页面的window是一样的。主要用在消息传递上（使用postMessage和onmessage）


具体使用是录入screen id 和 hour 用来拼接查询主键；
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc27bd0df7834282956928b6058e13da~tplv-k3u1fbpfcp-watermark.image?)

在查询过程中用到了`chrome.tabs.sendMessage` 用来做消息通信，上文有讲，`default_popup`无法直接在当前网页中执行js或者css，利用`content_scripts`注入脚本
```
    "content_scripts": [
        {
            "matches": [
                "https://*.easyv.cloud/*"
            ],
            "run_at": "document_end",
            "js": [
                "src/assets/scripts/getIndexdDB.js", // <= 获取indexDB数据，干活的就是俺
                "src/assets/scripts/screenExport.js"
            ]
        }
    ]
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46d816caf07f4a309d424d50baf9b70d~tplv-k3u1fbpfcp-watermark.image?)

```
let queryOptions = { active: true, currentWindow: true }; 
    let [tab] = await chrome.tabs.query(queryOptions);// 获取当前窗口激活选项卡信息
    chrome.tabs.sendMessage(tab.id, { // tabs.sendMessage 需要 tab.id
        eventType: 'import-data-copy', // 因为不止这一个功能需要数据交互，因此参数中维护了eventType
        key: `${screenId}-${hour}`
    }, {}, function (result) {
    // 业务代码
    });
```

`getIndexdDB.js ↓`

```
chrome.runtime.onMessage.addListener(
    function ({ key, eventType }, sender, _callback) {
        if (eventType === 'import-data-copy') {
            key && getIndexDB(key).then(result => {
                result && _callback(result);
            });
        }
        return true; //异步要这么写
    }
);

function getIndexDB(key) {
    return new Promise((resolve, reject) => {
        const EasyvLocalDB = window.indexedDB.open('EasyvLocalDB', 4);
        EasyvLocalDB.onsuccess =
            function (ev) {
                const db = ev.target.result;
                const request = db.transaction(['screen']).objectStore('screen').get(key);
                request.onsuccess = function (ev) {
                    resolve(ev.target.result)
                }
                request.onerror = function () {
                    reject()
                }
            }
        EasyvLocalDB.onerror = function () {
            reject();
        }
    })
}
```

为什么要专门贴一份`getIndexdDB.js`代码，其中有个细节，大家一定要留意，如果你的这部分逻辑是异步，一定要记得`chrome.runtime.onMessage.addListener function 中的 return true`

从chrome.runtime.onMessage.addListener的文档中：

`Function to call (at most once) when you have a response. The argument should be any JSON-ifiable object. If you have more than one listener in the same document, then only one may send a response. This function becomes invalid when the event listener returns, **unless you return true** from the event listener to indicate you wish to send a response asynchronously (this will keep the message channel open to the other end until is called)`

[↑ 相关文档链接]([chrome.runtime - Chrome Developers](https://developer.chrome.com/docs/extensions/reference/runtime/#event-onMessage))



# 结语
当然，我们上面仅仅摸索了chrome extension 很小很小一部分，在未来一段时间内，空闲之余，会持续增加完善相关功能，感谢阅读，共勉。

[easyv-develop-utils github 仓库地址]([easyv-develop-utils (github.com)](https://github.com/lidakai/easyv-develop-utils))