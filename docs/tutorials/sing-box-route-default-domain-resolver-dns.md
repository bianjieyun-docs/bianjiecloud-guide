> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-25。

# sing-box route.default_domain_resolver 怎么配置？DNS 解析器引用排查教程

## 一句话结论

`route.default_domain_resolver` 用来指定路由或出站进行域名解析时使用的默认解析器，通常填写 `dns.servers` 中对应 DNS 服务器的 `tag`，而不是直接填写 `1.1.1.1` 这类 DNS 地址；如果某个出站单独设置了 `domain_resolver`，则应优先检查出站自身的配置。官方 Route 文档还说明，该字段自 sing-box 1.12.0 起提供，并且可以被 `outbound.domain_resolver` 覆盖。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 适用场景

本文只讨论一个配置字段：`route.default_domain_resolver`。如果你在客户端中导入 sing-box 配置后遇到域名解析报错、出站服务器地址无法解析，或者不确定 `default_domain_resolver` 应该填写什么，可以按照本文的顺序检查。

常见适用情况包括：

- `dns.servers` 中已经配置了一个或多个 DNS 服务器，但不知道路由层应该引用哪个值。
- 出站服务器地址使用域名，运行时出现与域名解析有关的错误。
- 导入边界云加速提供的 sing-box 订阅后，需要确认 DNS 解析器引用是否指向了实际存在的服务器 tag。
- 从旧的 `domain_strategy` 配置迁移到 `domain_resolver` 配置。
- 配置检查时出现 JSON 结构、字段类型或 DNS 服务器引用方面的问题。

需要先区分两个概念：`route.default_domain_resolver` 不是 DNS 分流规则，也不是 `dns.final` 的别名。它属于路由层的默认域名解析器设置；DNS 查询本身仍然由 `dns.servers`、`dns.rules` 和 `dns.final` 等配置共同决定。官方 Dial Fields 文档说明，`domain_resolver` 可以使用字符串，也可以使用对象；字符串写法等价于对象中的 `server` 字段。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/shared/dial/))

## 1. 先理解 DNS server tag 和字段引用

一个最小的引用关系包含两部分：

1. 在 `dns.servers` 中声明 DNS 服务器，并设置 `tag`。
2. 在 `route.default_domain_resolver` 中填写这个 `tag`。

下面是一个使用本地系统解析器的最小示例：

```json
{
  "$schema": "https://sing-box.sagernet.org/schema.json%22,
  "dns": {
    "servers": [
      {
        "type": "local",
        "tag": "local"
      }
    ],
    "final": "local"
  },
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "default_domain_resolver": "local",
    "final": "direct"
  }
}
```

在这个示例中，`local` 同时出现在 DNS 服务器的 `tag` 和 `route.default_domain_resolver` 中，因此路由层能够引用这个 DNS 服务器。这里的 `local` 只是配置内部名称，不代表固定的公共 DNS 地址。

几个字段的含义不要混淆：

- `type` 表示 DNS 服务器类型，例如 `local`、`udp`、`tls` 或 `https`。
- `tag` 是 DNS 服务器在当前配置中的引用名称。
- `route.default_domain_resolver` 使用的是 DNS 服务器的 tag。
- DNS 服务器的具体地址或连接参数，应写在对应 DNS server 类型所支持的字段中。

例如，下面的写法把服务器地址误当成了引用名称：

```json
{
  "route": {
    "default_domain_resolver": "1.1.1.1"
  }
}
```

除非配置中确实存在 `tag` 为 `1.1.1.1` 的 DNS 服务器，否则这段配置并没有引用前文示例中的 `local` 服务器。正确做法是先在 `dns.servers` 中定义服务器，再填写它的 tag。

## 2. 使用远程 DNS 时如何填写

如果配置需要使用远程 DNS，应把远程 DNS 按官方文档支持的服务器类型写入 `dns.servers`，同时为它设置一个容易识别的 tag。下面仅展示字段关系：

```json
{
  "$schema": "https://sing-box.sagernet.org/schema.json%22,
  "dns": {
    "servers": [
      {
        "type": "tls",
        "tag": "remote-dns",
        "server": "1.1.1.1",
        "server_port": 853
      }
    ],
    "final": "remote-dns"
  },
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "default_domain_resolver": "remote-dns",
    "final": "direct"
  }
}
```

