# sapper-v2 静态安全审计

审计日期：2026-11-09  
对象：`Silver-cybersec/sapper-v2`（fork 自 `N1nEmAn/sapper-v2`）检出 `d4de0770ae1fba706f00b7e29197ce739adbea94`  
方式：只读。读了源码、脚本、Dockerfile、知识库、git 历史和公开索引。计算了仓库内 `docker/nuclei.zip` 的 SHA256，并与 ProjectDiscovery 发布的校验文件对照。查询了 OSV、GitHub Advisory、npm 注册表、PyPI 和 Docker Hub 的标签元数据。

没有做的事：没有运行仓库里的 Python、shell、安装脚本、二进制或 Docker 镜像，没有 `pip`/`npm`/`apt` 安装，没有 `docker build` / `docker load`，没有向任何靶机或比赛平台发请求，没有下载约 1.2GB 的预构建镜像，也没有下载 Claude Code 的平台原生包（解压后约 340MB）做字节复算。

## 结论

**有可疑点待确认。**

本仓库里的源码、脚本和已核对的官方 nuclei 3.3.7 压缩包，没有发现隐藏回连、混淆载荷、凭证外发给作者，或被改过的 nuclei 二进制。待确认的是两件仓库外或构建时才出现的东西：README 推荐的预构建镜像冻结在公开 git 历史中不存在的提交 `8e03190`，本次没有拆开；Docker 构建会执行未锁定依赖的安装脚本，本次只核对了 2026-11-09 注册表上的解析结果，不能代表下一次构建。

## 1. 供应链

仓库根目录没有 `go.mod`/`go.sum`、`package.json`/lock、`requirements.txt`/`poetry.lock`/`Pipfile`、`Cargo.lock`、`composer.lock`，没有 `docker-compose`，没有 `.gitmodules`，没有 `.github` 工作流。`bash install.sh` 只做符号链接，不下载包。会拉外部内容的路径是 Docker 构建、README 让人手动下载的镜像，以及运行中的 nuclei / 模型可能再去拉的模板和可选依赖。

### 1.1 会被拉取的外部内容

