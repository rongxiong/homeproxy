# 简洁模式直连响应变慢：原因分析与解决方案

| 项目 | 内容 |
| --- | --- |
| 分析日期 | 2026-11-07 |
| 代码基线 | `master` @ `46ce8bf`（修复前基线） |
| 分析范围 | 简洁模式的 DNS、路由、出站组、规则集下载及 nftables 接管路径 |
| 文档状态 | 2026-11-07 已按方案完成代码修复；第 2 节保留修复前分析，第 7 节记录实现状态，第 8 节记录免重新打包更新方式 |

> 第 2 节中的行号与行为描述以 `46ce8bf` 为准；当前实现请结合第 7 节和最新源码阅读。真机 A/B 测速仍需在实际 OpenWrt 设备上执行。

## 1. 结论

简洁模式下“直连网站响应慢”最值得优先处理的是 **DNS 与数据流量路径不一致**，而不是单纯减少路由规则数量。

当前默认配置可能形成以下路径：

```text
客户端访问一个未命中代理规则的网站
  ├─ DNS：第一路由组 → 代理节点 → TCP 远程 DNS
  └─ 数据：direct-out → 目标服务器
```

因此，即使数据最终完全直连，DNS 缓存未命中时仍可能先承担代理解析的建连和查询耗时。国内新域名、未被 `geolocation-cn` 收录的域名，以及国内规则集尚未就绪时，更容易出现该问题。

其他影响因素按优先级排序如下：

| 优先级 | 因素 | 判断 | 主要影响 |
| --- | --- | --- | --- |
| P0 | 未命中规则的直连流量默认使用代理远程 DNS | 代码行为明确，与现象高度吻合 | 冷 DNS 查询、首字节时间 |
| P0 | 国内 DNS 只依赖 `geolocation-cn`，缺少传统模式的组合回退 | 代码行为明确 | 未收录国内域名延迟、CDN 选址 |
| P1 | 大陆直连没有 nftables 提前放行，仍经过 sing-box | 代码行为明确，实际增幅依赖设备性能 | 用户态转发、规则匹配、CPU 开销 |
| P1 | 单节点规则组也强制生成 `urltest` | 代码行为明确，主要影响冷启动和故障场景 | 首次选路、辅助 DNS 和规则集下载 |
| P2 | 远程规则集下载与 DNS 共用第一路由组 | 代码行为明确，主要是启动期影响 | 服务就绪、规则暂时不命中 |
| 配置差异 | 简洁模式不等价于“绕过大陆”预设 | 语义差异，不是性能故障 | 海外流量可能按配置直连 |

> 当前没有真机日志和分段耗时数据，不能据此断言每一项增加了多少毫秒。下文将代码事实与性能推断分开，并给出验证方法。

## 2. 当前实现与代码证据

### 2.1 DNS 与数据出站不一致

简洁模式默认出站为 `direct-out`：

- `root/etc/config/homeproxy:58-66`：默认出站直连，远程 DNS 为 `8.8.8.8`，国内 DNS 为 `223.5.5.5`。
- `root/etc/homeproxy/scripts/generate_client.uc:558-563`：读取默认出站。
- `root/etc/homeproxy/scripts/generate_client.uc:1233`：写入 `route.final`。

只要存在可用代理节点，辅助出站又会影响全局 DNS：

- `generate_client.uc:565-576`：规则集下载与远程 DNS 共用 `aux_detour`；自动模式选择第一路由组。
- `generate_client.uc:775-791`：存在 `aux_detour` 时，`dns.final` 固定为 `simple-dns`，其 `detour` 指向该辅助出站。
- `generate_client.uc:155-168`、`778-790`：裸 IP DNS 默认转换为 TCP，因此默认是“经代理访问 TCP 8.8.8.8”。
- `generate_client.uc:798-814`：只有命中自动加入的 `geolocation-cn` 规则集时，才改用国内 DNS。

这意味着“添加任意一条代理规则”可能同时改变大量未命中规则的直连域名的 DNS 路径。数据虽然直连，首次访问却仍可能等待代理 DNS。

### 2.2 国内 DNS 分流覆盖不足

传统“绕过大陆”模式除了 `geosite-cn`，还使用“非 non-cn 且 geoip-cn”的组合规则，并支持 `direct-domain`：

