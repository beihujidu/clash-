# Clash 链式代理配置

虽然 新版Clash 本身已经支持链式代理配置，直接设置起来看似方便，但如果把所有流量都走住宅 IP，成本太高，带宽也浪费。所以有了下面这套分流方案：只让需要住宅 IP 的服务走链式代理，其他流量走普通节点或直连。

记录一套我自己在用的 Clash方案，核心思路特别简单：

OpenAI / Anthropic → 走美国住宅静态 ISP
其它国外网站（YouTube、Google、GitHub、Telegram 等）→ 走普通机场节点
国内网站和中国 IP → 直连

下面这张图能帮你快速理解访问路径：

![网络访问路径示意图](assets/network-access-path.png)

## 适合谁用？

你手头有一个普通机场订阅（节点可能不太干净，容易被 ChatGPT 等拒绝）。

你单独买了一个美国住宅静态 ISP 代理（网上很多，个人选择，我就不推荐具体商家了）。

你想让 Claude、ChatGPT 这类 AI 服务享受住宅 IP ，但又不希望 YouTube 之类的流量也跟着绕路。

你希望国内网站保持直连，省流量且不减速。

如果你符合上面几条，那这套配置会很适合你。

## 准备工作

一个可用的机场订阅（随便什么协议，支持 Clash 即可）

一个住宅 ISP 代理（SOCKS5 或 HTTP，我这里以 SOCKS5 为例）

注意：住宅代理的质量参差不齐，你需要自己找一个能用的。教程只讲配置方法，不提供代理。

## 找到 Clash的配置目录

Windows 下的配置目录一般在：

```text
C:\Users\你的用户名\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev
```
比如你的用户名是 Lenovo，那就是：

```text
C:\Users\Lenovo\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev
```
如果找不到，大概率是因为 AppData 文件夹隐藏了。可以这样打开：

按 Win + R

输入：

```text
%APPDATA%\io.github.clash-verge-rev.clash-verge-rev
回车
进去之后，你会看到 profiles.yaml。打开它，找到当前正在使用的订阅：
```

```yaml
current: xxxxxxxxxxxx   # ← 这个就是当前配置的 uid
```
往下翻，找到同一个 uid 对应的 option 段：

```yaml
option:
  script: xxxxxxxxxxxx.js
  proxies: xxxxxxxxxxxx.yaml
  groups: xxxxxxxxxxxx.yaml
  rules: xxxxxxxxxxxx.yaml
```
后面我们要修改的四个文件，都在同目录下的 profiles 文件夹里：

```text
C:\Users\你的用户名\AppData\Roaming\io.github.clash-verge-rev.clash-verge-rev\profiles
```
## 核心修改：不要直接改 clash-verge.yaml

很多人习惯直接修改 clash-verge.yaml，但这个文件是 Clash Verge Rev 动态生成的，订阅一更新就会被覆盖，你改得再细致也是白费功夫。

正确做法：修改 profiles 目录里的那四个配置文件（对应 script、proxies、groups、rules）。

### 1. 修改 proxies 文件

打开 proxies 对应的那个 YAML 文件（名字可能是类似 xxxxxxxxxxxx.yaml），在里面追加你的住宅代理。

```yaml
prepend: []
append:
  - name: "Claude-Residential-Chain"    # 这里取一个你喜欢的名字
    type: socks5
    server: 你的住宅IP
    port: 你的端口
    username: 用户名
    password: 密码
    udp: true
    dialer-proxy: "AI入口"               # 关键：通过 AI入口 这个代理组出去
delete: []
```
解释一下 dialer-proxy: "AI入口"：
这表示你这个住宅代理不会直接连接外网，而是先经过一个普通机场节点（被我们称为“入口”），再由那个节点去连住宅代理。这样可以避免住宅代理被墙，同时保持出口为住宅 IP。

注意：AI入口 这个组名我们会在后面的脚本里创建，这里先写上。

### 2. 清空 groups 文件

打开 groups 对应的文件（一般也是 .yaml），改成：