| 条目 | 触发方式 | 来源 | 版本或摘要是否锁在仓库里 | 本次核对 |
|---|---|---|---|---|
| `kalilinux/kali-rolling:latest` | `docker/Dockerfile` 第 11 行 `FROM` | Docker Hub 官方库 `kalilinux/kali-rolling` | 没有 digest。标签 `latest` 会漂 | 未拉取镜像。2026-11-09 查询时该标签的索引摘要是 `sha256:6ec743d9fc5cd397451eeddeff3869bcfab5de9bd5dec5b1f34cbfbd80313073`（amd64 层 `sha256:5cc5e418c51098ae3c48f9a1d5b3bc74a508588900765a3964259da783652b03`，标签更新时间 2026-11-08）。这只是查询当天的浮动标签，不是仓库里的钉死值 |
| `sapper-agent:latest`、`sapper-agent:tmux` | `docker/Dockerfile.tmux` 第 7 行、`docker/Dockerfile.commander` 第 7 行 | 本机已经构建的镜像名，不是注册表地址 | 没有 digest | 仓库不包含这两层的镜像内容。它们继承上面的 Kali 基础镜像 |
| Kali `apt` 包 | `docker/Dockerfile` 第 29–52 行；`docker/Dockerfile.tmux` 第 10 行装 `tmux`；`docker/Dockerfile.commander` 第 17–20 行再装一批 | 第 26–27 行把源写成 `http://mirrors.tuna.tsinghua.edu.cn/kali`，套件 `kali-rolling` | 包名有，版本没有，也没有 `apt` 哈希钉 | 未安装。版本和 CVE 适用性未确认 |
| `@anthropic-ai/claude-code` | `docker/Dockerfile` 第 71–72 行 `npm install -g`，并加 `--allow-scripts` | 默认 npm 注册表。作用域 `@anthropic-ai`。维护者邮箱为 `anthropic.com`。近月下载量约 4434 万（2026-10-10 至 2026-11-08，包名合计，不是某一个版本） | 仓库没有版本、没有 lock、没有 integrity | 2026-11-09 注册表 `latest` 为 `2.1.319`。该版本 OSV 无漏洞记录，也没有 `MAL-*`。Dockerfile 不锁这个版本，下次构建可能变 |
| `@anthropic-ai/claude-code-linux-x64`（以及同版本的 darwin/win32/arm64/musl 可选包） | 上一条的 `optionalDependencies`，由 npm 在安装时拉取；`postinstall` 再把它拷进 `bin/claude.exe` | 同一官方作用域。linux-x64 近月下载量约 3320 万 | 只出现在当天 `2.1.319` 的包元数据里，仓库没写 | 注册表 integrity：`sha512-FO4Sd23BO1VCQPiJe5vG3bNnmV5+YiTau7o1lqK3Fr5VlE4TLdKiyYpwQRwEXO91kE2l12Ep2i7J19T8Md2oOQ==`。解压大小约 340MB。没有下载这个 tarball 复算，原生二进制内容未确认 |
| `@openai/codex@0.42.0` | `docker/Dockerfile.commander` 第 32–33 行 | `registry.npmjs.org`，仓库字段 `git+https://github.com/openai/codex.git`，维护者为 OpenAI 的发布账号。包名近月下载量约 9206 万（含更新版本，不能当成 0.42.0 的下载量） | 版本钉在 `0.42.0`。仓库没有 lock，也没有把 integrity 写进文件。npm 客户端安装这个精确版本时会按注册表返回的 integrity 核对 tarball | 发布于 2025-09-26。当天 `latest` 已是 `0.167.0`。元数据里 `scripts` 为空，`dependencies` 为空。OSV 对该版本无记录。没有拆开 tarball，首次运行是否再下载二进制未确认 |
| Python 包 | `docker/Dockerfile` 第 76–82 行，`docker/Dockerfile.commander` 第 21–25 行 | 默认 PyPI。包名是规范名：`requests`、`pyjwt`、`unicorn`、`pwntools`、`pycryptodome`、`sympy`、`ROPgadget`、`ropper`，以及失败可忽略的 `gmpy2`、`z3-solver` | 只有 `unicorn==2.0.1.post1` 锁了版本。其余没有版本、没有哈希、没有 constraints 文件 | 见 1.4。传递依赖没有 lock，本次没有安装，传递闭包未确认 |
| `docker/nuclei.zip` | 已在 git 里。构建时 `docker/Dockerfile` 第 64–66 行 `COPY` 后解压出 `nuclei`，不在构建时从网络下载 | 压缩包内 README 指向 `github.com/projectdiscovery/nuclei`。文件时间 2024-12-02，与 v3.3.7 发布时间一致 | 仓库没有附带 checksum 文件。二进制被整包提交 | **已核对，与官方包一致。** 见 1.2 |
| nuclei 模板 | 仓库和 zip 里都没有模板目录。zip 内 README 第 310–314 行有 `-update`、`-update-templates`、`-disable-update-check` | 官方模板库是 `github.com/projectdiscovery/nuclei-templates`（工具自身的行为，不是本仓库写死的 URL） | 没有模板 commit，脚本也没有加 `-duc` | 没有运行 nuclei，所以没有观察到实际请求。默认会不会在第一次扫描时检查更新：未在本仓库的脚本里关闭，工具 README 把 `-duc` 写成关闭自动检查的开关 |
| `sapper-v18-mixed.tar.gz` | 不在 git 里。`README.md` 第 228–230 行、`README.zh-CN.md` 第 158–160 行让用户 `docker load` | `https://github.com/N1nEmAn/sapper-v2/releases/download/v18-champion/sapper-v18-mixed.tar.gz` | 发布说明和 GitHub asset digest 都写了 `sha256:22b0014b55c23f5e6ef8fd976045e6a551d91b814ae82921996f8f9af3c3fb77`，大小 1227645241 字节 | 没有下载复算。发布说明写冻结在 `8e03190`（2026-10-18）。这个对象在本机和 `N1nEmAn/sapper-v2` 的 GitHub API 里都不存在（API 422）。镜像内容与本次审过的树是否相同：未确认 |
| `nvwabase` | 只有 README 里的手动 `git clone`（`README.md` 第 116 行、`README.zh-CN.md` 第 79 行） | `https://github.com/N1nEmAn/nvwabase` | 没有 submodule，没有 commit pin | `install.sh` 不会克隆它。那个仓库本次没有审 |
| 运行时 HTTP | 容器入口和 `lib/arena.py` | `BENCHMARK_BASE_URL`（环境变量，无默认主机名）；模型网关默认写在脚本和文档里，见 1.3 | 由操作者注入 | 没有发这些请求 |