- `generate_client.uc:640-645`：传统直连域名 DNS 规则。
- `generate_client.uc:673-694`：`geosite-cn` 与 `geoip-cn` 组合回退。
- `generate_client.uc:1069-1075`：传统直连域名路由规则。

简洁模式只有 `geolocation-cn` 一条国内 DNS 规则，且在 `64-89` 的传统配置读取分支中不会读取 `direct_list.txt` 和 `proxy_list.txt`。

由此可确定：未被 `geolocation-cn` 收录的国内域名会落入代理远程 DNS。它可能进一步带来两类问题：

1. 增加首次解析耗时；
2. 从代理出口参与 CDN 调度，返回不适合本地直连的地址。

第二项需要通过比较解析结果的 IP 归属和实际连接目标验证，不能仅凭代码认定已经发生。

### 2.3 大陆直连缺少内核态快速路径

传统模式在 TCP、UDP 和 TUN 路径中都会对大陆 IP 提前 `return`：

- `root/etc/homeproxy/scripts/firewall_post.ut:347-362`：TCP redirect 路径。
- `firewall_post.ut:468-487`：UDP TProxy 路径。
- `firewall_post.ut:629-648`：TUN 路径。

简洁模式没有这些分支。其实际直连路径为：

```text
nftables redirect / tproxy / tun
  → sing-box 入站
  → sniff 与规则匹配
  → route.final
  → direct-out
  → 目标服务器
```

`firewall_post.ut` 的注释说明：classic 主节点模式使用 nft 捷径，custom 可选开启大陆捷径，而 simple 始终把所有路由决定交给 sing-box。因此它是架构开销，不应直接当作缺陷修复。

该路径必然比内核态提前放行增加处理环节，但对响应时间的影响取决于路由器 CPU、连接数和规则规模。高性能设备上可能不明显，低性能设备或高并发下更值得关注。

### 2.4 强制 URLTest 与辅助路径耦合

- `generate_client.uc:1005-1035`：每个简洁规则组都生成 `urltest`，即使只有一个节点且没有 fallback。
- `generate_client.uc:565-576`：远程 DNS 和规则集下载默认复用第一组。
- `generate_client.uc:1235-1243`：所有简洁规则集均为远程规则集，并使用同一个辅助出站下载。

这可能放大冷启动、健康检查目标不可达或第一组节点异常时的等待。但已经确定为 `direct-out` 的稳定数据连接并不经过 URLTest，所以不能用健康检查解释所有稳态直连延迟。

远程规则集加载完成后，本地规则匹配也不等于“每次请求都联网下载规则”；其主要风险在启动、更新和下载失败阶段。

### 2.5 简洁模式不等于“绕过大陆”

传统“绕过大陆”模式的未命中流量走主代理，见 `generate_client.uc:1085`；大陆 IP 则在 nftables 中直接放行。

简洁模式默认是：

```text
命中用户规则 → 对应代理组
未命中规则 → direct-out
```

因此，如果用户期待简洁模式复刻“绕过大陆”，却没有配置非大陆代理规则，海外网站直连慢属于配置语义差异。仅自动添加国内 DNS 规则，不代表路由自动具备“非大陆走代理”的能力。

## 3. 临时缓解与定位方法

以下措施不需要修改代码，但部分有副作用，只适合诊断或短期使用。

### 3.1 优先做 DNS A/B 测试

修复后，下载和 DNS 已是两个独立选项，可在 LuCI 的“简洁分流”中分别测试：

1. **Remote DNS node = Automatic**：默认行为，DNS 与匹配的数据出站对齐；
2. **Remote DNS node = 固定的稳定低延迟节点**：排除自动路径和节点异常的影响；
3. **Remote DNS node = WAN DNS only**：所有查询使用 WAN DNS，用于判断代理 DNS 是否仍是瓶颈；
4. **Rule set download node = Direct**：仅让规则集下载直连，不再隐式改变 DNS 行为。

`WAN DNS only` 是明确的“停用远程 DNS 和国内 DNS 分流”选项，不再是下载节点的副作用。旧配置中的 `Download / DNS node = Direct` 会在升级时迁移为 `Remote DNS node = WAN DNS only`，保持原有语义。

每次只修改一个变量，使用同一批直连网站比较冷 DNS 和热 DNS 的首字节时间。若 WAN DNS only 明显改善冷查询，而热查询差异缩小，基本可确认 DNS 路径是主要因素。

