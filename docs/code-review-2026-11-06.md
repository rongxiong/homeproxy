# HomeProxy 代码审查报告

| 项目 | 内容 |
| --- | --- |
| 审查日期 | 2026-11-06 |
| 代码基线 | `master` @ `df8a958`（`feat(client): 扩充 GeoIP 规则集选项`） |
| 审查范围 | 仓库全量：LuCI 前端 4 个视图 + 公共库、6 个 ucode 脚本、nft 模板、`init.d`、RPC/ACL、菜单、打包与 CI、i18n |
| 文档状态 | 初版（审查基线 `df8a958`）仅记录问题、未改动代码；同日修复完成后补充第 8 节「修复状态与验证记录」。正文行号一律以审查基线为准 |
| 结论 | 未发现语法/打包级致命问题；共记录 17 项（5 高 / 6 中 / 6 低）。修复后 16 项完全解决，1 项（HP-06）作为文档化风险保留，详见第 8 节 |

---

## 1. 审查方法

### 1.1 执行过的检查

| 检查项 | 方法 | 结果 |
| --- | --- | --- |
| JSON 合法性 | `python3 -c "json.load(...)"` 遍历所有 `*.json` | 全部通过 |
| Shell 语法 | `sh -n`（对 `#!/bin/bash` 脚本改用 bash 语义人工核对） | 全部通过 |
| 前端 JS 语法 | `node --check` 覆盖 `homeproxy.js` + 4 个视图 | 全部通过 |
| i18n 同步 | 新增 `_()` 文案逐一比对 `.pot` / `zh_Hans.po` | 同步（含简洁模式全部新词条） |
| ACL / 菜单 | 与前端 `rpc.declare` 的方法、`fs` 读取路径逐一对照 | 完整 |
| 前后端字段一致性 | 提取 UI option 名，与 `generate_client.uc` / `generate_server.uc` 实际读取的键交叉比对 | 发现 2 处不一致（HP-02、N-04） |
| 模式分支登记 | 对 `simple` 模式核对 `init.d` / `firewall_pre.uc` / `firewall_post.ut` / `migrate_config.uc` / `update_subscriptions.uc` | 均已登记（见附录 A） |

### 1.2 未能执行的检查

- **ucode 运行时验证**：本机无 `ucode`（`which ucode` 为空），无法实际运行 `generate_client.uc` / `firewall_post.ut`。
- **`sing-box check` 验证**：无 OpenWrt 设备/容器，无法对生成的 JSON 做 schema 校验。
- 因此涉及 sing-box 字段类型/语义的判断，均基于 sing-box 官方 schema（严格 JSON 反序列化）推导，并在各条目中标注置信度。

### 1.3 近期改动背景

本地新增 6 个提交（作者 `chenrx`，2026-11-06），核心是「简洁分流」模式：

```
df8a958 feat(client): 扩充 GeoIP 规则集选项
c52973b fix(client): 增强 DNS 服务器校验并优化辅助节点选择
afdca3d fix(client): 清理合并冲突标记残留
0a4778d Merge branch 'master' ...
6e4ea76 feat(client): 简洁模式新增国内 DNS 分流
a7d4f00 feat(client): 新增简洁分流路由模式
```

本次审查对这部分做了重点核对（数据模型、生成器分支、防火墙/启动脚本登记、UI 显隐依赖、文案）。整体设计成立：`urltest + 有序 outbounds + 极大 tolerance` 实现「按序故障转移」，规则集按 URL 去重、出站组按签名复用，未发现组合爆炸或循环依赖问题。以下问题中 HP-01（守卫为本次新增但未生效）、HP-06、HP-09、HP-11 与 N-03、N-05 与本轮改动直接相关；HP-05 属既有缺陷，但在简洁模式的端口跳跃等场景同样会暴露。

---

## 2. 项目结构与关键数据流

```
htdocs/luci-static/resources/
  homeproxy.js                    公共库：DNS 策略枚举、加密套件、规则集精选清单、校验器
  view/homeproxy/client.js        客户端设置（路由模式、简洁分流、自定义路由/DNS/规则集、ACL）
  view/homeproxy/node.js          节点管理、分享链接导入、订阅更新触发
  view/homeproxy/server.js        服务端入站管理、证书/ECH/ACME
  view/homeproxy/status.js        连接检测、资源版本、运行日志

root/
  etc/config/homeproxy            UCI 默认配置（infra / config / control / routing / simple / dns / subscription / server）
  etc/init.d/homeproxy            procd 启动脚本：生成配置 → sing-box check → dnsmasq/ip rule/nft → 启进程
  etc/homeproxy/scripts/
    homeproxy.uc                  公共工具（ISO/URL/Base64/校验/命令执行）
    generate_client.uc            客户端 sing-box JSON 生成（5+1 种路由模式分支）
    generate_server.uc            服务端 sing-box JSON 生成
    update_subscriptions.uc       订阅拉取 / 节点差分写入 UCI
    migrate_config.uc             版本升级迁移（uci-defaults 触发一次）
    firewall_pre.uc               写 fw4 include 片段（forward/input 放行）
    firewall_post.ut              nft 模板（dstnat / mangle / tproxy / tun / ACL 集合）
  usr/share/rpcd/ucode/luci.homeproxy   RPC：ACL 列表读写、证书写入、连接检测、生成器、资源更新
  usr/share/rpcd/acl.d / luci/menu.d   权限与菜单
```

### 2.1 路由模式分支矩阵

