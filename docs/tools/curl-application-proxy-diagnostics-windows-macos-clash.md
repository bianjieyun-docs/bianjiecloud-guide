> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-10-25。

# curl 应用层代理诊断：Windows PowerShell 与 macOS 如何确认 Clash 是否生效

## 一句话结论

不要用 `ping` 或 `tracert` 判断系统代理是否生效：ICMP 和路由探测通常不会经过 HTTP/SOCKS 代理。更可靠的方法是使用 `curl` 发起真实的 HTTP/HTTPS 请求，分别执行“显式指定代理”和“强制直连”两组测试，再结合 `remote_ip`、`-v` 连接日志、HTTP 状态码和客户端日志，判断问题位于本地端口、代理转发、DNS、TLS，还是目标网站响应。

## 适用场景

本文适合以下情况：

- Clash Verge Rev 已经启动，但浏览器访问网页仍然失败；
- 浏览器可以打开部分网站，某个命令行工具或 API 客户端却连接超时；
- 想确认请求是否真的经过本地代理端口，而不是误用了系统环境变量；
- 需要区分“代理端口没有监听”“代理能连上但远端失败”“目标站点返回错误”这几类问题；
- 在 Windows PowerShell、macOS 或 Linux 上进行应用层对照测试。

本文不把 curl 的耗时当作线路测速结果。`time_connect`、`time_total` 只能帮助定位一次请求的阶段和异常，不能代表长期速度、线路质量、节点排名或服务在线率。

## 第 1 步：确认客户端和本地代理端口

先打开正在使用的代理客户端，确认它已经启动，并找到当前配置中的本地代理端口。Clash Verge Rev 的配置中可能使用 `mixed-port`，也可能分别配置 HTTP 和 SOCKS 端口；实际端口以客户端界面或当前配置为准，不要直接套用其他教程中的固定数字。

如果你导入了边界云加速的订阅，请先在客户端中选择一个代理组或节点，并确认客户端没有处于暂停、停止或仅直连状态。随后把下面示例中的 `7897` 替换成你看到的本地混合端口。例如：

```text
http://127.0.0.1:7897/
```

这里的 `127.0.0.1` 表示本机。命令只应访问本地监听端口，不要把订阅链接、节点密码或带有个人令牌的配置内容发给他人。

如果客户端只提供 SOCKS 端口，应根据实际端口类型改写代理地址，例如：

```text
socks5h://127.0.0.1:7898
```

不要把 SOCKS 端口直接当作 HTTP 代理端口使用。`http://xn--``socks5-fm3g//%60 和 `socks5h://` 的含义不同，具体支持情况以客户端和当前配置为准。

## 第 2 步：先检查 curl 是否可用

### Windows PowerShell

Windows 自带 curl，但 Windows PowerShell 5.1 中，`curl` 可能是 `Invoke-WebRequest` 的别名，不一定是真正的 curl。为避免参数行为不同，建议始终明确调用 `curl.exe`：

```powershell
curl.exe --version
```

如果能看到 curl 版本和支持的协议，说明命令可执行。PowerShell 7 及更高版本通常不再定义这个别名，但使用 `curl.exe` 仍然最不容易混淆。

同时可以查询当前会影响 curl 的代理环境变量：

```powershell
Get-ChildItem Env: | Where-Object { $_.Name -match '^(HTTP|HTTPS|ALL|NO)_PROXY$' } | Format-Table -AutoSize
```

如果结果中存在 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 或 `NO_PROXY`，curl 可能会根据这些变量选择代理或绕过代理。后文使用 `--proxy` 和 `--noproxy` 进行显式控制，可以减少环境变量造成的干扰。

### macOS 或 Linux

在终端执行：

```bash
curl --version
env | grep -iE '^(http|https|all|no)_proxy='
```

没有输出代理环境变量并不代表系统代理没有配置；它只说明当前 shell 环境中没有这些变量。图形化客户端的系统代理设置与命令行环境变量是两套机制，最好使用显式代理参数完成对照。

## 第 3 步：用显式代理发送一次 HTTPS 请求

下面以 `https://example.com/%60 作为测试地址。它的作用是验证一次普通 HTTPS 请求是否能够完成，不代表任何具体网站的可用性。`curl` 的 `--proxy` 或短参数 `-x` 可以指定 HTTP、SOCKS4 或 SOCKS5 代理；代理协议应与客户端端口实际支持的类型一致。

### Windows PowerShell 写法

```powershell
$proxy = "http://127.0.0.1:7897"