### 3.2 检查规则集和第一路由组

- 确认 `geosite/geolocation-cn` 已成功下载并加载；
- 避免让低质量节点所在组成为第一路由组；
- 检查 `/var/run/homeproxy/homeproxy.log` 和 `/var/run/homeproxy/sing-box-c.log` 中的规则集下载、DNS 和健康检查异常；
- 规则集未就绪时的测速结果应单独标记，不与稳定状态混合。

### 3.3 如果目标是“绕过大陆”，修正配置语义

可以暂时使用传统 `bypass_mainland_china` 模式进行对照。若继续使用简洁模式，需要显式为应代理的域名或 IP 配置规则，例如 `geosite/geolocation-!cn`；该域名规则不能覆盖所有直接访问纯 IP 的非大陆流量，不能把“只添加这一条规则”当成完全等价迁移。

## 4. 推荐改造方案

### 4.1 P0：让 DNS 路径跟随数据分流意图

目标是避免“默认直连的数据，其 DNS 却因存在任意代理组而全部绕代理”。

建议采用以下自动策略：

1. **默认出站为 Direct 时**：未命中域名使用 WAN `default-dns`，不再默认经过第一代理组；
2. **默认出站为代理节点时**：未命中域名使用远程 DNS，并保持与该默认代理出站一致；
3. **命中代理域名规则的域名**：在 DNS 规则中优先走远程 DNS，且尽量使用与数据相同的路由组；
4. **国内域名**：继续使用直连的国内 DNS，规则顺序放在代理域名规则之后；
5. **geoip 和 IP CIDR 规则**：只参与数据路由，不强行复制为域名 DNS 条件。

若多个规则组都需要远程 DNS，可按组生成 `simple-dns-<group>`，避免所有域名都挤到第一组。自定义规则集无法可靠判断是域名还是 IP 类型时，应保守处理并记录日志，不要生成可能导致 DNS 查询失败的条件。

预期生成的逻辑为：

```text
代理域名 / 域名型代理规则集 → 对应代理组远程 DNS
国内域名及国内组合回退     → china-dns（直连）
其他域名
  ├─ 默认数据直连           → default-dns（WAN）
  └─ 默认数据代理           → 默认代理远程 DNS
```

### 4.2 P0：拆分规则集下载与远程 DNS 出站

当前 `ruleset_download_detour` 同时承担两种职责，建议拆分为：

- `ruleset_download_detour`：只控制规则集下载；
- `remote_dns_detour`：控制远程 DNS，支持 Auto、Direct 和指定节点；Auto 按 4.1 的路径对齐策略工作。

同时修正 Direct 的语义：它应表示“直接连接配置的远程 DNS”，而不是顺带关闭国内 DNS 分流。默认 DNS 回退和远程 DNS 是否启用应分别表达，避免一个下载选项隐式改变整套 DNS 行为。

升级兼容建议：

- 旧的空值 Automatic 可直接迁移到新 Auto，以修复全局代理 DNS 问题；
- 旧的显式节点可一次性复制到 `remote_dns_detour`，保留用户选择；
- 旧的 Direct 配置需要在迁移中明确转换为“WAN DNS 直连”或“指定远程 DNS 直连”，并记录发布说明，避免原本可用的 DNS 在升级后变为直连不可达的 `8.8.8.8`；
- 无论采用哪种迁移，都不能继续用同一个字段隐式控制下载、DNS 和国内分流开关。

### 4.3 P0：补齐国内 DNS 回退

参考传统模式增加以下能力：

- 保留 `geolocation-cn` 国内域名规则；
- 增加“非 non-cn 且 geoip-cn”的组合回退；
- 确保代理域名 DNS 规则优先于国内回退，防止代理域名被国内规则提前截获；
- 对规则集缺失时的行为记录明确日志，避免静默退化为全部远程解析。

`direct_list.txt` 是否应在简洁模式生效需要单独定义兼容策略。不建议未加说明地让旧列表突然改变简洁模式路由；可以提供显式导入或独立直连域名设置。

### 4.4 P1：单节点规则组不生成 URLTest

当规则组只有一个有效节点且没有 fallback 时，直接引用 `simple-n-*-out`；只有两个及以上节点时才生成 `urltest`。