| 关注点 | gfwlist | bypass_mainland_china | proxy_mainland_china | global | custom | simple |
| --- | --- | --- | --- | --- | --- | --- |
| 出站来源 | `config.main_node` | 同左 | 同左 | 同左 | `routing_node` 表 | `simple_rule` 表 |
| `generate_client.uc` 主分支 | `main_node` | 同左 | 同左 | 同左 | `default_outbound` | `simple` 翻译层 |
| DNS 接管（dnsmasq） | 分流 + nftset | 全部转发 | 分流 | 全部转发 | 全部转发 | 全部转发 |
| nft geo 直连分支 | gfw 集合 | 大陆集合 | 反向大陆集合 | 无 | 可选 `bypass_cn_traffic` | 无（交给 sing-box） |
| tproxy UDP 生效条件 | `main_udp_node != nil` | 同左 | 同左 | 同左 | 恒生效 | 恒生效 |
| ujail 沙箱 | 启用 | 启用 | 启用 | 启用 | 关闭 | **关闭**（见 HP-06） |
| `experimental.cache_file` | 关闭 | 开启 | 关闭 | 关闭 | 开启 | 开启 |

---

## 3. 问题总览

| ID | 严重度 | 位置（审查基线） | 问题摘要 | 置信度 | 状态 |
| --- | --- | --- | --- | --- | --- |
| HP-01 | **高** | `client.js:1718-1755` | custom/simple 模式下保存表单会清空 `direct_list.txt` / `proxy_list.txt` | 高 | 已修复 |
| HP-02 | **高** | `generate_server.uc:70` | 字段名拼写错误导致服务端 `recv_window_client` 永远不生效 | 高 | 已修复 |
| HP-03 | **高** | `generate_server.uc:132` | `disable_tls_alpn_challenge` 缺少 `strToBool`，生成字符串导致 `sing-box check` 失败 | 高 | 已修复 |
| HP-04 | **高** | `client.js:974-1028` / `generate_client.uc:1164` | 路由规则 `action=resolve` 的 5 个选项后端从未读取 | 高 | 已修复 |
| HP-05 | **高** | `homeproxy.js:372-389` | 端口范围校验用字符串比较，合法范围被拒绝 | 高 | 已修复 |
| HP-06 | 中 | `init.d:157` | simple 模式跳过 ujail，进程以 root 运行（安全面回退） | 高 | 保留（文档化风险） |
| HP-07 | 中 | `update_subscriptions.uc:40` | `bypass_mainalnd_china` 拼写错误 | 高 | 已修复 |
| HP-08 | 中 | `update_subscriptions.uc:599-620` | `main_node='nil'` 会被订阅更新改写为第一个节点 | 高 | 已修复（复查补齐） |
| HP-09 | 中 | `client.js:426-428` | `ruleset_update_interval` 无格式校验，非法值会让服务起不来 | 高 | 已修复 |
| HP-10 | 中 | `init.d:285-291` | `stop_service` 清理的 nft 链名与模板不一致 | 高 | 已修复 |
| HP-11 | 中 | `generate_client.uc:552-568,762-801` | 选「Direct」会静默关闭远端 DNS 与国内 DNS 分流 | 高 | 已修复（文案） |
| N-01 | 低 | `node.js:41/57/77/100` 等 | TLS 类协议分享链接缺省端口回退为 `80` | 高 | 已修复（复查补齐 `hy2`） |
| N-02 | 低 | `firewall_pre.uc:49-53` | include 文件仅在非空时写入，存在残留风险 | 中 | 已修复 |
| N-03 | 低 | `generate_client.uc:426-437` | simple 分流行悬空引用被静默丢弃，无日志 | 高 | 已修复 |
| N-04 | 低 | `generate_server.uc:102` | `hysteria_protocol` 为死字段（UI 有、生成器不用） | 高 | 已修复（注释） |
| N-05 | 低 | `client.js:90-105,362-367` | `isValidDNSServer` 对 `simple.dns_server` 放行 `nil` | 高 | 已修复 |
| N-06 | 低 | `.github/workflows/build-ipk.yml` | CI 无语法/静态校验步骤 | 高 | 已修复 |

> 状态为 2026-11-06 复查结果，修复后的具体位置与验证方式见第 8 节。正文中的行号不再随修复更新，以便与初版审查结论对照。

---

## 4. 详细问题

### HP-01（高）custom/simple 模式保存表单会清空域名列表文件

**位置**

- `htdocs/luci-static/resources/view/homeproxy/client.js:1718-1723`（Proxy Domain List）
- `htdocs/luci-static/resources/view/homeproxy/client.js:1750-1755`（Direct Domain List）

**代码**

```js
so.remove = function(/* ... */) {
    let routing_mode = this.section.formvalue('config', 'routing_mode');
    if (!['custom', 'simple'].includes(routing_mode))
        return callWriteDomainList('proxy_list', '');
    return true;
}
```

**现象**

在「自定义路由」或「简洁分流」模式下打开 Client Settings 并点击「保存并应用」，`/etc/homeproxy/resources/proxy_list.txt` 与 `direct_list.txt` 会被写空；切回 GFWList / 绕过大陆等模式后，之前配置的代理/直连域名全部分流失效。

**根因（两层）**

1. `so.depends('homeproxy.config.routing_mode', legacyModes)` 使该选项在 custom/simple 下为 *inactive*。LuCI 的 `CBIValue.parse()` 对 inactive 且 `retain == false` 的选项会调用 `option.remove(section_id)`（这是 `remove` 覆写存在的原因）。
2. 覆写里的 `this.section.formvalue('config', 'routing_mode')` **永远返回 `null`**：
   - `this.section` 是该选项所属的 `NamedSection`（即 `o.subsection`，映射到 `control` 段），而 LuCI 的 `CBIAbstractSection.formvalue(section_id, option)` 只在本 section 的 `children` 中按 `o.option == option` 查找；
   - `control` 段没有名为 `routing_mode` 的选项 → 返回 `null`；
   - `!['custom','simple'].includes(null) === true` → 无条件执行清空写入。

