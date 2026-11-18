# 代理规则说明

本仓库有两份规则文件，说的是**同一套规则**，只是语法不同：

| 文件 | 用在哪 | 说明 |
|---|---|---|
| `default626.conf` | 小火箭（iPhone、Mac） | **主文件**，所有改动先改这里 |
| `clash-rules.yaml` | 3x-ui 的 Clash/Mihomo 订阅（Windows Clash Verge） | 照着小火箭的规则翻译成 Clash 语法，3x-ui 自动拉取 |
| `clash-dns.yaml` | Clash Verge 的「DNS 覆写」 | 对应小火箭 `[General]` 的 DNS 部分，**需要手动粘贴** |
| `clash-tun-exclude.txt` | Clash Verge 虚拟网卡的「排除自定义网段」 | 对应小火箭 `tun-excluded-routes`，**需要手动粘贴** |

---

## 一、整体思想

所有规则都围绕下面几条原则。新增或修改规则时，先看是否违背其中任何一条。

### 1. 国内网站白名单，其他一律走代理

- 只有**明确写进规则**的国内网站、国内 App、银行等才直连（DIRECT）。
- 没有写进规则的网站，默认都走代理。
- 宁可让一个没写规则的国内网站绕道美国变慢，也不让一个境外网站误走直连。

### 2. 兜底规则绝不直连

- 规则列表的最后一条永远是"其余全部走代理"：
  - 小火箭：`FINAL,PROXY`
  - Clash：`MATCH,PROXY`
- 不允许把兜底改成 DIRECT，也不允许加"不认识的就直连"之类的规则。

### 3. DNS 防泄漏：需要走代理的域名，绝不在本机查 DNS

- 走代理的域名，原样交给 VPS，由 VPS 在美国解析。手机和电脑本地不去查它的 IP。
- 原因：网站可以通过 DNS 泄露检测，看到"是谁替你查的 DNS"。如果本地用国内 DNS 查境外域名，对方就能发现"网页请求来自美国，DNS 查询来自中国"。
- 落实到规则上：
  - 所有按 IP 判断的规则（`IP-CIDR`、`IP-CIDR6`、`GEOIP`、`IP-ASN`）**都必须加 `no-resolve`**。不加的话，没写规则的域名走到这里时会被迫在本地解析，既泄露，又可能因为 DNS 不稳而卡住。
  - `GEOIP,CN,DIRECT,no-resolve` 只对"本来就是 IP"的连接生效，域名直接跳过，落到兜底的代理。

### 4. DNS 查询绝不退回国内 DNS

- 主 DNS 和备用 DNS 都用境外加密 DNS（DoH），**绝不填 `system` 或国内 DNS** 作为备用。
- 直连域名解析失败时，改走代理去解析，不退回本地。

### 5. 在以上基础上，用 DIRECT 给国内网站加速

- 国内常用网站、App、CDN 写进 DIRECT 白名单，让它们不绕道美国。
- 白名单里的国内域名，用本机（运营商）DNS 解析，拿到国内最近的 CDN 节点，速度最快。这些本来就是国内网站，不涉及泄露。
- 哪个国内网站变慢了，就给它补一条 `DOMAIN-SUFFIX,域名,DIRECT`。

### 6. 重要的境外服务要写域名规则

- 用到的境外服务（尤其是 AI、Meta 这类风控严的）都写 `DOMAIN-SUFFIX,域名,PROXY`，不要只靠兜底。
- 例子：Meta 的 Muse 需要 `metaaivm.com` 和 `muse.ai`。曾经因为没写这两条，加上 `GEOIP,CN` 没有 `no-resolve`，导致每次都要在本地查 DNS，App 冷启动时经常连不上。

---

## 二、Clash 版本怎么写

**规则：`clash-rules.yaml` 必须和 `default626.conf` 表达同样的规则和含义。** 改了小火箭的规则，要同步改 Clash 的。