实施时需要同步调整组标签复用和路由规则中的 outbound 引用，不能只删除 outbound 而留下悬空标签。多节点组仍保留现有“按顺序故障转移”语义，不能借此改成按最低延迟自动切换。

收益主要是减少不必要的健康检查和冷启动选路依赖，不应宣称它能降低所有直连连接的固定 RTT。

### 4.5 P1/P2：谨慎处理大陆 IP 快速路径

不建议给简洁模式无条件复制传统模式的大陆 IP nftables `return`。简洁模式允许用户把任意域名、geosite 或 geoip 规则指向代理，内核态提前放行可能绕过这些规则。例如“某代理域名解析到大陆 IP”时，提前放行会直接改变用户配置的分流结果。

可选方案按安全性排序：

1. 先完成 DNS 修复，并用真机数据确认是否仍存在明显直连开销；
2. 若确有性能瓶颈，提供独立的“绕过大陆预设”，明确其规则约束，而不是改变任意简洁配置的语义；
3. 只有在能够证明不存在会被提前放行覆盖的代理规则时，才考虑开放可选快速路径。

> 最终决定（2026-11-12）：不实现该快速路径，见第 7 节。简洁模式的全部流量交 sing-box 规则处理，需要国内直连时由内置直连节点表达。

### 4.6 P2：补充冷启动可观测性

在现有日志中增加以下信息：

- 实际 DNS final、远程 DNS detour 和国内 DNS 是否启用；
- 规则集下载开始、成功、失败和耗时；
- 单节点直连组与多节点 URLTest 的生成结果；
- 国内规则集不可用时的 DNS 降级行为。

日志不记录节点密钥或完整敏感配置。运行界面可显示“规则集就绪”和“代理 DNS 可用”状态，方便区分启动延迟与稳态网络延迟。

## 5. 验证方案

### 5.1 配置生成与功能测试

至少覆盖以下用例：

| 用例 | 预期结果 |
| --- | --- |
| 默认 Direct + 一条代理域名规则 | 未命中 DNS 走 WAN；代理域名 DNS 走对应代理组 |
| 默认 Direct + 无代理节点 | DNS 和数据均可直连，不生成无效辅助出站 |
| 默认出站为代理节点 | 未命中 DNS 与数据均走默认代理 |
| 国内域名 + 代理域名同时存在 | 规则顺序正确，互不抢占 |
| 单节点无 fallback | 不生成 URLTest，路由无悬空 outbound |
| 多节点有 fallback | 保留 URLTest 和原有故障转移顺序 |
| 规则集下载失败 | 日志明确，DNS 降级行为符合设计 |
| 旧配置升级 | Automatic、Direct、指定节点三种配置均有确定迁移结果 |

生成配置后执行 `sing-box check`，并核对实际 `/var/run/homeproxy/sing-box-c.json` 中的 `dns.final`、各 DNS 规则的 `server`、`detour` 和 `route.final`。

### 5.2 真机 A/B 性能测试

从被 HomeProxy 接管的 LAN 客户端测试，避免路由器本机流量绕过待测路径。选择以下固定样本：

1. 已被 `geolocation-cn` 收录的国内直连域名；
2. 未被收录的国内直连域名；
3. 明确命中代理规则的域名；
4. 纯 IP 直连目标，用于隔离 DNS 因素。

每个场景分别测冷 DNS 和热 DNS，每轮只改变一个配置项。建议每轮至少 20 次，记录中位数和 P95，不只比较单次浏览器体感。

HTTP/HTTPS 测试可记录：

```sh
curl -o /dev/null -sS \
  -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  'https://example.com/'
```

将 `example.com` 替换为实际测试地址。DNS 测试同时记录查询耗时和返回 IP，检查是否因代理解析获得异常 CDN 地址。冷查询应使用未查询过的样本或在受控条件下清理相关缓存，不能把 dnsmasq 已缓存的结果当作冷查询。

建议对照组：

- A：修复前基线的简洁模式 Automatic；
- B：修复前基线的 Download / DNS node = Direct，或修复后的 Remote DNS node = WAN DNS only；
- C：修复后的 Remote DNS node = Automatic；
- D：传统绕过大陆模式，仅作为路径参考，不要求所有场景语义相同。

### 5.3 验收标准

