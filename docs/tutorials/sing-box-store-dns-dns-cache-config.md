> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-13。

# sing-box store_dns 怎么配置？持久化 DNS 缓存字段排查教程

## 一句话结论

如果希望 sing-box 在进程重启后继续使用已经保存的 DNS 缓存，可以在 `experimental.cache_file` 下启用 `store_dns: true`；但必须同时启用 `cache_file.enabled`，并且这个字段只负责持久化 DNS 缓存，不负责选择 DNS 服务器，也不负责修改路由规则。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 适用场景

本文适合以下情况：

- 你正在维护 sing-box JSON 配置，希望了解 `store_dns` 的准确位置和用途；
- sing-box 已经配置了 DNS，但重启进程后缓存没有继续保留；
- 配置中仍然使用 `store_rdrc`，需要按照 sing-box 1.14.0 之后的结构迁移；
- 你想确认 `store_dns` 是否应该写在 `dns`、`route` 或 `rule-set` 下面；
- 你使用边界云加速提供的 sing-box 订阅或自定义配置，需要在本地客户端中补充缓存选项。

本文只讨论一个字段：`experimental.cache_file.store_dns`。不会展开 DNS 规则匹配、TUN、路由规则集或不同客户端的完整配置教程。

## 1. 先理解 store_dns 在配置中的位置

`store_dns` 的完整路径是：

```text
experimental.cache_file.store_dns
```

它不是顶层字段，也不是 `dns.store_dns`，更不是 `route.store_dns`。sing-box 官方文档将 `cache_file` 放在 `experimental` 对象下，而 `store_dns` 则是缓存文件对象中的字段。官方结构中，相关层级如下：([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/))

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

其中：

- `experimental`：实验性功能的配置对象；
- `cache_file`：缓存文件相关配置；
- `enabled`：启用缓存文件；
- `store_dns`：将 DNS 缓存保存到缓存文件。

官方资料明确说明，`store_dns` 自 sing-box 1.14.0 起提供，作用是将 DNS cache 存储到 cache file。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 2. 使用最小可用配置

如果你的现有配置中还没有 `experimental`，可以添加下面的对象：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

这段内容是一个配置片段。实际使用时，应将 `experimental` 合并到现有 sing-box 配置的顶层，而不是重复添加两个同名的 `experimental` 对象。

例如，已有配置包含 `inbounds` 和 `outbounds` 时，可以合并为：

```json
{
  "log": {
    "level": "info"
  },
  "inbounds": [],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

示例中的 `inbounds`、`outbounds` 只是展示 JSON 层级。接入实际配置时，应保留你原有的入站、出站、DNS 和路由内容，不要为了添加 `store_dns` 而覆盖原配置。

## 3. enabled 与 store_dns 为什么要同时配置

`store_dns` 是缓存保存选项，而 `enabled` 才是缓存文件开关。官方配置结构将两者放在同一个 `cache_file` 对象中，并分别定义为“启用缓存文件”和“将 DNS 缓存保存到缓存文件”。因此，排查时不要只写：

```json
{
  "experimental": {
    "cache_file": {
      "store_dns": true
    }
  }
}
```

更稳妥的写法是：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

如果 `enabled` 没有启用，配置中即使出现 `store_dns: true`，也不能把它当作缓存文件已经正常启用来判断。这里的结论来自官方字段定义和配置层级，而不是某个客户端的自定义行为。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 4. 不指定 path 时，缓存文件放在哪里

`cache_file.path` 用于指定缓存文件路径。官方文档说明，如果 `path` 为空，将使用 `cache.db`。因此，最小配置可以不填写 `path`：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

如果你需要明确指定路径，可以写成：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "path": "/path/to/cache.db",
      "store_dns": true
    }
  }
}
```

上面的 `/path/to/cache.db` 是路径格式示例，不能直接当作 Windows、macOS、Linux 或 Android 的通用实际路径。请根据运行 sing-box 的用户权限、客户端沙盒和系统目录规则替换它。若不确定客户端允许写入哪些目录，优先使用省略 `path` 的最小配置，或使用该客户端文档指定的数据目录。官方只确认空路径时使用 `cache.db`，没有为所有图形客户端规定统一的绝对路径。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 5. 从 store_rdrc 迁移到 store_dns

