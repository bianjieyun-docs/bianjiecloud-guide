> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-21。

# curl --noproxy 怎么用？排查 NO_PROXY 导致的代理绕过

一句话结论：`--noproxy` 用来指定哪些目标不使用 curl 的显式代理；排查 `NO_PROXY` 干扰时，可以用 `--noproxy ""` 覆盖环境中的绕过列表，而 `--noproxy "*"` 的意思恰好相反——所有目标都绕过代理。([github.com](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md))

## 适用场景

本文只解决一个问题：**已经给 curl 指定代理，为什么访问某些域名时仍没有连接该代理？** 官方文档将代理地址设置和代理排除列表分别说明，因此排查时需要同时检查这两部分，不能只看命令里是否存在 `--proxy`。([curl.se](https://curl.se/docs/manpage.html))

你可以在以下情况下使用本文的对照流程：

- 想确认一次请求是否受到 `NO_PROXY` 或 `no_proxy` 的影响。
- 想临时让某个域名绕过代理，但不希望修改长期环境设置。
- 看到命令中的 `--noproxy "*"`，不确定星号是否代表“所有请求使用代理”。
- 想通过连接日志区分“代理没有被选用”和“已经选用代理但连接失败”。

以下命令面向 **macOS、Linux 的 Bash 或 Zsh 终端**。本文不提供 Windows PowerShell 的空字符串传参写法，避免将不同命令行环境混在同一组操作中。

## 分步骤操作

### 第一步：准备真实的本地代理地址

如果你正在通过边界云加速的订阅使用代理客户端，请先启动客户端，再从客户端设置中确认本地 HTTP 代理的监听地址与端口。后续示例使用 `http://127.0.0.1:7890`，这只是示例值，不代表任何客户端的固定默认端口；请替换成自己的实际地址。

在终端中定义本次测试要使用的值：

```bash
PROXY_URL='http://127.0.0.1:7890'
TARGET_URL='https://example.com/'
```

这里的 `PROXY_URL` 应当指向本地 HTTP 代理入口，而不是订阅链接。curl 的 `--proxy` 接收代理地址，并可以通过地址中的协议前缀区分代理类型；本教程统一使用 HTTP 代理，不展开其他协议。([curl.se](https://curl.se/docs/manpage.html))

建议先使用不带账号、令牌或业务参数的测试 URL。整个流程只比较同一目标在不同绕过设置下的连接行为，不需要调用真实业务接口。

### 第二步：确认环境中是否存在绕过列表

先查看当前终端环境：

```bash
printenv NO_PROXY
printenv no_proxy
```

这一步只是读取变量，不修改设置。请记录是否存在星号、目标域名或目标域名所属的上级域。

官方文档说明，`NO_PROXY` 是逗号分隔的主机列表，其中的目标不使用代理；单独一个 `*` 会匹配全部主机。以点开头的域名条目还可以匹配该域下的主机，例如 `.example.com` 可以匹配 `www.example.com`。因此，列表中没有逐字出现完整目标主机名，并不一定意味着它不会被排除。([everything.curl.dev](https://everything.curl.dev/usingcurl/proxies/env.html))

此时先不要删除环境变量。保留现场，再通过命令行覆盖，方便判断究竟是哪项设置影响了这次请求。

### 第三步：记录没有覆盖绕过列表时的行为

运行下面的命令：

```bash
curl -q --verbose --max-time 20 --proxy "$PROXY_URL" "$TARGET_URL"
```

其中，放在首个参数位置的 `-q` 用于禁止读取默认 curl 配置文件，减少隐藏配置的干扰；`--max-time 20` 限制本次传输的最长时间，避免诊断请求一直等待。这两项只是辅助条件，本文要比较的核心仍是 `--noproxy`。([curl.se](https://curl.se/docs/manpage.html))

查看详细日志中尝试连接的地址和端口。`--verbose` 会提供连接过程信息以及发送、接收的协议头信息，适合用来观察 curl 实际做了什么。不要只根据有没有返回网页内容判断是否使用代理。([everything.curl.dev](https://everything.curl.dev/usingcurl/verbose/index.html))

### 第四步：使用空字符串覆盖绕过列表

保持代理地址和目标 URL 不变，只增加 `--noproxy ""`：

```bash
curl -q --verbose --max-time 20 --proxy "$PROXY_URL" --noproxy "" "$TARGET_URL"
```

官方文档明确说明，`--noproxy` 会覆盖 `no_proxy`、`NO_PROXY`；将它设置为空字符串，可以覆盖环境变量设置的代理排除列表。这里的空字符串必须保留，不能把整个参数删掉。([github.com](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md))

对照第三步与第四步的日志，重点检查：

1. 第三步是否绕过了你指定的本地代理。
2. 第四步是否改为尝试连接 `PROXY_URL` 对应的地址与端口。
3. 如果第四步失败，失败发生在连接本地代理之前，还是已经连接代理之后。

**判断建议：**如果只增加这一项，就让连接目标从网站地址变成本地代理地址，那么原有绕过列表是应优先检查的原因。这是根据两次命令差异作出的排查推断，不代表远端网站或代理出口已经通过完整验证。

本文没有在你的设备上执行命令，因此不提供虚构的成功日志，也不承诺命令一定返回某个固定状态码。

### 第五步：用星号建立相反的对照

再运行一次：

```bash
curl -q --verbose --max-time 20 --proxy "$PROXY_URL" --noproxy "*" "$TARGET_URL"
```

这次虽然仍然写着 `--proxy`，但 `--noproxy "*"` 匹配全部目标，会让 curl 绕过该代理。星号不是“全部使用代理”，而是“全部不使用代理”。([github.com](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md))

这里所谓的“绕过”仅指 curl 的代理选择。请勿把这条命令单独当作系统所有网络接管方式都已关闭的证明；本教程的观察范围是 curl 是否连接了你显式指定的代理入口。

### 第六步：按需要改为域名列表

如果你的目的不是诊断，而是只让指定域名绕过代理，可以写成：

```bash
curl -q --verbose --max-time 20 --proxy "$PROXY_URL" --noproxy "example.com,example.org" "$TARGET_URL"
```

列表项之间使用逗号。`--noproxy` 接收的是主机匹配列表，不是完整 URL：不要把 `https://example.com/path%60 这样的带协议、路径地址放进去。官方示例还说明，域名条目可以匹配该域自身及其子域，而不是任意包含同一段文字的无关域名。([github.com](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md))

建议先在单条命令中验证，再决定是否修改长期环境设置。一次只改变一个条件，能让日志差异更容易解释。

## 常见问题

### 1. 已经写了 --proxy，为什么还需要 --noproxy？

两者解决不同问题：`--proxy` 指定代理地址，`--noproxy` 指定哪些目标不使用代理。显式指定代理地址，不等于清除了代理绕过列表。排查环境变量干扰时，应保留前者，并用后者的空字符串形式覆盖排除列表。([curl.se](https://curl.se/docs/manpage.html))

### 2. --noproxy "" 和 --noproxy "*" 可以互换吗？

不能。空字符串表示本次命令不保留环境变量提供的代理排除项；单独一个星号则匹配所有目标，使所有目标绕过 curl 的代理。把两者写反，会直接改变对照实验的含义。([github.com](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md))

### 3. 命令里出现两次 --noproxy，会合并列表吗？

不会按多个参数自动合并。curl 手册说明，多次提供 `--noproxy` 时使用最后一次的值。需要排除多个主机时，请将它们写进同一个逗号分隔的列表，尤其要检查脚本或封装命令是否在末尾追加了另一项。([curl.se](https://curl.se/docs/manpage.html))

### 4. 加上空字符串之后仍然失败，能认定是 NO_PROXY 吗？

不能仅凭“仍然失败”下结论。建议先区分代理是否被选中，再检查连接失败发生在哪个阶段。详细日志能展示连接尝试、TLS 相关过程和协议头，但必须结合你实际收到的错误判断；网页没有打开，不足以单独证明代理绕过列表有问题。([everything.curl.dev](https://everything.curl.dev/usingcurl/verbose/index.html))

### 5. 可以直接把完整 verbose 日志发到公开仓库吗？

建议先脱敏。官方文档说明，详细模式会显示发送和接收的协议头；发布前应检查并移除可能涉及身份认证、Cookie、业务参数或其他私密信息的内容。保留参数结构、连接阶段和错误信息即可，不必公开真实业务请求。([everything.curl.dev](https://everything.curl.dev/usingcurl/verbose/index.html))

## 参考资料

以下四条官方资料已在本次写作中实际打开；同时列出各自对应的关键核验结论，便于维护者复查：

1. [curl 命令行手册](https://curl.se/docs/manpage.html)：核对 `--proxy`、`-q`、`--max-time`，以及多次设置 `--noproxy` 时采用最后一个值的行为。
2. [curl 官方仓库：noproxy 参数文档](https://github.com/curl/curl/blob/master/docs/cmdline-opts/noproxy.md)：核对空字符串覆盖环境排除列表、星号匹配全部目标、逗号列表及域名匹配语义。
3. [everything curl：代理环境变量](https://everything.curl.dev/usingcurl/proxies/env.html)：核对 `NO_PROXY` 的用途、星号及域名排除行为。
4. [everything curl：Verbose](https://everything.curl.dev/usingcurl/verbose/index.html)：核对详细日志所包含的连接过程及协议头信息。

## 关于 边界云加速

边界云加速文档仓库整理客户端使用、订阅导入与常见连接问题的操作说明，帮助读者按步骤排查配置。服务信息与使用入口请访问[边界云加速官网](https://bianjiecloud.com/)。