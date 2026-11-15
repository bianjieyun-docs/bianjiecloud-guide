> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-15。

# sing-box route.auto_detect_interface 配置与 TUN 路由环路排查

**一句话结论：**如果 sing-box 使用 TUN 接管流量后出现连接绕回 TUN、代理自身无法连接或部分网站反复超时，可以优先在顶层 `route` 对象中启用 `auto_detect_interface`，让出站连接默认绑定到系统默认网卡。

## 适用场景

本文只讨论一个配置字段：`route.auto_detect_interface`。它适合以下情况：

- 使用 sing-box 的 TUN 入站，并启用了 `auto_route`；
- 代理客户端启动后，订阅或远程规则无法更新；
- TUN 开启后出现连接循环、代理自身无法访问服务器、部分连接超时；
- 在 Windows、macOS 或 Linux 上，希望减少出站连接再次进入 TUN 的可能性；
- 需要确认 `auto_detect_interface` 应该放在什么位置，以及它与 `outbound.bind_interface`、`route.default_interface` 的关系。

sing-box 官方文档将 `auto_detect_interface` 定义为 `route` 层字段，并说明它只支持 Linux、Windows 和 macOS。启用后，出站连接默认绑定到系统默认网卡，主要用途是在 TUN 场景下避免路由环路；如果某个出站已经设置了 `bind_interface`，该字段对该出站不起作用。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

本文不展开 DNS 劫持、规则集、分流规则或测速配置，也不把 `auto_detect_interface` 当作“提升速度”的开关。它解决的是出站连接选择网卡和 TUN 路由回环方向的问题。

## 1. 先确认字段放置位置

sing-box 的配置文件使用 JSON 结构，顶层可以包含 `dns`、`inbounds`、`outbounds`、`route` 和 `experimental` 等对象。`auto_detect_interface` 位于顶层 `route` 对象内，不属于 `tun` 入站，也不属于某个单独的出站。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

正确的层级关系如下：

```json
{
  "route": {
    "auto_detect_interface": true
  }
}
```

以下写法属于常见位置错误：

```json
{
  "inbounds": [
    {
      "type": "tun",
      "auto_detect_interface": true
    }
  ]
}
```

`auto_detect_interface` 不是 TUN 入站字段。把它写在 `inbounds` 数组中的 TUN 对象里，不能达到官方文档描述的路由绑定效果。

同样，下面的写法也不是本文讨论的字段位置：

```json
{
  "outbounds": [
    {
      "type": "direct",
      "auto_detect_interface": true
    }
  ]
}
```

它应该位于 `route` 中，而不是直接放在某个 outbound 对象内。

## 2. 在已有配置中启用字段

如果你已经有完整的 sing-box 配置，只需要找到顶层 `route` 对象，并加入或修改这一项：

```json
{
  "route": {
    "auto_detect_interface": true,
    "final": "proxy"
  }
}
```

其中 `final` 只是示例，必须替换为你配置中实际存在的出站标签。本文的核心字段是 `auto_detect_interface`，不要为了测试而随意新增不存在的 outbound 标签。

如果原配置已经包含 `rules`、`rule_set` 或其他路由字段，应保留原字段，只在同一个 `route` 对象中加入该布尔值。例如：

```json
{
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "action": "hijack-dns"
      }
    ],
    "final": "proxy",
    "auto_detect_interface": true
  }
}
```

上面的示例只展示字段组合方式。实际使用时，`proxy` 标签必须与现有出站的 `tag` 一致；如果你的配置没有同名出站，就不能直接照抄 `final` 的值。

## 3. 与 TUN 入站配合使用

TUN 入站的 `auto_route` 用于把系统默认路由指向 TUN。官方 TUN 文档明确提示，为避免流量回环，可以设置 `route.auto_detect_interface`、`route.default_interface` 或某个出站的 `bind_interface`。因此，两者职责不同：

- `inbounds[].auto_route`：决定是否自动设置通向 TUN 的默认路由；
- `route.auto_detect_interface`：让出站连接默认绑定系统默认网卡；
- `outbound.bind_interface`：对指定出站明确绑定某个网卡；
- `route.default_interface`：对默认路由连接指定一个具体网卡名称。

官方文档还说明，`default_interface` 在 `auto_detect_interface` 生效时不起作用；而在出站已经设置 `bind_interface` 的情况下，`auto_detect_interface` 对该出站不起作用。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

一个只用于展示层级的最小片段如下：

```json
{
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "address": [
        "172.19.0.1/30"
      ],
      "auto_route": true
    }
  ],
  "route": {
    "auto_detect_interface": true
  }
}
```

这不是完整的代理配置，因为还需要根据实际订阅或自建服务补充出站、路由规则等内容。导入边界云加速提供的订阅后，如果客户端允许编辑生成的 sing-box 配置，可将该字段放入已有的顶层 `route` 对象；如果客户端会覆盖本地修改，应优先在客户端提供的高级配置或模板入口中调整。

## 4. 逐步排查配置是否生效

