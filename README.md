# 代理规则说明

本仓库有两份规则文件，说的是**同一套规则**，只是语法不同：

| 文件 | 用在哪 | 说明 |
|---|---|---|
| `default626.conf` | 小火箭（iPhone、Mac） | **主文件**，所有改动先改这里 |
| `clash-rules.yaml` | 3x-ui 的 Clash/Mihomo 订阅（Windows Clash Verge） | 照着小火箭的规则翻译成 Clash 语法，3x-ui 自动拉取 |
| `clash-dns.yaml` | Clash Verge 的「DNS 覆写」 | 对应小火箭 `[General]` 的 DNS 部分，**需要手动粘贴** |

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
4. **开启虚拟网卡模式（TUN）**：系统代理模式只接管走代理设置的程序，很多 App 的 DNS 查询会绕过 Clash；虚拟网卡模式会接管所有流量和 DNS 查询。
5. **重启内核并检查**：打开 `browserleaks.com/dns`，结果里只应出现 Cloudflare，不应出现中国的 DNS 服务器。

---

## 三、改规则的步骤

1. 先改 `default626.conf`。
2. 检查是否违背"整体思想"里的任何一条。
3. 按"语法对照"把同样的改动写进 `clash-rules.yaml`，放在对应的位置。
4. 如果改的是 DNS 设置，同步改 `clash-dns.yaml`。
5. 提交后，小火箭和 Clash 各自更新一次配置。改了 `clash-dns.yaml` 的话，要重新粘贴到 Clash Verge 的 DNS 覆写里。
