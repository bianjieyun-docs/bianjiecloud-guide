> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-10-16。

# sing-box 1.14+ hijack-dns 规则验证与最小修复教程

**一句话结论：**在 sing-box 1.14+ 中，判断 DNS 是否被 TUN 接管，不能只看配置文件里有没有 `hijack-dns`，应依次完成配置版本检查、`sing-box check` 校验、调试日志验证和外部 DNS 泄漏测试；最小修复通常是在 `route.rules` 中把 DNS 规则放在普通分流规则之前，并使用 `{"protocol":"dns","action":"hijack-dns"}`。

## 适用场景

本文适用于以下情况：

- 使用 sing-box 原始配置或基于 sing-box 内核的图形客户端；
- 使用 TUN 入站，希望系统或应用发出的 DNS 请求进入 sing-box 的 DNS 模块；
- 已经配置了 DNS 服务器，但怀疑请求仍然由系统、路由器或运营商 DNS 直接解析；
- 升级到 sing-box 1.14.0 或更高版本后，旧 DNS 配置无法启动，或者原有规则表现异常；
- 使用边界云加速提供的订阅或自定义配置时，需要定位是规则未命中、TUN 未接管，还是浏览器自身启用了独立 DNS。

本文只讨论一个问题：**如何验证和修复 `route.rules` 中的 `hijack-dns` 规则**。不展开完整的 TUN、分流、FakeIP 或多平台客户端配置。

## 第 1 步：先确认 sing-box 版本和配置来源

先查看客户端显示的内核版本，或在命令行执行：

```bash
sing-box version
```

如果配置使用了 TUN 的 `dns_mode` 或 `dns_address`，需要特别注意版本范围：这两个字段均自 sing-box 1.14.0 起提供。`dns_mode` 用于控制 TUN 接口上的 DNS 处理方式，`dns_address` 用于指定相关 DNS 服务器地址；如果设置了 `dns_address`，官方文档要求额外配置 `hijack-dns` 路由规则处理发往这些地址的 DNS 流量。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/inbound/tun/))

同时，不要把 1.12.0 以前的旧 DNS 服务器写法直接复制到新配置中。旧的 `address`、`address_resolver` 等 DNS server 结构已在 sing-box 1.12.0 废弃，并在 1.14.0 移除；新格式应使用 `type`、`server` 等字段。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/migration/))

例如，下面这种旧格式不应继续用于 1.14.0+：

```json
{
  "dns": {
    "servers": [
      {
        "tag": "remote",
        "address": "tls://dns.example.com",
        "address_resolver": "local"
      }
    ]
  }
}
```

不要为了修复 `hijack-dns` 而继续添加 `address_resolver`。新版 DNS server 应根据实际服务器类型改写，例如 HTTPS DNS 可以采用类似下面的结构；其中域名、服务器地址和解析器标签必须替换成你实际使用的值：

```json
{
  "dns": {
    "servers": [
      {
        "tag": "remote",
        "type": "https",
        "server": "dns.example.com",
        "path": "/dns-query",
        "domain_resolver": "local"
      },
      {
        "tag": "local",
        "type": "local"
      }
    ],
    "final": "remote"
  }
}
```

上面示例只用于说明新版字段关系，`dns.example.com` 不是可直接使用的公共服务地址。真实配置应使用你所选择的 DNS 服务及其官方参数。

## 第 2 步：检查 `hijack-dns` 是否位于正确位置

`hijack-dns` 是路由规则动作，作用是把匹配到的 DNS 请求交给 sing-box DNS 模块。官方示例通常将协议探测规则放在前面，再使用以下规则：

```json
{
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      }
    ]
  }
}
```

`hijack-dns` 的官方定义就是将 DNS 请求劫持到 sing-box DNS 模块；它不是 DNS 服务器地址，也不是 `dns.rules` 中的 DNS 选择动作。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/route/rule_action/))

检查时重点看三项：

1. `action` 是否拼写为 `hijack-dns`，中间使用半角连字符；
2. 规则是否位于 `route.rules`，而不是误放到 `dns.rules`；
3. 在它之前是否有一个会提前结束匹配的规则，例如针对全部流量的 `route`、`reject` 或其他最终动作。

如果希望同时覆盖通过 UDP 53 发出的 DNS 流量，也可以使用官方客户端文档中的逻辑规则写法：

```json
{
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "type": "logical",
        "mode": "or",
        "rules": [
          {
            "protocol": "dns"
          },
          {
            "port": 53
          }
        ],
        "action": "hijack-dns"
      }
    ]
  }
}
```