**影响面**

- custom 模式：该缺陷长期存在（本轮改动只是试图加守卫，但守卫未生效）。
- simple 模式：本轮新增，用户量预期更大；两个列表在该模式下虽然不参与生成，但一旦清空，切回旧模式即不可恢复（无备份）。

**复现步骤**

1. 路由模式选 `bypass_mainland_china`，在 Proxy/Direct Domain List 填入若干域名并保存 → 确认 txt 文件有内容。
2. 路由模式改为 `simple`（或 `custom`）→ 保存并应用。
3. `cat /etc/homeproxy/resources/{proxy,direct}_list.txt` → 内容为空。

**修复建议（未实施）**

```js
so.remove = function(/* ... */) {
    // 直接读 UCI，不要依赖 this.section（control 段里没有 routing_mode）
    let routing_mode = uci.get('homeproxy', 'config', 'routing_mode');
    if (!['custom', 'simple'].includes(routing_mode))
        return callWriteDomainList('proxy_list', '');
    return true;
}
```

两处（`proxy_list` / `direct_list`）都需要改。回归验证：在 6 种模式下各保存一次，确认非 legacy 模式下两个文件 mtime 与内容不变。

---

### HP-02（高）服务端 `recv_window_client` 字段名拼写错误

**位置**：`root/etc/homeproxy/scripts/generate_server.uc:70`

```ucode
recv_window_client: strToInt(cfg.hysteria_revc_window_client),
```

**根因**：服务端 UI 写入的键是 `hysteria_recv_window_client`（`server.js:307`），生成器读取的键是 `hysteria_revc_window_client`（`recv` → `revc`）。键名不匹配 → `cfg.hysteria_revc_window_client` 恒为 `null` → `recv_window_client` 被 `removeBlankAttrs` 移除。

**影响面**：Hysteria 入站配置页的「QUIC connection receive window」完全无效，即使填写也不会出现在 `sing-box-s.json` 中，服务端流控窗口始终保持 sing-box 默认值。

**旁证**：客户端侧是「一致的错拼」——`node.js:599` 写 `hysteria_revc_window`，`generate_client.uc:253` 读 `hysteria_revc_window`，因此客户端选项可用；这也说明服务端是单侧错拼。

**修复建议（未实施）**

1. 统一为正确拼写：`generate_server.uc` 改为 `cfg.hysteria_recv_window_client`；同时把 `node.js` / `generate_client.uc` 的 `hysteria_revc_window` 改为 `hysteria_recv_window`。
2. 由于 UCI 已存旧键，需要在 `migrate_config.uc` 增加一次 `uci.rename` 迁移（只对 `node` 段的旧键；服务端旧键未被使用过，可直接不需要迁移）。

**回归验证**：服务端启用 Hysteria，填写窗口值 → 生成 JSON 中出现 `recv_window_client`；客户端节点填写 → 出现 `recv_window`。

---

### HP-03（高）`disable_tls_alpn_challenge` 生成字符串，导致配置校验失败

**位置**：`root/etc/homeproxy/scripts/generate_server.uc:131-132`

```ucode
disable_http_challenge: strToBool(cfg.tls_acme_dhc),
disable_tls_alpn_challenge: (cfg.tls_acme_dtac),
```

**根因**：`tls_acme_dtac` 是 `form.Flag`，UCI 值为字符串 `'1'`/`'0'`。ucode 中非空字符串恒为真值，直接放入对象会原样序列化为 JSON 字符串：

```json
"disable_tls_alpn_challenge": "1"
```

sing-box 该字段类型为 `bool`，严格反序列化会失败。

**影响面**：服务端启用 ACME 且勾选「Disable TLS ALPN challenge」时，`sing-box check` 失败 → `start_service` 直接 `return 1`，客户端与服务端实例均不启动（该检查在 server 段落之前完成，属于启动关键路径）。

**修复建议（未实施）**

```ucode
disable_tls_alpn_challenge: strToBool(cfg.tls_acme_dtac),
```

同一对象内 `disable_http_challenge` 已是正确写法，可直接对齐。修复后建议对 `generate_server.uc` 全量扫一遍同类问题（本次仅发现此一处，`strToBool`/`strToInt` 其余使用均正确）。

---

### HP-04（高）路由规则 `action=resolve` 的选项后端从未读取

**位置**

- UI：`client.js:974-1028`（`resolve_server` / `resolve_strategy` / `resolve_disable_cache` / `resolve_rewrite_ttl` / `resolve_client_subnet`，均 `depends('action', 'resolve')`）
- 生成器：`generate_client.uc:1141-1174`（route 规则字段映射）

```ucode
push(config.route.rules, {
    ...
    action: cfg.action,
    outbound: get_outbound(cfg.outbound),
    override_address: cfg.override_address,
    ...
    tls_record_fragment: strToBool(cfg.tls_record_fragment)
});
```

**根因**：后端只映射 `outbound`，从未读取任何 `resolve_*` 键（对整个 `root/` 目录 grep 无命中）。因此选择「Resolve」动作的规则最终只生成 `{ "action": "resolve" }`。

**影响面**：路由规则中为 `resolve` 动作指定的 DNS 服务器、解析策略、缓存/rewrite TTL、EDNS Client Subnet 全部被忽略，sing-box 回退到默认解析器与默认策略。用户会得到「设置了但不生效」的错误体验，且无任何提示。

