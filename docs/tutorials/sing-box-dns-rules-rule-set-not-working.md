> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-10。

# sing-box dns.rules[*].rule_set 不生效怎么办？DNS 规则匹配与 final 排查教程

**一句话结论：**`rule_set` 在 DNS 场景中必须写在 `dns.rules[*]` 的规则对象内，并且规则对象还需要配置有效的 `action`；仅在其他位置声明规则集、写错 tag，或把 `route` 放在更早的匹配规则之后，都可能导致 DNS 请求继续使用 `dns.final`。

## 适用场景

本文适合以下情况：

- sing-box 配置能够启动，但指定域名没有按预期使用某个 DNS 服务器；
- 已经定义了一个 rule-set，却不知道 DNS 规则中应该如何引用；
- 配置中同时存在 `dns.rules` 和 `dns.final`，部分域名似乎一直走默认 DNS；
- 从其他客户端迁移配置时，将路由规则中的 `rule_set` 写法直接复制到 DNS 配置中；
- 使用 sing-box 图形客户端或自建配置文件，需要只排查 DNS 规则引用问题。

本文只讨论一个字段：`dns.rules[*].rule_set`。它与 `route.rule_set` 都可以引用规则集，但所在位置和作用不同：前者用于匹配 DNS 查询，后者用于匹配连接路由。sing-box 官方文档将 `route.rule_set` 定义在 `route` 对象中，而 DNS 规则的 `rule_set` 则位于 `dns.rules` 的单条规则内。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 一、先理解 `dns.rules[*].rule_set` 的作用

在 sing-box 中，`dns` 对象包含 `servers`、`rules` 和 `final` 等字段。`final` 是没有其他 DNS 规则命中时使用的默认 DNS server tag；如果为空，则使用列表中的第一个 DNS server。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/))

`dns.rules[*].rule_set` 的作用是：让当前这条 DNS 规则根据指定 rule-set 中的规则进行匹配。例如，rule-set 中包含若干域名后缀，DNS 规则引用它后，匹配到这些域名的查询就可以执行当前规则的 `action`。

需要注意，下面这种写法不是 DNS 规则的正确位置：

```json
{
  "dns": {
    "rule_set": ["海外域名"]
  }
}
```

DNS 对象本身没有对应的顶层 `rule_set` 字段。正确位置是 `dns.rules` 数组中的某一条规则：

```json
{
  "dns": {
    "rules": [
      {
        "rule_set": ["海外域名"],
        "action": "route",
        "server": "dns-remote"
      }
    ]
  }
}
```

官方 DNS Rule 文档将 `rule_set` 列为 DNS 规则匹配字段，并说明它用于匹配 rule-set；同时，DNS 规则必须有 `action`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule/))

## 二、准备一个最小可读的 rule-set

为了先排除远程下载、缓存和复杂规则的影响，可以使用内联 rule-set 做验证。官方 rule-set 结构支持 `type: inline`、`tag` 和 `rules`，其中 `rules` 是 Headless Rule 列表。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/rule-set/))

下面的片段表示：创建一个名为 `海外域名` 的内联规则集，并匹配 `example.com` 及其子域名。

```json
{
  "route": {
    "rule_set": [
      {
        "type": "inline",
        "tag": "海外域名",
        "rules": [
          {
            "domain_suffix": [
              "example.com"
            ]
          }
        ]
      }
    ]
  }
}
```

这里的 `route.rule_set` 只是负责定义一个可被引用的 rule-set。定义完成后，还需要在 `dns.rules[*].rule_set` 中引用它。不要把定义位置误认为 DNS 规则的引用位置。

如果你的实际配置使用的是本地或远程 rule-set，应确保该对象拥有正确的 `tag`，并在 DNS 规则中使用完全一致的 tag。例如，`海外域名`、`海外域名 ` 和 `海外域名规则` 是不同字符串。本文不展开 source rule-set 文件格式，只建议先用上面的内联示例确认 `dns.rules[*].rule_set` 的引用逻辑，再替换为实际规则集。

## 三、把 rule-set 接到 DNS 规则中

假设配置中已经存在两个 DNS server，tag 分别为 `dns-local` 和 `dns-remote`。一个最小的 DNS 规则片段可以写成：

```json
{
  "dns": {
    "servers": [
      {
        "tag": "dns-local",
        "type": "local"
      },
      {
        "tag": "dns-remote",
        "type": "local"
      }
    ],
    "rules": [
      {
        "rule_set": [
          "海外域名"
        ],
        "action": "route",
        "server": "dns-remote"
      }
    ],
    "final": "dns-local"
  }
}
```

这个片段使用两个 `local` DNS server 只是为了展示 tag 和规则关系；在实际使用中，应将 `dns-remote` 替换为你已经配置好的合法 DNS server tag。重点不在 server 类型，而在以下三处必须互相对应：

1. `route.rule_set` 中存在 `tag: "海外域名"`；
2. `dns.rules[0].rule_set` 引用同一个 `海外域名`；
3. `dns.rules[0].action` 为 `route`，并且 `server` 指向已存在的 DNS server tag。

DNS Rule Action 文档说明，`route` 用于将 DNS 请求路由到指定 server，`server` 是目标 DNS server 的 tag；`evaluate` 则是另一种动作，不会终止规则评估，不能把它当成普通的 DNS 转发动作使用。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule_action/))

在边界云加速的 sing-box 订阅或自定义配置中，如果你需要排查某个域名是否命中 DNS rule-set，建议先复制一份配置，只保留一个明确的 `dns.rules[*].rule_set` 规则和一个 `dns.final`，确认逻辑后再合并回完整配置。