curl.exe --proxy $proxy `
  --verbose `
  --connect-timeout 10 `
  --max-time 20 `
  --output NUL `
  --write-out "code=%{http_code} remote_ip=%{remote_ip} proxy_used=%{proxy_used} time_connect=%{time_connect} time_total=%{time_total}`n" `
  https://example.com/
```

PowerShell 的多行续行符是反引号 `` ` ``，不是 Bash 使用的反斜杠。若想避免换行符复制错误，也可以写成单行命令：

```powershell
curl.exe --proxy "http://127.0.0.1:7897" --verbose --connect-timeout 10 --max-time 20 --output NUL --write-out "code=%{http_code} remote_ip=%{remote_ip} proxy_used=%{proxy_used} time_connect=%{time_connect} time_total=%{time_total}`n" https://example.com/
```

### macOS 或 Linux 写法

```bash
proxy='http://127.0.0.1:7897'

curl --proxy "$proxy" \
  --verbose \
  --connect-timeout 10 \
  --max-time 20 \
  --output /dev/null \
  --write-out "code=%{http_code} remote_ip=%{remote_ip} proxy_used=%{proxy_used} time_connect=%{time_connect} time_total=%{time_total}\n" \
  https://example.com/
```

这里使用 `/dev/null` 是 macOS/Linux 的写法；Windows PowerShell 应使用 `NUL`。命令中的 `--output` 用于丢弃网页正文，`--write-out` 只输出请求摘要，`--verbose` 则把连接建立、TLS 握手和代理交互过程写到终端。

需要注意，`proxy_used` 变量并不是所有旧版本 curl 都支持。如果当前 curl 报告该变量不可用，可以删除 `proxy_used=%{proxy_used}`，保留 `remote_ip` 和其他字段。

## 第 4 步：用强制直连做对照

为了判断上一条命令是否确实使用了指定的本地代理，可以执行一条强制不使用代理的请求。`--noproxy "*"` 表示所有主机都不使用代理：

### Windows PowerShell

```powershell
curl.exe --noproxy "*" `
  --verbose `
  --connect-timeout 10 `
  --max-time 20 `
  --output NUL `
  --write-out "code=%{http_code} remote_ip=%{remote_ip} time_connect=%{time_connect} time_total=%{time_total}`n" `
  https://example.com/
```

### macOS 或 Linux

```bash
curl --noproxy '*' \
  --verbose \
  --connect-timeout 10 \
  --max-time 20 \
  --output /dev/null \
  --write-out "code=%{http_code} remote_ip=%{remote_ip} time_connect=%{time_connect} time_total=%{time_total}\n" \
  https://example.com/
```

### 如何理解 `remote_ip`

`%{remote_ip}` 表示 curl 最近一次建立连接的远端 IP。它不是固定意义上的“网站出口 IP”，必须结合当前命令是否使用代理来解释：

- **显式代理测试中**：如果 `--proxy http://127.0.0.1:7897` 成功，curl 的 TCP 连接对象通常是本机代理监听地址，因此 `remote_ip` 通常显示为 `127.0.0.1`，或显示本机回环地址的 IPv6 形式 `::1`。此时 curl 连接的是本地代理，而不是直接连接 `example.com` 的公网地址。
- **强制直连测试中**：使用 `--noproxy "*"` 后，`remote_ip` 通常显示目标域名解析得到的公网 IPv4 或 IPv6 地址，也可能是 CDN、负载均衡或目标站点的其他解析地址。

因此，在本地 HTTP 代理场景下，显式代理测试显示 `remote_ip=127.0.0.1`，而直连测试显示目标站点的公网地址，本身就是判断连接入口是否不同的直接依据。但它不能单独证明代理出口位于某个国家或地区，也不能代表代理服务器最终访问目标站点时使用的公网地址。

对照时重点看四项：

1. **`remote_ip` 是否发生预期变化**：显式本地代理通常是 `127.0.0.1` 或 `::1`；直连通常是目标域名解析得到的地址。
2. **详细日志中的连接对象**：显式代理测试应出现连接到 `127.0.0.1:本地端口` 的过程；直连测试则不会以该本地代理作为连接入口。
3. **命令是否在超时时间内完成**：如果显式代理立即提示连接被拒绝，优先检查端口、客户端状态和端口占用；如果本地端口连接成功但后续超时，问题可能在代理转发、DNS、TLS 或远端响应。
4. **HTTP 状态码**：`2xx` 通常表示请求成功，`3xx` 表示重定向，`4xx` 或 `5xx` 是服务器或中间层返回的错误。状态码本身不能单独证明“节点好”或“节点差”，需要结合 `-v` 日志判断错误出现在哪一段。

