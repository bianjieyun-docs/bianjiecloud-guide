> 本文由 [边界云加速](https://bianjiecloud.com/) 整理，更新于 2026-11-06。

# sing-box check 怎么判断配置检查成功？用退出状态排查 JSON 配置

**一句话结论：**`sing-box check` 的可靠判断方式是看进程是否无错误退出，而不是等待某一行固定的“检查成功”提示。

## 适用场景

本文只讨论一个问题：使用 sing-box 命令行工具检查 JSON 配置后，如何确认检查通过，以及检查失败时如何定位错误。

适合以下情况：

- 手动编辑了 `config.json`，想在启动前先检查配置；
- 从订阅转换工具或客户端导出了 sing-box 配置；
- 修改了 `inbounds`、`outbounds`、`dns`、`route` 等字段后，担心 JSON 或字段值写错；
- 使用 Linux、macOS、Windows 命令行运行 sing-box，希望在脚本中自动判断检查结果；
- 配置启动失败，但不确定是文件路径错误、JSON 语法错误，还是 sing-box 不接受某个字段。

sing-box 官方配置文档将配置文件定义为 JSON，并列出了 `sing-box check` 作为配置检查命令。官方实现中，`check` 会读取并合并配置，然后尝试创建 sing-box 实例；如果过程中出现错误，命令会记录致命错误并以失败状态结束。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

## 分步骤操作

### 第一步：确认 sing-box 命令可用

先打开终端或命令提示符，执行：

```bash
sing-box version
```

如果系统提示找不到命令，先安装 sing-box，或改用二进制文件的完整路径。例如，官方文档列出了 Windows 下通过 Scoop、Chocolatey 或 winget 安装的方式，也列出了 macOS 下通过 Homebrew 安装的方式。([sing-box.sagernet.org](https://sing-box.sagernet.org/installation/package-manager/))

```powershell
# Windows，示例
winget install sing-box
```

```bash
# macOS，示例
brew install sing-box
```

安装方式会随系统和发行版不同而变化，本文不假定某个特定版本号。如果你使用的是图形客户端内置的 sing-box 核心，应以该客户端实际提供的命令路径为准。

### 第二步：准备要检查的配置文件

假设配置文件位于当前目录，文件名为 `config.json`。可以先确认文件确实存在：

```bash
ls -l ./config.json
```

Windows PowerShell 可以使用：

```powershell
Get-Item .\config.json
```

Windows 命令提示符可以使用：

```cmd
dir config.json
```

如果文件不存在，`sing-box check` 无法读取目标配置。此时应先修正路径，不要把“文件找不到”误判为配置字段错误。

### 第三步：执行 sing-box check

在配置文件所在目录运行：

```bash
sing-box check -c ./config.json
```

这里的 `-c` 用于指定配置文件。官方命令实现会读取配置并尝试初始化配置对应的 sing-box 实例；配置检查并不是只做括号或逗号检查，因此某些字段值、组件初始化问题也可能在这里暴露。([github.com](https://github.com/SagerNet/sing-box/blob/testing/cmd/sing-box/cmd_check.go))

如果你的配置文件不在当前目录，请使用绝对路径或正确的相对路径：

```bash
sing-box check -c /etc/sing-box/config.json
```

Windows PowerShell 示例：

```powershell
sing-box.exe check -c "C:\sing-box\config.json"
```

路径中包含空格时，必须用引号包住整个路径。

### 第四步：用退出状态判断结果

#### Linux 或 macOS：查看 `$?`

```bash
sing-box check -c ./config.json
printf 'exit_code=%s\n' "$?"
```

判断原则如下：

- `exit_code=0`：该进程无错误退出，可以继续进行启动测试；
- 非 `0`：检查失败，应根据终端输出修正配置、路径或运行环境；
- 不要把某一条固定的成功提示当作必然输出，官方文档只定义了 `sing-box check` 命令，源码也没有要求成功时打印固定文本。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

如果希望在脚本中直接中止，可以这样写：

```bash
sing-box check -c ./config.json
if [ "$?" -ne 0 ]; then
  echo "sing-box 配置检查失败"
  exit 1
fi

echo "sing-box 配置检查通过"
```

也可以使用更简洁的写法：

```bash
if sing-box check -c ./config.json; then
  echo "sing-box 配置检查通过"
else
  echo "sing-box 配置检查失败"
  exit 1
fi
```

#### Windows PowerShell：查看 `$LASTEXITCODE`

```powershell
sing-box.exe check -c ".\config.json"
$LASTEXITCODE
```

可以用条件判断：

```powershell
sing-box.exe check -c ".\config.json"
if ($LASTEXITCODE -eq 0) {
    Write-Host "sing-box 配置检查通过"
} else {
    Write-Host "sing-box 配置检查失败，退出码：$LASTEXITCODE"
    exit $LASTEXITCODE
}
```

#### Windows 命令提示符：查看 `%ERRORLEVEL%`

```cmd
sing-box.exe check -c config.json
if %ERRORLEVEL% equ 0 (
  echo sing-box 配置检查通过
) else (
  echo sing-box 配置检查失败，退出码为 %ERRORLEVEL%
)
```

### 第五步：根据错误输出分类处理

#### 1. 文件路径或权限错误

如果提示无法打开文件、文件不存在或权限不足，优先检查：

- `-c` 后面的路径是否拼写正确；
- 当前用户是否有读取文件的权限；
- Windows 路径是否需要引号；
- Linux 服务用户是否能读取配置和配置引用的证书、密钥等文件。

这类问题还没有进入具体字段排查阶段，先把文件读取问题解决。

#### 2. JSON 语法错误

如果提示逗号、括号、引号或 JSON 解析错误，检查最近修改的区域。常见错误包括：

```json
{
  "log": {
    "level": "info"
  },
  "outbounds": []
}
```

上面的结构使用双引号，字段之间用逗号分隔，最后一个字段后没有多余逗号。JSON 不接受注释，因此不要直接加入 `//` 或 `/* ... */` 说明文字。

可以先使用编辑器的 JSON 格式化功能，但格式化只能帮助发现结构问题，不能保证所有 sing-box 字段都符合当前版本要求。

#### 3. 字段或组件初始化错误

如果 JSON 能够解析，但某个字段值、协议参数、路径资源或组件初始化失败，终端通常会显示更具体的错误位置。此时不要只重复执行命令，应记录：

- 报错涉及的配置区块；
- 是否是最近新增或修改的字段；
- 配置使用的 sing-box 核心版本；
- 是否依赖证书、私钥、规则集或外部文件；
- 是否在不同操作系统或不同用户权限下执行。

官方配置文档将 `log`、`dns`、`inbounds`、`outbounds`、`route` 等列为顶层配置区域；排错时应先确认报错字段位于正确的配置层级，再检查字段值。([sing-box.sagernet.org](https://sing-box.sagernet.org/configuration/))

### 第六步：检查通过后再启动

`sing-box check` 通过只说明检查进程无错误退出，不等于网络连接、远端服务器或系统代理一定可用。检查通过后，可以再执行实际启动命令，例如：

```bash
sing-box run -c ./config.json
```

如果配置由 systemd 管理，可以结合服务日志继续观察：

```bash
sudo systemctl restart sing-box
sudo journalctl -u sing-box --output cat -e
```

官方安装文档列出了 sing-box 的 systemd 服务管理命令和日志查看方式。([sing-box.sagernet.org](https://sing-box.sagernet.org/installation/package-manager/))

如果你是在边界云加速后台获取订阅后转换为 sing-box 配置，建议先将配置文件保存到本地，完成 `sing-box check`，再导入客户端或启动服务。这样可以把“配置格式问题”和“连接问题”分开处理。

## 常见问题

### 1. `sing-box check` 没有输出，是不是没有执行？

不一定。成功时不要依赖固定文本判断结果。直接查看退出状态：

```bash
sing-box check -c ./config.json
printf '%s\n' "$?"
```

如果结果为 `0`，表示进程无错误退出。PowerShell 使用 `$LASTEXITCODE`，Windows 命令提示符使用 `%ERRORLEVEL%`。

### 2. 为什么 `sing-box check -c config.json` 报“文件不存在”？

通常是当前工作目录不对，或者文件名、大小写、扩展名与命令不一致。先执行以下命令确认位置：

```bash
pwd
ls -l
```

Windows PowerShell：

```powershell
Get-Location
Get-ChildItem
```

确认后使用配置文件的绝对路径，避免相对路径受当前目录影响。

### 3. 检查通过后，运行 sing-box 仍然失败，说明 check 不可靠吗？

不代表检查命令失效。`check` 主要用于读取、合并并初始化配置；实际运行时还可能受到端口占用、权限、系统 TUN 能力、证书文件权限、网络环境或远端服务状态影响。因此应继续查看 `run` 命令输出或 systemd 日志，而不是只重复执行 `check`。

### 4. 能不能用编辑器的 JSON 校验代替 `sing-box check`？

不能完全替代。编辑器通常只能检查 JSON 语法和基础结构，而 `sing-box check` 会按 sing-box 自身的配置逻辑读取并初始化配置。两者适合配合使用：先用编辑器处理括号、引号和逗号，再用 `sing-box check` 检查 sing-box 能否接受该配置。

### 5. 是否必须看到 `configuration parsed successfully` 才算成功？

不需要把这句话作为判断条件。不同版本、构建方式或日志设置下，成功时的终端表现可能不同。更稳妥的判断是：命令进程无错误退出，并返回退出码 `0`；失败时则根据错误输出和非零退出状态处理。

## 参考资料

以下资料均为本次检索到的 sing-box 官方文档或官方源代码：

1. [sing-box 配置介绍与 check 命令](https://sing-box.sagernet.org/configuration/)：说明配置使用 JSON，并列出 `sing-box check`。
2. [sing-box check 命令官方源代码](https://github.com/SagerNet/sing-box/blob/testing/cmd/sing-box/cmd_check.go)：展示命令如何读取、初始化配置，并在错误时调用错误处理逻辑。
3. [sing-box 安装与包管理](https://sing-box.sagernet.org/installation/package-manager/)：列出 Windows、macOS、Linux 等平台的安装方式及 systemd 管理命令。
4. [sing-box 日志配置](https://sing-box.sagernet.org/configuration/log/)：说明日志级别、输出文件和时间戳字段。
5. [sing-box Desktop 官方文档](https://sing-box.sagernet.org/clients/desktop/)：介绍 sing-box 桌面客户端对本地或远程配置文件的管理方式。

## 关于 边界云加速

[边界云加速](https://bianjiecloud.com/) 提供跨平台网络连接与订阅使用服务，适用于 Windows、macOS、Android、iOS 以及常见代理客户端。使用相关服务和网络工具时，请遵守当地法律法规。