### 要保持一致的地方

- **规则内容一致**：小火箭里有的域名规则，Clash 里也要有，策略（PROXY / DIRECT / REJECT）相同。
- **规则顺序一致**：两边都是从上往下匹配、命中第一条就停。小火箭里写着"必须在某某规则之前"的，Clash 里也要保持这个先后。
- **上面"整体思想"里的每一条，两边都要满足**：IP 类规则都加 `no-resolve`，兜底永远是代理。

### 语法对照

| 含义 | 小火箭 `default626.conf` | Clash/Mihomo `clash-rules.yaml` |
|---|---|---|
| 规则写在哪 | `[Rule]` 段，每行一条 | `rules:` 下，每条前面加 `- ` |
| 精确域名 | `DOMAIN,a.com,PROXY` | `- DOMAIN,a.com,PROXY` |
| 域名后缀 | `DOMAIN-SUFFIX,a.com,PROXY` | `- DOMAIN-SUFFIX,a.com,PROXY` |
| 域名关键字 | `DOMAIN-KEYWORD,google,PROXY` | `- DOMAIN-KEYWORD,google,PROXY` |
| IP 段 | `IP-CIDR,17.0.0.0/8,PROXY,no-resolve` | `- IP-CIDR,17.0.0.0/8,PROXY,no-resolve` |
| IPv6 段 | `IP-CIDR6,fe80::/10,DIRECT,no-resolve` | `- IP-CIDR6,fe80::/10,DIRECT,no-resolve` |
| 国家 IP | `GEOIP,CN,DIRECT,no-resolve` | `- GEOIP,CN,DIRECT,no-resolve` |
| ASN | `IP-ASN,132203,DIRECT,no-resolve` | `- IP-ASN,132203,DIRECT,no-resolve` |
| 按协议 | `PROTOCOL,UDP` | `NETWORK,UDP` |
| 组合规则 | `AND,((PROTOCOL,UDP),(DST-PORT,443)),REJECT-NO-DROP` | `- AND,((NETWORK,UDP),(DST-PORT,443)),REJECT` |
| 兜底 | `FINAL,PROXY` | `- MATCH,PROXY` |
| 按程序名（仅电脑） | `PROCESS-NAME,YoudaoDict,DIRECT` | `- PROCESS-NAME,YoudaoDict,DIRECT` |

策略名两边一样：`PROXY`、`DIRECT`、`REJECT`。Clash 里的 `PROXY` 是文件开头 `proxy-groups` 定义的节点选择组，自动包含面板生成的所有节点，不要在规则里写具体节点名。

### DNS 设置

- 小火箭的 DNS 写在 `default626.conf` 的 `[General]` 段。
- 3x-ui 只从 `clash-rules.yaml` 里取 `proxy-groups` 和 `rules`，DNS 写进去不会生效。
- Clash 的 DNS 单独放在 `clash-dns.yaml`，同样要符合"整体思想"第 3、4 条。改了小火箭的 DNS 设置，也要同步改这个文件。

### Clash Verge 使用步骤

1. **订阅**：在 Clash Verge 里添加 3x-ui 的订阅链接。节点由面板生成，规则来自 `clash-rules.yaml`，点更新就会同步 GitHub 上的最新规则。
2. **替换 DNS 覆写**：设置 → 打开「DNS 覆写」，把 `clash-dns.yaml` 的全部内容替换进去，保存。Clash Verge 自带的默认 DNS 用的是国内 DNS，会泄露，必须替换。
3. **关闭 IPv6 开关**：和 DNS 里的 `ipv6: false` 保持一致，避免 IPv4 / IPv6 来回切换。
4. **开启虚拟网卡模式（TUN）并按下面设置**
   - **为什么要用虚拟网卡模式**：系统代理模式只接管主动走代理设置的程序（主要是浏览器），很多 App 和它们的 DNS 查询会绕过 Clash；虚拟网卡模式接管电脑上所有流量和 DNS 查询。
   - 「自动设置全局路由」：打开。把所有流量导进虚拟网卡。
   - 「严格路由」：**打开**。理由见下方"严格路由"一节。
   - 「自动选择流量出口接口」：打开。Clash 自己连 VPS 时走真实网卡，否则会绕回虚拟网卡形成死循环。
   - 「DNS 劫持」：保持 `any:53`。截住所有发往 53 端口的 DNS 请求交给 Clash 回答，对应小火箭的 `hijack-dns`，范围更大。
   - 「DNS 模式」：保持 Auto。
   - 「排除自定义网段」：把 `clash-tun-exclude.txt` 的全部内容复制进去，保存。理由见下方"排除网段"一节。