如果代理测试和直连测试都成功，说明 curl 的基本网络能力正常，浏览器问题可能来自浏览器自己的代理设置、扩展、证书缓存或 DNS 缓存。如果只有显式代理测试成功，说明本地直连路径不可用，但这不等于所有应用都会自动使用代理；需要检查目标应用是否读取系统代理、环境变量，或是否需要单独填写代理地址。

## 第 5 步：保存 `-v` 日志并定位失败阶段

可以把 curl 的详细日志保存到文件，便于查看连接入口和错误行。curl 官方提供了 `--stderr` 参数，可将 curl 写入标准错误输出的内容直接保存到指定文件。

### Windows PowerShell

```powershell
curl.exe --proxy "http://127.0.0.1:7897" --verbose --connect-timeout 10 --max-time 20 --output NUL --stderr .\curl-proxy.log https://example.com/
Get-Content .\curl-proxy.log
```

在 Windows PowerShell 5.1 中，不建议使用下面这种方式保存 curl 的详细日志：

```powershell
curl.exe --proxy "http://127.0.0.1:7897" --verbose --output NUL https://example.com/ 2> .\curl-proxy.log
```

PowerShell 5.1 对外部程序的 stderr 重定向可能把内容包装成 `ErrorRecord`，并出现 `NativeCommandError` 等 PowerShell 格式；同时，使用 PowerShell 重定向写文件时还可能受到 Windows PowerShell 编码规则影响。这样得到的日志不一定保持 curl 原始输出的形态，不方便直接复制分析。使用 curl 自己的 `--stderr .\curl-proxy.log` 可以绕开这层 PowerShell 错误流包装。

如果使用的是 PowerShell 7，外部程序 stderr 的处理方式有所改善，但为了让命令在不同 PowerShell 版本中保持一致，仍建议优先使用 `--stderr`。

### macOS/Linux

```bash
curl --proxy 'http://127.0.0.1:7897' \
  --verbose \
  --connect-timeout 10 \
  --max-time 20 \
  --output /dev/null \
  --stderr curl-proxy.log \
  https://example.com/
cat curl-proxy.log
```

可按下面方式初步分类：

| 现象 | 优先检查方向 | 下一步 |
| --- | --- | --- |
| `Connection refused`，连接本地端口被拒绝 | 客户端未启动、端口填错、端口被占用 | 回到客户端查看实际 HTTP/SOCKS/mixed 端口 |
| `remote_ip` 为 `127.0.0.1`，但随后超时 | 本地代理已接收请求，后续代理转发或远端连接异常 | 查看客户端日志、代理组选择和规则命中情况 |
| 直连测试的 `remote_ip` 为公网地址 | curl 正在直接连接目标域名解析地址 | 这是直连对照的正常表现，不代表代理测试失败 |
| 出现证书验证错误 | 系统时间、证书存储、HTTPS 检查或代理链路 | 不要直接长期使用 `-k`，先确认系统时间和证书环境 |
| 返回 `407 Proxy Authentication Required` | 代理要求认证但命令未提供凭据 | 核对端口类型和认证方式，不要把密码写入公开脚本 |
| 返回 `403`、`404` 或 `5xx` | 目标服务器或中间层响应 | 结合响应头和目标地址判断，不要把 HTTP 错误直接归因于本地端口 |

`-k` 或 `--insecure` 可以绕过证书验证，但不建议把它作为常规修复方式。它会降低 HTTPS 连接的身份验证保障；若只是为了确认“是否能建立连接”，也应在测试后恢复正常证书校验。

## 第 6 步：检查客户端日志，而不是用 ping 判断代理线路

当 curl 已经确认本地端口可连接，但请求仍然失败时，应查看客户端的运行日志。日志通常比 ICMP 延迟更能说明问题，例如是否命中了某条规则、使用了哪个代理组、域名解析是否失败、远端连接是否被关闭。

需要注意：Clash Verge Rev 项目说明中，核心日志可能包含请求域名、IP 地址和连接元数据；提交日志前应删除订阅链接、令牌、服务器地址、UUID、密码以及不希望公开的访问记录。不要为了排查问题直接上传完整的 `profiles`、配置文件或备份压缩包。

如果使用的是 TUN 模式，ICMP 可能由本机网络栈或 TUN 内核直接处理，结果并不等于浏览器 HTTPS 请求的代理路径。因此，本文的 curl 显式代理测试、直连对照和客户端日志，应作为主要排查依据。

## 常见问题

### 1. PowerShell 输入 `curl -x` 后提示参数无法识别，怎么办？

在 Windows PowerShell 5.1 中，`curl` 可能映射到 `Invoke-WebRequest`，它不接受 curl 的全部参数。改用 `curl.exe` 即可，例如：

