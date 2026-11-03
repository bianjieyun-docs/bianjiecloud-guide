> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-03。

# sing-box rule_set 配置报错排查：source 规则集格式与 DNS 引用位置

**一句话结论：**sing-box 的 `rule_set` 报错，优先检查规则集声明位置、`source` 文件结构和 `version` 兼容性；不要把普通域名数组或文本列表直接当作 source 规则集，也不要把 DNS 规则引用写成 `dns.rule_set`。

## 适用场景

本文适合以下情况：

- 启动 sing-box 时提示 `rule-set`、`source`、`format` 或字段解析错误；
- 配置中出现 `dns.rule_set`，但 DNS 分流没有按预期生效；
- 使用本地 JSON 规则集时，程序提示格式不正确或缺少字段；
- 规则集文件可以打开，但路由规则始终没有命中；
- 需要在导入完整订阅前，用最小配置定位单个字段错误。

sing-box 使用 JSON 作为配置文件格式，配置通常包含 `dns`、`inbounds`、`outbounds` 和 `route` 等顶层对象。官方文档也提供了 `sing-box check`、`format` 和 `merge` 等配置处理命令。实际排查时，应先确认当前安装版本，再按照对应版本的官方文档检查字段。

## 第一步：区分规则集声明和规则集引用

### 1. `route.rule_set`：声明规则集

规则集通常在顶层 `route` 对象的 `rule_set` 数组中声明。下面示例使用本地 source 文件：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "local",
        "tag": "work-domains",
        "format": "source",
        "path": "./work-domains.json"
      }
    ],
    "rules": [
      {
        "rule_set": ["work-domains"],
        "action": "route",
        "outbound": "proxy"
      }
    ],
    "final": "direct"
  }
}
```

这里的 `route.rule_set` 负责注册规则集，`tag` 是后续规则引用时使用的标签。`path` 指向 sing-box 运行时能够读取的本地文件，`format` 需要与文件实际格式一致。

不要把规则集写成顶层的 `rule_set`，也不要把规则集声明写成 `dns.rule_set`。规则集定义属于 `route.rule_set`；DNS 场景只是通过 DNS 规则引用已经声明的规则集。

### 2. `route.rules[*].rule_set`：用于连接路由

在 `route.rules` 中，`rule_set` 是匹配条件，值应与 `route.rule_set[*].tag` 完全一致：

```json
{
  "route": {
    "rule_set": [
      {
        "type": "local",
        "tag": "work-domains",
        "format": "source",
        "path": "./work-domains.json"
      }
    ],
    "rules": [
      {
        "rule_set": ["work-domains"],
        "action": "route",
        "outbound": "proxy"
      }
    ],
    "final": "direct"
  }
}
```

`work-domains`、`work_domains` 和 `Work-Domains` 是不同的标签。排查“找不到规则集”或“规则不命中”时，应逐字比较标签内容，并检查规则顺序是否被更早的规则匹配。

### 3. `dns.rules[*].rule_set`：用于 DNS 规则

如果希望 DNS 查询按照某个规则集选择 DNS server，应在 `dns.rules` 中引用规则集：

```json
{
  "dns": {
    "servers": [
      {
        "tag": "local",
        "address": "local"
      },
      {
        "tag": "remote-dns",
        "address": "https://dns.example/dns-query%22
      }
    ],
    "rules": [
      {
        "rule_set": ["work-domains"],
        "action": "route",
        "server": "remote-dns"
      }
    ],
    "final": "local"
  }
}
```

上例中的 `https://dns.example/dns-query%60 只是格式示意，不能直接作为实际 DNS 地址使用。实际配置时，应替换为你确认可访问、且符合所在网络环境的 DNS server。

需要特别注意的是：规则集仍然要在 `route.rule_set` 中声明，DNS 场景通过 `dns.rules[*].rule_set` 引用。不要根据字段名称猜测存在一个 `dns.rule_set` 顶层配置项。

## 第二步：检查 source 规则集文件

### 错误写法：普通 JSON 数组

下面只是一个普通字符串数组，不符合 sing-box source rule-set 文件结构：

```json
[
  "example.com",
  "example.org"
]
```

普通文本列表也不能直接声明为 `format: "source"`：

```text
example.com
example.org
```

如果手里只有 TXT、CSV 或其他域名列表，需要先转换成 sing-box 支持的 source JSON 结构，不能只修改文件扩展名。

### 正确写法：source rule-set JSON

下面是只使用基础域名匹配字段的示例：

```json
{
  "version": 1,
  "rules": [
    {
      "domain_suffix": [
        "example.com",
        "example.org"
      ]
    }
  ]
}
```

最外层必须是对象，并包含 `version` 与 `rules`。`rules` 是规则对象列表，规则对象中的字段必须符合 sing-box 的 Headless Rule 定义。

`version` 是 source rule-set 格式版本，不是随意填写的配置版本号。本文示例使用 `1`，对应 sing-box 初始 source 格式，适合示例中的基础 `domain_suffix` 规则。较新的 sing-box 文档还列出了后续格式版本；如果配置使用了新版本才支持的规则项，应以当前安装版本和官方 Source Format 文档为准，不能机械地把所有文件都改成 `5`。旧版 sing-box 读取较新的格式版本时，可能出现版本不兼容或解析失败。

