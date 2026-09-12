# surge-rules

个人 Surge 分流规则集。核心目的只有一个：**把指定服务的域名钉死在固定代理节点上**，让出口 IP 保持稳定，避免节点漂移触发风控。

覆盖 Anthropic / Claude、Google 全家桶、Spotify、TikTok、欧美流媒体，以及一份约 2 万条的成人域名拦截表。适用于 Surge Mac 与 Surge iOS。

---

## 一、解决什么问题

Surge 的规则自上而下匹配、首条命中即生效。当服务走 `url-test` / `fallback` / `load-balance` 这类自动策略组时，出口节点会随延迟波动、节点故障或手动切换而改变，同一账号的出口 IP 就在多个地理位置之间跳动。

典型后果：

| 服务 | 表现 |
|---|---|
| Claude / Anthropic | 判定为异常登录（"不可能旅行"），强制重新验证或直接中断会话 |
| Google | 触发可疑登录验证；`challenges.cloudflare.com` 过不去时表现为"反复要求重登" |
| Spotify | 账号区域异常、内容库与推荐区漂移 |
| TikTok | 风控触发、推荐区漂移 |
| 流媒体 | 地区内容库跳变、播放中断 |

所以本仓库把这些域名单独拎出来，指向**固定出口**（单个代理策略，或 `select` 手动组），而不是丢给自动测速组。

> **前提**：这些 RULE-SET 指向的策略必须是固定出口。指到 `url-test` / `fallback` 组上，等于没做。

---

## 二、文件清单

| 文件 | 规则条数 | 规则类型 | 建议策略 | 用途 |
|---|---:|---|---|---|
| `claude.list` | 16 | DOMAIN-SUFFIX ×12、DOMAIN、DOMAIN-KEYWORD、IP-CIDR、IP-CIDR6 | 固定节点 | Claude / Anthropic 全链路（含 Google 登录链、Cloudflare 验证、IP 段兜底） |
| `Google Core Services.list` | 41 | DOMAIN-SUFFIX ×33、DOMAIN ×5、DOMAIN-KEYWORD ×3 | 固定节点 | Google 分层域名：核心 / Gemini / YouTube / 云与办公 |
| `spotify.list` | 26 | DOMAIN-SUFFIX ×16、DOMAIN ×9、USER-AGENT | 固定节点 | Spotify 主站 + CDN + 第三方 CDN 别名 |
| `tiktok.list` | 38 | DOMAIN-SUFFIX ×28、DOMAIN ×6、DOMAIN-KEYWORD ×2、USER-AGENT、PROCESS-NAME | 固定节点 | TikTok 及字节系海外服务域名 |
| `stream_us.list` | 40 | DOMAIN-SUFFIX | 美区节点 | 美区 IP 可用的流媒体（17 个平台） |
| `stream_other.list` | 10 | DOMAIN-SUFFIX | `REJECT` 或 `DIRECT` | 非美区独占流媒体（7 个平台），美区 IP 不可用 |
| `AdultDomain.list` | 19,976（唯一域名 19,738） | DOMAIN-SUFFIX | `REJECT` | 成人 / NSFW 域名拦截，约 1.0 MB |

`main.py` 是早期占位脚本（Hello World），不参与规则生成，可忽略或删除。

---

## 三、快速开始

### 3.1 引用远程规则（推荐）

在 profile 的 `[Rule]` 段之前或之中加入下面的行。**顺序很重要**：固定出口类必须排在 `GEOIP,CN` / `FINAL` 等通用规则之前，否则永远轮不到它们匹配。

```ini
[Proxy Group]
# 固定出口：用 select 手动锁死，不要用 url-test / fallback
US-Fixed  = select, 你的固定节点A, 你的固定节点B
US-Stream = select, 你的美区节点

[Rule]
# ---- 1. 需要固定出口 IP 的服务（必须排最前）----
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/claude.list,US-Fixed
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/Google%20Core%20Services.list,US-Fixed
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/spotify.list,US-Fixed
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/tiktok.list,US-Fixed

# ---- 2. 流媒体 ----
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/stream_us.list,US-Stream
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/stream_other.list,REJECT

# ---- 3. 拦截（pre-matching：REJECT 类规则集可在 DNS 解析前先判定）----
RULE-SET,https://raw.githubusercontent.com/bianJuFeng/surge-rules/main/AdultDomain.list,REJECT,pre-matching

# ---- 4. 通用规则放最后 ----
RULE-SET,SYSTEM,DIRECT
RULE-SET,LAN,DIRECT
GEOIP,CN,DIRECT
FINAL,Proxy,dns-failed
```

中国大陆访问 `raw.githubusercontent.com` 经常不通，可换 jsDelivr 镜像（`@main` 分支引用，同样内容）：

```ini
RULE-SET,https://cdn.jsdelivr.net/gh/bianJuFeng/surge-rules@main/claude.list,US-Fixed
```