## 四、按顺序排查为什么仍然走 `dns.final`

### 1. 检查引用位置

先搜索配置中的 `dns` 对象，确认 `rule_set` 是否位于 `dns.rules` 的单条规则内。正确结构通常类似：

```json
"dns": {
  "rules": [
    {
      "rule_set": ["海外域名"],
      "action": "route",
      "server": "dns-remote"
    }
  ],
  "final": "dns-local"
}
```

如果 `rule_set` 与 `rules` 同级，或直接放在 `dns` 下，通常就是字段层级写错。

### 2. 检查 tag 是否完全一致

定义和引用必须使用相同 tag：

```json
"tag": "海外域名"
```

对应：

```json
"rule_set": ["海外域名"]
```

不要只根据显示名称判断。建议直接复制 tag，避免中英文标点、空格和大小写差异。

### 3. 检查 rule-set 是否真的包含可匹配规则

`rule_set` 只是引用关系，不会自动产生匹配条件。内联 rule-set 需要在 `rules` 中提供规则，例如 `domain`、`domain_suffix`、`domain_keyword` 或其他官方支持的 Headless Rule 字段。DNS Rule 文档将 `domain_suffix` 定义为域名后缀匹配，将 `domain` 定义为完整域名匹配。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule/))

例如：

```json
"rules": [
  {
    "domain_suffix": ["example.com"]
  }
]
```

这类规则可以匹配 `example.com` 及其后缀范围；如果你只希望匹配一个完整域名，则应考虑使用 `domain`，不要把两者混用后再判断结果。

### 4. 检查 DNS 规则顺序

DNS 规则按配置中的顺序进行判断。若前面已经有一条更宽泛的规则，并且它的 action 已经处理了请求，后面的 rule-set 规则就可能不会按你的预期发挥作用。

建议把更具体的 rule-set 规则放在更宽泛的规则之前，例如：

```json
"rules": [
  {
    "rule_set": ["海外域名"],
    "action": "route",
    "server": "dns-remote"
  },
  {
    "domain_suffix": ["local"],
    "action": "route",
    "server": "dns-local"
  }
]
```

`route` 动作会把 DNS 请求交给指定 server；官方文档还特别区分了 `evaluate`，后者保存响应供后续 `match_response` 使用，并不会像 `route` 一样结束普通的规则处理流程。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule_action/))

### 5. 检查测试域名是否真的属于规则集

不要只测试一个与规则集无关的域名，然后据此判断 `rule_set` 失效。例如规则集只包含 `example.com`，测试 `example.net` 时自然会落到 `dns.final`。

排查时可以先使用规则集中明确写入的域名，再测试其子域名，最后测试不在规则集中的域名。这样能够区分“规则没有命中”和“规则命中后 server 不可用”这两类问题。

### 6. 检查缓存影响

如果 DNS 查询结果已经被缓存，修改规则后立即重复访问同一域名，观察结果可能不明显。可以在客户端重启 sing-box，或使用客户端提供的清理 DNS 缓存功能后再测试。不要把一次缓存命中直接当成规则匹配失败。

## 常见问题

### Q1：`dns.rules[*].rule_set` 和 `route.rule_set` 可以互相替代吗？

不能。两者都可以引用 rule-set，但作用域不同。`route.rule_set` 属于路由配置，供 route rule 匹配连接；`dns.rules[*].rule_set` 属于 DNS 规则，供 DNS 查询匹配。官方文档分别在 Route 和 DNS Rule 中定义了这两个字段。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

### Q2：只写 `rule_set`，不写 `action` 可以吗？

不建议。DNS Rule 文档将 `action` 标记为必需字段。若希望匹配后将查询交给某个 DNS server，应明确写出 `"action": "route"` 和对应的 `"server"`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule/))

### Q3：为什么 `rule_set` 引用正确，但所有请求仍然使用 `final`？

优先检查三点：rule-set 的 tag 是否完全一致；rule-set 内是否存在能匹配测试域名的规则；`dns.rules` 中是否有更早的宽泛规则处理了请求。还要确认测试域名确实属于规则集，并在修改后清理或等待 DNS 缓存更新。

### Q4：能不能把 `dns.rule_set` 写成一个数组？

不能按照该结构理解。DNS 配置中的规则集引用应放在 `dns.rules` 的规则对象中，即 `dns.rules[*].rule_set`。规则集定义本身可以位于 `route.rule_set` 配置中，但这不意味着 DNS 对象存在同名顶层字段。

### Q5：`evaluate` 可以代替 `route` 吗？

不能简单替代。`route` 用于将 DNS 查询路由到指定 server；`evaluate` 用于查询并保存响应，供后续带 `match_response` 的规则使用，并且不会终止规则评估。只有在确实需要基于 DNS 响应内容继续匹配时，才考虑 `evaluate` 方案。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/dns/rule_action/))

## 参考资料

- https://sing-box.sagernet.org/configuration/dns/rule/
- https://sing-box.sagernet.org/configuration/dns/rule_action/
- https://sing-box.sagernet.org/configuration/dns/
- https://sing-box.sagernet.org/configuration/route/
- https://sing-box.sagernet.org/configuration/rule-set/

## 关于 边界云加速

边界云加速提供跨平台网络连接与订阅使用服务，支持结合常见客户端进行配置导入和连接排查。更多平台说明与使用教程请访问官网：[https://bianjiecloud.com/](https://bianjiecloud.com/)。