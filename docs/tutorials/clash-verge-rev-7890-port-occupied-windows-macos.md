> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-10-27。

# Clash Verge Rev 7890 端口被占用怎么办？Windows/macOS 排查教程

## 一句话结论

Clash Verge Rev 出现“端口被占用”“核心启动失败”或系统代理突然失效时，应先确认客户端当前实际使用的代理端口，再分别检查 TCP 和 UDP 是否已有进程占用，最后选择安全退出进程或修改端口；不要把 `external-controller` 控制端口误当成浏览器代理端口。

## 适用场景

本文适用于以下情况：

- Clash Verge Rev 启动后提示 `Port is already in use`、`bind failed` 或类似端口监听错误；
- 客户端界面显示核心没有正常运行，系统代理开关无法生效；
- 更新、崩溃或强制退出客户端后，再次启动时提示端口冲突；
- Windows 或 macOS 上同时使用多个代理客户端，怀疑它们监听了相同端口；
- 客户端看起来已经启动，但浏览器、终端或开发工具仍然连接到旧端口；
- 修改端口后，浏览器插件、Git、终端环境变量或其他设备仍然使用原来的端口；
- macOS 上 TCP 检查没有结果，但核心仍然提示端口无法绑定，需要进一步确认 UDP 占用。

Clash Verge Rev 是一个适用于 Windows、macOS 和 Linux 的图形客户端，并使用 mihomo 相关核心能力。其实际端口由当前配置和客户端设置决定，不能只依据网上常见的 `7890` 示例判断。排查时应以客户端当前显示的端口、配置文件和日志为准。[Clash Verge Rev 官方仓库](https://github.com/clash-verge-rev/clash-verge-rev)

## 第一步：先确认报错对应的实际端口

不要看到“7890 被占用”就直接结束进程。一个配置中可能同时存在多个入站端口，它们用途不同：

| 配置项 | 主要用途 | 是否用于浏览器代理 |
|---|---|---|
| `port` | HTTP 代理端口 | 是，适用于支持 HTTP 代理的程序 |
| `socks-port` | SOCKS 代理端口 | 是，适用于支持 SOCKS 代理的程序 |
| `mixed-port` | 同时接收 HTTP(S) 与 SOCKS5 请求的代理端口 | 是，适用于支持 HTTP 或 SOCKS5 代理的程序 |
| `external-controller` | 管理核心状态的控制接口 | 否，不是普通浏览器代理端口 |

mihomo 官方文档将 HTTP、SOCKS 和 Mixed 端口分开说明，其中 `mixed-port` 用于同时接收 HTTP(S) 与 SOCKS5 请求。本文将 `socks-port` 按 SOCKS 代理端口处理，不把它扩展解释为 SOCKS4 支持；如果你的客户端或核心有特殊实现，应以对应版本的官方文档为准。[mihomo Proxy Ports](https://wiki.metacubex.one/en/config/inbound/port/)

打开 Clash Verge Rev 的设置或端口配置页面，记录以下信息：

1. 当前启用的是 `mixed-port`、`port` 还是 `socks-port`；
2. 报错日志中的端口号，例如 `7890`、`7897` 或其他自定义端口；
3. 系统代理当前填写的地址和端口；
4. 是否开启了后台服务、开机启动或其他代理客户端；
5. 端口冲突是否发生在 TCP、UDP，或两者都可能涉及。

如果你使用边界云加速提供的订阅，请先在客户端中确认订阅已经成功加载，再检查本地端口。订阅内容和本地端口设置属于两个不同层次的问题：端口冲突通常发生在本机监听阶段，并不等同于远端节点连接失败。

## 第二步：Windows 查找占用端口的进程

### 1. 使用 netstat 查看监听状态和 PID

打开 PowerShell 或命令提示符，将 `7890` 替换为客户端实际报错的端口：

```powershell
netstat -ano | findstr :7890
```

Windows 官方文档说明，`netstat` 可以显示活动连接和监听端口；`-n` 让地址和端口以数字形式显示，`-o` 会附带进程 ID，也就是 PID。典型输出可能类似下面这样：

```text
TCP    0.0.0.0:7890    0.0.0.0:0    LISTENING    12345
```

这里最后一列 `12345` 是 PID，不是端口号。`LISTENING` 表示该进程正在监听本地 TCP 端口。[Microsoft Learn：netstat 命令](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)

如果希望让 `netstat` 尝试显示可执行文件，可以使用：

```powershell
netstat -anobq | findstr :7890
```

其中 `-b` 可能需要足够的权限，并且执行时间可能较长。若输出过多，先使用 `netstat -ano` 获取 PID，再通过任务管理器或 `tasklist` 查询进程名称，通常更容易判断。

### 2. 根据 PID 查询进程

假设上一步得到的 PID 是 `12345`，执行：

```powershell
tasklist /FI "PID eq 12345"
```

也可以打开任务管理器，在“详细信息”页查看对应 PID。重点区分以下情况：

- 另一个代理客户端正在使用该端口；
- Clash Verge Rev 的旧核心进程没有退出；
- 某个开发工具、虚拟机或本地服务占用了端口；
- 端口属于系统服务或安全软件，不能贸然结束。

### 3. 只结束已经确认的进程

确认进程确实属于旧的代理核心，或确认目标程序可以安全关闭后，可以根据 PID 结束进程：

```powershell
taskkill /PID 12345 /T
```

如果正常结束失败，再考虑强制结束：

```powershell
taskkill /PID 12345 /T /F
```

Microsoft 文档说明，`taskkill` 可以按 PID 或镜像名称结束任务，`/T` 会连同目标进程启动的子进程一起结束，`/F` 表示强制终止。不要对来源不明的系统进程直接使用 `/F`；无法确认进程归属时，优先关闭相关应用或重启电脑。[Microsoft Learn：taskkill 命令](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/taskkill)

结束后重新执行：

```powershell
netstat -ano | findstr :7890
```

如果没有输出，说明当前没有进程显示为 TCP 监听该端口。此时重新启动 Clash Verge Rev，并观察它是否能够继续使用原端口。若客户端明确报 UDP 绑定错误，应结合客户端日志和 Windows 的其他网络诊断方式继续确认，不要仅凭 TCP 的 `LISTENING` 结果下结论。

## 第三步：macOS 同时检查 TCP 和 UDP

在 macOS 中打开“终端”，将 `7890` 替换为实际端口。先执行完整检查：

```bash
lsof -nP -i :7890
```

`lsof` 的 `-i` 网络文件筛选器可以按端口筛选网络套接字；只写 `:7890` 时，可用于查看该端口相关的网络文件，而不是只限制为 TCP。这样适合先判断同一端口是否存在 TCP、UDP 或其他网络套接字占用。[lsof 手册](https://man7.org/linux/man-pages/man8/lsof.8.html)

如果需要分别确认 TCP 和 UDP，可以执行：

```bash
lsof -nP -iTCP:7890
```

```bash
lsof -nP -iUDP:7890
```

如果只想查看 TCP 监听项，可以使用：

```bash
lsof -nP -iTCP:7890 -sTCP:LISTEN
```

需要注意，不能把“没有 TCP `LISTEN` 输出”理解为“7890 端口完全没有被占用”。UDP 不使用 TCP 的 `LISTEN` 状态，在 macOS 上应通过单独的 `-iUDP:7890` 查询，或者直接使用前面的完整命令 `lsof -nP -i :7890`。Apple Darwin 的 UDP 状态信息也可能不同于 TCP，因此排查 UDP 时应重点查看进程和端口，而不是等待 `LISTEN` 字样。[lsof 手册](https://man7.org/linux/man-pages/man8/lsof.8.html)

常见输出可能类似下面这样：

```text
COMMAND   PID   USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
mihomo   12345  user   12u  IPv4 0x...      0t0  TCP 127.0.0.1:7890 (LISTEN)
```

需要重点查看两列：

- `COMMAND`：进程名称；
- `PID`：进程编号。

如果 UDP 命令有输出，可能不会出现 `(LISTEN)`，但仍应记录 `COMMAND`、`PID` 和 `NAME` 等信息。确认 PID 后，先尝试正常结束：

```bash
kill 12345
```

如果进程没有退出，再在确认它属于旧代理核心或可安全关闭的程序后使用：

```bash
kill -9 12345
```

`kill -9` 会跳过进程的正常清理流程，可能导致应用来不及保存状态，因此不建议把它作为第一选择。对于会被后台服务、登录项或开机启动重新拉起的进程，单纯执行一次 `kill` 可能不能彻底解决问题；这时应先退出相关客户端，再重新执行 TCP 和 UDP 检查。

## 第四步：区分代理端口和控制端口

配置中可能同时出现代理端口和控制端口。例如：

```yaml
mixed-port: 7890
external-controller: 127.0.0.1:9090
```

在这个示例中：

- 浏览器或支持代理的终端工具通常应连接 `127.0.0.1:7890`；
- 面板或管理脚本如果需要访问控制接口，才连接 `127.0.0.1:9090`；
- 不能把 `127.0.0.1:9090` 填到浏览器的 HTTP 或 SOCKS 代理设置中；
- 控制器端口能够访问，也不能证明普通代理流量一定可用。

mihomo 的入站配置包含 `port`、`socks-port` 和 `mixed-port` 等代理入口；控制接口属于管理用途，不应与普通代理入站端口混淆。[mihomo Inbound Configuration](https://wiki.metacubex.one/en/config/inbound/)

如果客户端界面显示核心正在运行，但浏览器仍无法访问，应分别检查系统代理填写的端口和当前配置中的实际代理端口。如果只有某个浏览器插件无法使用，还要检查插件是否手动写死了旧端口。

## 第五步：修改端口后检查关联配置

如果原端口确实无法释放，可以在 Clash Verge Rev 的端口设置中改为一个未被占用的端口，例如将 `7890` 改为 `7898`。保存后重新检查 TCP 和 UDP 状态。

Windows：

```powershell
netstat -ano | findstr :7898
```

macOS：

```bash
lsof -nP -i :7898
```

修改端口后，逐项检查以下位置：

1. **系统代理**：确认地址仍为 `127.0.0.1`，端口已经变更为新端口；
2. **浏览器扩展**：代理切换插件可能保存了旧端口；
3. **终端环境变量**：检查 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 等变量是否仍指向旧端口；
4. **Git 或包管理器**：如果曾手动配置代理，需要同步更新；
5. **局域网设备**：如果启用了局域网共享，其他设备连接时也必须使用新的端口；
6. **脚本和开发工具**：项目配置文件中可能仍然写着旧的 `127.0.0.1:7890`；
7. **UDP 相关功能**：如果客户端或配置使用了 UDP 入站，还要确认新端口没有被其他 UDP 套接字占用。

只修改客户端端口而不修改这些手动配置，可能出现“浏览器正常、Git 失败”或“系统代理正常、某个插件无法连接”的局部故障。

## 常见问题

### 1. 7890 一定是 Clash Verge Rev 的默认端口吗？

不一定。端口可能受到客户端版本、配置文件和用户设置影响，也可能在不同安装环境中使用其他数值。排查时应以客户端设置页、当前配置和日志中的实际端口为准，不要只根据网上示例操作。

### 2. macOS 用 `lsof -iTCP` 没有结果，为什么仍然提示端口冲突？

因为 `-iTCP` 只筛选 TCP 网络套接字。如果报错涉及 UDP，TCP 查询自然不会显示对应进程。请改用：

```bash
lsof -nP -iUDP:7890
```

或者直接使用同时覆盖网络类型的查询：

```bash
lsof -nP -i :7890
```

其中 `7890` 应替换为日志中的实际端口。

### 3. 端口没有被其他程序占用，为什么仍然提示冲突？

可能存在旧的代理核心、后台服务或刚刚退出但尚未完全释放的进程。先完全退出客户端，再分别用 `netstat` 或 `lsof` 复查 TCP 和 UDP；如果端口仍然被同一客户端的旧进程占用，结束已经确认的旧进程或重启系统后再次启动。若问题在更新后反复出现，应记录客户端版本、操作系统、报错端口、网络类型和进程 PID，再到官方仓库查找相近问题。

### 4. 客户端自动使用了新的端口，是不是订阅失效了？

不一定。端口是本机监听配置，订阅主要提供代理节点、代理组或规则等内容。若日志显示原端口被占用，应先确认系统代理是否同步指向实际运行端口，再判断订阅和远端连接问题。

### 5. 能不能直接把 `external-controller` 端口当作代理端口？

不能。控制器端口用于管理接口或面板通信，不负责普通浏览器代理流量。浏览器和终端工具应使用当前实际启用的 `port`、`socks-port` 或 `mixed-port`。

### 6. 修改端口后仍然无法上网，下一步查什么？

先确认客户端核心确实在新端口监听，或在涉及 UDP 的情况下确认新端口没有被其他 UDP 套接字占用；然后检查系统代理地址、代理协议类型以及浏览器是否绕过系统代理。如果端口状态正常但请求仍失败，再继续检查规则匹配、代理组选择、DNS 或目标网站可达性，不要把所有问题都归因于端口冲突。

### 7. 结束占用端口的进程会不会影响系统？

有可能。未知 PID 可能属于系统组件、开发服务、虚拟机或安全软件。结束前应确认进程名称、路径和用途；优先正常退出相关应用，只有在确认目标进程可以安全终止时才使用强制结束命令。

## 参考资料

- [Clash Verge Rev 官方 GitHub 仓库](https://github.com/clash-verge-rev/clash-verge-rev)
- [mihomo 官方文档：Proxy Ports](https://wiki.metacubex.one/en/config/inbound/port/)
- [mihomo 官方文档：Inbound Configuration](https://wiki.metacubex.one/en/config/inbound/)
- [Microsoft Learn：netstat 命令](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)
- [Microsoft Learn：taskkill 命令](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/taskkill)
- [lsof 手册：网络文件筛选器与端口查询](https://man7.org/linux/man-pages/man8/lsof.8.html)

## 关于 边界云加速

边界云加速提供跨平台代理使用相关的订阅与客户端配置指引，帮助用户完成常见客户端导入和连接排查。使用相关服务时，请遵守当地法律法规。

官网：[https://bianjiecloud.com/](https://bianjiecloud.com/)