> 文件名里的空格在 URL 中必须写成 `%20`，即 `Google%20Core%20Services.list`。小写 `claude.list` 是仓库里的真实文件名，写 `Claude.list` 会 404。

### 3.2 引用本地文件

把 `.list` 放在 profile 同目录（或写绝对路径），Surge 会监听本地文件变化并自动重载，改完立即生效，不依赖网络：

```ini
RULE-SET,claude.list,US-Fixed
RULE-SET,Google Core Services.list,US-Fixed
RULE-SET,AdultDomain.list,REJECT,pre-matching
```

### 3.3 RULE-SET 行可用参数

| 参数 | 作用 |
|---|---|
| `no-resolve` | 为整个规则集强制 `no-resolve`：IP 类子规则对未解析的域名直接跳过，不触发 DNS 查询 |
| `extended-matching` | 域名子规则额外匹配 TLS SNI 与 HTTP Host 头 |
| `update-interval=N` | URL 规则集重新下载间隔（秒，默认 `86400`；`-1` 关闭自动更新） |
| `pre-matching` | 仅当策略是 `REJECT` 家族时可加，整个规则集参与解析前匹配 |

---

## 四、各文件说明

### 4.1 `claude.list` — Claude / Anthropic

按用途分四段：

- **自有业务**：`claude.ai`、`claude.com`、`clau.de`、`anthropic.com`、`claudeusercontent.com`、`claudemcpclient.com`
- **登录链路**：`accounts.google.com`、`google.com`、`gstatic.com` —— 用 Google 账号登录时必须整段一起固定，否则登录态与主站出口 IP 不一致
- **人机验证**：`challenges.cloudflare.com` —— 最容易被误判为"强制重登录"的环节
- **第三方依赖**：`sentry.io`、`statsig.anthropic.com`、`statsigapi.net`、`datadoghq`（KEYWORD）
- **IP 兜底**：`160.79.104.0/21`、`2607:6bc0::/48`，均带 `no-resolve`，用于兜住未来新增子域名漏配的情况

> 注意：`DOMAIN-SUFFIX,google.com` 覆盖的是**全部** google.com 流量，不只登录。若不希望 Gmail / 搜索跟着 Claude 的节点走，应把登录链路拆成单独文件引用。

### 4.2 `Google Core Services.list` — Google 四层

1. **核心服务**（12 条）：`google.com`、`gstatic.com`、`googleapis.com`、`googleusercontent.com`、`ggpht.com`、`googleadservices.com`、`googlesyndication.com`、`google-analytics.com`、`gvt1.com` 等
2. **Gemini / 生成式 AI**（15 条）：`gemini.google.com`、`bard.google.com`、`deepmind.*`、`aistudio.google.com`、`vertexai.google`、`ai.google.dev`、`makersuite.google.com`，以及 `generativelanguage` / `developerprofiles` / `colab` 三个 KEYWORD
3. **YouTube**（7 条）：`youtube.com`、`youtube-nocookie.com`、`youtu.be`、`ytimg.com`、`ytstatic.com`、`googlevideo.com`、`youtubei.googleapis.com`
4. **Cloud 与办公套件**（7 条）：`cloud.google.com`、`gmail.com`、`mail/drive/docs/sheets/slides.google.com`

前三段与第四段可以拆开引用：把前三段指向固定节点、第四段交给普通策略，也是常见用法。

### 4.3 `spotify.list`

- 主站 / 品牌子域 12 条：`spotify.com`、`spoti.fi`、`spotify.link`、`spotifycharts.com`、`spotify.design`、`byspotify.com` 等
- CDN 4 条：`scdn.co`、`pscdn.co`、`spotifycdn.com`、`spotifycdn.net`
- **第三方 CDN 别名 9 条**：Akamai / Fastly 共享域名（如 `audio-ak-spotify-com.akamaized.net`、`spotify.map.fastly.net`）。这类域名必须用 `DOMAIN` 精确匹配，写成 `DOMAIN-SUFFIX,akamaized.net` 会把整个 Akamai 网络的流量一起带走
- 客户端兜底 1 条：`USER-AGENT,*Spotify*`

### 4.4 `stream_us.list` / `stream_other.list`

两份互补的流媒体清单：

- `stream_us.list`（美区 IP 可访问，17 个平台）：Netflix、Disney+ / Hulu / ESPN / Star+、Max / HBO、Paramount+ / Showtime、Apple TV+、Prime Video、Peacock、Starz、Tubi、DAZN、MUBI、Criterion Channel、Kanopy、Crunchyroll、HIDIVE、Viki、JustWatch
- `stream_other.list`（严格限定对应国家 IP，美区节点打不开，7 个平台）：BBC iPlayer、ITV / ITVX、Channel 4、My5、日本 niconico、加拿大 Crave、澳洲 Stan