这里真正被 `route.default_domain_resolver` 引用的是 `remote-dns`，不是 `tls`，也不是 `1.1.1.1`。`tls` 是服务器类型，`1.1.1.1` 是服务器地址，二者都不能自动替代 tag。

这个示例只用于说明 JSON 结构。远程 DNS 是否能够连接，还会受到当前网络、服务器地址、端口、TLS 参数和防火墙策略影响。配置关系正确，不等于目标 DNS 在所有网络环境中都一定可访问。

## 3. 字符串写法和对象写法

如果只需要指定 DNS server tag，可以使用字符串写法：

```json
{
  "route": {
    "default_domain_resolver": "local"
  }
}
```

如果还需要指定解析策略，可以使用对象写法：

```json
{
  "route": {
    "default_domain_resolver": {
      "server": "local",
      "strategy": "prefer_ipv4"
    }
  }
}
```

官方 Dial Fields 文档说明，`domain_resolver` 对象使用与 DNS 规则 action 类似的格式，但不包含 `action` 字段；其中 `server` 用于指定 DNS server tag，`strategy` 可用于选择域名解析策略。官方迁移文档也使用了包含 `server` 和 `strategy` 的对象示例。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/shared/dial/))

如果只是想指定默认解析器，建议优先使用字符串写法。对象写法应只添加当前官方文档和目标版本确实支持、且配置确实需要的字段，不要为了“完整”而随意增加未确认的参数。

## 4. 出站级 domain_resolver 会覆盖默认值

`route.default_domain_resolver` 是默认设置，不是对所有出站的强制覆盖。官方文档明确说明，出站可以通过 `outbound.domain_resolver` 单独指定解析器，并覆盖路由层的默认值。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

示意配置如下：

```json
{
  "dns": {
    "servers": [
      {
        "type": "local",
        "tag": "local"
      },
      {
        "type": "tls",
        "tag": "remote-dns",
        "server": "1.1.1.1",
        "server_port": 853
      }
    ],
    "final": "local"
  },
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "socks",
      "tag": "proxy",
      "server": "example.org",
      "server_port": 443,
      "domain_resolver": "remote-dns"
    }
  ],
  "route": {
    "default_domain_resolver": "local",
    "final": "proxy"
  }
}
```

在这个结构中，路由默认解析器是 `local`，但 `proxy` 出站单独指定了 `remote-dns`。排查某个出站服务器地址无法解析时，不能只查看 `route.default_domain_resolver`，还要继续检查该出站是否存在 `domain_resolver`。

官方文档还指出，当配置中只有一个 DNS 服务器时，`domain_resolver` 或 `route.default_domain_resolver` 可以省略；当存在多个 DNS 服务器时，明确填写 tag 通常更容易确认实际使用的解析器。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/shared/dial/))

## 5. 按步骤排查配置

### 第一步：确认字段层级

`default_domain_resolver` 应位于顶层 `route` 对象内，而不是放在 `dns`、`dns.servers` 或某一条 route rule 中。

正确结构：

```json
{
  "dns": {
    "servers": [
      {
        "type": "local",
        "tag": "local"
      }
    ]
  },
  "route": {
    "default_domain_resolver": "local"
  }
}
```

常见错误结构：

```json
{
  "dns": {
    "default_domain_resolver": "local"
  }
}
```

### 第二步：逐字核对 tag

检查大小写、连字符、下划线和空格。例如 DNS server 使用：

```json
"tag": "remote-dns"
```

那么引用必须写成：

```json
"default_domain_resolver": "remote-dns"
```

`remote_dns`、`Remote-DNS` 和 `tls` 都不是同一个引用名称。不要把服务器类型或服务器地址当作 tag 使用。

### 第三步：用 sing-box check 检查配置

将配置保存为 `config.json` 后，可以运行：

```bash
sing-box check -c config.json
```

如果不确定当前本地构建版本支持哪些参数，先查看帮助：

```bash
sing-box check --help
```

sing-box 官方配置引言列出了 `sing-box check` 命令，并将 JSON 配置作为程序配置格式。实际检查时，应观察终端输出是否指出 JSON 语法、未知字段、字段类型或配置引用方面的问题；不要仅凭某一条命令的输出形式推断特定退出状态。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

### 第四步：确认实际加载的配置文件

如果检查通过但运行结果没有变化，先确认运行中的客户端是否加载了你刚刚修改的文件。重点检查：