### 第一步：确认运行平台

官方文档列出的支持平台是 Linux、Windows 和 macOS。Android 和 Apple 图形客户端不能简单按照桌面端的支持范围推断行为；移动端通常还会受到系统 VPN、NetworkExtension 或客户端实现限制。因此，在 Android 或 iOS 上看到字段不生效时，先确认所使用的图形客户端是否支持对应配置项，而不要仅凭 JSON 结构判断。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 第二步：确认字段层级

检查 JSON 中是否是以下结构：

```json
{
  "route": {
    "auto_detect_interface": true
  }
}
```

重点检查三点：

1. 字段拼写是 `auto_detect_interface`，中间使用下划线；
2. 值是 JSON 布尔值 `true` 或 `false`，不是字符串 `"true"`；
3. 字段位于顶层 `route` 对象内。

错误示例：

```json
{
  "route": {
    "auto_detect_interface": "true"
  }
}
```

这里的值是字符串，不是布尔值。修改后还要确认逗号位置正确，避免因为 JSON 语法错误导致整个配置无法加载。

### 第三步：检查是否被更具体的绑定设置覆盖

如果某个出站对象已经设置了 `bind_interface`，官方说明 `route.auto_detect_interface` 对它不起作用。排查时应搜索整个配置中的 `bind_interface`，确认是否存在针对代理出站、DNS 出站或其他连接的显式绑定。

也要检查是否设置了 `route.default_interface`。官方文档说明，`default_interface` 与 `auto_detect_interface` 在默认网卡选择上不能同时按预期叠加使用；前者指定具体接口，后者自动选择默认网卡。若配置目标是自动跟随当前系统默认网络，通常不要同时保留一个过时或不匹配的固定接口名。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 第四步：观察问题是否只发生在 TUN 开启时

临时关闭 TUN 或关闭 `auto_route` 后，如果普通系统代理可以连接，而开启 TUN 后代理自身无法连接，说明问题更接近路由回环或网卡选择，而不是订阅链接本身。

这一步只能用于定位方向，不能据此断定所有连接失败都由 `auto_detect_interface` 引起。还应结合客户端日志、系统路由表和实际出站绑定设置进行判断。

## 常见问题

### 1. `auto_detect_interface` 是不是速度优化开关？

不是。官方定义是让出站连接默认绑定到系统默认网卡，主要用于 TUN 场景下避免路由环路。它不会承诺提升带宽、降低延迟或改变代理线路质量。启用后速度没有变化，并不代表字段无效；该字段的目标是连接路径和网卡选择，而不是测速表现。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 2. Windows 和 macOS 都可以使用吗？

官方 Route 文档列出的支持平台包括 Linux、Windows 和 macOS。因此，桌面端可以按本文的顶层 `route` 结构检查。实际是否能修改，还取决于所使用的 sing-box 客户端是否允许编辑或覆盖生成的配置文件。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 3. Android 上为什么加了字段仍然没有效果？

该字段的官方支持范围不包括 Android。Android 图形客户端通常通过系统 VPN 机制接管流量，不能把桌面端 TUN 配置的行为直接等同过来。对于 Android，应查看具体客户端的配置支持说明；不要仅因为配置文件能保存，就认为该字段一定会参与路由处理。

### 4. 已经设置了 `outbound.bind_interface`，还需要保留它吗？

如果某个出站已经设置了 `bind_interface`，官方说明 `auto_detect_interface` 对该出站不起作用。是否删除 `bind_interface`，要看你是否确实需要固定某个网卡；不要为了“叠加保险”同时设置互相冲突的接口策略。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 5. 能不能把字段放到 `tun` 入站里？

不能按官方字段结构这样配置。`auto_route` 属于 TUN 入站，而 `auto_detect_interface` 属于顶层 `route`。两者经常一起出现，但不是同一个对象下的字段。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### 6. 启用后仍然连接失败，下一步查什么？

先确认配置文件能够被当前 sing-box 版本和客户端加载，再检查是否存在 `bind_interface`、`default_interface` 或错误的默认出站标签。随后观察日志中是否出现代理服务器连接失败、DNS 解析失败或系统网络接口变化等信息。不要把所有 TUN 连接问题都归因于这一个字段；如果配置语法、出站参数或订阅内容本身有误，启用 `auto_detect_interface` 也无法修复。

## 参考资料

- [sing-box Route 官方文档](https://sing-box.sagernet.org/configuration/route/)
- [sing-box TUN 入站官方文档](https://sing-box.sagernet.org/configuration/inbound/tun/)
- [sing-box 配置结构官方文档](https://sing-box.sagernet.org/configuration/)
- [sing-box Migration 官方文档](https://sing-box.sagernet.org/migration/)

## 关于 边界云加速

边界云加速提供跨境网络连接相关的订阅与客户端使用指引，覆盖 Windows、macOS、Android、iOS 及常见代理客户端配置场景。使用相关服务时，请遵守当地法律法规。

官网：[https://bianjiecloud.com/](https://bianjiecloud.com/)