**附带一致性说明**：DNS 规则页的 action 列表没有 `resolve`，其 `server` / `domain_strategy` / `dns_disable_cache` / `rewrite_ttl` / `client_subnet` 与生成器（`generate_client.uc:743-751`）是对齐的，因此问题仅限路由规则页。

**修复建议（未实施）**

```ucode
const rule = {
    ip_version: strToInt(cfg.ip_version),
    /* ... 既有字段 ... */
    action: cfg.action,
    outbound: get_outbound(cfg.outbound)
};

if (cfg.action === 'resolve') {
    rule.server        = get_resolver(cfg.resolve_server);
    rule.strategy      = cfg.resolve_strategy;
    rule.disable_cache = strToBool(cfg.resolve_disable_cache);
    rule.rewrite_ttl   = strToInt(cfg.resolve_rewrite_ttl);
    rule.client_subnet = cfg.resolve_client_subnet;
}
```

**回归验证**：新建一条 `domain_suffix = example.com` + `action = resolve` + `server = <自建 DNS>` 的规则，检查 `sing-box-c.json` 中该规则包含 `server` 与 `strategy` 键。

---

### HP-05（高）端口范围校验使用字符串比较

**位置**：`htdocs/luci-static/resources/homeproxy.js:372-389`

```js
validatePortRange(section_id, value) {
    if (section_id && value) {
        value = value.match(/^(\d+)?\:(\d+)?$/);
        if (value && (value[1] || value[2])) {
            if (!value[1])      value[1] = 0;
            else if (!value[2]) value[2] = 65535;

            if (value[1] < value[2] && value[2] <= 65535)
                return true;
        }
        return _('Expecting: %s').format(_('valid port range (port1:port2)'));
    }
    return true;
}
```

**根因**：`String.prototype.match` 的捕获组是字符串，`'5' < '100'`、`'999' < '1000'` 均按字典序比较，结果为 `false`。

**影响面**（所有使用该校验器的地方）

| 使用点 | 文件 |
| --- | --- |
| `port_range` / `source_port_range`（路由规则） | `client.js:1070,1080` |
| `port_range` / `source_port_range`（DNS 规则） | `client.js:1504,1514` |
| `hysteria_hopping_port`（节点端口跳跃） | `node.js:532` |

**触发示例**：`5:100`、`9:80`、`999:1000`、`3000:10000` 全部被判为非法；`:100`、`100:` 反而正常（因为一侧被赋值为数字）。

**修复建议（未实施）**

```js
const start = value[1] ? parseInt(value[1], 10) : 0;
const end   = value[2] ? parseInt(value[2], 10) : 65535;

if (start < end && end <= 65535)
    return true;
```

建议同时补上 `start <= 65535` 的边界判断，并保留「单端口不接受范围语法」的既有语义。

---

### HP-06（中）simple 模式跳过 ujail，sing-box 以 root 运行

**位置**：`root/etc/init.d/homeproxy:157`

```sh
if [ -x "/sbin/ujail" ] && [ "$routing_mode" != "custom" ] && [ "$routing_mode" != "simple" ] && ! grep -Eq '"type": "(wireguard|tun)"' "$RUN_DIR/sing-box-c.json"; then
```

**分析**

- `custom` 被排除有其原因：`/etc/homeproxy/ruleset` 本地规则集目录未挂载进 jail。
- `simple` 模式并不需要该目录（全部为 remote rule set），其缓存写入 `$RUN_DIR/cache.db`；而 `bypass_mainland_china` 分支在 jail 下已通过 `procd_add_jail_mount_rw "$RUN_DIR/cache.db"` 解决（`init.d:161`）。
- 因此 simple 的排除看起来是「保守起见」而非必需，代价是：进程失去 namespace/capability 收敛与 `no_new_privs`，与同一份代码在其他模式下运行 root→sing-box 用户的最小权限原则不一致。

**建议（未实施）**：复核 simple 模式在 jail 下的实际读写路径（`cache.db`、`/etc/homeproxy/certs`、`/etc/ssl`、`/etc/localtime`），确认无额外访问后去掉 `!= "simple"`，并补充 `procd_add_jail_mount_rw "$RUN_DIR/cache.db"` 分支。

**若确认需要保留 root**：建议在代码注释与 README 中写明原因，避免后续维护者误删。

---

### HP-07（中）拼写错误 `bypass_mainalnd_china`

**位置**：`root/etc/homeproxy/scripts/update_subscriptions.uc:40`

```ucode
const routing_mode = uci.get(uciconfig, ucimain, 'routing_mode') || 'bypass_mainalnd_china';
```

**影响**：当前只参与 `routing_mode !== 'custom'` 判断，拼错不改变行为，属于「潜在陷阱」——一旦后续以该变量做精确分支（例如加入 `simple` 判断），就会静默走错分支。建议修正为 `bypass_mainland_china`。

---

### HP-08（中）订阅更新把「主节点 = 禁用(nil)」改写为第一个节点

**位置**：`root/etc/homeproxy/scripts/update_subscriptions.uc:598-620`

```ucode
let need_restart = (via_proxy !== '1');
if (!isEmpty(main_node)) {
    const first_server = uci.get_first(uciconfig, ucinode);
    if (first_server) {
        ...
        if ((main_node === 'urltest') ? !length(main_urltest_nodes) : !uci.get(uciconfig, main_node)) {
            uci.set(uciconfig, ucimain, 'main_node', first_server);
            uci.commit(uciconfig);
            need_restart = true;
            log('Main node is gone, switching to the first node.');
        }
```