`docker/Dockerfile` 第 29–52 行列出的 apt 包（全部未锁版本）是：`ca-certificates` `curl` `wget` `git` `jq` `unzip` `zip` `file` `xxd` `less` `vim` `procps` `psmisc` `lsof` `tcpdump` `iproute2` `iputils-ping` `net-tools` `python3` `python3-pip` `nodejs` `npm` `netcat-traditional` `socat` `dnsutils` `whois` `openssl` `nmap` `masscan` `whatweb` `wafw00f` `nikto` `gobuster` `ffuf` `dirb` `wfuzz` `sqlmap` `hydra` `binwalk` `steghide` `exiftool` `libgmp-dev` `libmpfr-dev` `libmpc-dev` `redis-tools` `default-mysql-client` `postgresql-client` `gdb` `checksec` `smbclient` `smbmap` `ldap-utils` `enum4linux` `responder` `impacket-scripts` `proxychains4` `seclists` `sudo`。tmux 变体另加 `tmux`。这些是 Kali 仓库里的既有包名，没有看到拼写仿冒。

知识库不是安装清单，但模型有 shell 时会按文字去下载。`kb/entries/kb012.json` 第 5 行写「linpeas.sh（离线版随仓库带）」。当前树里没有 `linpeas.sh`。`kb016.json`、`kb031.json` 的 `source` 写了 `nuclei-templates`，仓库没带模板。这会让运行中的 worker 自己去网上找工具。那些 URL 不在脚本里，本次无法列出具体地址。

### 1.2 已核对的预编译制品

`docker/nuclei.zip` 的 SHA256 是 `725ef892fcffd1b03ad4f0874942fc4b623c0419b6b6c6c91fe4a5a65671f77c`，大小 29058291 字节。这与 ProjectDiscovery 发布文件 `nuclei_3.3.7_checksums.txt` 里 `nuclei_3.3.7_linux_amd64.zip` 的一行相同：

https://github.com/projectdiscovery/nuclei/releases/download/v3.3.7/nuclei_3.3.7_checksums.txt

zip 内是 ELF64、静态链接、已 strip 的 Go 二进制，外加 LICENSE 和多语言 README，没有第二个可执行文件。包内字符串 `8.8.8.8:53` 和 `8.8.4.4:53` 在文件里紧挨着，拼读会变成 `8.8.8.8:538`。公开沙箱 Triage 对这个同一哈希报过 Gafgyt，C2 写成 `8.8.8.8:538`。二进制里没有独立的 `gafgyt`/`BASHLITE` 字符串；`sapper` 出现在 Svelte Sapper 的指纹规则（`https://sapper.svelte.dev`）里，不是这个项目的名字。因此这个 zip 是官方 nuclei 3.3.7，不是本仓库改过的植入包。它仍然是过时扫描器，见 1.4。

预构建镜像只核对了 GitHub 上的摘要字段与发布说明一致。字节没有在本地重算，标为未确认。

### 1.3 未锁定版本和运行时再拉取

风险在于仓库把「装什么」留到构建或第一次运行时才决定。

