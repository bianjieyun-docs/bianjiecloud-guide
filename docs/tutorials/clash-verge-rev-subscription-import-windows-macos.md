> 本文整理自 [边界云加速 官网《Clash Verge Rev 怎么导入订阅：Windows / macOS 使用教程 · 边界云加速》](https://www.bianjiecloud.com/help/clash-verge-rev)，更新于 2026-10-20。

# Clash Verge Rev 订阅导入与系统代理配置：Windows / macOS 完整步骤

## 一句话结论

在 Windows 或 macOS 上使用 Clash Verge Rev，通常只需要完成四件事：安装对应系统版本、从服务商复制 `clash_meta` 订阅链接、在“订阅”页面导入并更新配置，最后开启“系统代理”；如果仍有应用不走代理，再根据软件是否支持系统代理决定是否使用 TUN 模式。

## 适用场景

本文适合以下情况：

- 第一次在 Windows 或 macOS 上使用 Clash Verge Rev；
- 已经拥有订阅链接，但不知道应该选择哪一种订阅格式；
- 导入订阅后看不到节点，或导入时提示连接失败；
- 浏览器可以使用，但某些桌面软件仍然无法连接；
- 想让订阅每天自动更新，减少手动维护配置的次数。

Clash Verge Rev 是一个运行在 Windows、macOS 和 Linux 上的图形客户端，项目基于 Tauri，并集成 Clash.Meta（mihomo）内核。官方仓库同时提供 Windows、Linux 和 macOS 安装包；macOS 支持 Intel 与 Apple 芯片版本。不同版本的菜单名称和布局可能略有差异，本文按照常见的 2.x 界面说明，截图或按钮位置变化时以实际版本为准。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev))

> 注意：订阅链接通常相当于账号的使用凭证。不要把链接发到群聊、工单截图、公开仓库或社交平台；如果怀疑泄露，应尽快在服务商后台重置或重新生成订阅链接。

## 步骤一：下载并安装 Clash Verge Rev

建议从项目的 GitHub Releases 页面下载，不要从搜索结果中的不明下载站获取安装包。官方仓库把 Stable 标记为适合日常使用的正式版本，同时还提供 AutoBuild 等其他构建渠道；普通用户优先选择 Stable 即可。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev))

### Windows

1. 打开 Clash Verge Rev 官方 GitHub 仓库的 Releases 页面。
2. Windows 电脑一般选择带有 `x64-setup.exe` 的安装包；如果设备是 ARM 架构，应根据发布页提供的架构选择对应文件。
3. 双击安装包完成安装。首次启用系统代理或 TUN 相关功能时，Windows 可能会弹出管理员权限提示。
4. 安装后启动 Clash Verge Rev，并确认左侧能够看到“订阅”“代理”“首页”等功能页面。

### macOS

1. Apple 芯片设备选择 `aarch64.dmg` 或发布页中标注为 Apple Silicon、arm64 的版本。
2. Intel 芯片设备选择 `x64.dmg` 或标注为 Intel、x86_64 的版本。
3. 将应用拖入“应用程序”文件夹后启动。如果 macOS 阻止首次打开，可以在“系统设置 → 隐私与安全性”中允许已下载的应用继续打开。
4. 如果后续启用系统代理或虚拟网卡功能，按照系统提示授予必要权限。

官方仓库说明 Clash Verge Rev 支持 Windows、Linux，以及 macOS 11 及以上系统；实际安装前仍建议查看当前发布页的系统要求。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev))

## 步骤二：复制 clash_meta 订阅链接

以边界云加速为例，登录用户中心后进入套餐详情，在“我的订阅”或类似位置找到“订阅链接”。格式选择 `clash_meta`，然后点击复制。

复制完成后，不要直接把链接粘贴到公共笔记、在线转换站或公开代码仓库。订阅链接中可能包含用于识别账号或套餐的访问凭据，保存时可以使用本地密码管理器，并避免在命令行中留下长期可见的历史记录。

如果服务商提供多种格式，不要把 `singbox`、`v2ray` 或其他格式的链接直接导入 Clash Verge Rev。Clash Verge Rev 的远程订阅导入应使用与 Clash/Mihomo 兼容的订阅格式；本文以服务商页面中的 `clash_meta` 选项为准。

## 步骤三：在 Clash Verge Rev 中导入订阅