**根因**：`main_node` 上一步取值为 `uci.get(...) || 'nil'`，`'nil'` 非空字符串 → 进入分支；`uci.get(uciconfig, 'nil')` 因不存在该 section 而返回 `null` → 判定「主节点已消失」→ 写入第一个节点。

**影响面**

1. 旧模式用户勾选「Main node: Disable」后执行订阅更新 → 代理被静默重新启用（用户可能完全不知情），且 `need_restart = true` 会触发 stop/start。
2. simple 模式下同样会写入一个无意义的 `main_node`，污染配置；日后切回旧模式出现「意外已启用」。

**修复建议（未实施）**：在 `if (!isEmpty(main_node) && main_node !== 'nil')` 中排除 `'nil'`（同时 `main_udp_node` 的 `'nil'` 语义已由 `!isEmpty(...) && !== 'same'` 部分覆盖，但仍建议统一判定）。

---

### HP-09（中）`ruleset_update_interval` 缺少格式校验

**位置**：`client.js:426-428`

```js
so = ss.option(form.Value, 'ruleset_update_interval', _('Rule set update interval'),
    _('How often remote rule sets are refreshed, e.g. 1d.'));
so.placeholder = '1d';
```

**根因**：既无 `datatype` 也无 `validate`，任意字符串都会写入 UCI 并进入 `route.rule_set[].update_interval`。

**影响**：填成 `1day`、`abc`、`每天` 等值时，`sing-box check` 在启动阶段失败 → `start_service` 记录 "wrong client configuration detected" 并 `return 1`，用户表现为「服务无法启动」，但 UI 无任何提示，排查成本高。

**修复建议（未实施）**

```js
so.datatype = 'uinteger';   // 或允许单位：so.validate 用 /^\d+[smhd]$/
```

若希望支持 `1d`/`12h`，建议用显式正则（`/^\d+(ms|s|m|h|d)$/`）并给出示例文案，不要复用 `uinteger`。同页 `health_check_interval` 已有 10–1800 的范围校验，可作为风格参照。

---

### HP-10（中）`stop_service` 清理的 nft 链名与模板不一致

**位置**：`root/etc/init.d/homeproxy:285-291`

```sh
for i in "homeproxy_dstnat_redir" "homeproxy_output_redir" \
     "homeproxy_redirect" "homeproxy_redirect_proxy" \
     "homeproxy_redirect_proxy_port" "homeproxy_redirect_lanac" \
     "homeproxy_mangle_prerouting" "homeproxy_mangle_output" \
     "homeproxy_mangle_tproxy" "homeproxy_mangle_tproxy_port" \
     "homeproxy_mangle_tproxy_lanac" "homeproxy_mangle_mark" \
     "homeproxy_mangle_tun" "homeproxy_mangle_tun_mark"; do
```

**实际模板中的链名**（`firewall_post.ut`）

- `dstnat`（fw4 自有链，不应删除）
- `homeproxy_mangle_lanac`（tproxy 分支与 tun 分支都定义）

**结论**

- `homeproxy_dstnat_redir` 不存在（历史遗留名）；
- `homeproxy_mangle_tproxy_lanac` 与实际 `homeproxy_mangle_lanac` 不符，导致该链不会被显式删除。

**为什么目前不影响功能**：`stop_service` 随后会 truncate `fw4_post.nft` 并 `fw4 reload`，整表重建时残留链会被清除。属于「靠兜底掩盖的失效清理」，一旦以后改成局部 reload 就会暴露。建议同步为实际链名，并在注释中标注 `dstnat` 为 fw4 所有、禁止删除。

---

### HP-11（中）simple 模式选「Direct」会静默关闭远端 DNS 与国内 DNS 分流

**位置**

- `generate_client.uc:552-568`（`aux_detour` 决策）
- `generate_client.uc:762-801`（DNS 生成）
- UI 文案：`client.js:419-424`

**机制**

```ucode
const aux_node = simple_cfg.ruleset_download_detour;
if (aux_node === 'direct-out')
    simple.aux_detour = null;
...
if (simple.aux_detour) {
    // 生成 simple-dns（远端 DNS），config.dns.final = 'simple-dns'
}
if (simple.aux_detour && !isEmpty(simple.china_dns_server) && ...)
    simple.cn_ruleset_tag = simple_add_ruleset(simple, base, 'geosite/geolocation-cn');
```

即：`ruleset_download_detour = direct-out` 时 → `aux_detour = null` → 远端 DNS 服务器不生成、`dns.final` 退回 `default-dns`（WAN DNS）、国内 DNS 分流所需的 CN 规则集也不下载。

**问题**：一个下拉框同时控制两件事（规则集下载路径、远端 DNS 可达性）。用户完全可能只想「规则集直连下载（避免代理流量/下载失败）」而仍希望「远端域名走加密 DNS + 国内域名走国内 DNS」。当前行为无提示、无日志，只有注释与一句文案说明，容易误判为 bug。

**建议（未实施，二者择一）**

1. 拆成两个选项：`ruleset_download_detour` 与 `remote_dns_detour`；或
2. 保留单选项，但在 `download_detour = direct-out` 时仍生成 `simple-dns`（detour 用 `direct-out`）并保留 CN 分流，仅在「完全没有节点」时才退回 `default-dns`。

无论哪种，建议 generator 在降级路径写一条日志到 `$RUN_DIR/homeproxy.log`，便于用户自查。

---

## 5. 低优先级问题

### N-01 TLS 类协议的缺省端口为 80

**位置**：`node.js:41`（anytls）、`57`（https）、`77`（hysteria）、`104`（hysteria2）、`127`（socks）、`211`（trojan）、`246`（tuic）等

```js
port: url.port || '80',
```