5. **重启内核并检查**：打开 `browserleaks.com/dns`，结果里只应出现 Cloudflare，不应出现中国的 DNS 服务器。

### 为什么要关闭浏览器的「安全 DNS」

操作：Chrome → 设置 → 隐私和安全 → 安全 → 关闭「使用安全 DNS」。Edge：设置 → 隐私、搜索和服务 → 安全性 → 关闭「使用安全 DNS」。

- 开着时，浏览器不用系统 DNS，自己用加密 DNS（DoH）去查域名，绕过了 Clash 的整套 DNS 处理。
- **会泄露**：如果电脑设的是国内 DNS（如阿里 `223.5.5.5`），浏览器会自动改用它家的加密 DNS。这种查询直接连国内 IP，命中 `GEOIP,CN,DIRECT`，从国内直连发出，阿里能看到你查的每个境外网站。
- **规则变弱**：浏览器自己查到 IP 后直接拿 IP 去连，Clash 往往只看到 IP、看不到域名，按域名写的分流规则就失效了。
- 关掉后，浏览器问系统 DNS → 被虚拟网卡截住 → Clash 用 fake-ip 回答 → 真正的解析由 VPS 完成，符合"整体思想"第 3 条。
- 浏览器设置里的 OpenDNS、CleanBrowsing 等选项都属于这个功能，同样不要用。

### 为什么要开启「严格路由」

- 开了虚拟网卡后，电脑上同时有虚拟网卡和真实网卡（Wi-Fi / 有线）。不开严格路由时，Clash 只是"建议"流量走虚拟网卡，并不禁止走真实网卡。
- Windows 查 DNS 时会**同时问所有网卡**上的 DNS 服务器，谁先回答用谁的。所以即使 Clash 接管了，Windows 仍会通过真实网卡去问运营商 DNS，造成 DNS 泄露。
- 开启后，Clash 用路由和防火墙规则把真实网卡这条路堵死，所有流量和 DNS 查询只能走虚拟网卡。
- 代价：局域网设备（打印机、NAS、投屏）可能连不上，靠"排除网段"解决；少数软件可能异常；Clash 崩溃时可能整台电脑断网，重启 Clash 或关掉虚拟网卡模式即可恢复。

### 为什么要填「排除网段」

- 对应小火箭的 `tun-excluded-routes`，列表完全相同（见 `clash-tun-exclude.txt`）。
- 这些是局域网、本机、组播和各类保留地址，本来就不该走代理。开了严格路由后，不排除它们，访问路由器、NAS、打印机等可能出问题。
- `100.64.0.0/10` 是 Tailscale 使用的网段，排除后 Tailscale 直接走自己的网卡，不经过 Clash。

---

## 三、改规则的步骤

1. 先改 `default626.conf`。
2. 检查是否违背"整体思想"里的任何一条。
3. 按"语法对照"把同样的改动写进 `clash-rules.yaml`，放在对应的位置。
4. 如果改的是 DNS 设置，同步改 `clash-dns.yaml`。
5. 提交后，小火箭和 Clash 各自更新一次配置。改了 `clash-dns.yaml` 的话，要重新粘贴到 Clash Verge 的 DNS 覆写里。