```yaml
prepend: []
append: []
delete: []
为什么清空？
因为所有分组逻辑我们统一在脚本里生成，避免订阅自带的 groups 混进来捣乱。
```

### 3. 清空 rules 文件

同样，打开 rules 对应的文件，改成：

```yaml
prepend: []
append: []
delete: []
规则也完全交给脚本来控制。
```

### 4. 修改 script 文件（最关键的一步）

打开 script 对应的 .js 文件，把内容全部替换成下面这个版本。

重要：你需要把脚本里的 AI_PROXY 变量改成你在 proxies 文件中填写的代理名称（例如 "Claude-Residential-Chain"）。

```js
function main(config, profileName) {
  // ========== 请务必修改这一行 ==========
  const AI_PROXY = "Claude-Residential-Chain";  // 改成你在 proxies 里填写的 name
  // ====================================
  const AI_GROUP = "AI住宅ISP";          // 这个组会直接选择住宅代理
  const ENTRY_GROUP = "AI入口";           // 这个组用来选入口节点
  const NORMAL_GROUP = "普通节点";        // 普通国外网站走的组
  // 需要走住宅 ISP 的域名（AI 服务）
  const aiDomains = [
    "openai.com",
    "chatgpt.com",
    "chat.openai.com",
    "oaistatic.com",
    "oaiusercontent.com",
    "openaiapi-site.azureedge.net",
    "auth0.com",
    "featureassets.org",
    "prodregistryv2.org",
    "anthropic.com",
    "claude.ai",
    "claude.com",
    "claudeusercontent.com",
  ];
  // 需要直连的国内域名（可根据自己需要增减）
  const directDomains = [
    "apple.com", "icloud.com", "icloud-content.com", "mzstatic.com",
    "126.com", "126.net", "127.net", "163.com", "qq.com", "wechat.com",
    "tencent.com", "taobao.com", "tmall.com", "alipay.com", "alicdn.com",
    "baidu.com", "bdstatic.com", "bilibili.com", "jd.com", "douyin.com",
    "amap.com", "meituan.com", "zhihu.com", "xiaomi.com",
  ];
  // 识别订阅里的“信息节点”（比如显示流量、到期时间的假节点）
  const isInfoNode = (name, index) => {
    if (index < 3) return true;
    return ["剩余流量", "距离下次", "套餐到期"].some((word) => name.includes(word));
  };
  config.proxies = config.proxies || [];
  // 提取所有普通节点（排除住宅代理、排除信息节点）
  const normalNodes = config.proxies
    .map((proxy, index) => ({ name: String(proxy.name || ""), index }))
    .filter(({ name, index }) => name && name !== AI_PROXY && !isInfoNode(name, index))
    .map(({ name }) => name);
  // 入口节点列表：默认取前 8 个普通节点（你也可以手动指定，见下文）
  const entryNodes = normalNodes.length ? normalNodes.slice(0, 8) : ["DIRECT"];
  config["proxy-groups"] = [
    // 普通国外网站组
    {
      name: NORMAL_GROUP,
      type: "select",
      proxies: ["自动选择", "故障转移", ...normalNodes, "DIRECT"],
    },
    // AI 专用组（只包含住宅代理）
    {
      name: AI_GROUP,
      type: "select",
      proxies: [AI_PROXY],
    },
    // 入口节点组（住宅代理的“跳板”）
    {
      name: ENTRY_GROUP,
      type: "select",
      proxies: entryNodes,
    },
    // 自动测速组
    {
      name: "自动选择",
      type: "url-test",
      proxies: normalNodes.length ? normalNodes : ["DIRECT"],
      url: "https://www.gstatic.com/generate_204",
      interval: 300,
    },
    // 故障转移组
    {
      name: "故障转移",
      type: "fallback",
      proxies: normalNodes.length ? normalNodes : ["DIRECT"],
      url: "https://www.gstatic.com/generate_204",
      interval: 300,
    },
    // 总开关（方便全局切换）
    {
      name: "GLOBAL",
      type: "select",
      proxies: [NORMAL_GROUP, AI_GROUP, "DIRECT"],
    },
  ];
  // 规则：AI 域名走 AI 组，国内域名/ IP 直连，其余走普通组
  config.rules = [
    ...aiDomains.map((domain) => `DOMAIN-SUFFIX,${domain},${AI_GROUP}`),
    `DOMAIN,api.anthropic.com,${AI_GROUP}`,
    `DOMAIN,console.anthropic.com,${AI_GROUP}`,
    ...directDomains.map((domain) => `DOMAIN-SUFFIX,${domain},DIRECT`),
    "DOMAIN-SUFFIX,local,DIRECT",
    "IP-CIDR,127.0.0.0/8,DIRECT,no-resolve",
    "IP-CIDR,172.16.0.0/12,DIRECT,no-resolve",
    "IP-CIDR,192.168.0.0/16,DIRECT,no-resolve",
    "IP-CIDR,10.0.0.0/8,DIRECT,no-resolve",
    "IP-CIDR,100.64.0.0/10,DIRECT,no-resolve",
    "IP-CIDR6,fe80::/10,DIRECT,no-resolve",
    "DOMAIN-SUFFIX,cn,DIRECT",
    "DOMAIN-KEYWORD,-cn,DIRECT",
    "GEOIP,CN,DIRECT",
    `MATCH,${NORMAL_GROUP}`,
  ];
  config.mode = "rule";
  return config;
}
关于手动指定入口节点
脚本里入口节点默认取前 8 个普通节点（normalNodes.slice(0, 8)）。如果你不想让它自动选，可以改成固定节点，例如：
```