分享链接省略端口时统一回退 `80`。对 `anytls` / `hysteria2` / `tuic` / `trojan` 这类 TLS/QUIC 协议，`443` 更符合实际且不会产生「导入后连不上」的困惑。建议按协议区分缺省值（TLS 类 `443`，明文类 `80`）。

### N-02 fw4 include 片段仅在非空时写入

**位置**：`firewall_pre.uc:49-53`

```ucode
if (!isEmpty(forward))
    writefile(RUN_DIR + '/fw4_forward.nft', join('\n', forward) + '\n');
if (!isEmpty(input))
    writefile(RUN_DIR + '/fw4_input.nft', join('\n', input) + '\n');
```

配置从「有放行规则」变为「无」时，旧文件会保留，直到 `stop_service` 的 truncate 或重启。建议无条件写入（空文件亦可），使 fw4 include 内容始终与当前配置一致。

### N-03 simple 分流行悬空引用被静默丢弃

**位置**：`generate_client.uc:426-437`

```ucode
const node = uci.get_all(uciconfig, name) || {};
/* Silently drop dangling references, e.g. a node deleted on the node page. */
if (isEmpty(node.type))
    return null;
```

节点被删除（手动删除或订阅更新移除）后：
- 主节点失效 → 整行规则被丢弃 → 该行流量落到 `default_outbound`；
- 仅 fallback 失效 → 静默少一个后备。

无任何日志与 UI 提示，用户会以为分流仍在生效。建议在丢弃时写日志（`RUN_DIR/homeproxy.log`），或在 `update_subscriptions.uc` 清理引用后同步刷新相关行。

### N-04 `hysteria_protocol` 为死字段

`server.js:250` 提供服务端「Protocol」下拉（值仅 `udp`），但 `generate_server.uc` 从不读取（Hysteria 入站无该配置项）；`node.js:543` / `update_subscriptions.uc:157` 写入的 `hysteria_protocol` 同样不被 `generate_client.uc` 使用（分享链接中非 UDP 的节点已在解析阶段被拒绝）。建议移除该选项或加注释说明其仅用于导入兼容。

### N-05 `isValidDNSServer` 对 `simple.dns_server` 放行 `nil`

**位置**：`client.js:90-105`（`['wan','nil'].includes(value)` 直接返回 `true`）+ `client.js:362-367`

`nil` 对 `china_dns_server` 有意义（表示禁用国内 DNS），但对 `simple.dns_server` 无意义：`parse_dnsserver('nil')` 会生成 `udp://nil`，最终得到一个没有有效 `server` 的 DNS 服务器条目或直接解析失败。建议把 `nil` 的放行限制在 `china_dns_server` 的校验器内。

### N-06 CI 缺少静态校验

**位置**：`.github/workflows/build-ipk.yml`

当前仅在 `push`/`PR` 时打包 apk/ipk。建议增加轻量校验步骤，可在合并前拦住语法级问题：

```yaml
- name: Static checks
  run: |
    for f in htdocs/luci-static/resources/homeproxy.js htdocs/luci-static/resources/view/homeproxy/*.js; do
      node --check "$f"
    done
    python3 - <<'PY'
    import glob, json, sys
    for p in glob.glob('**/*.json', recursive=True):
        json.load(open(p))
    PY
    # 如有 ucode 环境，可追加: ucode -c 或对 .ut 做 utpl -S 渲染
```

---

## 6. 建议修复顺序与回归验证

### 6.1 顺序建议

1. **HP-03**（启动阻断）→ 1 行改动，风险最低，收益最大。
2. **HP-02**（字段名）+ 迁移 → 影响单协议单选项，改动可控。
3. **HP-05**（端口范围校验）→ 纯前端，注意补边界判断。
4. **HP-04**（`resolve_*` 映射）→ 需要确认 sing-box `resolve` 动作的字段集合后再落地。
5. **HP-01**（域名列表被清空）→ 涉及数据丢失，改动小但必须回归 6 种模式。
6. **HP-08 / HP-07 / HP-09** → 配置健壮性。
7. **HP-06 / HP-10 / HP-11** → 需要设备验证或产品决策，可独立排期。

### 6.2 回归验证清单

| 场景 | 期望 |
| --- | --- |
| 6 种路由模式各保存一次 Client Settings | `direct_list.txt` / `proxy_list.txt` 仅在不含 `depends` 的模式下按用户输入变化；custom/simple 下内容保持不变 |
| 服务端 Hysteria + 填写窗口 | `sing-box-s.json` 含 `recv_window_client` |
| 服务端 ACME + 勾选「Disable TLS ALPN challenge」 | `sing-box-s.json` 中为 `true`，`sing-box check` 通过 |
| 路由规则 `action=resolve` + 自建 DNS | 生成的规则含 `server` / `strategy` |
| 端口范围 `5:100`、`999:1000`、`:1024`、`1024:`、`100:99` | 前四个通过，最后一个报错 |
| 主节点设为 Disable 后执行订阅更新 | `main_node` 仍为 `nil`，代理未被启用 |
| simple 模式非法 `ruleset_update_interval` | 保存时被拦截（修复后） |
| simple 模式删除被规则引用的节点 | 有日志告警（修复后），且不产生无效 sing-box 配置 |
| `stop` 后 `nft list chains inet fw4` | 无 `homeproxy_*` 残留链 |

### 6.3 建议的验证手段