不要盲目同时保留多条互相重叠的 DNS 规则。先使用最小的 `protocol: dns` 规则验证是否能命中；只有在日志或抓包确认某些流量没有被识别为 DNS 时，再考虑加入 `port: 53` 条件。官方示例也展示了把 `protocol: dns` 与 `port: 53` 放入逻辑规则的方式。([github.com](https://github.com/SagerNet/sing-box/blob/testing/docs/manual/proxy/client.md))

## 第 3 步：用 `sing-box check` 验证配置能否加载

在修改配置后，先不要直接启动代理，执行配置检查：

```bash
sing-box check -c config.json
```

如果你的发行版或客户端对配置文件路径有额外要求，也可以按照客户端提供的配置目录方式执行检查。官方文档将 `sing-box check` 作为配置检查命令，并将 `sing-box format` 作为格式化命令。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/))

检查通过只代表 JSON 结构和当前内核支持的字段基本可加载，不代表 DNS 一定已经被接管。尤其要注意以下误区：

- 配置能启动，但 TUN 没有启用，系统 DNS 自然不会经过 TUN；
- 配置启用了 `dns_mode: native`，但没有达到你预期的流量接管范围；
- 配置启用了 `dns_address`，却没有为对应地址配置 `hijack-dns`；
- `route.rules` 中存在更早命中的规则，DNS 请求在到达劫持规则前已经被其他动作处理。

## 第 4 步：开启 debug 日志，确认规则是否实际命中

将日志等级临时改为 `debug`：

```json
{
  "log": {
    "level": "debug",
    "timestamp": true
  }
}
```

sing-box 支持 `trace`、`debug`、`info`、`warn`、`error`、`fatal` 和 `panic` 等日志等级；排查完成后建议恢复为 `info`，避免长期产生过多日志。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/log/))

重启 sing-box 后，执行一次新的 DNS 查询，例如：

```bash
nslookup example.com
```

或：

```bash
dig example.com
```

然后查看日志中是否出现与 DNS、路由匹配或 `hijack-dns` 相关的记录。判断标准不是某一行固定文本，而是确认该查询被路由规则识别，并进入 sing-box DNS 处理流程。不同图形客户端可能会对日志格式进行包装，因此不要机械地寻找完全相同的日志行。

如果完全没有相关命中记录，可以按以下顺序排查：

1. 当前应用是否真的使用了这份配置；
2. TUN 入站是否已启动并获得系统权限；
3. DNS 查询是否由浏览器自己的 DNS over HTTPS 产生；
4. 查询是否走了 IPv6、虚拟网卡或其他 VPN 接口；
5. `route.rules` 是否被订阅更新或客户端模板覆盖。

## 第 5 步：通过外部检测确认是否存在 DNS 泄漏

日志能说明规则是否命中，但不能单独证明所有应用都没有绕过代理。可以在启用配置后访问 [BrowserLeaks DNS Leak Test](https://browserleaks.com/dns)，运行测试并记录显示的 DNS 服务器；该页面用于检测浏览器实际使用的 DNS 服务器。([browserleaks.com](https://browserleaks.com/dns))

建议按照下面的顺序测试：

1. 关闭 sing-box，运行一次 DNS 测试并保存结果；
2. 启用 TUN 和当前配置，再运行一次；
3. 对比两次结果中的 DNS 服务商、网络归属和地址；
4. 同时查看 sing-box debug 日志，确认测试期间产生了 DNS 规则命中记录；
5. 在浏览器中暂时关闭独立的 Secure DNS、DNS over HTTPS 或类似功能，再重复测试。

如果外部页面显示的解析器与系统网络、路由器或运营商相关，而 sing-box 日志中没有对应查询，通常说明浏览器或系统绕过了 TUN；如果日志显示命中但外部检测结果仍不符合预期，则应继续检查 DNS server 的实际出站、浏览器缓存以及 IPv6 路径。

这里的“无泄漏”应理解为测试期间没有观察到预期之外的 DNS 解析器，不应把一次网页检测结果当作所有网络环境下的绝对保证。

## 第 6 步：应用最小修复配置

在已有配置中，优先只加入下面这条规则：

```json
{
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      }
    ]
  }
}
```

如果配置已经有 `route.rules`，不要再新增一个同名的 `route` 对象，而是将第二条规则插入现有数组，并放在可能提前结束匹配的普通分流规则之前。修改后重新执行：

```bash
sing-box check -c config.json
```

然后重启客户端，清理浏览器 DNS 缓存，再按“日志命中 + 外部检测”两种方法复核。

对于 sing-box 1.14.0+ 的 TUN 配置，如果显式设置了 `dns_address`，应确保它与 `hijack-dns` 规则配套。例如：

```json
{
  "inbounds": [
    {
      "type": "tun",
      "address": [
        "172.19.0.1/30"
      ],
      "auto_route": true,
      "dns_mode": "hijack",
      "dns_address": [
        "172.19.0.2"
      ]
    }
  ],
  "route": {
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      }
    ]
  }
}
```

这是字段关系示例，不是包含出站、DNS server 和平台权限的完整可运行配置。`dns_mode` 与 `dns_address` 的适用版本及行为以 sing-box 1.14.0+ 官方文档为准。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/inbound/tun/))

