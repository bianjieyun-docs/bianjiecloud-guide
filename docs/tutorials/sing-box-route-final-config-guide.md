> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-17。

# sing-box route.final 怎么配置？默认出站与规则匹配排查教程

**一句话结论：**`route.final` 用来指定没有被前面路由规则明确处理的连接所使用的默认出站，它的值必须对应 `outbounds` 中已有的 `tag`；如果留空，sing-box 会使用第一个出站。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 适用场景

本文适合以下情况：

- sing-box 配置中已经有多个出站，但不知道未命中规则的流量最终从哪里出去；
- 修改订阅或手写 JSON 后，出现“配置能启动，但流量没有按预期走代理”的问题；
- 想让大部分流量默认走代理，仅将局域网、国内网站或特定地址通过前置规则直连；
- 使用 TUN、混合入站或图形客户端时，需要确认默认路由出口，而不是反复修改 DNS 或规则集；
- 使用边界云加速提供的 sing-box 订阅或配置时，需要根据配置中实际存在的出站标签设置默认出口。

本文只讨论 `route.final` 一个字段，不展开 DNS、TUN 环路、规则集 source 格式或测速问题。

## 一、先理解 route.final 的作用

sing-box 的配置文件使用 JSON 结构，顶层可以包含 `outbounds` 和 `route` 等部分。`route.final` 位于 `route` 对象内部，官方定义是“默认出站标签”；当该字段为空时，使用第一个出站。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

可以把它理解成路由规则处理完成后的“兜底出口”：

```text
连接进入 sing-box
        │
        ▼
依次检查 route.rules
        │
        ├─ 命中规则：按规则 action.route.outbound 选择出站
        │
        └─ 没有命中：使用 route.final
```

因此，`route.final` 不是规则集名称，也不是协议名称，更不是服务器地址。它只是一个字符串，用来引用某个出站的 `tag`。

## 二、最小可用配置示例

下面是一个只用于理解字段关系的最小示例。示例中的出站标签 `proxy-out` 和 `direct-out` 是人为定义的占位名称，实际使用时必须与自己配置中的 `outbounds[*].tag` 完全一致。

```json
{
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 2080
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct-out"
    },
    {
      "type": "socks",
      "tag": "proxy-out",
      "server": "proxy.example.com",
      "server_port": 443,
      "ク" : "此处不是有效示例"
    }
  ],
  "route": {
    "final": "proxy-out"
  }
}
```

上面的配置故意包含了一个无效字段和无效内容，不能直接运行。实际配置中应删除该部分。下面给出可用于说明字段关系的有效版本，但其中的代理服务器地址、端口和认证信息仍需替换为真实参数：

```json
{
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct-out"
    },
    {
      "type": "socks",
      "tag": "proxy-out",
      "server": "192.0.2.10",
      "server_port": 1080
    }
  ],
  "route": {
    "final": "proxy-out"
  }
}
```

这里的关键关系只有两点：

1. `route.final` 的值是 `proxy-out`；
2. `outbounds` 中存在一个 `tag` 同样为 `proxy-out` 的出站。

如果将 `route.final` 改成 `direct-out`，未被其他规则处理的连接就会交给直连出站。如果删除 `route.final`，则会回退到 `outbounds` 数组中的第一个出站，因此不能只看出站类型，还要注意出站排列顺序。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 三、按步骤配置 route.final

### 第 1 步：找到 outbounds 数组

打开 sing-box 配置文件，搜索：

```json
"outbounds": [
```

逐个查看出站对象中的 `tag`，例如：

```json
{
  "type": "direct",
  "tag": "direct-out"
}
```

这里真正可供引用的是 `direct-out`，不是 `direct`。`type` 表示出站类型，`tag` 才是配置中用于引用该出站的标签。

如果配置来自订阅转换、客户端导入或服务商生成的配置，不要凭经验猜测标签名称，应直接以当前文件中的 `tag` 为准。

### 第 2 步：在 route 对象中添加 final

如果配置中已经有 `route`，只需在其中添加字段：

```json
"route": {
  "final": "proxy-out"
}
```

如果原来已经有其他字段，需要注意 JSON 逗号：

```json
"route": {
  "rules": [
    {
      "domain_suffix": ["example.com"],
      "action": "route",
      "outbound": "direct-out"
    }
  ],
  "final": "proxy-out"
}
```

这个例子表示：匹配 `example.com` 后缀的连接使用 `direct-out`；其他没有被该规则处理的连接使用 `proxy-out`。在当前 sing-box 文档中，路由规则的 `outbound` 位于 `action: route` 对应的规则动作中，并且要求填写目标出站标签。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule/))

### 第 3 步：确认 final 没有写到 DNS 配置中

`route.final` 只属于顶层 `route` 对象，不能写入 `dns`、`dns.rules` 或某个出站对象内部。以下写法位置错误：

```json
{
  "dns": {
    "final": "proxy-out"
  }
}
```

正确位置是：