- **ucode 语法**：`ucode -c <file>`（或在容器中 `apk add ucode`）。
- **ut 模板渲染**：`utpl -S root/etc/homeproxy/scripts/firewall_post.ut > /tmp/fw4.nft`，再对 6 种模式 × 3 种 proxy_mode 组合各渲染一次做 diff review。
- **sing-box 配置**：`sing-box check --config <json>`；建议补齐边界用例：空分流表、仅自定义 URL、单行多规则集、同一节点多行、fallback 为空/重复、节点被删除后的悬空引用。
- **ACL**：以非 root 的 LuCI 账号（仅有 `luci-app-homeproxy` ACL）验证 `resources_update` / `acllist_write` / `certificate_write` 可用。

---

## 7. 附录

### A. 已确认无问题的检查项

- 全部 JSON 配置文件语法合法。
- 全部 shell 脚本语法合法（`.github/*.sh` 使用 bash 特性，shebang 正确）。
- 全部前端 JS 通过 `node --check`。
- 简洁模式新增文案全部进入 `po/templates/homeproxy.pot` 与 `po/zh_Hans/homeproxy.po`（抽查 11 条关键 msgid 均命中）。
- RPC 方法与 ACL 匹配：`luci.homeproxy` 授权 `*`；`update_subscriptions.uc` 有 `exec` 权限；日志有 read 权限；证书临时文件有 write 权限。
- 菜单四项（client/node/server/status）与视图文件一一对应。
- `simple` 模式在以下位置均已登记，未发现遗漏：`init.d`（出站判定、dnsmasq 全量接管、proxy_list 跳过、tproxy 规则、jail 判定、cache.db 准备）、`firewall_pre.uc`（tun 出站判定）、`firewall_post.ut`（`legacy_mode` 判定、tproxy 端口条件）、`migrate_config.uc`（默认值补全）、`update_subscriptions.uc`（不读取 `main_node` 主分支）。
- 简洁模式的「按序故障转移」实现符合 sing-box 的 `urltest` 语义（有序 `outbounds` + 极大 `tolerance` + `interrupt_exist_connections`），规则集按 URL 去重、出站组按签名复用，未发现重复下载或循环依赖。

### B. 本轮审查中确认「正常」但值得留意的设计取舍

| 取舍 | 说明 |
| --- | --- |
| `simple` 的规则集与 DNS 共用 `ruleset_download_detour` | 见 HP-11，行为已文档化但易误用 |
| 故障转移不保证快速回切 | `tolerance=30000` 使恢复节点不会立即抢回；UI 文案已说明「failback 不是瞬时」 |
| `simple` 模式全部交给 sing-box 分流 | nft 侧只做「全量接管 + 本地/保留地址例外」，与 custom 一致 |
| `simple` 模式关闭 ujail | 见 HP-06，需要确认是否有意为之 |
| `cache_file` 仅部分模式开启 | `gfwlist`/`proxy_mainland_china`/`global` 不开启，与这些模式不依赖 rule set 缓存有关 |

### C. 术语与文件对应

| 术语 | 对应实现 |
| --- | --- |
| 简洁分流 | `routing_mode = simple` + `simple` 命名段 + `simple_rule` 动态段 |
| 辅助节点（aux） | `ruleset_download_detour` → `simple.aux_detour` |
| 健康检查 / 故障转移 | `urltest` 出站组（`health_check_url` / `health_check_interval`） |
| 主节点模式（legacy） | `gfwlist` / `bypass_mainland_china` / `proxy_mainland_china` / `global` |

---

## 8. 修复状态与验证记录（2026-11-06 复查）

> 本节记录初版报告之后的修复情况。改动仍处于工作区未提交状态，第 4 节的详细分析保留原样，便于对照问题成因。

### 8.1 逐项落点

| ID | 状态 | 修复后位置 | 修复方式 |
| --- | --- | --- | --- |
| HP-01 | 已修复 | `client.js:1725-1731`、`1758-1764` | `remove()` 改用 `this.isActive(section_id)`：字段被 `depends()` 隐藏时直接返回，不再清空文件 |
| HP-02 | 已修复 | `generate_server.uc:70` | 键名改为 `hysteria_recv_window_client`，与服务端 UI 写入的键一致 |
| HP-03 | 已修复 | `generate_server.uc:132` | 补 `strToBool()`，输出布尔而非 `"1"` 字符串 |
| HP-04 | 已修复 | `generate_client.uc:1189-1195` | `action === 'resolve'` 时映射 `server` / `strategy` / `disable_cache` / `rewrite_ttl` / `client_subnet` |
| HP-05 | 已修复 | `homeproxy.js:372-389` | 捕获组 `parseInt` 后再比较，并补 `start >= 0` 边界 |
| HP-06 | 保留 | `init.d:157-160` | 判定未改，补 3 行注释说明 custom/simple 无沙箱的原因与「待真机验证」状态 |
| HP-07 | 已修复 | `update_subscriptions.uc:40` | 修正拼写；同时把 `simple` 纳入「不读取 main_node」的判断 |
| HP-08 | 已修复 | `update_subscriptions.uc:605`、`627` | 读取与改写分支均排除 `'nil'`（见 8.2） |
| HP-09 | 已修复 | `client.js:427-435` | 增加 `/^\d+(ms|s|m|h|d)$/` 校验与提示文案（pot/po 已同步） |
| HP-10 | 已修复 | `init.d:287-299` | 移除不存在的 `homeproxy_dstnat_redir`，`homeproxy_mangle_tproxy_lanac` → `homeproxy_mangle_lanac`，与模板 14 个链名逐一核对 |
| HP-11 | 已修复 | `client.js:421` | 保留单下拉框方案，文案明确「Direct 会同时停用远端 DNS 与国内 DNS 分流」 |
| N-01 | 已修复 | `node.js:18-20` | 新增 `defaultSharePort()`：TLS/QUIC 类协议回退 443，其余 80；复查后补齐 `hy2`（见 8.2） |
| N-02 | 已修复 | `firewall_pre.uc:49-52` | 无条件重写两个 include（空内容写单个换行），并移除不再使用的 `isEmpty` 导入 |
| N-03 | 已修复 | `generate_client.uc:426-433` | 新增 `simple_log()`，悬空引用先写 `homeproxy.log` 再丢弃 |
| N-04 | 已修复 | `node.js:548`、`server.js:250` | 保留选项，补注释说明仅为分享链接导入兼容，sing-box 无对应字段 |
| N-05 | 已修复 | `client.js:90-105`、`380` | 增加 `allowNil` 参数，仅 `china_dns_server` 放行 `nil` |
| N-06 | 已修复 | `.github/workflows/build-ipk.yml:43-49` | 新增 Static checks：`node --check` × 5、`sh -n`、全仓 JSON 解析 |

