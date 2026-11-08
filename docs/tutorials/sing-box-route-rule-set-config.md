> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-08。

# sing-box route.rule_set 怎么配置？规则集引用与路由匹配排查教程

## 一句话结论

`route.rule_set` 用来**声明可供路由规则使用的规则集**，而真正执行匹配的是 `route.rules[*].rule_set`。只在 `route.rule_set` 中登记规则集，并不会自动改变流量去向；还需要在 `route.rules` 的具体规则中引用对应的 `tag`，并配置 `action` 与目标出站。官方文档将 `route.rule_set` 定义为规则集列表，将路由规则中的 `rule_set` 定义为匹配规则集的字段。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 适用场景

本文适用于以下情况：

- 使用 sing-box 配置文件进行分流，希望让某个域名集合走直连、代理或拦截；
- 配置中已经有规则集，但修改后流量仍然没有按照预期分流；
- 不确定 `route.rule_set` 和 `route.rules[*].rule_set` 是否应该写成同一个字段；
- 使用边界云加速提供的 sing-box 配置或自行维护 sing-box 配置时，需要确认规则集声明和引用位置。

本文只讲 `route.rule_set` 这一项及其在路由规则中的引用，不展开 DNS 规则、TUN 接管或复杂的多层规则组合。

## 1. 先区分两个容易混淆的位置

sing-box 中常见的两个位置作用不同：

| 配置位置 | 作用 | 是否直接决定流量去向 |
| --- | --- | --- |
| `route.rule_set` | 声明规则集对象，例如 inline、local 或 remote 规则集 | 否 |
| `route.rules[*].rule_set` | 在某一条路由规则中引用已声明的规则集 `tag` | 参与匹配，但仍需要动作 |
| `route.rules[*].action` | 指定匹配后执行的动作 | 是 |
| `route.rules[*].outbound` | 当动作是 `route` 时指定目标出站 | 是 |

`route.rule_set` 的值是一个列表，每个规则集通常需要有唯一的 `tag`。规则集可以是内联规则，也可以来自本地文件或远程 URL；官方结构中，local 和 remote 类型需要指定规则集格式、路径或 URL。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

因此，下面这种写法只完成了“登记规则集”，并不会自动让 `example.com` 走直连：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "inline",
        "tag": "example-sites",
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

还需要在 `route.rules` 中引用 `example-sites`，并指定匹配后的动作。

## 2. 使用 inline 规则集完成最小可用配置

为了便于验证，下面使用 `inline` 规则集。它不依赖外部文件，适合先确认字段位置和匹配逻辑是否正确。示例包含一个直连出站和一个代理出站名称；其中 `proxy` 只是示例标签，必须替换为你的配置中实际存在的出站标签。

```json
{
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "rule_set": [
      {
        "type": "inline",
        "tag": "example-sites",
        "rules": [
          {
            "domain_suffix": [
              "example.com",
              "example.org"
            ]
          }
        ]
      }
    ],
    "rules": [
      {
        "rule_set": [
          "example-sites"
        ],
        "action": "route",
        "outbound": "direct"
      }
    ],
    "final": "direct"
  }
}
```

这段配置的逻辑是：

1. 在 `route.rule_set` 中创建名为 `example-sites` 的规则集；
2. 规则集中的 `domain_suffix` 匹配 `example.com` 和 `example.org` 的域名后缀；
3. `route.rules[0].rule_set` 引用这个标签；
4. 匹配成功后执行 `action: "route"`；
5. 通过 `outbound: "direct"` 将流量交给名为 `direct` 的出站；
6. 未匹配的流量按照 `route.final` 处理。

官方文档说明，`route` 动作需要指定目标出站标签；出站标签必须与 `outbounds` 中已有的 `tag` 对应。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/rule_action/))

## 3. 使用本地或远程规则集时怎么写

如果规则集较大，通常不会全部内嵌在配置中，而是使用本地文件或远程文件。`route.rule_set` 负责声明文件来源，`route.rules[*].rule_set` 仍然只引用它的 `tag`。

本地二进制规则集的配置结构可以写成：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "local",
        "tag": "company-sites",
        "format": "binary",
        "path": "./rules/company-sites.srs"
      }
    ],
    "rules": [
      {
        "rule_set": [
          "company-sites"
        ],
        "action": "route",
        "outbound": "proxy"
      }
    ]
  }
}
```

使用远程规则集时，官方结构支持 `type: "remote"`、`format`、`url` 和可选的更新间隔。例如：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "remote",
        "tag": "company-sites",
        "format": "binary",
        "url": "https://example.com/rules/company-sites.srs%22,
        "update_interval": "1d"
      }
    ]
  }
}
```

上面的 URL 仅用于展示配置结构，不能直接当作可用规则集地址。实际使用时，应替换为你确认过的规则集 URL，并确保运行环境能够访问该地址。远程规则集还涉及下载客户端、缓存和更新行为，不能只凭 `url` 字段判断规则内容一定有效。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/rule-set/))

