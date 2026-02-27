# 🚀 Sublink Worker 终极网页端定制分流指南

如果你发现在 Sublink Worker 的“基础配置 (Base Config)”中填写的路由规则（rules）总是不生效、被强行覆盖，那是因为程序的**底层硬编码逻辑**拦截并删除了自定义规则。

本指南提供了一套**“顺应底层逻辑”**的完美解决方案：**在 Base Config 中只保留节点与策略组，在 Custom Rules（自定义规则）中用 JSON 喂给它路由规则。**

---

## 🛠️ 部署步骤

### 第一步：修改基础配置 (Base Config)
在 Sublink Worker 网页端，找到底部 **“基础配置 (Base Config)”**，格式选择 `Clash`。
**将以下代码完整粘贴进去（注意：这里绝对不能包含 `rules:` 部分）：**

```yaml
port: 7890
socks-port: 7891
allow-lan: false
mode: Rule
log-level: info
dns:
  enable: true
  ipv6: true
  respect-rules: true
  enhanced-mode: fake-ip
  nameserver:
    - [https://120.53.53.53/dns-query](https://120.53.53.53/dns-query)
    - [https://223.5.5.5/dns-query](https://223.5.5.5/dns-query)
  proxy-server-nameserver:
    - [https://120.53.53.53/dns-query](https://120.53.53.53/dns-query)
    - [https://223.5.5.5/dns-query](https://223.5.5.5/dns-query)

proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies: [♻️ 自动选择, ⚖️ 负载均衡, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🇰🇷 韩国节点, 📥 下载节点, 🚀 手动切换, 🐢 慢速节点, DIRECT]
  
  - name: ♻️ 自动选择
    type: url-test
    url: [http://cp.cloudflare.com/generate_204](http://cp.cloudflare.com/generate_204)
    interval: 600
    tolerance: 50
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|官网|剩余|到期|hk|港|hongkong)).*'
  
  - name: ⚖️ 负载均衡
    type: fallback
    url: [http://cp.cloudflare.com/generate_204](http://cp.cloudflare.com/generate_204)
    interval: 600
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|hk|港|hongkong)).*'
  
  - name: 💰 加密货币
    type: select
    proxies: [🇯🇵 日本节点, 🇺🇲 美国节点, 🚀 手动切换]
  - name: 💬 Ai平台
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🐢 慢速节点]
  - name: 🔞 成人内容
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择, 🐢 慢速节点]
  - name: ☁️ 海外网盘
    type: select
    proxies: [📥 下载节点, 🚀 节点选择, 🐢 慢速节点]
  - name: 📺 哔哩哔哩
    type: select
    proxies: [🎯 全球直连, 🇭🇰 香港节点, 📥 下载节点]
  - name: 📹 油管视频
    type: select
    proxies: [♻️ 自动选择, 🚀 节点选择, 🇯🇵 日本节点, 🇭🇰 香港节点, 🇺🇲 美国节点, 📥 下载节点, 🐢 慢速节点]
  - name: 📲 电报消息
    type: select
    proxies: [♻️ 自动选择, 🚀 节点选择, 📥 下载节点, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🐢 慢速节点]
  - name: 🌍 国外媒体
    type: select
    proxies: [🚀 节点选择, 📥 下载节点, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🐢 慢速节点]
  - name: 🎮 游戏平台
    type: select
    proxies: [DIRECT, 📥 下载节点, 🐢 慢速节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 📢 谷歌FCM
    type: select
    proxies: [♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: Ⓜ️ 微软Bing
    type: select
    proxies: [♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: Ⓜ️ 微软服务
    type: select
    proxies: [DIRECT, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 🍎 苹果服务
    type: select
    proxies: [DIRECT, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 🎯 全球直连
    type: select
    proxies: [DIRECT]
  - name: 🚫 广告拦截
    type: select
    proxies: [REJECT, DIRECT]
  - name: 🐟 漏网之鱼
    type: select
    proxies: [⚖️ 负载均衡, ♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🇰🇷 韩国节点]
  
  # --- 正则筛选节点 ---
  - name: 📥 下载节点
    type: select
    include-all: true
    filter: '(?i)(wx|s801)'
  - name: 🇭🇰 香港节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(hk|港|hongkong)'
  - name: 🇯🇵 日本节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(jp|日本|tokyo|c93s4)'
  - name: 🇰🇷 韩国节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(kr|韩|seoul)'
  - name: 🇺🇲 美国节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|c93s4|s801)).*(us|美|unitedstates|c93s|\.top-(?:VLESS_Reality_Vision|singbox_naive|anytls))'
  - name: 🐢 慢速节点
    type: select
    include-all: true
    filter: '(?i)(jgw|Oracle|甲骨文)'
  - name: 🚀 手动切换
    type: select
    include-all: true
    filter: '.*'
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

