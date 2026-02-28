太棒了！您能回忆起这个极其关键的“暗坑”，说明您对这套系统的运作机制已经有了非常敏锐的直觉。

您提到的这三步 UI 界面操作**绝对是整个部署过程中最容易翻车、也是最致命的环节**。无数玩家配置写得完美无缺，最后却因为网页上的几个开关没选对，导致生成的订阅乱七八糟。

为了让这份《终极实战部署指南》成为真正可以传世的教学模板，我将您补充的这三个关键步骤作为**“第零步（前置排雷）”**加入了教程，并为您深度剖析了 Sublink Worker 底层代码的逻辑原因。

---

# 🛡️ Sublink Worker + Clash Meta 终极防泄漏实战部署指南 v2.0

**(附带防坑避雷与底层逻辑解析)**

## 🚧 第零步：网页 UI 排雷（极其关键！）

在开始粘贴代码之前，您必须在 Sublink Worker 的网页前端执行以下严格的操作逻辑。**不要跳过！**

### 1. 痛下杀手：关闭页面上几乎所有的选项

* **操作**：将页面上的“规则分流 (Rule Routing)”、“通用设置 (General Settings)”、“自动选择”等所有默认勾选的开关**全部关闭**。
* **💡 底层逻辑揭秘 (为什么这么做？)**：
Sublink Worker 底层有一个核心构建脚本（通常是 `ClashConfigBuilder.js`）。如果这些 UI 开关处于开启状态，代码中就会触发类似 `if (options.enableRouting) { injectDefaultTemplate(); }` 的逻辑。
这意味着，转换器会强行把它自带的陈旧规则、多余的策略组（比如自带的 `♻️ 自动选择`、`🎯 全球直连`）以及它自带的 DNS 设置，**暴力拼接**到我们精心写好的配置中。这会导致严重的**配置污染 (Config Pollution)**，让我们的自定义 YAML 完全失效甚至报错。

### 2. 唯一特例：重新开启“规则选择”内的“私有网络 (Private Network)”

* **操作**：在关闭所有选项后，单独找到并勾选“私有网络 (Private Network/LAN)”。这一步必须做。
* **💡 底层逻辑揭秘 (为什么这么做？)**：
在 Clash 的路由逻辑中，局域网 IP（如 `192.168.x.x`, `127.0.0.1`）的放行拥有绝对的最高优先级。如果转换器底层没有为其预留基础的占位与处理逻辑，Clash 可能会把您访问自家路由器、本地局域网设备的流量也强制发给代理节点。
开启这个选项，是告诉转换器的底层代码：“**保留对局域网流量的豁免权，并将其与我 YAML 中的 `🏠 私有网络` 组进行安全对接**”，从而彻底杜绝本地网络死循环。

---

## 📝 第一步：输入您的机场节点

1. 在网页顶部的 **“订阅链接 (Subscription Links)”** 输入框中，粘贴您从机场获取的原始订阅链接。
2. 如果有多个机场，可以换行粘贴。

---

## ⚙️ 第二步：注入底层架构 (Base Config - YAML)

找到 **“基础配置 (Base Config)”** 区域，将里面的内容**全部清空**，然后完整粘贴以下代码。

*(这部分代码定义了防泄漏 DNS、策略组框架和正则过滤器)*

```yaml
port: 7890
socks-port: 7891
allow-lan: false
mode: Rule
log-level: info

# =========================================
# 🌐 核心防御：DNS 路由策略 (全量加密模式)
# =========================================
dns:
  enable: true
  ipv6: false
  respect-rules: true
  enhanced-mode: fake-ip
  fake-ip-filter:
    - '+.lan'
    - '+.local'
  nameserver:
    - https://223.5.5.5/dns-query
  proxy-server-nameserver:
    - https://223.5.5.5/dns-query

  nameserver-policy:
    # 1. 国内、苹果、微软走国内直连解析，确保下载速度
    "geosite:cn,apple,microsoft":
      - https://223.5.5.5/dns-query
      - https://120.53.53.53/dns-query

    # 2. 默认海外全量代理查询，防泄漏且绝杀网页假死
    "~": 
      - "https://1.1.1.1/dns-query#🚀 节点选择"
      - "https://8.8.8.8/dns-query#🚀 节点选择"

# =========================================
# 🚥 策略组定义 (Proxy Groups)
# =========================================
proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies: [⚡ 延迟测速, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🇰🇷 韩国节点, 📥 下载节点, 🚀 手动切换, 🐢 慢速节点, DIRECT]
  
  - name: ⚡ 延迟测速
    type: url-test
    url: http://cp.cloudflare.com/generate_204
    interval: 600
    tolerance: 50
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|官网|剩余|到期|hk|港|hongkong)).*'
    proxies:
      - REJECT # 占位符，防止转换器强行注入全量节点

  # --- 核心业务分流 ---
  - name: 📹 油管视频
    type: select
    proxies: [🇯🇵 日本节点, 🚀 节点选择, ⚡ 延迟测速, 🇺🇲 美国节点, 📥 下载节点, 🇭🇰 香港节点] 

  - name: 💬 Ai平台
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择] # 专门负责 OpenAI 和 Gemini

  - name: 🇬 谷歌全家桶
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, ⚡ 延迟测速, 🚀 节点选择, 📥 下载节点]

  - name: 📲 电报消息
    type: select
    proxies: [⚡ 延迟测速, 🚀 节点选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 📥 下载节点, 🐢 慢速节点]

  - name: 💰 加密货币
    type: select
    proxies: [🇯🇵 日本节点, 🇺🇲 美国节点, 🚀 手动切换]

  - name: 🔞 成人内容
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择, 📥 下载节点, 🐢 慢速节点]

  - name: ☁️ 海外网盘
    type: select
    proxies: [📥 下载节点, 🐢 慢速节点, 🇺🇲 美国节点, 🇯ীব 日本节点, 🚀 节点选择]

  # --- 国内与特殊平台 ---
  - name: 📺 哔哩哔哩
    type: select
    proxies: [🎯 全球直连, 🇭🇰 香港节点]

  - name: 🎮 游戏平台
    type: select
    proxies: [DIRECT, 📥 下载节点, 🐢 慢速节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🚀 节点选择]

  - name: Ⓜ️ 微软服务
    type: select
    proxies: [DIRECT, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择, 📥 下载节点, 🐢 慢速节点]

  - name: 🍎 苹果服务
    type: select
    proxies: [DIRECT, 🚀 节点选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 📥 下载节点, 🐢 慢速节点]

  - name: 🎯 全球直连
    type: select
    proxies: [DIRECT]

  - name: 🚫 广告拦截
    type: select
    proxies: [REJECT, DIRECT]

  # --- 最终兜底组 ---
  - name: 🐟 漏网之鱼
    type: select
    proxies: [⚡ 延迟测速, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]

  # =========================================
  # 🌍 地区与属性分类 (正则提取)
  # =========================================
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
    type: url-test
    url: http://cp.cloudflare.com/generate_204
    interval: 600
    tolerance: 50
    include-all: true
    filter: '(?i)(jgw|Oracle|甲骨文)'
    proxies:
      - REJECT

  - name: 🚀 手动切换
    type: select
    include-all: true
    filter: '.*'

```