## 常见问题

### 1. 配置里有 `hijack-dns`，为什么 DNS 泄漏测试仍显示系统 DNS？

先确认测试时 TUN 已经启动，并确认 debug 日志中确实有 DNS 请求命中。之后检查浏览器是否开启了 Secure DNS 或 DNS over HTTPS。浏览器自己的加密 DNS 连接可能不表现为普通的 UDP 53 请求，因此不能只依赖 `port: 53` 规则判断。还应检查 IPv6 是否使用了另一条未被接管的网络路径。

### 2. `dns_mode`、`native` 和 `hijack` 是不是可以用于所有 sing-box 版本？

不是。`dns_mode` 和 `dns_address` 自 sing-box 1.14.0 起提供；官方文档中 `disabled`、`native`、`hijack` 是 `dns_mode` 的取值，不能把它们当成通用的 `route.rules` 动作。路由规则中负责把 DNS 请求送入 DNS 模块的是 `{"action":"hijack-dns"}`。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/configuration/inbound/tun/))

### 3. 旧配置中的 `address_resolver` 应该改成什么？

不要简单地把 `address_resolver` 改名。1.14.0 已移除旧 DNS server 格式，应该按照新 DNS server 类型重新组织配置；对于需要解析域名形式服务器地址的场景，应查看当前版本的 DNS server 与拨号字段文档，使用新版结构中的 `domain_resolver`，而不是继续沿用旧版 `address_resolver`。([sing-box.sagernet.org](https://sing-box.sagernet.org/zh/migration/))

### 4. `sing-box check` 通过了，但启动后仍然没有 DNS 命中日志，怎么办？

这通常说明问题不在 JSON 语法，而在运行路径。依次确认：客户端实际加载的文件、TUN 权限、系统代理模式、浏览器独立 DNS、IPv6 路由，以及订阅更新是否覆盖了手动修改。必要时可以先关闭复杂分流，只保留一个 TUN、一个 DNS server 和一条 `hijack-dns` 规则进行最小化测试。

### 5. 是否应该把所有 DNS 规则都改成 `port: 53`？

不建议。先使用官方示例中的 `protocol: dns` 规则验证协议识别；如果确认有 UDP 53 流量未被识别，再使用逻辑规则将 `protocol: dns` 与 `port: 53` 合并。规则越宽并不一定越可靠，过宽的规则可能把不应交给 DNS 模块的流量也纳入排查范围。([github.com](https://github.com/SagerNet/sing-box/blob/testing/docs/manual/proxy/client.md))

## 研究记录

- 检索日期：2026-10-16。
- 版本边界：重点核对 sing-box 1.14.0 的 TUN DNS 字段、1.14.0 移除旧 DNS server 格式，以及当前官方文档中的 `hijack-dns` 规则动作。
- 采用的最小验证链路：`sing-box check` → debug 日志 → `nslookup` 或 `dig` 触发查询 → 外部 DNS 泄漏测试。
- 未采用未标版本的 TUN DNS 模式说法，也未使用已在 1.14.0 移除的 `address_resolver` 作为新版配置示例。

## 参考资料

- https://sing-box.sagernet.org/zh/configuration/route/rule_action/
- https://sing-box.sagernet.org/zh/configuration/inbound/tun/
- https://sing-box.sagernet.org/zh/configuration/dns/server/legacy/
- https://sing-box.sagernet.org/zh/migration/
- https://sing-box.sagernet.org/zh/configuration/log/
- https://sing-box.sagernet.org/zh/configuration/
- https://sing-box.sagernet.org/zh/configuration/route/
- https://github.com/SagerNet/sing-box/blob/testing/docs/manual/proxy/client.md
- https://browserleaks.com/dns

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供跨平台网络加速与订阅使用指引，覆盖 Windows、macOS、Android 和 iOS 等常见客户端场景。你可以访问官网了解服务说明及客户端配置相关信息。