如果旧配置中存在下面的写法：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_rdrc": true
    }
  }
}
```

需要注意：官方已将 `store_rdrc` 标记为 sing-box 1.14.0 弃用字段，并说明它计划在 1.16.0 移除。官方迁移文档给出的替代方式是使用 `store_dns`，后者保存完整 DNS 缓存，而不是只保存旧版 rejected DNS response cache。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

迁移后的写法是：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

迁移时不要同时保留两个字段作为长期方案：

```json
{
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_rdrc": true,
      "store_dns": true
    }
  }
}
```

这样做会让配置意图变得不清晰，也不能解决旧字段即将移除带来的兼容性问题。按照官方迁移说明，将 `store_rdrc` 替换为 `store_dns` 即可。([sing-box.sagernet.org](https://sing-box.sagernet.org/migration/))

## 6. store_dns 不负责哪些事情

`store_dns` 只涉及 DNS 缓存是否保存到缓存文件，不能用来完成以下操作：

- 指定 DNS 服务器；
- 修改 `dns.rules` 的匹配顺序；
- 让某个域名强制走代理或直连；
- 替代 `dns.final`；
- 下载或更新远程 rule-set；
- 修复代理出站连接失败。

例如，下面的配置仍然需要由 `dns` 部分负责 DNS 服务器和规则：

```json
{
  "dns": {
    "servers": [],
    "rules": [],
    "final": ""
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "store_dns": true
    }
  }
}
```

这里的 `store_dns` 不会自动补充 `servers`，也不会自动生成 DNS 规则。它和 DNS 服务器、DNS 规则是不同层级的功能。这个区分可以避免把“缓存持久化”误认为“DNS 分流配置”。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 7. 远程 rule-set 缓存与 DNS 缓存不是一回事

sing-box 官方 rule-set 文档还提到，远程 rule-set 会在 `experimental.cache_file.enabled` 时被缓存。这说明 `cache_file` 可能同时服务于不同类型的缓存，但不要因此把 `store_dns` 理解为“开启所有缓存”。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/rule-set/))

可以这样区分：

| 配置项 | 作用 |
| --- | --- |
| `experimental.cache_file.enabled` | 启用缓存文件功能 |
| `experimental.cache_file.store_dns` | 将 DNS 缓存保存到缓存文件 |
| `route.rule_set` 中的远程 rule-set | 定义路由使用哪些规则集 |
| `dns.rules[*].rule_set` | 定义 DNS 规则使用哪些规则集 |

本文只处理 `store_dns`，不改变 `route.rule_set` 或 `dns.rules[*].rule_set` 的写法。

## 8. 排查步骤

### 第一步：确认字段路径

检查 `store_dns` 是否位于：

```text
experimental.cache_file.store_dns
```

如果写成 `dns.store_dns`、`route.store_dns` 或顶层 `store_dns`，就不是官方文档中的字段路径。

### 第二步：确认启用了 cache_file

确保同一个 `cache_file` 对象中存在：

```json
{
  "enabled": true,
  "store_dns": true
}
```

不要只看 `store_dns` 是否存在，还要确认 `enabled` 的值确实为布尔值 `true`，而不是字符串：

```json
{
  "enabled": "true",
  "store_dns": true
}
```

上面的写法不是与官方结构一致的布尔配置。

### 第三步：确认版本兼容性

官方文档标注 `store_dns` 自 sing-box 1.14.0 起提供；如果运行环境版本低于该版本，需要核对对应版本的配置文档和客户端实际支持情况。本文不对未提供版本号的客户端或打包版本作兼容性承诺。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

### 第四步：检查缓存文件路径与权限

如果配置了 `path`，确认运行 sing-box 的进程对该路径具有读写权限，并且父目录已经存在。不要把桌面端、移动端和命令行程序的配置目录混为一谈；移动端通常还会受到应用沙盒限制。

### 第五步：重新检查配置并观察实际行为

修改后，使用你当前客户端或发行包提供的配置检查流程确认 JSON 和字段能够被接受。成功判断应以进程无错误退出或客户端明确显示配置加载成功为准，不要根据某一条固定提示文本判断所有平台都成功。

## 常见问题

### 1. `store_dns` 应该放在 `dns` 下面吗？

不应该。官方路径是 `experimental.cache_file.store_dns`。`dns` 对象负责 DNS 服务器、DNS 规则和默认 DNS 服务器等配置；缓存文件选项属于 `experimental.cache_file`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

### 2. 只写 `store_dns: true`，不写 `enabled` 可以吗？

不建议这样配置。官方把 `enabled` 定义为启用缓存文件，把 `store_dns` 定义为保存 DNS 缓存。实际配置应同时写入：

```json
{
  "enabled": true,
  "store_dns": true
}
```

### 3. `store_dns` 能解决域名解析失败吗？

不能把它当作通用的 DNS 故障修复开关。它的作用是保存 DNS 缓存；DNS 服务器不可用、DNS 规则顺序错误、上游网络不可达等问题，仍需分别检查 `dns.servers`、`dns.rules`、`dns.final` 以及网络连接。

### 4. `store_rdrc` 还能继续使用吗？

官方已将 `store_rdrc` 标记为 sing-box 1.14.0 弃用字段，并说明将在 1.16.0 移除。新配置应按照迁移文档改用 `store_dns`。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

### 5. 配置了 `store_dns`，为什么没有看到固定的成功提示？

不同客户端对配置加载、日志输出和缓存文件管理方式可能不同，官方字段文档也没有规定一个跨平台统一的成功提示。应以配置检查流程无错误退出、客户端接受配置，以及实际文件权限和运行日志为准；不要臆测某个固定输出一定会出现。

### 6. `store_dns` 会自动缓存远程 rule-set 吗？

不能这样理解。官方说明远程 rule-set 会在 `experimental.cache_file.enabled` 时缓存，而 `store_dns` 的定义是保存 DNS cache。两者都使用 cache file，但用途不同。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/experimental/cache-file/))

## 参考资料

- [sing-box Cache File 官方文档](https://sing-box.sagernet.org/configuration/experimental/cache-file/)
- [sing-box Experimental 官方文档](https://sing-box.sagernet.org/configuration/experimental/)
- [sing-box Migration 官方文档](https://sing-box.sagernet.org/migration/)
- [sing-box rule-set 官方文档](https://sing-box.sagernet.org/configuration/rule-set/)
- [sing-box DNS 官方文档](https://sing-box.sagernet.org/configuration/dns/)

## 关于 边界云加速

边界云加速提供跨平台网络连接与订阅使用服务，本文仅整理 sing-box 官方配置字段，具体客户端界面和可用配置请以实际版本为准。更多平台说明与使用教程，请访问 [边界云加速官网](https://bianjiecloud.com/)。