- 默认直连的未命中域名，其生成配置不再经过代理 DNS；
- 代理域名的 DNS 与数据出站符合规则，未被国内 DNS 回退提前截获；
- 冷 DNS 的直连网站 TTFB 相较修复前改善，且差异不只出现在单次测试中；
- 热 DNS 下没有引入新的明显回退；
- 单节点、无节点、规则集失败和旧配置迁移场景均可正常启动和解析；
- 内核快速路径最终未启用（见第 7 节）；若未来重新引入，必须通过“大陆 IP 仍需代理”的反向用例，证明没有绕过用户规则。

## 6. 推荐实施顺序

1. **先测量**：完成 Automatic / Direct / 固定节点的 DNS A/B，保存分段耗时和解析 IP；
2. **第一阶段**：修复默认直连时的 DNS final，并为代理域名生成对应 DNS 规则；
3. **第二阶段**：拆分下载与 DNS 配置，补齐国内 DNS 回退和升级迁移；
4. **第三阶段**：优化单节点 URLTest，补充冷启动日志；
5. **最后评估**：评估结论是不启用大陆 IP 快速路径；改为由内置直连节点 + sing-box 规则表达国内直连策略（见第 7 节）。

这样可以用最小行为变更先解决最符合现象的瓶颈，同时避免为了性能牺牲简洁模式的规则正确性。

---

## 7. 修复状态（2026-11-07）

| 方案 | 实现状态 | 说明 |
| --- | --- | --- |
| 4.1 DNS 与数据路径对齐 | 已完成 | DNS 规则与路由规则按同一行顺序、同一批域名条件生成：代理行用对应路由组的远程 DNS，直连行用本地/国内 DNS；默认出站直连用 WAN DNS，默认出站代理用对应远程 DNS |
| 4.2 下载与 DNS 出站拆分 | 已完成 | 新增 `simple.remote_dns_detour`；`ruleset_download_detour` 仅控制规则集。支持 Automatic、WAN DNS only、Direct 和指定节点；Automatic 复用第一个分流出站及其 fallback |
| 旧配置迁移 | 已完成 | 新增一次性 `migration.simple_dns_split`；旧 Automatic 采用新 Auto，旧显式节点复制到新字段，旧 Direct 转为 WAN DNS only |
| 4.3 国内 DNS 回退 | 已完成 | 加入 `geolocation-cn`、`geolocation-!cn` 取反与 `geoip/cn` 组合回退；代理域名 DNS 规则排在前面 |
| 4.4 单节点 URLTest | 已完成 | 单节点规则直接引用节点出站；多节点规则保留 URLTest 与有序故障转移 |
| 4.5 大陆 IP 快速路径 | 已移除 | 简洁模式删除 `simple.bypass_cn_traffic`、生成器状态文件与 nft 层中国 IP 提前放行；所有被接管的流量都进入 sing-box，由分流规则决定去向。需要国内流量直连时，可在规则中显式选用内置的直连节点 |
| 4.6 冷启动可观测性 | 已完成 | `homeproxy.log` 记录规则集排队与下载出站、DNS final、代理 DNS 规则数、国内分流状态与 URLTest 组数 |
| UI、默认配置与 i18n | 已完成 | 新选项和提示已加入 LuCI、UCI 默认值、迁移脚本及中英文词条；默认出站说明明确“简洁模式不等于绕过大陆” |

实现约束与剩余工作：

- 规则集的实际下载成功/失败仍由 sing-box 在运行时写入 `sing-box-c.log`；生成器只能记录请求、URL 和出站，不能在不接入 sing-box 日志的情况下给出每次下载的准确完成耗时。
- 简洁模式不再有跨进程状态文件：`simple-bypass-cn` 的生成器写入、防火墙读取与 `init.d` 清理均已删除，生成器与防火墙之间不再有隐式运行时依赖。
- 内置直连节点（`config node 'builtin_direct'`，`type=direct`）可作为规则主节点、备用节点或默认出站；指向纯直连节点的规则在 DNS 侧走本地 `default-dns` / `china-dns`，不会再落到可能是代理的 `dns.final`；规则集下载的 Automatic 会跳过纯直连出站。
- 规则集下载的 Automatic 复用第一个代理分流出站；若该出站是多节点 URLTest 组，则保留 fallback 顺序，优先保证下载可用性。
- DNS 阶段拿不到目标 IP / 端口 / 协议，故 `geoip/*` 规则集、IP/CIDR 与端口条件不生成 DNS 条件；明确为 GeoIP 的自定义规则集会跳过并记录日志，避免把 IP 规则集误用于域名解析。这类规则的数据出站与 DNS 解析器可能不一致，属固有限制。
- `direct_list.txt` 未在简洁模式自动生效，避免旧域名列表静默改变用户当前策略；如需支持，应另行设计显式导入。
- 大陆流量不再由静态 `china_ip4.txt` / `china_ip6.txt` 在内核态提前放行，口径与远程 GeoSite/GeoIP 规则集保持一致；代价是国内流量多一次用户态转发。
- 真机 `sing-box check`、生成 JSON 字段核对和第 5 节的冷/热 DNS A/B 测试仍需在 OpenWrt 设备上执行；静态代码修复不能代替实测性能结论。

