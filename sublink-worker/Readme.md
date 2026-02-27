# 🚀 Sublink Worker 终极网页端定制分流指南

如果你发现在 Sublink Worker 的“基础配置 (Base Config)”中填写的路由规则（rules）总是不生效、被强行覆盖，那是因为程序的**底层硬编码逻辑**拦截并删除了自定义规则。

本指南提供了一套**“顺应底层逻辑”**的完美解决方案：**在 Base Config 中只保留节点与策略组，在 Custom Rules（自定义规则）中用 JSON 喂给它路由规则。**

---

## 🛠️ 部署步骤

### 第一步：修改基础配置 (Base Config)
在 Sublink Worker 网页端，找到底部 **“基础配置 (Base Config)”**，格式选择 `Clash`。
**将以下代码完整粘贴进去（注意：这里绝对不能包含 `rules:` 部分）：**

```yaml
https://github.com/0x1233333/ACL4SSR/edit/master/sublink-worker/%E5%9F%BA%E7%A1%80%E9%85%8D%E7%BD%AE.ini
   
```


### 第二步：配置 Custom Rules (自定义规则)
在网页上找到 【自定义规则 (Custom Rules)】 区域，切换到** JSON **视图。
**这套 JSON 包含了满血补全的“拦截、网盘、成人、电报”规则，会直接转换为高性能的 RULE-SET。请完整粘贴进去：**

```JSON
[
  {
    "name": "🚫 广告拦截",
    "site": "category-ads-all"
  },
  {
    "name": "💰 加密货币",
    "site": "category-cryptocurrency"
  },
  {
    "name": "💬 Ai平台",
    "site": "category-ai-!cn"
  },
  {
    "name": "🔞 成人内容",
    "site": "category-porn",
    "domain_keyword": "hanime1,bika"
  },
  {
    "name": "☁️ 海外网盘",
    "site": "dropbox,mega,pikpak,terabox,usersdrive,onedrive,pstorage",
    "domain_suffix": "mediafire.com,pcloud.com,1fichier.com,uptobox.com,gofile.io,catbox.moe,filen.io,sync.com",
    "domain_keyword": "rapidgator"
  },
  {
    "name": "📲 电报消息",
    "ip": "telegram"
  },
  {
    "name": "📹 油管视频",
    "site": "youtube"
  },
  {
    "name": "📺 哔哩哔哩",
    "site": "bilibili"
  },
  {
    "name": "🍎 苹果服务",
    "site": "apple"
  },
  {
    "name": "Ⓜ️ 微软服务",
    "site": "microsoft"
  },
  {
    "name": "🌍 国外媒体",
    "site": "geolocation-!cn"
  },
  {
    "name": "🎯 全球直连",
    "site": "geolocation-cn",
    "ip": "cn"
  }
]
```


### 第三步：网页端防覆盖生成技巧
在顶部输入框填好你的原始节点链接。

【防覆盖关键】：在“基础规则 (Rulesets)”复选框区域，请至少随便打勾保留一个规则（比如只勾选“🔒 国内服务”）。千万不要全部取消勾选！ 否则系统会触发“最小化防御机制”，强行抹除我们刚写的 JSON 规则。

（不用担心底部附带的系统规则，因为我们在 JSON 里设置了很高的优先级，它会自动跑到配置文件的最前面拦截流量，底部的规则纯属摆设不影响）。

点击 “生成订阅”，享受完美的节点与规则分流吧！