```powershell
curl.exe --proxy "http://127.0.0.1:7897" https://example.com/
```

不建议只修改命令参数来适配别名，因为这样容易把 PowerShell 命令和 curl 命令混在一起。

### 2. 显式代理测试失败，但浏览器可以打开网页，是否说明客户端坏了？

不一定。浏览器可能使用了系统代理、浏览器扩展自己的代理，或者缓存了连接；curl 的显式代理测试则直接访问你填写的本地端口。请先确认端口类型和数字，尤其不要把 SOCKS 端口当作 HTTP 端口使用。再用 `-v` 判断是本地连接失败，还是本地端口连接成功后远端失败。

### 3. 显式代理测试中的 `remote_ip` 为什么是 `127.0.0.1`？

这是本地 HTTP 代理正常工作的常见表现。curl 使用 `--proxy http://127.0.0.1:7897` 时，先与本机代理端口建立 TCP 连接，因此 `%{remote_ip}` 通常是 `127.0.0.1`，或本机回环地址的 IPv6 形式 `::1`。这表示 curl 的连接入口是本地代理，并不表示目标网站的公网地址就是 `127.0.0.1`。

相反，使用 `--noproxy "*"` 强制直连时，`remote_ip` 通常会显示目标域名解析得到的公网 IPv4 或 IPv6 地址。两组命令中的 `remote_ip` 出现这种差异，可以作为判断请求是否经过本地代理的直接依据。

但无论哪种测试，`remote_ip` 都不能直接当作“代理出口 IP”。显式代理测试中它反映的是 curl 连接到的代理地址；直连测试中它反映的是 curl 连接到的目标地址。若要确认对外显示的公网地址，应使用你信任的 HTTPS 检测页面或服务，并结合客户端日志判断。

### 4. 使用 `--noproxy "*"` 仍然能访问目标网站，是否代表代理没有生效？

不代表。它只说明当前网络的直连路径能够访问这个测试地址。要判断代理是否实际参与，应比较 `--proxy` 测试中的 `remote_ip`、详细连接日志、客户端日志和直连对照结果。某个公开测试站点直连成功，也不能推断其他目标网站都能直连。

### 5. Windows PowerShell 中用 `2> curl-proxy.log` 保存日志，为什么文件里有 `NativeCommandError`？

这通常与 Windows PowerShell 5.1 对外部程序 stderr 的处理方式有关。curl 的 `--verbose` 内容写入 stderr，而 PowerShell 5.1 可能把这些内容包装为错误记录，文件中就会出现 PowerShell 的错误格式；此外，PowerShell 5.1 的文件输出编码也可能与 curl 原始日志不同。

建议直接使用 curl 的 `--stderr` 参数：

```powershell
curl.exe --proxy "http://127.0.0.1:7897" --verbose --output NUL --stderr .\curl-proxy.log https://example.com/
```

这样由 curl 自己打开日志文件，不经过 PowerShell 的 stderr 重定向格式化。分享日志前仍要检查其中是否含有域名、IP、认证信息或其他敏感内容。

### 6. curl 显式代理可以访问，某个 API 程序仍然失败，下一步查什么？

先确认该程序是否支持 HTTP、HTTPS 或 SOCKS 代理，以及它读取的是环境变量、系统代理还是独立配置。再确认程序是否使用自己的 DNS、是否要求特殊 TLS 参数、是否有证书校验策略。curl 成功只能证明一条具体 HTTPS 请求成功，不能替代目标程序自身的协议测试。

## 参考资料

- curl 官方命令手册：<https://curl.se/docs/manpage.html%3E
- curl 官方使用教程：<https://curl.se/docs/tutorial.html%3E
- curl FAQ：<https://curl.se/docs/faq.html%3E
- Microsoft Learn：Windows 中的 curl：<https://learn.microsoft.com/en-us/windows/curl/%3E
- Microsoft Learn：PowerShell 重定向：<https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_redirection?view=powershell-5.1%3E
- Microsoft Learn：PowerShell 字符编码：<https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding?view=powershell-7%3E
- Microsoft Learn：Invoke-WebRequest：<https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-7.5%3E
- Clash Verge Rev 配置说明：<https://github.com/clash-verge-rev/clash-verge-rev.github.io/blob/main/docs/guide/config.md%3E
- Clash Verge Rev 隐私与日志说明：<https://github.com/clash-verge-rev/clash-verge-rev/blob/dev/PRIVACY.md%3E

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供跨境网络访问相关的订阅与使用指引，支持结合常见客户端进行配置和排查。使用相关服务时，请遵守当地法律法规，并妥善保护订阅链接、账号凭据和配置文件。