---

## 8. 免重新打包更新到 OpenWrt 设备

本节用于把当前工作区的修复直接同步到已安装 HomeProxy 的设备，无需重新构建 ipk。以下命令在本机执行，按实际情况替换 `ROUTER`。

### 8.1 必须使用 `scp -O`，不要使用普通 `scp`

新版 OpenSSH（9.0+，含当前 macOS 自带 `scp`）默认使用 SFTP 子协议。很多 OpenWrt / Dropbear 环境没有 `/usr/libexec/sftp-server`，普通 `scp` 会出现：

```text
ash: line 0: /usr/libexec/sftp-server: not found
scp: Connection closed
```

解决方法是给所有 `scp` 增加大写字母 `-O`，强制使用旧版 SCP 协议：

```sh
scp -O ...
```

注意：

- `-O` 是大写字母 O，不是小写 `-o`；小写表示传递 ssh 配置选项。
- 不需要在路由器上安装 `sftp-server`。
- 本机可用 `scp -O` 查看 usage 来确认支持该参数；不支持时改用 8.5 的 `tar` 方案。
- 每条 `scp` / `ssh` 都可能要求一次密码，多次提示是正常的。

### 8.2 路径映射

打包脚本 `.github/build-ipk.sh` 使用以下映射：

| 工作区路径 | 设备路径 |
| --- | --- |
| `htdocs/luci-static/resources/...` | `/www/luci-static/resources/...` |
| `root/etc/homeproxy/scripts/...` | `/etc/homeproxy/scripts/...` |
| `root/etc/init.d/homeproxy` | `/etc/init.d/homeproxy` |
| `po/zh_Hans/homeproxy.po` | 需先由 `po2lmo` 编译为 `/usr/lib/lua/luci/i18n/homeproxy.zh-cn.lmo` |

`docs/*.md`、`po/templates/homeproxy.pot` 不参与运行时，不需要复制。

> 不要直接覆盖 `/etc/config/homeproxy`：这是用户配置，覆盖会丢失节点、规则和现有选项。新增字段由 `migrate_config.uc` 补齐。

### 8.3 设备已应用主要修复，只需更新 LuCI 前端时

例如只应用「Automatic 选项允许为空、表单可保存」这类前端修复：

```sh
SRC=/Users/chenrongxiong/GitHub/homeproxy
ROUTER=root@192.168.6.1

scp -O "$SRC/htdocs/luci-static/resources/view/homeproxy/client.js" \
    "$ROUTER:/www/luci-static/resources/view/homeproxy/client.js"

ssh "$ROUTER" 'chmod 644 /www/luci-static/resources/view/homeproxy/client.js'
```

随后强制刷新浏览器缓存（`Cmd/Ctrl + Shift + R`）。必要时可执行：

```sh
ssh "$ROUTER" '/etc/init.d/uhttpd restart'
```

这种情况不需要执行 UCI 迁移，也不需要重启 HomeProxy。

### 8.4 设备仍是旧版时，同步全部运行文件