```json
{
  "route": {
    "final": "proxy-out"
  }
}
```

DNS 规则和连接路由是不同层次的配置。即使 DNS 查询使用了某个 DNS server，也不等于普通连接会自动使用同名出站；不要用 DNS 字段替代 `route.final`。

### 第 4 步：检查标签是否完全一致

以下内容都可能导致引用失败或行为不符合预期：

- `proxy-out` 与 `Proxy-Out` 大小写不同；
- 标签前后多了空格；
- `route.final` 写成了出站的 `type`，例如 `socks`，但实际标签是 `proxy-out`；
- 修改了出站标签，却没有同步修改 `route.final`；
- 复制配置时保留了不存在的旧标签。

建议使用文本搜索同时搜索 `"final"` 和对应的 `"tag"`，确认两处字符串一致。

### 第 5 步：执行配置检查

sing-box 官方配置说明提供了 `check` 命令。命令形式如下：

```bash
sing-box check -c config.json
```

其中 `config.json` 替换成实际配置文件路径。对于自动化脚本或命令行排查，成功时应以进程无错误退出作为判断依据，不应依赖某一行固定提示文字。配置检查通过，只能说明配置结构和字段关系通过检查；它不代表代理服务器一定可连接，也不代表所有域名都能访问。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

## 四、route.rules 与 route.final 的先后关系

`route.final` 是默认出口，不会自动覆盖已经匹配到的规则。比如：

```json
{
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct-out"
    },
    {
      "type": "socks",
      "tag": "proxy-out",
      "server": "192.0.2.10",
      "server_port": 1080
    }
  ],
  "route": {
    "rules": [
      {
        "domain_suffix": ["intranet.example"],
        "action": "route",
        "outbound": "direct-out"
      }
    ],
    "final": "proxy-out"
  }
}
```

可以按以下方式理解：

- 访问 `intranet.example` 后缀域名，并且命中该规则：使用 `direct-out`；
- 访问没有命中这条规则的其他目标：使用 `proxy-out`；
- `route.final` 不会把已经由规则指定的 `outbound` 改掉；
- 如果规则中使用了 `action: route`，其 `outbound` 应填写目标出站的标签。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule/))

因此，排查“所有流量都走直连”时，应同时检查规则是否覆盖范围过大，以及 `route.final` 是否指向了直连出站，而不能只看其中一个字段。

## 常见问题

### 1. route.final 可以直接写服务器地址吗？

不可以。`route.final` 写的是出站标签，不是 IP 地址、域名、端口或订阅链接。服务器地址应放在具体出站对象中，`route.final` 只引用该对象的 `tag`。

### 2. route.final 写成 socks、vmess 或 direct 可以吗？

只有当配置中确实存在相同名称的 `tag` 时才可以。`socks`、`vmess`、`direct` 通常是出站的 `type`，不一定是标签。应以 `outbounds` 中实际的 `tag` 为准。

### 3. 不写 route.final 会怎样？

按照官方说明，如果 `final` 为空，sing-box 会使用第一个出站。因此，不写该字段并不等于“没有默认出口”。如果第一个出站是直连，未命中规则的流量就可能直接连接；如果第一个出站是代理，则可能表现为默认走代理。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 4. route.final 配置正确，但访问仍然失败，说明字段错了吗？

不一定。`route.final` 只负责选择默认出站。配置检查通过后，仍可能存在服务器地址、端口、认证、TLS、网络连通性或客户端权限等其他问题。可以先确认 `sing-box check -c 配置文件` 进程无错误退出，再查看运行日志和目标出站本身的参数。

### 5. route.final 和规则里的 outbound 应该写成同一个值吗？

不要求相同。它们都引用出站标签，但用途不同：规则中的 `outbound` 处理命中的特定流量，`route.final` 处理没有被规则明确处理的流量。常见做法是让规则把局域网或特定域名交给直连出站，同时让 `final` 指向代理出站。

### 6. 修改 route.final 后需要重新导入订阅吗？

如果配置由订阅自动生成，手工修改可能在下一次更新时被覆盖。建议先确认客户端是否支持本地覆写或独立配置；如果配置由服务端生成，则应以当前订阅内容中的出站标签为准，并在更新后重新检查 `route.final` 是否仍然引用存在的标签。

## 参考资料

1. [sing-box Route 官方文档](https://sing-box.sagernet.org/configuration/route/)
2. [sing-box Route Rule 官方文档](https://sing-box.sagernet.org/configuration/route/rule/)
3. [sing-box Rule Action 官方文档](https://sing-box.sagernet.org/configuration/route/rule_action/)
4. [sing-box Configuration 官方文档](https://sing-box.sagernet.org/configuration/)
5. [sing-box Outbound 官方文档](https://sing-box.sagernet.org/configuration/outbound/)

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供跨平台网络连接与订阅使用相关服务，本文仅用于说明 sing-box 配置字段及排查思路。使用相关服务和网络工具时，请遵守当地法律法规。