1. **Kali `latest` + `kali-rolling` + 清华镜像。** `docker/Dockerfile` 第 11、26–27、29 行。镜像源是清华大学公开镜像，不是短链，也不是个人仓库；它也不是 `http.kali.org`，并且没有 snapshot 日期。镜像或滚动套件被换掉时，同一次 Dockerfile 会装进另一套包。构建机若设置了 `http_proxy`，基础镜像的 apt 走代理；commander 变体第 13–16 行会主动去掉代理再访问清华源。
2. **`@anthropic-ai/claude-code` 不锁版本，且打开安装脚本。** 第 69–72 行注释写 postinstall 必须去取原生二进制。本次读了 2026-11-09 的 `2.1.319` 包里的 `install.cjs`（只读，没有执行）：它不自己 `curl` 一个 URL，而是从 npm 已安装的 optional 平台包拷贝 `claude` 并 `chmod 755`。平台包仍是另一次注册表下载。仓库没有把版本和 integrity 写下来，所以下一次 `latest` 的脚本可以不同。`--allow-scripts=@anthropic-ai/claude-code` 会让这个包的生命周期脚本执行。
3. **PyPI 除 unicorn 外都不锁版本。** `pip3 install` 没有 `--require-hashes`。`pwntools` 会再拖入自己的依赖树，那些包名没有出现在 Dockerfile 里。
4. **nuclei 模板和引擎更新。** 仓库脚本没有调用 `nuclei -update`。zip 内 README 仍提供 `-up` / `-ut`，并用 `-duc` 关闭自动检查。本仓库的运行命令没有加 `-duc`。worker 一旦执行 nuclei，模板来自当时的官方模板库，而不是这次审过的文件。
5. **知识库指向仓库里不存在的离线工具。** 见 `kb012.json` 第 5 行。模型可以改去下载 linpeas 或其他二进制。
6. **`docker/local-sim.sh` 第 13、17 行执行 `/tmp/mock_bench.py`。** 这个文件不在仓库里。脚本第 28 行使用 `docker run --network host`。只有有人运行该脚本时才会发生。
7. **模型网关是运行时出站，不是包管理器。** 默认主机名写在：`docker/ENV.md` 第 23、33 行（`open.bigmodel.cn`、`api.moonshot.cn`）、`docker/commander/codex-home/config.toml` 第 8、14 行和 `docker/commander/worker.sh` 第 43 行、`watchdog.sh` 第 278 行（`api.deepseek.com`）、`docker/local-sim.sh` 第 31 行。入口脚本把 `DISABLE_TELEMETRY`、`DISABLE_AUTOUPDATER`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 设为 1（`docker/entrypoint.sh` 第 22–25 行，commander 入口第 38–39 行）。这关的是 Claude Code 的遥测和自更新，不管 nuclei。

`install.sh` 没有 `curl`/`wget`。第 59 行会执行本地的 `sapper --version`，那是本仓库的 Python，不是下载后执行。

### 1.4 公开漏洞库（2026-11-09）

查询的是 OSV `query`/`querybatch` 和 GitHub Advisory。恶意包库使用 OSV 的 `MAL-*` 编号。下面规范包名都没有返回 `MAL-*`。

**nuclei 模块 `github.com/projectdiscovery/nuclei/v3` 版本 3.3.7 被 OSV 标为受影响。** 这些是官方扫描器的已知问题，触发条件是加载不可信模板，或处理目标返回的数据。影响写到「能在运行 nuclei 的机器上读文件、读环境变量或执行命令」为止，这里不写触发方法。

| 公开编号 | OSV 记录的影响（v3） | 标题中的影响 |
|---|---|---|
| CVE-2026-41646 / GHSA-29rg-wmcw-hpf4 / GO-2026-4965 | `>=3.0.0`，`3.8.0` 修复 | 本地文件读取 |
| CVE-2026-41282、CVE-2026-41645 / GHSA-jm34-66cf-qpvr / GO-2026-5473 | `>=3.0.0`，`3.8.0` 修复 | 环境变量泄露 |
| CVE-2026-76802 / GHSA-jpf4-98qj-qr67 / GO-2026-6570 | `>=3.0.0`，`3.10.0` 修复 | 任意命令执行 |
| CVE-2026-76805 / GHSA-jpvm-9frm-hjcq / GO-2026-6571 | `>=3.0.0`，`3.10.0` 修复 | 环境变量泄露 |
| CVE-2026-76804 / GHSA-qgw5-7j4f-fg97 / GO-2026-6574 | `>=3.0.0`，`3.10.0` 修复 | 本地文件读取 |
| CVE-2026-76819 / GHSA-vxg7-f2jj-jmqm / GO-2026-6575 | `>=3.0.0`，`3.10.0` 修复 | 任意代码执行 |
| CVE-2026-76803 / GHSA-xhmx-w2j4-rw3q / GO-2026-6577 | `>=3.0.0`，`3.10.0` 修复 | 本地文件读取 |

已抽查的 GitHub Advisory 把其中几条标为 medium。OSV 对这个精确模块版本没有返回 CVE-2024-27920、CVE-2024-40641、CVE-2024-43405。GitHub 对 CVE-2024-43405 的易受攻击范围是 `>=3.0.0, <3.3.2`，按该范围 3.3.7 不在其中。

**同一天、若按「索引上的最新版」安装，OSV 对下列精确版本返回 0 条，也没有 `MAL-*`。** Dockerfile 并没有锁这些版本，所以这不是「本仓库已经钉死的安全版本」：