如果使用 `format: "source"`，文件也不能是任意 JSON 数组或普通文本域名列表。官方 source rule-set 结构需要包含 `version` 和 `rules` 字段，`rules` 中放置符合 sing-box Headless Rule 结构的规则；这是规则集文件格式要求，不等同于普通 JSON 列表。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/rule-set/source-format/))

## 4. 修改配置后如何排查 route.rule_set 不生效

### 检查一：`tag` 是否完全一致

声明处使用的是：

```json
"tag": "example-sites"
```

引用处就必须写：

```json
"rule_set": ["example-sites"]
```

大小写、连字符、下划线和空格都应保持一致。`example-sites`、`example_sites` 和 `Example-Sites` 应视为不同字符串，不要凭记忆修改标签。

### 检查二：是否把声明和引用写在了同一层

正确结构是：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "inline",
        "tag": "example-sites",
        "rules": [
          {
            "domain_suffix": ["example.com"]
          }
        ]
      }
    ],
    "rules": [
      {
        "rule_set": ["example-sites"],
        "action": "route",
        "outbound": "direct"
      }
    ]
  }
}
```

不要把规则集对象直接放到 `route.rules[*].rule_set` 中，也不要把 `route.rules[*].rule_set` 误写成顶层的独立配置项。

### 检查三：目标 outbound 是否存在

如果规则匹配成功，但 `outbound` 写成了不存在的标签，配置仍然不能按预期工作。先检查：

```json
"outbound": "proxy"
```

是否能在配置的 `outbounds` 数组中找到：

```json
{
  "type": "某个出站类型",
  "tag": "proxy"
}
```

如果只想验证规则是否命中，可以先将目标改成已确认存在的 `direct` 出站，排除代理出站自身配置的问题。

### 检查四：确认匹配字段符合预期

`domain` 用于匹配完整域名，`domain_suffix` 用于匹配域名后缀，`domain_keyword` 用于关键词匹配。不要把一个本应匹配后缀的规则误写成完整域名，也不要把带路径的 URL 直接放进域名规则中。例如，`https://example.com/path%60 不是合法的 `domain_suffix` 值；应填写域名部分，例如 `example.com`。

### 检查五：观察规则顺序和 final

路由规则通常按配置顺序参与匹配。若前面已有一条更宽泛的规则先处理了流量，后面的 `rule_set` 可能没有机会按预期发挥作用。可以暂时把目标规则放到更靠前的位置，并为未匹配流量设置明确的 `route.final`，避免通过空值推测默认行为。官方文档说明，`final` 是默认出站标签；为空时会使用第一个出站。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/route/))

## 常见问题

### 1. 只写 `route.rule_set`，为什么没有分流？

因为 `route.rule_set` 只是声明规则集。必须在 `route.rules` 的某一条规则中通过 `rule_set` 引用对应标签，并配置动作，例如 `action: "route"` 和 `outbound`。没有引用规则时，规则集不会自动改变流量路径。

### 2. `route.rule_set` 和 `route.rules[*].rule_set` 要写成一样吗？

它们的字符串标签通常要对应，但所在位置和作用不同。前者放规则集对象，后者放要匹配的标签数组。例如前者的 `tag` 是 `company-sites`，后者就引用 `company-sites`。

### 3. `rule_set` 能不能直接写域名数组？

不能把域名数组直接当作完整的规则集对象使用。inline 规则集需要包含 `type`、`tag` 和 `rules`；local 或 remote 规则集则需要符合对应的文件来源和格式配置。source 文件也必须遵守 sing-box 的 source rule-set 结构。

### 4. 为什么 `format` 写了 `source` 仍然加载失败？

需要同时检查文件内容和文件扩展名、路径或 URL。source 格式不是普通文本列表，也不是任意 JSON 数组；至少要符合官方定义的版本字段与规则列表结构。若只是想先确认路由字段位置，可以先使用 inline 规则集进行最小化验证。

### 5. 规则匹配后能不能直接写代理地址？

不能。`outbound` 应填写配置中已有出站的 `tag`，而不是服务器地址、订阅名称或 URL。目标出站的具体协议、服务器和认证信息应在 `outbounds` 中单独配置。

## 参考资料

- https://sing-box.sagernet.org/configuration/route/
- https://sing-box.sagernet.org/configuration/route/rule/
- https://sing-box.sagernet.org/configuration/route/rule_action/
- https://sing-box.sagernet.org/configuration/rule-set/
- https://sing-box.sagernet.org/configuration/rule-set/source-format/

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供跨平台网络连接与订阅使用相关服务，本文用于帮助用户理解 sing-box 配置中的规则集字段。使用相关服务时，请遵守当地法律法规。