### 8.2 复查时发现并补齐的两处

首轮修复后复查发现两项未真正闭环，已补齐。

**（1）HP-08 只修了一半。** 首轮仅让 custom/simple 不读取 `main_node`；legacy 模式下 `main_node = 'nil'` 仍会在 `!uci.get('nil')` 处被判定为「节点消失」，进而写入第一个节点：

- 旧模式用户勾选「Main node: Disable」且库中有节点 → 订阅更新后代理被静默重新启用（并触发 stop/start）；
- `main_node` 有效但「Main UDP node: Disable」→ `main_udp_node` 被改写，UDP 开始走代理。

补齐方式为两处各加一个判断：

```ucode
/* root/etc/homeproxy/scripts/update_subscriptions.uc:605 */
if (!isEmpty(main_node) && main_node !== 'nil') {

/* :627 */
if (!isEmpty(main_udp_node) && main_udp_node !== 'same' && main_udp_node !== 'nil') {
```

`644-651` 行「完全没有节点时写回 `nil`」的逻辑对 legacy 仍有意义（关闭代理），保持不动。

**（2）N-01 漏了 `hy2`。** `node.js` 的解析分支同时接受 `hysteria2://` 与 `hy2://`（`case 'hysteria2': case 'hy2':`），但 `defaultSharePort()` 白名单只含 `hysteria2`，导致 `hy2://` 链接仍回退 80。已把 `'hy2'` 加入 443 列表。

### 8.3 保留项与非阻塞瑕疵

| 项 | 说明 |
| --- | --- |
| HP-06 未启用 ujail | 未改判定，仅以注释记录「尚未在真机 ujail 下验证」，属于显式风险接受；恢复前需设备验证 |
| `recv` / `revc` 命名不对称 | 服务端已修正为 `hysteria_recv_window_client`；客户端 `hysteria_revc_window` 在 UI 与生成器两侧自洽、功能正常。统一命名需配 `uci.rename` 迁移，否则会静默丢失既有配置，故本次不动 |
| `init.d:157-159` 注释缩进 | 多行注释与 `if` 少一层 tab（shell 语义无影响，`sh -n` 通过），仅影响可读性 |
| HP-11 未拆分选项 | 维持「一个下拉框同时控制下载与远端 DNS」的设计，靠文案说明；若后续拆分，需同步调整生成器与迁移默认值 |

### 8.4 验证记录

| 检查 | 命令 | 结果 |
| --- | --- | --- |
| 前端 JS 语法 | `node --check` × 5 个文件 | 通过 |
| 启动脚本语法 | `sh -n root/etc/init.d/homeproxy` | 通过 |
| JSON 合法性 | `python3 -c 'import glob, json; ...'` | 通过 |
| workflow YAML | `ruby -ryaml -e 'YAML.load_file(...)'` | 通过 |
| 翻译文件 | `msgfmt -c -o /dev/null`（`.po` / `.pot`） | 通过（仅既有 header 警告） |
| i18n 一致性 | 比对新增/修改 msgid | `a duration such as 1d or 12h` 与改写后的 Download/DNS node 说明均 pot=1、po=1；旧 msgid 已清除 |
| nft 链名 | `init.d` 清理列表 ↔ `firewall_post.ut` 链定义 | 14/14 匹配；旧链名无残留 |
| 未使用导入 | `grep isEmpty firewall_pre.uc` | 无残留 |
| CI 步骤 | 本地复现 `Static checks` 三条命令 | 全部通过 |

尚未执行（同 1.2，需要设备/运行环境）：

- `ucode -c` / `utpl -S` 渲染（本机无 ucode）；
- `sing-box check --config`：建议在真机重点回归 HP-03（ACME + TLS-ALPN）、HP-04（`resolve` 规则）与 HP-08（legacy + Disable 主节点后更新订阅）。
- HP-04 的 `disable_cache` / `rewrite_ttl` / `client_subnet` 自 sing-box 1.12 起受支持；项目既有配置已使用 1.12+ 的 DNS server 新格式，版本要求一致，仍建议以目标固件的 sing-box 实测一次。

### 8.5 后续建议

1. 真机执行 `sing-box check`，覆盖 8.4 中三个重点场景后再合并。
2. HP-06：在设备上确认 simple 模式的 `cache.db` 与证书路径可访问后恢复 ujail，并补 `procd_add_jail_mount_rw "$RUN_DIR/cache.db"`。
3. 可选：统一 `hysteria_revc_window` → `hysteria_recv_window`，先在 `migrate_config.uc` 增加 `uci.rename` 再改，避免静默丢配置。
4. 可选：为发布流程补一次 `sing-box check` 冒烟（N-06 目前只覆盖语法与 JSON）。