| 包 | 2026-11-09 索引最新版 | 仓库里的约束 | OSV |
|---|---|---|---|
| requests | 2.34.2 | 无 | 0 |
| pyjwt | 2.15.1 | 无 | 0 |
| pwntools | 4.15.0 | 无 | 0 |
| pycryptodome | 4.4.0 | 无 | 0 |
| sympy | 1.14.0 | 无 | 0 |
| ROPgadget | 7.7 | 无 | 0 |
| ropper | 1.13.13 | 无 | 0 |
| gmpy2 | 2.3.2 | 无 | 0 |
| z3-solver | 5.3.3.0 | 无 | 0 |
| unicorn | 索引最新 2.1.4；仓库钉的是 `2.0.1.post1`（2022-11-22 上传） | `==2.0.1.post1` | 对该钉死版本 0 |
| `@openai/codex` | 索引最新 0.167.0 | `0.42.0` | 对 0.42.0 为 0 |
| `@anthropic-ai/claude-code` | 2.1.319 | 无 | 对 2.1.319 为 0 |

不带版本去查 OSV 时，`requests`、`pyjwt`、`pwntools`、`pycryptodome`、`@anthropic-ai/claude-code` 会返回多条历史 GHSA。那些记录不能直接套到上表的最新版。Kali 包没有版本，CVE 未确认。

`unicorn==2.0.1.post1` 在 PyPI 上仍有文件。linux x86_64 的 manylinux wheel 摘要之一是 `sha256:45be711d2bc8b82d07f2e0bdde128f3e9408c42978a3ed5edf709bce5350fcdf`（文件名 `unicorn-2.0.1.post1-py2.py3-none-manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl`）。Dockerfile 没有使用这个哈希。本次没有下载 wheel 复算。

### 1.5 仿冒、新包、作者、安装钩子、非官方来源

- 包名没有看到缺字、多字或相近仿冒。`pyjwt` 是 PyPI 上 PyJWT 的规范名。`pycryptodome` 是独立项目名，不是 `pycrypto` 的误写。npm 用的是 `@anthropic-ai` 和 `@openai` 作用域。
- 没有发现从个人 git 仓库、`bit.ly` 一类短链或来历不明的小站下载。唯一改写的镜像是 `mirrors.tuna.tsinghua.edu.cn`。
- 下载量：上面两个 npm 包名的近月下载都是千万级，不像新注册的冷门包。`codex@0.42.0` 是旧版本，包名下载量不能证明这个旧版本仍被广泛安装。PyPI 这几个规范名没有在 OSV 标成恶意包；具体上传者当日下载量未逐个统计。
- 安装钩子：claude-code `2.1.319` 有 `postinstall`: `node install.cjs`。平台包自身元数据没有 scripts。codex `0.42.0` 元数据没有 scripts。pip 这几行没有使用会在安装时执行 `setup.py` 的本地路径包；`pip install` 仍会构建带原生扩展的包（`unicorn`、`pwntools` 的依赖等），构建脚本内容未确认，因为没有安装。
- 作者异常：npm 维护者与作用域组织一致。git 作者见第 2 节，那是仓库提交身份，不是包维护者。

### 1.6 更稳妥的拉取方式

本地只用 CLI 时，走 `bash install.sh`。它不拉包。模型 CLI 若已装在本机，那是另一条供应链，不在这个仓库里。

要构建镜像时：

1. 把 `FROM kalilinux/kali-rolling:latest` 换成带 digest 的官方库镜像，或改用带日期的 snapshot，不要长期追 `latest`。需要国内镜像时，固定镜像站点和套件日期，并保留官方校验路径。
2. 提交 `package-lock.json`（或 npm 的等价锁），用 `npm ci`。把 `@anthropic-ai/claude-code` 和 `@openai/codex` 写成精确版本加 integrity。先 `--ignore-scripts` 安装，再只对审过的 `install.cjs` 打开脚本。
3. 用 `pip` 的 requirements 加 `--require-hashes`，包括传递依赖。`unicorn` 继续钉版本，并把 PyPI 上的 wheel sha256 写进文件。
4. nuclei 换到 OSV 范围之外的版本（上表修复版本为 3.8.0 与 3.10.0，取更高的那档），用发布页 checksums 核对后再放进构建上下文。运行时加 `-duc`，模板单独钉一个 git commit，不要在扫描时 `-ut`。
5. 不要把 `sapper-v18-mixed.tar.gz` 当作已审计制品。若仍要下载，先 `sha256sum` 对比 `22b0014b55c23f5e6ef8fd976045e6a551d91b814ae82921996f8f9af3c3fb77`。摘要对上，也只说明文件和 GitHub 发布页一致，对不上公开提交 `8e03190`（该提交在公开仓库中不存在）。