```sh
SRC=/Users/chenrongxiong/GitHub/homeproxy
ROUTER=root@192.168.6.1

scp -O "$SRC/htdocs/luci-static/resources/view/homeproxy/client.js" \
    "$ROUTER:/www/luci-static/resources/view/homeproxy/client.js"

scp -O "$SRC/htdocs/luci-static/resources/homeproxy.js" \
    "$ROUTER:/www/luci-static/resources/homeproxy.js"

scp -O \
    "$SRC/root/etc/homeproxy/scripts/generate_client.uc" \
    "$SRC/root/etc/homeproxy/scripts/firewall_post.ut" \
    "$SRC/root/etc/homeproxy/scripts/migrate_config.uc" \
    "$SRC/root/etc/homeproxy/scripts/update_subscriptions.uc" \
    "$ROUTER:/etc/homeproxy/scripts/"

scp -O "$SRC/root/etc/init.d/homeproxy" \
    "$ROUTER:/etc/init.d/homeproxy"

ssh "$ROUTER" '
chmod 644 /www/luci-static/resources/view/homeproxy/client.js /www/luci-static/resources/homeproxy.js
chmod 755 /etc/init.d/homeproxy /etc/homeproxy/scripts/*.uc /etc/homeproxy/scripts/*.ut
ucode /etc/homeproxy/scripts/migrate_config.uc
/etc/init.d/homeproxy restart
/etc/init.d/dnsmasq restart
'
```

同步的是**当前工作区文件**，不要只复制某个旧 commit 的快照；未提交的前端保存修复也包含在工作区文件中。

中文翻译是可选的。路由器上存在 `po2lmo` 时可执行：

```sh
scp -O "$SRC/po/zh_Hans/homeproxy.po" "$ROUTER:/tmp/homeproxy.po"

ssh "$ROUTER" '
command -v po2lmo >/dev/null || exit 0
po2lmo /tmp/homeproxy.po /usr/lib/lua/luci/i18n/homeproxy.zh-cn.lmo
chmod 644 /usr/lib/lua/luci/i18n/homeproxy.zh-cn.lmo
rm -f /tmp/homeproxy.po
/etc/init.d/uhttpd restart
'
```

没有 `po2lmo` 时可以跳过；功能不受影响，但新增或修改的文案可能暂时显示英文。

### 8.5 只想输入一次密码的替代方案

不依赖 SFTP、SCP 的稳妥方式是用 `tar` 通过一次 SSH 连接传输所有文件：

```sh
SRC=/Users/chenrongxiong/GitHub/homeproxy
ROUTER=root@192.168.6.1
TMP=$(mktemp -d)

mkdir -p "$TMP/www/luci-static/resources/view/homeproxy" \
         "$TMP/etc/homeproxy/scripts" "$TMP/etc/init.d"

cp "$SRC/htdocs/luci-static/resources/view/homeproxy/client.js" \
   "$TMP/www/luci-static/resources/view/homeproxy/client.js"
cp "$SRC/htdocs/luci-static/resources/homeproxy.js" \
   "$TMP/www/luci-static/resources/homeproxy.js"
cp "$SRC/root/etc/homeproxy/scripts/generate_client.uc" \
   "$SRC/root/etc/homeproxy/scripts/firewall_post.ut" \
   "$SRC/root/etc/homeproxy/scripts/migrate_config.uc" \
   "$SRC/root/etc/homeproxy/scripts/update_subscriptions.uc" \
   "$TMP/etc/homeproxy/scripts/"
cp "$SRC/root/etc/init.d/homeproxy" "$TMP/etc/init.d/homeproxy"

tar -C "$TMP" -cf - www etc | ssh "$ROUTER" '
tar -C / -xf - &&
chmod 644 /www/luci-static/resources/view/homeproxy/client.js /www/luci-static/resources/homeproxy.js &&
chmod 755 /etc/init.d/homeproxy /etc/homeproxy/scripts/*.uc /etc/homeproxy/scripts/*.ut &&
ucode /etc/homeproxy/scripts/migrate_config.uc &&
/etc/init.d/homeproxy restart &&
/etc/init.d/dnsmasq restart
'

rm -rf "$TMP"
```

### 8.6 更新后验证

```sh
ssh "$ROUTER" '
uci show homeproxy.simple
uci show homeproxy.migration
sing-box check --config /var/run/homeproxy/sing-box-c.json
logread -e homeproxy | tail -n 50
'
```

然后在 LuCI 保存一次表单，确认 Automatic 可以保存，并强制刷新浏览器缓存。

> 手动覆盖的文件会在以后通过 `opkg` / `apk` 升级 HomeProxy 包时被覆盖；正式发布时应重新打包，或在每次升级后重新执行本节同步步骤。