- 启动参数中的配置路径是否与修改的文件一致。
- 图形客户端是否使用订阅生成的配置，而不是本地手工编辑文件。
- 修改后是否执行了重新加载或重启。
- 是否存在多个 profile、配置合并或客户端自动生成配置的情况。

### 第五步：区分配置问题和网络可达性问题

`sing-box check` 主要帮助发现配置格式和结构方面的问题，不能证明远程 DNS 服务器在当前网络中一定可达。如果 tag、字段层级和 JSON 结构都正确，但运行时仍然解析失败，应继续检查服务器地址、端口、TLS 设置、网络防火墙以及客户端实际日志。

## 常见问题

### 1. `default_domain_resolver` 能不能直接填写 DNS IP？

通常不应把 DNS IP 直接当作该字段的值。推荐先在 `dns.servers` 中声明 DNS server，并设置 `tag`，然后在 `route.default_domain_resolver` 中引用这个 tag。例如服务器 tag 是 `remote-dns`，路由层就填写 `"remote-dns"`。

### 2. `route.default_domain_resolver` 和 `dns.final` 是同一个字段吗？

不是。`dns.final` 用于指定 DNS 查询在没有更具体规则匹配时使用的默认 DNS server；`route.default_domain_resolver` 用于指定路由或出站进行域名解析时采用的默认解析器。二者可以填写相同 tag，也可以根据用途使用不同 tag。DNS 配置和路由配置在官方文档中属于不同的配置层级。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 3. 已经配置了 `dns.rules`，还需要配置 `route.default_domain_resolver` 吗？

不一定。是否需要配置，取决于是否存在需要通过域名解析的出站或端点、配置中有几个 DNS server，以及目标 sing-box 版本的具体要求。官方 Dial Fields 文档说明，只有一个 DNS server 时，`domain_resolver` 或 `route.default_domain_resolver` 可以省略；多个 DNS server 并存时，明确填写 tag 更有利于排查。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/shared/dial/))

### 4. 修改默认解析器后，DNS 分流规则没有变化，正常吗？

可能正常。`route.default_domain_resolver` 本身不是 `dns.rules` 的匹配条件，也不会自动改写 DNS rule 的顺序。如果需要按域名、入站或其他条件选择不同 DNS server，应继续检查 `dns.rules` 及其 action 配置，而不是只修改路由默认解析器。

### 5. 为什么配置检查通过，运行时仍然解析失败？

配置检查通过只能说明当前文件至少通过了检查命令能够发现的结构或格式验证，不能证明远程 DNS 服务器、出站服务器地址和当前网络之间一定连通。可以按以下顺序缩小范围：先确认客户端加载的是目标文件，再确认引用的 tag 确实存在，随后检查 DNS server 的地址、端口和协议参数，最后查看运行日志中的实际错误。

### 6. 对象写法中可以随便添加 `timeout` 吗？

不要把其他配置示例中的字段直接复制到 `default_domain_resolver` 对象中。本文只保留官方 Dial Fields 和迁移示例中直接对应的 `server`、`strategy` 字段；如果确实需要设置 DNS 查询超时，应以目标 sing-box 版本的 DNS、DNS rule action 或 domain resolver 官方文档为准，并先使用对应版本的 schema 或 `sing-box check` 验证。当前官方 DNS 文档对 DNS 查询超时的配置位置有单独说明，不能据此把任意超时字段放入所有对象中。 ([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/shared/dial/))

## 参考资料

1. [sing-box Route 官方文档](https://sing-box.sagernet.org/configuration/route/)
2. [sing-box Dial Fields 官方文档](https://sing-box.sagernet.org/configuration/shared/dial/)
3. [sing-box 配置引言、check 与 format 命令](https://sing-box.sagernet.org/configuration/)
4. [sing-box DNS 官方文档](https://sing-box.sagernet.org/configuration/dns/)
5. [sing-box Migration：Migrate outbound domain strategy option to domain resolver](https://sing-box.sagernet.org/migration/)
6. [sing-box DNS Rule Action 官方文档](https://sing-box.sagernet.org/configuration/dns/rule_action/)

## 关于 边界云加速

边界云加速提供跨平台网络连接与订阅使用相关服务，方便用户根据设备和客户端选择合适的配置方式。更多服务说明、客户端与订阅信息，请访问官网：[https://bianjiecloud.com/](https://bianjiecloud.com/)。