`stream_other.list` 建议直接给 `REJECT` 或 `DIRECT`，避免发无效的代理请求（发到美区节点上也只会报错）。

### 4.5 `tiktok.list`

基于 [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) 的 TikTok 规则（文件头声明上游更新至 2025-08-10），本地在其上补充了 5 条字节翻译接口：`bytedance.net`、`bytedanceapi.com`、`volcengineapi.com`、`translate.byteoversea.com`、`translate.bytedance.net`。

清单里除 TikTok 本体外，还含 CapCut、Trae、MarsCode、`byteoversea.com`、`ibyteimg.com`、`muscdn.com` 等字节系海外域名，以及 `DOMAIN-KEYWORD,tiktok` / `musical.ly` 两条宽匹配。

### 4.6 `AdultDomain.list`

成人 / NSFW 域名拦截表，19,976 条全部为 `DOMAIN-SUFFIX`，来源为 [Thoseyearsbrian/Aegis](https://github.com/Thoseyearsbrian/Aegis) 的 `rules/AdultDomain.list`（本地域名集合与上游完全一致，仅删除了文件头的元信息注释）。每条规则带行尾中文分类注释（成人广告推广平台 / 成人相关域名）。

引用时建议 `REJECT` + `pre-matching`，让这批域名在 DNS 解析前就被判掉。

---

## 五、规则文件书写规范

本仓库所有 `.list` 均为 Surge **外部 RULE-SET** 文件，遵循以下约束：

- 每行一条子规则，**不写策略名** —— 策略在 profile 的 `RULE-SET,...` 行统一指定，所以同一个文件可以被不同 profile 复用到不同策略上
- 注释支持 `#`、`//`、`;`，行首与行尾都可以
- 允许逐行参数：`no-resolve`、`extended-matching`
- **不允许**出现 `FINAL` 和 `pre-matching` 标记（后者只能加在 `[Rule]` 的 RULE-SET 行上）
- 非法行会被跳过并告警，不会导致整个规则集失效
- 单个规则集上限 1,000,000 条；超过 1,000 条域名规则会使用磁盘索引，匹配性能不受影响
- 同一个 URL / 文件路径不能同时被用作 RULE-SET 和 DOMAIN-SET

常见行示例：

```text
DOMAIN-SUFFIX,example.com
DOMAIN,cdn.example.org
DOMAIN-KEYWORD,generativelanguage
IP-CIDR,160.79.104.0/21,no-resolve
USER-AGENT,*Spotify*
```

---

## 六、维护与更新

**添加 / 修改规则**：直接编辑对应 `.list` → `git commit` → `git push`。Surge 端的 URL 规则集默认每 24 小时重新拉取一次；想立即生效就改用本地文件引用（见 3.2），或临时调小 `update-interval`。

**上游来源**：

| 文件 | 来源 | 本地改动 |
|---|---|---|
| `AdultDomain.list` | Thoseyearsbrian/Aegis | 仅删除文件头元信息注释 |
| `tiktok.list` | blackmatrix7/ios_rule_script（头部声明 2025-08-10） | 新增 5 条字节翻译接口 |
| 其余文件 | 自建 | — |

---

## 七、已知问题

1. **文件名不一致**：仓库里是 `claude.list`（小写），本地磁盘上是 `Claude.list`。macOS 大小写不敏感所以看不出来，但引用 URL 必须用小写，否则 GitHub 返回 404。建议后续统一改成小写。
2. **文件名含空格**：`Google Core Services.list` 在 URL 中必须转义为 `Google%20Core%20Services.list`。建议按早期版本重命名为 `google.list`，引用更省事。
3. **`DOMAIN-KEYWORD` 是宽匹配**：`tiktok.list` 的 `DOMAIN-KEYWORD,tiktok` 会连带匹配任何域名中带 `tiktok` 字样的第三方统计 / 追踪；`claude.list` 的 `DOMAIN-KEYWORD,datadoghq` 同理。
4. **`USER-AGENT` 规则的生效范围有限**：仅对 HTTP/HTTPS 请求有效；iOS 15 起系统不再在 CONNECT 请求中携带 UA，未开启 MITM 时 HTTPS 请求的 UA 规则实际不生效。因此 `spotify.list` 与 `tiktok.list` 里的 `USER-AGENT` 行只能算明文流量的兜底。
5. **`AdultDomain.list` 有 238 处重复域名**（19,976 行 → 19,738 个唯一域名），可用 `sort -u` 压缩。
6. **`main.py`** 是早期占位脚本，未参与规则生成。
7. 仓库未声明开源许可证。若要让他人复用，建议补一个（如 MIT）。

---

## 八、免责声明

本仓库为个人自用配置，规则内容按"能用就行"维护，不保证完整性与时效性。成人域名清单来自第三方开源项目，仅用于网络层拦截；请自行确认所在地区的合规要求，使用风险自负。