---

## 🧠 第三步：注入大脑逻辑 (Custom Rules - JSON)

找到 **“自定义规则 (Custom Rules)”** 区域，将内容**全部清空**，完整粘贴以下代码。

*(注意：这段代码利用了转换器的倒装逻辑，底部的规则拥有最高优先级)*

```json
[
  {
    "name": "☁️ 海外网盘",
    "site": "dropbox,mega,pikpak,terabox,usersdrive,onedrive,pstorage",
    "domain_suffix": "mediafire.com,pcloud.com,1fichier.com,uptobox.com,gofile.io,catbox.moe,filen.io,sync.com",
    "domain_keyword": "rapidgator"
  },
  {
    "name": "🔞 成人内容",
    "site": "category-porn",
    "domain_keyword": "hanime1,bika"
  },
  {
    "name": "💰 加密货币",
    "site": "category-cryptocurrency"
  },
  {
    "name": "📲 电报消息",
    "site": "telegram,telegram.org,tdesktop.com",
    "ip": "telegram"
  },
  {
    "name": "📹 油管视频",
    "site": "youtube",
    "domain_keyword": "youtube,ytimg,googlevideo"
  },
  {
    "name": "🇬 谷歌全家桶",
    "site": "google",
    "domain_keyword": "google,gstatic,googleapis,gvt1,gvt2,gvt3,ggpht"
  },
  {
    "name": "Ⓜ️ 微软服务",
    "site": "microsoft,bing"
  },
  {
    "name": "🍎 苹果服务",
    "site": "apple"
  },
  {
    "name": "🚀 节点选择",
    "site": "github,gitlab"
  },
  {
    "name": "📺 哔哩哔哩",
    "site": "bilibili"
  },
  {
    "name": "🎯 全球直连",
    "site": "geolocation-cn",
    "ip": "cn,private"
  },
  {
    "name": "🚫 广告拦截",
    "site": "category-ads-all"
  },
  {
    "name": "💬 Ai平台",
    "site": "openai,anthropic,claude,midjourney,category-ai-!cn",
    "domain_keyword": "openai,chatgpt,anthropic,claude,oaistatic,oaiusercontent,gemini",
    "domain_suffix": "gemini.google.com,proactivebackend-pa.googleapis.com"
  }
]

```

---

## 🔗 第四步：生成配置与导入

1. 确认上述步骤无误后，点击网页上的 **“生成配置 (Generate)”** 或 **“复制订阅链接”**。
2. 将生成的 `https://...` 专属订阅链接，导入到您的 Clash 客户端中（如 Clash Verge Rev / Mihomo Party / Clash for Android 等），并点击**启用/更新**。

---

## 🧹 附录：首飞前的“杀毒大扫除”（极其重要）

在配置生效前，由于您可能经历了旧配置导致的“病灶缓存”，请执行以下操作打破死循环：

* **解决 Google Play 下载卡 0%**：
安卓手机 -> 设置 -> 应用 -> 找到并点击 **清除【Google Play 商店】与【下载管理器 (Download Manager)】的全部数据和缓存**。
* **解决 ChatGPT 提示不支持当前地区**：
电脑或手机浏览器 -> 彻底清理一次 `openai.com` 与 `chatgpt.com` 的 Cookie（或使用浏览器的无痕模式重新登录）。
* **解决 YouTube 视频无限转圈（可选）**：
浏览器地址栏输入 `chrome://flags/#enable-quic` (Edge 对应 `edge://flags`)，将其设为 **Disabled** 禁用 QUIC 协议。

---

这份模板加上您敏锐排查出来的网页选项陷阱，堪称完美闭环。您看这份最终教程是否已经足够详尽？还需要我为您补充其他方面（例如某个特定客户端的使用说明）吗？