---

## 9. 用空闲内存换 DNS 性能：简洁模式缓存（2026-11-08）

简洁模式的 DNS 链路是 `dnsmasq → sing-box`，两层默认都使用保守缓存：dnsmasq 缓存条目少，sing-box 按 TTL 过期。设备内存有余量时，可以在不改变分流语义的前提下提高命中率。

| UCI 选项（`config homeproxy 'simple'`） | 默认值 | 作用层 | 行为 |
| --- | --- | --- | --- |
| `dns_cache_size` | `4096` | dnsmasq + sing-box | 简洁模式下写入 dnsmasq `cache-size`，并按同一数值生成 sing-box `dns.cache_capacity`；dnsmasq 侧只增不减（系统 `cachesize` 更大时保持系统值），设为 `0` 时不覆盖 dnsmasq 设置、sing-box 保持默认容量 |
| `dns_optimistic` | `0` | sing-box | 生成 `dns.optimistic`（sing-box 1.14+），过期条目在陈旧窗口内立即返回并后台刷新，默认窗口 3 天 |

约束与风险：

- 容量基线：dnsmasq 编译默认 150 条，OpenWrt 典型 `cachesize=1000`；sing-box 的 `dns.cache_capacity` 默认 1024 条。默认 4096 将两层同时提升，主要影响活跃域名超过 1000 个的环境；家庭网络常见 100–500 个活跃域名时，命中率提升有限。
- 内存开销：dnsmasq 每条约 100–200 字节（4096 条约 0.5–1 MB）；sing-box 的 LRU 保存完整 DNS 消息，4096 条通常为数 MB 量级，应以设备 RSS 实测为准。dnsmasq 官方提示缓存过大反而影响性能，`10000` 是本项目为控制内存设置的上限，不是 dnsmasq 的硬上限。
- 覆盖顺序：OpenWrt 生成的 dnsmasq 主配置中 `cache-size` 先于 `conf-dir` 写出，因此 drop-in 中的 `cache-size` 会覆盖 `/etc/config/dhcp` 的值；本实现只提升不降低，`0` 表示不覆盖，系统显式 `cachesize=0`（禁用缓存）时也不会覆盖。
- `dns_optimistic` 默认关闭。开启后 sing-box 会把已过期但仍在陈旧窗口（默认 3 天）内的答案立即返回，并在后台刷新；超过窗口后恢复正常解析。它比 `disable_expire` 有明确边界，但 CDN 调度、节点切换或域名换 IP 后仍可能短暂使用旧地址，不适合出口频繁变化的环境。`optimistic` 与 `disable_cache` / `disable_expire` 互斥，简洁模式不生成后两者。
- 版本要求：`optimistic` 只在 sing-box 1.14+ 可用；低版本上手动开启会导致 `sing-box check` 失败、服务无法启动。默认关闭时不生成该字段，低版本升级后仍可正常运行。
- 两项都只在简洁模式生效；其他路由模式的 DNS 配置不变。旧配置由 `migrate_config.uc` 补齐默认值；`init.d` 记录 dnsmasq 缓存大小，生成器记录 sing-box 容量与乐观缓存策略，均写入 `homeproxy.log`。

真机验证：

```sh
grep -R 'cache-size' /tmp/dnsmasq.d /tmp/etc/dnsmasq.d 2>/dev/null
grep 'cache_capacity' /var/run/homeproxy/sing-box-c.json
grep 'optimistic' /var/run/homeproxy/sing-box-c.json
sing-box check --config /var/run/homeproxy/sing-box-c.json
kill -USR1 "$(pidof dnsmasq)" && logread -e dnsmasq | tail -n 20
```

`optimistic` 默认不写入生成配置，因此对应 `grep` 无输出属于正常；`cache_capacity` 默认应为 4096。`kill -USR1` 输出的缓存统计（cache size、insertions、evictions、hits/misses）可用于判断扩容是否真正减少淘汰；若 evictions 长期接近 0，继续增大容量不会带来收益。

缓存效果应沿用第 5 节的冷/热 DNS A/B 方法对比：热查询的 `time_namelookup` 与直连 TTFB 是主要观察指标，冷查询不应因缓存而变差。只有 dnsmasq 的 hits 显著增加、上游查询减少时，才能认为缓存优化产生了实际性能收益。