## 2. 其他发现

### 2.1 网络出站

**等级：信息（功能所需）和低（文档外链）。**

源码里的 HTTP 客户端是 `lib/arena.py` 第 132–144 行，只请求 `BENCHMARK_BASE_URL` 加上固定路径 `/openapi/v1/challenges` 及其 `start`/`hint`/`submit`/`close`。主机名来自环境变量或 `.sapper/arena.json`，仓库里的模板值是 `https://benchmark.example.com`（`lib/arena.py` 第 120 行）。

容器入口在启动时用 `curl` 打同一路径：`docker/entrypoint.sh` 第 42–46 行，`docker/entrypoint-tmux.sh` 第 34–38 行，`docker/commander/entrypoint.sh` 第 66–70 行。随后入口会执行比赛循环（`docker/entrypoint.sh` 第 67–74 行）或拉起指挥官（commander 入口第 105–125 行）。

没有在 Python 或 shell 里看到写死的 webhook、Discord、Telegram、私有 IP 回连或 sapper 自己的遥测域名。README 里的微信文章、linux.do、star-history 图片是文档外链，`install.sh` 不会请求它们。`docs/events-2026-10-17.md` 里的 `10.0.162.x` 是比赛记录表，不是程序默认目标。

nuclei 二进制带有 interactsh / `oast.` 字符串。那是官方扫描器的带外检测功能，只有运行 nuclei 且模板使用它时才会出网。

### 2.2 混淆、编码、二进制

**等级：信息。官方二进制另有已知 CVE，见 1.4。**

- `docker/commander/worker.sh` 第 379–385 行把任务说明做 base64，是为了看门狗重启窗格后重放，不是隐藏载荷。`watchdog.sh` 第 336–338 行解码同一字段。
- `lib/sapper.py` 第 1720–1737、1840–1848、3364–3374 行用 `importlib` 加载同目录的 `arena.py`、`kb.py`、`_settings_patch.py`，不加载远程代码。
- 没有 `eval` 执行网络响应，没有压缩混淆的 JS。`lib/view_template.html` 的 `fetch` 只请求本页给出的本地轮询路径。
- 除 `docker/nuclei.zip` 外，`file` 结果里没有其他可执行文件。`docs/assets/*.jpg` 的 JPEG 结束标记之后没有尾随字节。
- 全库源码检索没有零宽字符。

### 2.3 安装脚本与「下载后执行」

**等级：高（仅 Docker 构建路径）；低（`install.sh`）。**

`install.sh` 不下载。Docker 构建会 `npm install` 并允许 claude-code 的 postinstall（`docker/Dockerfile` 第 71–72 行），commander 镜像对 codex 也传了 `--allow-scripts`（`docker/Dockerfile.commander` 第 32 行）。codex 0.42.0 的注册表元数据没有脚本；这个开关仍会作用于该次安装里任何带脚本的包。

没有 CI 工作流，因此没有 Actions 里的第三方 `curl | sh`。

### 2.4 凭证、浏览器数据、SSH、环境变量

**等级：中（密钥落在本机可复制的位置）；没有发现发往作者服务器。**

程序读取的是进程环境里的 `BENCHMARK_TOKEN`、`ANTHROPIC_AUTH_TOKEN`、`DEEPSEEK_API_KEY`、`SAPPER_KEY_POOL`。没有代码去读 `~/.ssh`、浏览器配置目录或云厂商家目录。知识库文字会让模型在靶机上找凭证并记进事实图，那是渗透流程，数据默认留在本地 `.sapper/`。

密钥会以明文离开环境变量、进入工作目录和进程命令行：

- `docker/tmux-drive.sh` 第 72–76 行把 `ANTHROPIC_AUTH_TOKEN` 写入 `$SAPPER_ROOT/rotated.env`。
- `docker/commander/watchdog.sh` 第 145–150 行写入 `rotated.env` 和 `rotated-<family>.env`。
- `docker/commander/worker.sh` 第 388–398 行把 `DEEPSEEK_API_KEY` 插进 tmux 要执行的命令字符串。`watchdog.sh` 第 329 行同样。命令行和 tmux 的 `pane_start_command` 能看到完整密钥。
- `lib/sapper.py` 第 1420–1423 行在轮换时把 token 末 6 位打到 stderr。