## 第三步：编译为 binary 规则集

如果需要把 source JSON 编译为二进制规则集，可以使用官方提供的命令：

```bash
sing-box rule-set compile --output work-domains.srs work-domains.json
```

编译后，配置可以改为：

```json
{
  "type": "local",
  "tag": "work-domains",
  "format": "binary",
  "path": "./work-domains.srs"
}
```

`.json` source 文件不能标记为 `binary`，`.srs` 二进制文件也不能标记为 `source`。如果只想先排查字段问题，建议先直接使用本地 source JSON；确认能够解析后，再进行编译。

## 第四步：用 `sing-box check` 检查配置

将配置保存为 `config.json` 后，先执行：

```bash
sing-box check -c ./config.json
```

Windows PowerShell 可以使用：

```powershell
sing-box.exe check -c .\config.json
```

macOS 或 Linux 如果程序位于当前目录，可以使用：

```bash
./sing-box check -c ./config.json
```

不要依赖某一段固定的成功提示文本。更稳妥的判断方式是查看命令是否以错误退出，并检查终端是否出现配置解析错误。如果提示找不到 `sing-box`，那是可执行文件路径或 PATH 问题，不等同于 `rule_set` 配置错误。

建议采用由小到大的排查顺序：

1. 只保留一个本地 source 规则集；
2. 只保留一条 `route.rules` 引用；
3. 确认 `sing-box check` 不再报告结构错误；
4. 再加入 DNS 规则、远程规则集和其他出站；
5. 每加入一组配置就重新检查一次。

如果你的订阅来自边界云加速，建议先在客户端中重新导入订阅，并查看客户端生成的 sing-box 配置是否已经包含规则集声明。不要把订阅链接的返回内容直接复制成普通域名数组，再手动标记为 `source`。

## 常见问题

### 1. 为什么 `dns.rule_set` 不生效？

因为规则集声明与 DNS 规则引用是两个位置。规则集应在 `route.rule_set` 中声明，DNS 规则应写在 `dns.rules` 中，并在其中使用 `rule_set` 作为匹配条件。`dns.rule_set` 不是本文所述场景下对应的顶层声明位置。

### 2. `source` 能否直接读取 TXT 或域名列表？

不能把普通 TXT、CSV 或逐行域名列表直接声明为 `format: "source"`。source 文件必须符合 sing-box 规定的 JSON 结构，至少包含 `version` 和 `rules`。需要使用文本规则时，应先转换为官方支持的格式，或按照官方文档提供的转换方式生成规则集。

### 3. `version` 应该填写 1 还是其他数字？

它取决于 source rule-set 格式和规则项需求。本文基础示例使用 `version: 1`，避免将较新的格式版本误用于旧版 sing-box。当前官方文档列出了多个格式版本，并说明各版本新增的规则项；如果使用了新规则字段，应以当前安装版本的官方文档为准。

### 4. 配置能通过检查，但规则没有命中怎么办？

配置检查通过只表示结构和字段基本可解析，不代表业务一定会匹配。应依次确认 source 文件中是否确实包含目标域名、`domain` 与 `domain_suffix` 是否使用正确、规则集标签是否完全一致，以及前面是否存在更早命中的规则。

### 5. 相对路径为什么在图形客户端中找不到文件？

`path` 是 sing-box 运行环境解析的路径，相对路径可能受工作目录影响，不一定等于配置文件所在目录。排查时应确认规则文件确实随配置一起部署，并检查文件名大小写；如果客户端对工作目录处理不明确，可按照客户端文档改用绝对路径或其支持的配置目录。

### 6. 远程规则集加载失败，是否说明 source 内容错误？

不一定。远程规则集还涉及 URL 可访问性、TLS、下载客户端、缓存和初始文件路径等因素。建议先用本地 source 文件验证 `version`、`rules` 和引用关系，再单独排查远程下载问题，这样可以区分文件格式错误与网络加载失败。

## 参考资料

- [sing-box 配置引言](https://sing-box.sagernet.org/configuration/)
- [sing-box Route Rule](https://sing-box.sagernet.org/configuration/route/rule/)
- [sing-box Rule Action](https://sing-box.sagernet.org/configuration/route/rule_action/)
- [sing-box DNS Rule](https://sing-box.sagernet.org/configuration/dns/rule/)
- [sing-box Rule-set](https://sing-box.sagernet.org/configuration/rule-set/)
- [sing-box Source Format](https://sing-box.sagernet.org/configuration/rule-set/source-format/)
- [sing-box Headless Rule](https://sing-box.sagernet.org/configuration/rule-set/headless-rule/)

## 关于 边界云加速

边界云加速提供跨平台网络连接与订阅使用相关服务，本文仅整理 sing-box 配置排查方法，不对第三方客户端或具体规则内容作保证。更多平台介绍与使用教程，请访问[边界云加速官网](https://bianjiecloud.com/)。