```js
const entryNodes = ["香港01", "日本01", "新加坡01"];
```
注意：节点名字必须和订阅里完全一致，否则会报错。

## 验证配置是否生效

改完上面所有文件后，重启 Clash ，确认模式为 规则模式，并打开 TUN 模式（如果不打开 TUN 模式，部分应用可能不走代理）。

### 1. 看日志

打开 Clash 的日志页面，然后访问 chatgpt.com 或 claude.ai。

如果配置生效，你应该会看到类似下面的记录：

```text
chatgpt.com:443 match DomainSuffix(chatgpt.com) using AI住宅ISP[Claude-Residential-Chain]
```
重点看两处：

match DomainSuffix(chatgpt.com)
using AI住宅ISP[Claude-Residential-Chain]

![日志中显示 ChatGPT 使用 AI住宅ISP](assets/clash-log-ai-group.png)

### 2. 测试住宅 IP 属性

临时把 Clash 的 GLOBAL 组切到 AI住宅ISP（相当于全局走住宅代理），然后访问 https://ipinfo.io：

```text
AS Type: isp
Privacy: false
```

![ipinfo 显示 AS Type 为 isp，Privacy 为 false](assets/ipinfo-isp-privacy.png)

注意：住宅 IP 质量完全取决于你买的代理，如果显示 hosting 或 Privacy: true，说明这个 IP 不够“干净”，可能会被 ChatGPT 等识别为机房 IP。这种情况请换一家住宅代理服务。

测试完记得把 GLOBAL 切回 普通节点 或 AI住宅ISP 以外的选项。
## 日常使用流程

打开 Clash Verge Rev，确保模式为 规则模式，TUN 模式打开。

在 普通节点 组里选一个你日常用的节点，或者直接选 自动选择 让它自动测速。

在 AI入口 组里选一个稳定、低延迟的普通节点（最好是非香港、非特殊地区的普通节点，避免住宅代理连接失败）。

AI住宅ISP 组保持为你配置的那个链式代理（名字叫 Claude-Residential-Chain 或你自定义的）。

国内网站自动直连，不用管。

Claude / ChatGPT 会自动走住宅 ISP，无需手动切换。

只有当你需要临时验证住宅 IP 质量时，才把 GLOBAL 切到 AI住宅ISP 去访问 ipinfo.io，测完立刻切回来。

希望这份教程能帮你一次搞定链式代理，省去以后反复折腾的烦恼。有问题欢迎交流。