`lib/sapper-hook.py` 只在项目已有 `.sapper/` 时，把图摘要交给 Claude Code 的钩子输出。它会跑本地 `date`（第 46–50 行），不上传。

`sapper view --serve` 默认绑 `127.0.0.1:8765`（`lib/sapper.py` 第 4550–4551 行）。若改成非回环地址，第 3647–3649 行会警告图可被该端口的访问者读到。轮询路径带随机 token，响应没有 CORS 头。

### 2.5 持久化、提权、自更新

**等级：低（安装器）；信息（容器内看门狗是比赛循环的一部分）。**

- `install.sh` 第 49、73 行在 `~/.local/bin/sapper` 和 `~/.claude/skills/sapper-setup` 放符号链接。不写 crontab、systemd、SSH authorized_keys。
- `sapper setup` 只改当前项目的 `./.claude/settings.json` 和可选的 `AGENTS.md`（`lib/sapper.py` 第 3379–3443 行）。注释写明不改家目录。`lib/_settings_patch.py` 第 6 行的说明提到 `install.sh --global-hooks`，当前 `install.sh` 没有这个参数。直接运行 `_settings_patch.py add` 且不给路径时，第 243 行会落到 `~/.claude/settings.json`。正常的 `sapper setup` 传的是项目路径。
- 镜像里装了 `sudo`（`docker/Dockerfile` 第 52 行），随后 `useradd` 的用户是 `agent`（第 93–102 行），没有看到给 `agent` 的 sudoers。`agent` 能否提权：未确认，取决于基础镜像的默认组，本次没有构建镜像。
- sapper 自己没有自更新器。入口关闭 Claude Code 自更新。`DISCLAIMER.md` 第 26–31 行和第 43 行写未来版本「包括静默更新」可能带示踪剂。当前树里没有示踪剂实现，也没有静默更新代码。以后的版本不在本次范围内。
- 看门狗会在截止时间前把退出的 tmux 窗格拉起来（`docker/commander/watchdog.sh` 第 490 行之后的循环）。这是容器里的进程守护，不是主机开机启动项。

### 2.6 git 历史与作者

**等级：信息。fork 没有多出提交。**

- 本 fork 与上游 `N1nEmAn/sapper-v2` 的 `master` 在 2026-11-09 为 `identical`，`ahead_by=0`，`behind_by=0`，同一提交 `d4de077`。历史里没有 `Silver-cybersec` 的作者。
- 18 个提交，都未签名（`git log %G?` 为 `N`）。作者只有 `N1nEmAn <1332037591@qq.com>` 和 `VarFusion <varfusion@localhost>`。`VarFusion` 的 11 个提交都在 2026-11-01，改的是 README 和 `docs/assets` 图片。邮箱是 `localhost`，不能据此核对真人身份。
- 初始提交 `6098db0`（2026-10-30，N1nEmAn）一次加入包括 `nuclei.zip` 在内的整棵树。之后唯一的代码改动是 `c91881d`：`lib/sapper-hook.py` 把 `import fg` 改成 `import sapper`，并加了测试。diff 已读，是导入名修复。提交说明含 `Co-Authored-By: Claude <noreply@anthropic.com>`。
- 没有看到在后续小提交里塞入二进制或外连的情况。

## 3. 作为渗透工具的风险边界

这是工具的设计行为，不是第 2 节那种隐藏通道。

**默认对什么动手。** `bash install.sh` 和 `sapper setup` 不会扫描网络。攻击发生在有人执行 `sapper auto`、`sapper bench`、`sapper dispatch`，或启动 Docker 入口之后。Docker 入口没有交互确认：`docker/entrypoint.sh` 第 67–74 行直接 `sapper bench`；commander 入口第 125 行要求模型读军规并自主循环。目标地址来自平台 `start` 接口返回的 `container_addr`（`lib/sapper.py` 第 2051、2158 行），不是仓库里写死的受害者。本地若自己写了 goal 或事实图，执行器就在当前工作目录里按图里的意图行动。