1. 打开 Clash Verge Rev，点击左侧的“订阅”或“Profiles”。
2. 在页面顶部的输入框粘贴刚刚复制的订阅链接。
3. 点击“导入”。
4. 等待客户端完成请求和解析。成功后，订阅列表中应出现一张新的配置卡片。
5. 点击这张卡片，确认它处于选中或启用状态；如果界面提供“更新”按钮，可以手动执行一次更新。

官方文档将远程订阅导入分为订阅链接导入、订阅链接配置和 URL Schemes 等方式。通过普通订阅链接导入时，如果出现 `client error(Connect)`，官方文档建议尝试其他导入方式；使用订阅链接配置方式时，也可能需要检查证书相关选项。([clashverge.dev](https://www.clashverge.dev/guide/profile.html))

### 可选：用命令检查订阅地址是否能返回内容

这一步只用于排查“链接本身是否能访问”，不能代替 Clash Verge Rev 的配置解析。请把示例中的订阅地址替换为自己的地址，并注意不要把包含真实订阅链接的命令复制到公开日志中。

#### Windows PowerShell

```powershell
$url = "https://example.com/your-clash-meta-subscription%22
$out = Join-Path $env:TEMP "clash-subscription.yaml"
Invoke-WebRequest -Uri $url -Method Get -MaximumRedirection 5 -TimeoutSec 20 -OutFile $out
Get-Content $out -TotalCount 12
Remove-Item $out -Force
```

如果命令能够完成下载，且输出看起来像 YAML 配置或服务商返回的订阅内容，说明地址至少可以从当前网络请求到。若返回 HTML 登录页、错误页或直接超时，优先检查链接是否复制完整、是否已经失效，以及当前网络能否打开订阅域名。

#### macOS / Linux

```bash
url='https://example.com/your-clash-meta-subscription'
out="${TMPDIR:-/tmp}/clash-subscription.yaml"
curl --location --fail --max-time 20 --output "$out" "$url"
sed -n '1,12p' "$out"
rm -f "$out"
```

macOS 和 Linux 使用 `curl` 时，建议用双引号包裹变量，用 `--output` 指定临时文件；不要把真实订阅链接写进会提交到 Git 的脚本或配置文件。Windows PowerShell 与 macOS/Linux 的重定向和空设备路径写法不同，不要直接混用命令。

## 步骤四：修改名称并设置自动更新

导入成功后，右键订阅卡片，选择“编辑信息”或同等菜单项：

1. 将名称改成容易识别的名称，例如“边界云”；
2. 将更新间隔设置为 `1440` 分钟，表示每天更新一次；
3. 点击“保存”；
4. 返回订阅列表，确认名称和更新时间已经生效。

自动更新只负责重新获取订阅内容，不代表客户端一定会自动切换到新的节点或修改你的代理模式。遇到配置变化后，仍应检查当前配置是否处于启用状态，以及代理组是否选择了可用的线路。

## 步骤五：选择代理组和节点

进入左侧“代理”页面，确认当前模式为“规则”或 Rule。找到名为 `PROXY`、`Proxy` 或服务商自定义名称的代理组，再从组内选择一个节点。

如果客户端显示节点延迟或可用性图标，可以把它作为当前客户端的连接参考，但不要把一次显示结果理解为长期线路质量，也不要使用 ICMP 的 `ping` 或 `tracert` 来判断系统代理线路：普通系统代理通常不接管 ICMP 流量，TUN 模式也可能在本地处理 ICMP，因此这类结果不能代表浏览器或应用通过代理访问目标网站的实际表现。

更稳妥的验收方式是：

1. 先在 Clash Verge Rev 的“代理”页面确认代理组已选中节点；
2. 在“连接”或 Connections 页面观察实际应用请求是否出现；
3. 用浏览器打开一个你平时需要访问的 HTTPS 网站；
4. 如果只想验证代理是否生效，可以查看浏览器访问结果或应用连接日志，而不是根据本地 `ping` 延迟下结论。

## 步骤六：开启系统代理

回到“首页”或主界面，按以下顺序检查：

1. 当前配置是刚刚导入并启用的订阅；
2. 代理模式选择“规则”；
3. 代理组已经选择节点；
4. 打开“系统代理”开关；
5. 用浏览器访问目标网站，并回到“连接”页面确认有请求记录。

在此步骤中，边界云加速的订阅只负责向客户端提供配置和节点信息，实际的分流行为仍由 Clash Verge Rev、内置内核以及当前规则共同决定。部分软件不读取 Windows 或 macOS 的系统代理设置，因此即使浏览器已经正常工作，其他软件仍可能保持直连。

## 部分软件不走代理怎么办？

先确认软件是否支持系统代理。浏览器、部分开发工具和常见桌面应用通常会读取系统代理；某些游戏、命令行程序、虚拟机、容器或自带网络栈的软件可能不会。

排查顺序建议如下：

1. 确认系统代理开关仍处于开启状态；
2. 确认 Clash Verge Rev 的代理组没有回到“直连”；
3. 在“连接”页面观察该软件是否产生请求；
4. 检查软件自身是否有 HTTP、HTTPS 或 SOCKS5 代理设置；
5. 最后再考虑启用 TUN 模式。

Clash Verge Rev 官方仓库列出了系统代理、系统代理守卫以及 TUN（虚拟网卡）模式等功能。TUN 需要更高权限，并可能改变本机更多流量的处理方式，因此不建议在没有明确需求时一开始就启用。([github.com](https://github.com/Clash-Verge-rev/clash-verge-rev))

## 常见问题

### 1. 导入时提示 `client error(Connect)`，怎么处理？

先确认订阅链接没有被截断，并重新从服务商后台选择 `clash_meta` 格式后复制。然后检查当前网络是否能打开订阅域名。如果普通链接导入仍失败，可以尝试 Clash Verge Rev 文档提供的其他远程导入方式；对于“订阅链接配置”方式，官方文档还提示可以检查“允许无效证书（危险）”选项，但只有在确认订阅地址可信、且理解安全风险时才考虑使用。([clashverge.dev](https://www.clashverge.dev/guide/profile.html))

### 2. 导入成功，但订阅卡片里没有节点怎么办？

先手动点击更新，并确认导入的是 `clash_meta` 格式而不是 sing-box 或其他客户端格式。若命令行下载结果是 HTML、登录页或错误提示，说明服务商没有返回有效的 Clash 配置；若能下载但客户端仍无法解析，建议查看 Clash Verge Rev 日志，并确认客户端版本与订阅格式兼容。

### 3. 开启系统代理后浏览器仍然直连怎么办？

检查系统代理开关、当前配置、代理模式和代理组是否都正确。随后关闭浏览器后重新打开，避免浏览器保留旧连接。如果 Clash Verge Rev 的“连接”页面完全没有浏览器请求，说明浏览器可能没有使用系统代理，或者系统代理设置未成功写入；可以检查系统网络代理设置，或在浏览器自身设置中确认是否启用了代理。

### 4. 某个软件始终不走代理，需要直接开启 TUN 吗？

不一定。先查看软件是否支持 HTTP、HTTPS 或 SOCKS5 代理；如果软件完全不读取系统代理，再考虑 TUN。启用 TUN 前应保存当前配置，按客户端提示安装服务并授予管理员权限；测试完成后，如果不再需要接管更多流量，可以关闭 TUN，回到系统代理模式。

### 5. 自动更新设置为 1440 分钟后，为什么没有立即更新？

更新间隔表示下一次自动更新的时间，不一定会在保存后立刻执行。需要立即获取新内容时，可以在订阅卡片菜单中手动点击更新。更新完成后检查更新时间、节点列表和当前代理组，避免只看到“更新成功”提示却仍在使用旧的配置。

## 参考资料

- [Clash Verge Rev 官方 GitHub 仓库](https://github.com/Clash-Verge-rev/clash-verge-rev)
- [Clash Verge Rev 官方 Releases 页面](https://github.com/Clash-Verge-rev/clash-verge-rev/releases)
- [Clash Verge Rev 官方文档：订阅导入](https://www.clashverge.dev/guide/profile.html)
- [边界云加速官网原文：Clash Verge Rev 怎么导入订阅](https://www.bianjiecloud.com/help/clash-verge-rev)

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供面向 Windows、macOS、Android 和 iOS 等设备的网络接入与客户端使用说明。本文仅介绍 Clash Verge Rev 的安装、订阅导入和基础代理设置，具体服务内容以官网页面为准；使用网络服务时请遵守当地法律法规。