**用什么权限。** 镜像以用户 `agent` 运行（`docker/Dockerfile` 第 102 行）。同时给 Claude 写了 `bypassPermissions`，并加上 `--dangerously-skip-permissions`（`docker/entrypoint.sh` 第 35 行，commander 入口第 47、105 行，`docker/commander/worker.sh` 第 200、398 行）。`lib/sapper.py` 第 1455–1456 行在 `--skip-permissions` 时把同一开关传给模型 CLI。效果是模型 CLI 可以不经逐次确认就在该用户权限下执行 shell。`docker/local-sim.sh` 第 28 行额外使用宿主机网络命名空间。知识库和镜像里的工具包括扫描、口令尝试、利用辅助和隧道一类能力；具体命令在 `kb/entries/*.json`，这里不抄录。

**破坏性操作。** `templates/explore.md` 第 110–111 行要求避免 `rm -rf`、删库、关机，除非意图写明。`templates/reason.md` 第 51–53 行把 `requires_confirmation` 留给破坏性步骤，并写 CTF 环境里几乎不用。这是提示词，不是程序里的拦截。`sapper teardown --purge` 会删 `.sapper/`，并拒绝删除文件系统根和家目录（`lib/sapper.py` 第 1548–1557、3495–3500 行）。平台侧的 `close` 会关掉题目实例。

**会不会无人值守扩散。** 会在「平台返回的题目集合」里继续派工，直到时间用完或题目完成，不会在代码里写死向公网新网段蠕虫式传播。

- `sapper bench` 对未完成题目循环 `start → auto → 提交 flag → close`，默认可并行（`lib/sapper.py` 第 2001–2031、2105–2177 行）。入口默认总预算 470 分钟（`docker/Dockerfile` 第 17 行，`docker/entrypoint.sh` 第 62 行）。
- commander 军规要求自己不打目标，只通过 `worker.sh` 派工，循环直到截止（`docker/commander/SKILL.md` 第 18–19、35–37、41–67 行）。硬上限写的是平台同时 3 个实例、默认 8 个 worker 窗格（`docker/commander/worker.sh` 第 46 行）。
- 将军任务书允许再拉最多 2 个约 5 分钟的小队（`docker/commander/worker.sh` 第 242–245 行）。这是写给模型的上限，执行靠模型遵守；窗格总数仍受 `SAPPER_MAX_WORKERS` 约束。
- 看门狗在截止前重启死掉的窗格并重发任务（`docker/commander/watchdog.sh` 第 309–356、490 行起）。截止后 `finish_all` 提交已记录 flag 并关闭实例（第 423–458 行）。
- `sapper auto` 还有步数上限（默认最多 8 轮 reason、6 次 explore，`lib/sapper.py` 第 4688–4689 行）。容器入口把 bench 的 `--max-steps` 设为 10。
- 提示词要求只打题目给出的地址，不要打平台和别的机器（`templates/explore.md` 第 95–97 行，`templates/reason.md` 第 87–89 行）。同一文件写明这是文字约束，不是沙箱。目标响应被记进事实图后，后续模型仍可能按注入内容行动。

本地不启动 `bench`/`command`/`auto`、也不跑上述镜像时，worker 不会自己扩散。

## 4. 未确认清单

- `sapper-v18-mixed.tar.gz` 的字节与发布摘要是否一致（未下载）；镜像内部与提交 `8e03190` 的内容（该提交在公开仓库不存在）。发布说明写 63 条知识库，当前树是 `kb001.json`–`kb056.json` 共 56 个文件。
- 未来某次 `docker build` 解析到的 Kali 包、npm `latest`、未锁的 pip 包及其传递依赖。
- `@anthropic-ai/claude-code-linux-x64@2.1.319` 的 340MB 原生包是否与注册表 integrity 一致（未下载）。
- `@openai/codex@0.42.0` tarball 内部，以及它第一次运行会不会再下载文件。
- 镜像里的 `agent` 用户是否因为 Kali 默认策略而拥有 sudo。
- nuclei 在默认参数下第一次运行时的实际更新请求（未运行）。
- `DISCLAIMER.md` 所说的后续示踪剂和静默更新。
- `nvwabase` 仓库，以及作者机器上被 `.gitignore` 排除、树里不存在的 `HANDOVER.md`、`PREP-INTERVIEW.md`、`docker/ENV-COMMANDER.md`、`COMMANDER-HANDOVER.md`。

## 5. 审计限制

结论只覆盖这次检出的 git 树和上面点名的公开索引查询。没有跑测试，也没有做动态沙箱。官方 nuclei 包与公开校验和一致，不能推出「跑 nuclei 是安全的」：3.3.7 仍落在第 1.4 节的公告范围内，并且扫描器会按模板访问操作者指定的目标。
