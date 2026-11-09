# sapper-v2 只读细查：v18-champion 镜像与 nvwabase

细查日期：2026-11-09  
对照的原审计：`cursor/security-audit-f1ff` 上的 `SECURITY_AUDIT.md`，提交 `9f7cf18290106eefa43457868db2f0283800c2cc`（2026-11-09）。那份审计审的是本 fork 的树 `d4de0770ae1fba706f00b7e29197ce739adbea94`。本文件只补充它当时标成未确认的两件事，不改原审计、不改代码。

对象：

- GitHub Release `v18-champion` 的 `sapper-v18-mixed.tar.gz`（`N1nEmAn/sapper-v2`）
- `N1nEmAn/nvwabase` 的 `master`，提交 `6a55bf5793a948f0d1a5f9f790acbeba4c6cc891`（2026-11-09）

方式：只读。下载发布包后用 tar 解出 OCI/docker save 布局，按层读文件名和文本，没有 `docker load`，没有运行镜像、二进制或仓库脚本。本环境没有 `crane`/`skopeo`，注册表只走了 HTTPS API。敏感口令字段没有写入本文。

## 1. 结论摘要

**(1) v18-champion 预构建镜像。** 发布包字节与 GitHub 上的摘要一致；25 层都能对上「Kali 根文件系统 + 三份 Dockerfile 叠出来的构建历史」，没有发现构建历史解释不了的私钥、SSH 密钥或额外开机项。与当前仓库文本有冲突的是：基础阶段少装了一批后来写进 `docker/Dockerfile` 的包、多删了 `seclists/Passwords`、Codex 配置里仍指向竞赛网关主机、发布说明中的提交 `8e03190` 和「63 条知识库」对不上镜像。对原审计「镜像没拆开，所以可疑」：**下调**。对「冻结提交 `8e03190`」：**维持无法确认**。对「`agent` 会不会因为 Kali 默认策略拥有 sudo」：**下调**为已核实没有。

**(2) nvwabase。** 它不是本仓镜像的 `FROM`，而是同一作者的公开 Python 标准库仓库；没有 tag、没有 lock、没有第三方依赖，也没有读到外联载荷或写死的凭据。`fgc capture` / `nvwa listen` 是文档里的本机 tmux 读写。对原审计第 4 节「nvwabase 未审」：**由无法确认改为已核实**，不需要上调成发现了隐藏通道。对「sapper 没有把这个仓库钉在某个提交」：**维持**。

发行版依赖里新确认的一条，不改变上面两条的方向：镜像中的 `cryptography` 49.0.0 落在 GHSA-g6cj-pr64-35w5 的影响范围内。见第 3 节。原审计说 Kali 包没有版本、CVE 未确认；这一条改为已确认，其余 apt 包没有逐个对公告。

## 2. 逐项细查

### 2.1 v18-champion 镜像

**发布物身份：已核实。**

| 字段 | 值 |
|---|---|
| 标签 | `v18-champion`，发布于 2026-11-01T13:31:33Z，非 draft |
| 文件 | `sapper-v18-mixed.tar.gz`，1,227,645,241 字节 |
| 本次 SHA256 | `22b0014b55c23f5e6ef8fd976045e6a551d91b814ae82921996f8f9af3c3fb77` |
| GitHub asset digest | 同一个 SHA256 |
| 解压后布局 | `oci-layout`、`index.json`、`manifest.json`、`repositories`、`blobs/sha256/` |
| 镜像名 | `sapper-agent:commander`（注解写着 `docker.io/library/sapper-agent:commander`） |
| 配置 blob | `sha256:f463b5ebc8b9e61b545013ebffab4d48332270776d30a06308ceec79fc0715d3`（13,354 字节，与文件名一致） |
| 架构 / 用户 / 工作目录 | `linux/amd64`，`User=agent`，`WorkingDir=/work` |
| 入口 | `["/opt/sapper/docker/commander/entrypoint.sh"]` |
| 配置里的创建时间 | `2026-10-17T10:51:10+08:00` |
| 构建器标签 | `io.buildah.version=1.43.2` |

`index.json` 里的镜像清单是 `sha256:e52316dd41ef35764588ff29a490aaa359431b12f80c1b53eac6dcf5c86ad594`。`repositories` 把 `sapper-agent:commander` 指到最上一层 `66f6007a370a…`，这是 docker save 的旧式顶层指针，不是另一份根文件系统。

环境变量里没有密钥，只有 `PATH`、`LANG`、`DEBIAN_FRONTEND`、`PYTHONUNBUFFERED` 和 `SAPPER_*`。最终 `SAPPER_MODEL=glm-5.3[1m]`，与当前 `docker/Dockerfile.commander` 第 51 行一致，不是基础 Dockerfile 第 16 行的 `glm-5.3-flash[1m]`。

**发布说明里的提交号：有冲突，且该提交无法确认。**

发布说明写「Frozen at commit 8e03190 (2026-10-18)」，并写知识库有 63 条。

- `GET /repos/N1nEmAn/sapper-v2/commits/8e03190` 返回 422，`No commit found for SHA: 8e03190`。原审计已经记录过这一点。
- 在全部层 tar、配置和抽取出的 sapper 文本里检索 `8e03190`，没有命中。
- 配置创建时间是 2026-10-17，比发布说明的 2026-10-18 早。
- 最终知识库是 `kb001.json`–`kb056.json`，56 个文件。两层 `COPY kb/` 都没有 `kb057` 及以上。这 56 个文件的 SHA256 与当前树 `d4de077` 相同。

所以：镜像是 2026-10-17 的一次构建；「它就是公开提交 `8e03190`」无法确认，而且和镜像元数据、知识库个数冲突。

**25 层与构建历史：已核实，历史命令和层一一对应。**

配置里 43 条 history、25 个非空层。每个层文件的 SHA256 都等于它的文件名，也等于 `rootfs.diff_ids` 里的那一项。下表「历史」列是对应的 `created_by` 摘要。成员很多的层只按目录汇总；小层把路径写在后面。

| # | 层（前 12 位） | 字节 | 成员 | 历史在说什么 | 层里实际有什么 |
|---|---|---:|---:|---|---|
| 1 | `52ed1a34df98` | 125,312,512 | 5,640 | Kali 镜像的 `CMD ["bash"]`。前面的 `ADD` 被标成 empty layer，根文件系统落在这一条上 | Kali 最小根文件系统。`/usr/lib/os-release` 为 Kali Rolling，`VERSION_ID=2026.3`。`kali-defaults` 2026.3.2。标签写官方镜像、revision `4d1ddb93`、版本 `2026.10.11`、创建于 2026-10-11 |
| 2 | `346dbc420675` | 4,608 | 5 | 把 apt 源写成清华 Kali rolling | 只改了 `/etc/apt/sources.list` 和 `apt.conf.d/80-retries` |
| 3 | `dbe82ff308b3` | 1,982,462,976 | 67,596 | 基础 apt：证书、网络工具、nmap、sqlmap、hydra、seclists、sudo 等，并删文档 | 约 5.1 万个路径在 `usr/share`，1.2 万在 `usr/lib`。白文件只有删掉的 `/usr/share/doc`、`/usr/share/man` 和 apt lists。没有 `seclists/Passwords`、`IOCs` |
| 4 | `8029d3e78b37` | 29,060,608 | 2 | `COPY docker/nuclei.zip` | `/tmp/nuclei.zip`，29,058,291 字节 |
| 5 | `891e80b41086` | 85,188,608 | 6 | 解压 nuclei 并删除 zip | `/usr/local/bin/nuclei`，以及 `tmp/.wh.nuclei.zip` |
| 6 | `ea2d590a6c66` | 270,177,280 | 630 | `npm install -g` claude-code，并清 npm cache | `/usr/local/lib/node_modules/@anthropic-ai/claude-code`，`/usr/local/bin/claude` 符号链接。另有 `/tmp/node-compile-cache`（601 项）和两份 npm 调试日志 |
| 7 | `c43074bbafb0` | 201,013,248 | 10,958 | `pip3 install` requests、pyjwt、`unicorn==2.0.1.post1`、pwntools | 几乎全在 `/usr/local`（Python 3.14 site-packages） |
| 8 | `c7b9e3c624d4` | 615,424 | 15 | `COPY lib/` | `/opt/sapper/lib/` 下 6 个 `.py`、1 个 HTML，以及 5 个 `cpython-314.pyc` |
| 9 | `5bc2a9b8721d` | 29,696 | 9 | `COPY templates/` | 6 个模板 markdown |
| 10 | `d3dc8f2c0b2f` | 79,872 | 48 | 第一次 `COPY kb/` | `kb/entries` 下的一批 json |
| 11 | `93cee127a1c3` | 26,624 | 5 | `COPY README.md LICENSE SKILL.md` | 这三个文件 |
| 12 | `d46b5fb833e8` | 795,648 | 104 | `useradd agent`，做 sapper 符号链接并 `chown` | `/etc/passwd`、`shadow`、`group` 更新；`/home/agent` 来自 `/etc/skel`；`/home/agent/.local/bin/sapper` → `/opt/sapper/lib/sapper.py`；`chown` 把 `/opt/sapper` 再写进这一层 |
| 13 | `68774421382e` | 6,656 | 4 | `COPY docker/entrypoint.sh` | `/usr/local/bin/sapper-entrypoint`（3,305 字节） |
| 14 | `75ba930695a5` | 6,656 | 4 | `chmod +x` 该入口 | 同一文件，模式变为可执行 |
| 15 | `5f70bf18a086` | 1,024 | 0 | `WORKDIR /work` | 空层。这是 Docker 用来表示「这一步不产生文件」的固定空 tar |
| 16 | `b9dab970b8db` | 8,475,136 | 867 | apt 安装 tmux | `tmux` 3.7c-1 及其库。主要落在 `/var/lib`、`/usr/bin` |
| 17 | `f982b7022417` | 18,432 | 4 | `COPY docker/tmux-drive.sh` | `/opt/sapper/docker/tmux-drive.sh` |
| 18 | `9769b7a41213` | 6,144 | 4 | `COPY docker/entrypoint-tmux.sh` 覆盖入口路径 | `/usr/local/bin/sapper-entrypoint` 换成 2,947 字节 |
| 19 | `b998d5f2e0d1` | 23,552 | 8 | `chmod +x` tmux 入口和 drive 脚本 | 上述两个文件的可执行副本 |
| 20 | `ab6d0ed0e503` | 393,943,040 | 7,670 | commander 阶段 apt（binwalk、steghide、exiftool、数据库客户端、gdb、checksec、GMP）再加 pip | `/usr` 下的这些包，以及 `/usr/local` 里的 pycryptodome、sympy、ROPgadget、gmpy2、z3-solver。`/etc/profile.d/gawk.sh`、`gawk.csh` 随包进来。`/root/.cache/pip/http-v2` 留下 3 个缓存体 |
| 21 | `5a686f92b8fc` | 286,197,760 | 65 | `npm install -g @openai/codex@0.42.0` | `/usr/local/lib/node_modules/@openai/codex`，含各平台 `vendor/`。`/usr/local/bin/codex` 为符号链接。`/root/.npm/_cacache` 留下 tarball 缓存和一份调试日志 |
| 22 | `a435ebdadbf1` | 120,320 | 11 | `COPY docker/commander/` | `SKILL.md`、`SPEC.md`、`entrypoint.sh`、`watchdog.sh`、`worker.sh`、`codex-home/config.toml` |
| 23 | `86a2af104ecb` | 99,328 | 60 | 第二次 `COPY kb/` | 最终的 56 个条目，盖过第一次拷贝 |
| 24 | `c4ef18b8e9aa` | 4,096 | 4 | 把 codex 配置拷进 agent 家目录 | `/home/agent/.codex/config.toml`（635 字节） |
| 25 | `66f6007a370a` | 120,320 | 11 | `chmod +x` commander 的三个脚本并 `chown` | 与第 22 层相同的 6 个文件，模式为可执行 |

空历史（ARG、LABEL、ENV、USER、最终 ENTRYPOINT）没有对应层。最终入口是配置里的 commander 脚本，不是 `/usr/local/bin/sapper-entrypoint`。后者留在镜像里，是 tmux 阶段盖过基础入口后的文件，哈希 `d588ac9d849155242ddc64460c3595265e59fb69f08cd674357bce1235b1bd17`，与当前 `docker/entrypoint-tmux.sh` 一致。commander 入口、`worker.sh`、`watchdog.sh`、`tmux-drive.sh`、`SKILL.md`、`SPEC.md` 的哈希也与当前树一致。

**和当前 Dockerfile 的差异：有冲突，差异都在构建命令文本里，不是层外来的文件。**

当前 `docker/Dockerfile` 第 42–45 行在基础阶段就安装 `binwalk`、`steghide`、`exiftool`、`libgmp-dev`、`libmpfr-dev`、`libmpc-dev`、数据库客户端、`gdb`、`checksec`。镜像的基础 apt 命令没有这些。除 `libmpc-dev` 外，它们出现在第 20 层的 commander apt 命令里，和当前 `docker/Dockerfile.commander` 第 17–20 行一致。`libmpc-dev` 这个包名没有出现在任何 apt history 里；镜像里名为 libmpc 的文件是 gmpy2 wheel 自带的 `gmpy2.libs/libmpc-*.so`，不是那个 apt 包。

当前 `docker/Dockerfile` 第 55–57 行删除 seclists 的 IOCs、Misc、Username_Lists、Discovery/DNS，不删 `Passwords`。镜像基础 apt 命令多删了 `/usr/share/seclists/Passwords`。层内检索没有这个目录。

当前基础 Dockerfile 第 76–82 行还 pip 安装 pycryptodome、sympy、ROPgadget、ropper，以及可失败的 gmpy2、z3。镜像把前三个和 gmpy2、z3 放到 commander 阶段。`ropper` 的 dist-info 不存在，和 commander Dockerfile 里「ropper 失败则继续」一致。

被盖掉的第一份入口（第 13 层）和当前 `docker/entrypoint.sh` 第 17 行只差错误提示里的示例主机：仓库写 `https://open.bigmodel.cn/api/anthropic`，镜像那一版写 `http://open.bigmodel.cn.tsecbench.gw/api/anthropic`。最终根文件系统里的入口已经换成与仓库一致的 tmux 脚本和 commander 脚本，这行示例不在最终入口里。

**源码树里没有、但镜像里有的 sapper 文件：5 个 pyc。已核实它们是同目录 `.py` 的 3.14 字节码，没有看到额外 URL。**

`/opt/sapper` 最终 83 个文件。与 `d4de077` 对比：73 个哈希相同；5 个不同（下面）；5 个只在镜像里。`COPY lib/` 层和后面 `chown` 层里的这 5 个 pyc 哈希相同，`chown` 没有改内容。没有执行它们。从字节码里抽出的可打印字符串，和同层 `.py` 相比，多出来的是代码对象名、类型名和已经写在源码里的句子。`sapper.cpython-314.pyc` 里的 `http://` 紧挨着源码里的 `[view] serving fact-graph`，对应 `http://{host}` 这种拼接，不是另一个主机名。

| 文件 | 字节 | SHA256 |
|---|---:|---|
| `lib/__pycache__/_settings_patch.cpython-314.pyc` | 13,905 | `02419cdbdbc7ecb9d6d6b639f1150d615c2a7322ad6ff8772e887523631ee117` |
| `lib/__pycache__/arena.cpython-314.pyc` | 15,194 | `f28b7cd00aa598f68f9ff6f78986186e7fb4806d728a7deb947a70f4ff5cfb93` |
| `lib/__pycache__/kb.cpython-314.pyc` | 14,989 | `507fec3081799fd941a1eb17e5c290f61893e6e2d9f1e674649da183bb788b28` |
| `lib/__pycache__/sapper-hook.cpython-314.pyc` | 11,046 | `4171576002f63ecb72a45fde622c03884521ad9f0c93e38e5c62202b9cb18d2e` |
| `lib/__pycache__/sapper.cpython-314.pyc` | 290,269 | `d12488a3e39b4228072b1cbc7f1b79487f0c5cd7e424fd2d7064fae6bada74c1` |

当前 `.dockerignore` 第 4–5 行忽略 `*.pyc` 和 `__pycache__`。镜像里仍有它们，说明 2026-10-17 那次构建没有用今天这份忽略规则。它们不在 git 里。

五个与当前树不同的文本文件：

| 路径 | 镜像 SHA256 | 当前树 SHA256 | 差异 |
|---|---|---|---|
| `README.md` | `c61f8a8a51eb2d739c2d215c30cb2ab2946457c2e7ef60a0fb82b147809172c9` | `900a0ef215bee4c65964b2fc308b68e0e3cccbb44a69b6001850ff9668b8fe7a` | 镜像是 5,436 字节的旧简介。当前 README 是后来的冠军介绍。文档差异 |
| `docker/commander/codex-home/config.toml` | `aecf34936056f4d0f7099591d38acf9e4ba48370c2c45b0363c007e229f3c548` | `55e35c42ca34b2cb870b45bbd77a15ee48618865d2b23ea2c3174f94994f25e4` | 见下。同一份也在 `/home/agent/.codex/config.toml` |
| `lib/arena.py` | `611263ce45b26b4e89f0e332f22a49c21220146d0c22a6b9a4889c11df148763` | `99296ebaceb793fd5e3a9f0e5b3b8e845c4d5d72a2175a8913c9951b95fe4451` | 只差第 23–24 行注释。镜像写竞赛网关使用 `http`；当前仓库改成「direct vendor endpoints」 |
| `lib/sapper-hook.py` | `99d1d36cc76a93778656abbb97ca0392a8fa8d02352dd82b94df849fee4d592f` | `fc1595f8908c1e194cdc0c32b966915d2c02bf113bdc6904dfd9b282d8ec5ff3` | 镜像第 38 行附近仍是 `import fg`。当前树在 `c91881d` 改成 `import sapper`。镜像里的钩子会去找一个不存在的模块名 |
| `lib/sapper.py` | `6eedb2f4519a19ee75799b0f46b4709834332047d69e0347adc5ea83fcf533b2` | `24663ab6810e21926fb6f99e153a8e02ce2866c9bd36f0fa78c9407619872895` | 4,726 行里只有 arena 子命令的调用名不同：镜像在约 1787、1819–1829 行调用 `submit` / `challenges` / `start` / `stop` / `hint`。同文件约 1986 行的 bench 路径，以及两边的 `arena.py` 第 179–209 行，用的是 `submit_flag` 等名字。这是镜像内部调用名不一致，不是多出来的请求目标 |

`config.toml` 第 8 行：镜像是 `base_url = "http://api.deepseek.com.tsecbench.gw/v1"`，当前仓库是 `https://api.deepseek.com/v1`。两边第 14 行的 `deepseek-local` 都是 `https://api.deepseek.com/v1`。文件里没有密钥，只有环境变量名 `DEEPSEEK_API_KEY`。这是构建上下文里的配置，不是层外塞进来的文件。

仓库里有、镜像 `/opt/sapper` 里没有的，是 Dockerfile 本来就没 COPY 的脚本和文档：`docker/Dockerfile*`、`docker/build.sh`、`docker/export.sh`、`docker/local-sim.sh`、`docker/ENV.md`、`docker/entrypoint.sh`、`docker/entrypoint-tmux.sh`。后两个入口以 `/usr/local/bin/sapper-entrypoint` 的形式存在，不在 `/opt/sapper`。

**家目录里看起来陌生的文件：已核实来自基础镜像的 `kali-defaults`，不是 sapper 拷进去的。**

`useradd -m` 会把 `/etc/skel` 复制到 `/home/agent`。基础层的 `var/lib/dpkg/info/kali-defaults.list` 列出了这些路径：

- `/etc/skel/.bashrc`、`.zshrc`、`.zprofile`
- `/etc/skel/.config/powershell/Microsoft.PowerShell_profile.ps1`
- `/etc/skel/.java/.userPrefs/burp/prefs.xml`

`/etc/skel/.bashrc` 的 MD5 `475e1be6622d3102c94f277c6f06cb49` 与该包 `md5sums` 里 `usr/share/kali-defaults/etc/skel/.bashrc` 的一行相同，文件都是 5,578 字节。PowerShell 文件是 Kali 的双行提示符函数。Burp 的 `prefs.xml` 只关了 JRE 提示和更新提示，没有许可证或服务器地址。zshrc 是 Kali 的提示符配置。这三份在包的文件清单里，但不在 `md5sums` 里，所以没有用包内校验和再对一次字节。文本里没有 `curl`/`wget`/远程下载。

**凭据与账号：已核实没有预置私钥；口令字段是锁定状态。**

全层文件名检索没有 `id_rsa`、`id_ed25519`、`authorized_keys`、`.ssh`、`.npmrc`、`.netrc`、ssh host key。`.pem` 命中的是发行版 CA 证书链接和 `cacert.pem`。

`/etc/shadow` 里 `root` 和 `agent` 的口令字段长度为 1，属于锁定标记，这里不抄字段本身。`agent:x:1000:1000`，主组是 `agent`，不在 `sudo` 组。`sudo:x:27:` 的成员为空。`/etc/sudoers` 只有 `root` 和 `%sudo` 两条授权，外加 `#includedir /etc/sudoers.d`。没有名为 agent 的授权行。`sudoers.d` 在 apt 层里是空目录。这把原审计「agent 是否因 Kali 默认策略而有 sudo」从无法确认改为已核实没有。

对 `/opt/sapper`、`/home`、`/root` 下抽出的文本做了私钥头、云厂商访问密钥前缀、常见聊天机器人口令前缀和 `ANTHROPIC_AUTH_TOKEN=` / `DEEPSEEK_API_KEY=` 赋值扫描。命中的都是脚本里的变量名，赋值右侧是引用或引号，不是一段写死的密钥。npm 调试日志里的 URL 只有 `https://registry.npmjs.org/...`，没有认证头。

**启动项：已核实。最终入口就是仓库里的 commander 脚本。**

- 配置入口：`/opt/sapper/docker/commander/entrypoint.sh`，用户 `agent`，工作目录 `/work`。该脚本哈希与当前仓库一致。
- 基础层自带 `/etc/cron.daily/apt-compat` 和 `dpkg`。sapper 的层没有再加 cron。
- `kali-defaults` 的文件清单包含 `regenerate-ssh-host-keys.service`，校验和与包内 `md5sums` 一致。镜像里没有配好的 host key。sapper 的层没有新增 systemd unit。
- `/etc/profile.d` 里随后出现的是包自带的 `nmap.sh`、systemd 的两个脚本，以及 commander apt 带进的 `gawk.sh` / `gawk.csh`。
- 没有 `rc.local`，没有写 crontab、`authorized_keys` 或家目录启动钩子。`/home/agent` 的 shell 启动文件就是上面的 skel。

**基础镜像是不是「当天的官方 Kali」：内容与包清单相符，字节级相等无法确认。**

层内标签：`org.opencontainers.image.source=https://gitlab.com/kalilinux/build-scripts/kali-docker`，revision `4d1ddb93`，vendor OffSec，title 为 Kali Linux kali-rolling。GitLab 上 `4d1ddb93a862f6713ca34cd5d83abf4d89150513` 存在，标题是 apt 源格式改动，作者日期 2026-05-14，早于镜像标签上的 2026-10-11。revision 没有随每次镜像构建更新。

2026-11-09 查询 Docker Hub `kalilinux/kali-rolling`，`latest` 更新于 2026-11-08，digest 前缀 `sha256:6ec743d9fc5cd`，与原审计记录的 `sha256:6ec743d9fc5cd397451eeddeff3869bcfab5de9bd5dec5b1f34cbfbd80313073` 一致。那不是本镜像的基础层 `52ed1a34df98…`。Hub 上这个仓库当前只有 latest 和架构标签，没有 2026-10-11 的日期标签，所以不能把官方历史层拉下来做字节对比。

**这只是 GitHub Release 上的 tar，不是已发布的注册表镜像：已核实抽查的仓库名不存在；GHCR 无法确认。**

Docker Hub `library/sapper-agent`、`n1neman/sapper-agent`、`n1neman/sapper` 均为 404。配置注解里的 `docker.io/library/...` 是 buildah 给本地标签加的名字。`https://ghcr.io/token?service=ghcr.io&scope=repository:n1neman/sapper-v2:pull` 返回 403，没有列出标签。

### 2.2 nvwabase

**它在本仓里的角色：已核实，不是镜像基础，也没有被钉住。**

当前 Dockerfiles 的 `FROM` 是 `kalilinux/kali-rolling:latest`、`sapper-agent:latest`、`sapper-agent:tmux`。没有任何一行 `FROM` nvwabase，镜像层里也没有 `nvwa` 或 `fg.py`。

本仓只在 README 里让人自己克隆：

- `README.md` 第 116–131 行：`git clone https://github.com/N1nEmAn/nvwabase`，然后运行名为 `nvwabase` 的命令。
- `README.zh-CN.md` 第 76–90 行：同样使用 `nvwabase` 这个命令名，并写成 `` `nuwa spawn` ``。

`install.sh` 不克隆它。没有 submodule，没有 lock。镜像是 2026-10-17 构建的；nvwabase 里实现 spawn/say/listen 的提交在 2026-11-01 之后，所以这份冠军镜像不可能含有今天的 `nvwa` 脚本。

**仓库身份：已核实是公开的同作者仓库；提交未签名，没有 tag。**

| 字段 | 值 |
|---|---|
| 仓库 | `N1nEmAn/nvwabase`，2026-06-14 创建，非 fork，MIT，默认分支 `master` |
| 本次 HEAD | `6a55bf5793a948f0d1a5f9f790acbeba4c6cc891` |
| 发布 / tag / `.github` | 都没有。GitHub contents API 对 `.github` 返回 404 |
| 体积 | 树 29 个 blob。最大的是 `lib/fg.py`，134,348 字节，SHA256 `afdbc965e328c87001b2063ed10e2dd90a43b8c12006fa88e3eb3dea0ff01fad` |
| `nvwa` | 8,715 字节，SHA256 `23da9c7916db619bfa61df97b9925183d80005203aed21f414c336505ef0cf18` |
| `install.sh` | 3,660 字节，SHA256 `68ee7fc981ade5bde573a2146634f012b70551a25ec23df6bc448faf203e278c` |
| 版本字符串 | `lib/fg.py` 第 32 行 `__version__ = "0.2.0"`。没有 git tag 把 0.2.0 钉住 |

全部提交的 `%G?` 都是 `N`（未签名）。作者是 `N1nEmAn`（`1332037591@qq.com` 或 `users.noreply.github.com`）、`VarFusion <varfusion@localhost>`，以及 2026-07-02 的两条 `Codex Ops <codex@localhost>`：`7502b2c072b3`（tmux peer messaging）和 `aac0a0b631b9`（authorized peer capture）。这两条改动今天仍在 `lib/fg.py` 里，没有被后续提交藏进别的文件。`localhost` 邮箱不能用来核对真人。这和 sapper 仓库里 `VarFusion <varfusion@localhost>` 的情况同类。

**内容和可疑代码：已核实没有网络载荷；本机 tmux 能力是写在文档里的。**

树里没有 Dockerfile、没有 `package.json`、没有 requirements、没有已编译的可执行文件。两张 JPEG 都以 JPEG 结束标记结尾，后面没有尾随字节。源码检索没有零宽字符。

`install.sh` 只做符号链接：`~/.local/bin/fgc` → `lib/fg.py`，`~/.local/bin/nvwa` → `./nvwa`，以及 `~/.claude/skills/fgc-setup`。它不下载。第 72 行会执行刚链上的 `fgc --version`。本次没有执行它。它不创建名为 `nvwabase` 的命令。这和 sapper README 第 118 行起写的 `nvwabase setup` 对不上：**有冲突**，冲突在文档里的命令名，不在多出来的二进制。

`nvwa` 第 40–62、87–145 行用 tmux 在会话 `NVWA_SESSION`（默认 `nvwa`）里起窗格、粘贴文字、读窗格。默认 `--agent` 是 `claude`。没有命令白名单，也没有把读到的文本发到网络。`listen` 只接受这个会话里已经存在的窗口名。

`lib/fg.py` 第 2624–2897 行是另一套：`peers --add` 把 tmux target 写进项目里的 `.fg/ai-peers.json`，`send` 往该窗格粘贴，`capture` 读取该窗格最近输出并可追加到本地 `ai-channel.txt`。第 2682 行的发现函数列出本机 tmux 服务上的窗格。代码在调用 capture 前要求目标已经写在这个 JSON 里；同一个 CLI 的 `--add` 就能写这个 JSON。文档（`SKILL.md`、生成的 AGENTS 片段第 2945–2951 行）要求先由使用者授权。这是本地文件门闩，不是单独的审批凭据，也没有外联。2026-07-02 那两次提交加的就是这段，不是后来塞进去的混淆代码。

其他边界，和原审计对 sapper 的描述同类，不重复展开成操作步骤：

- `run_claude`（第 1234–1254 行）只起本地 `claude` 子进程。`--skip-permissions` 会加上 Claude Code 自己的权限开关。没有在这里写死网关主机。
- `cmd_setup`（第 2126–2144 行）只改当前项目的 `.claude/settings.json`。`_settings_patch.py` 第 6 行的说明提到 `install.sh --global-hooks`，当前 `install.sh` 没有这个参数。直接运行 `_settings_patch.py` 且不给路径时，第 240–243 行会落到用户级 `~/.claude/settings.json`。
- `fg view --serve` 默认 `127.0.0.1:8765`（第 3219–3220 行）。非回环地址会打印警告（第 2394–2396 行）。
- 钩子 `lib/fg-hook.py` 只在当前目录已有 `.fg/` 时往模型上下文注入图的文本，并声明图内容不可当指令。它调用本地 `date`。

**仓库自己的文档和代码不一致：有冲突，不影响「没有隐藏包」这一条。**

- `README.md` 第 49–53 行写本仓只交付历史能力，造人和语言在 sapper 里。同文件第 97–108 行和第 4 节却在教 `nvwa spawn` / `say` / `listen`。`nvwa` 文件已经实现这三件事。`README.zh-CN.md` 第 35 行仍写只交付历史支柱。
- `README.md` 第 77 行写 `fgc fact ... -k host/service/cred/vuln/loot/flag`。`fact` 子命令（`lib/fg.py` 第 2996–3007 行）没有 `-k`。类型化事实不在这个解析器里。
- 路线图第 120–122 行仍把通用的 spawn/say 写成尚未抽出的计划，和已经在树里的 `nvwa` 重复。

最新提交 `6a55bf57` 把 CLI 名从 `nuwa` 改成 `nvwa`。sapper 的中文 README 还写着 `nuwa`。克隆今天的默认分支，得到的命令名和 sapper 文档都不一样。因为没有 tag，下一次 `git clone` 还会再变。

## 3. 供应链补充

镜像把「仓库没锁版本」冻成了 2026-10-16/17 的具体版本。下面是从层里的 `dist-info`、`package.json` 和 `dpkg status` 读到的，再和注册表、OSV 对过。原审计 1.4 节查的是 2026-11-09 的索引最新版；那些「最新版 0 条」不能直接套到这个镜像。

**锁死情况。**

| 组件 | 镜像里的版本 | 仓库有没有写死 | 与上游字节 |
|---|---|---|---|
| nuclei | 与仓库 `docker/nuclei.zip` 相同 | zip 在 git 里，仓库没有附 checksum 文件 | **已核实。** zip SHA256 `725ef892fcffd1b03ad4f0874942fc4b623c0419b6b6c6c91fe4a5a65671f77c`，等于 ProjectDiscovery `nuclei_3.3.7_linux_amd64.zip` 那一行（原审计已对过）。解压出的 `/usr/local/bin/nuclei` 85,184,512 字节，SHA256 `9121735553c5bd2bf0a2785677b8457c01b99dacc361e83e3a8734843fe5bc57`，等于该 zip 内 ELF |
| `@anthropic-ai/claude-code` | 2.1.301 | Dockerfile 不锁版本 | **已核实。** 包装 tarball 的 sha512 与 registry 一致。`package.json` SHA256 `e2c5eab93f177f3f3df1ce2757c66245a054bee6f89373c5400b6f8a0fe2962b`，`install.cjs` SHA256 `32a7b2429d7447c5aa043e383bcf9a72e0b233dff66336ea4b5b32531da981ad`，都与 `@anthropic-ai/claude-code-2.1.301.tgz` 内文件相同 |
| `@anthropic-ai/claude-code-linux-x64` | 2.1.301 | 不在仓库里，由上面的 postinstall 选择平台包 | **已核实。** 官方 tarball sha512 核对通过。包内 `package/claude` 267,955,384 字节，SHA256 `bc1aad9982c7b59c7d889ff92595d61cf46e0a6eae074cb551d932a989bcd222`，与镜像 `bin/claude.exe` 相同。构建日志只下载了这个平台的 tarball，没有把 darwin/win32 二进制装进层里 |
| `@openai/codex` | 0.42.0 | `docker/Dockerfile.commander` 第 32 行钉了版本，没有把 integrity 写进仓库 | **已核实。** 官方 `codex-0.42.0.tgz` sha512 核对通过。`package.json`、`bin/codex.js`、`bin/rg`、`vendor/x86_64-unknown-linux-musl/codex/codex`（31,691,168 字节，SHA256 `fd337adc73eb863cd1b879caf382a03cf056af60fba2e437dea8273ec2927b72`）和同目录的 `rg` 都与镜像相同。这个包自己带了 darwin、windows 和另一种 Linux 的 vendor 二进制，所以第 21 层有它们；不是单独再下载的。`bin/rg` 是包内的 dotslash 文本，指向上游 ripgrep 发布物；对应平台的 `rg` 二进制已经在 vendor 里，哈希与 tarball 一致 |
| Python 直接依赖 | 见下表 | 只有 unicorn 在 Dockerfile 里写了 `==` | 没有把 wheel 再下载一遍做哈希。版本来自镜像内 METADATA |
| apt | 见下表 | 没有版本，也没有哈希 | 版本来自镜像内 dpkg status / apt history，没有再对 Kali 仓库的 Packages 文件 |
| nvwabase | HEAD `6a55bf57`，自称 0.2.0 | sapper 没有 pin | 无第三方包，不适用 wheel/tarball 校验 |

镜像内直接相关的 Python 版本：`pyjwt` 2.15.1，`pwntools` 4.15.0，`unicorn` 2.0.1.post1，`pycryptodome` 4.0.0，`sympy` 1.14.0，`ROPgadget` 7.7，`gmpy2` 2.3.2，`z3-solver` 5.3.1.0，`requests` 2.34.2（在 `/usr/lib/python3/dist-packages`，没有看到 `/usr/local` 下的第二份）。传递依赖里抽到的包括 `paramiko` 5.0.0、`cryptography` 49.0.0、`urllib3` 2.8.0。`ropper` 不在镜像里。

apt 抽查：`python3` 3.14.7-3，`nodejs` 24.21.0，`npm` 12.0.2，`nmap` 7.991+dfsg-1kali1，`tmux` 3.7c-1，`sudo` 1.9.17p2-8，`kali-defaults` 2026.3.2。基础层 dpkg 状态约 78 个包，加上两轮 apt 之后约 844 个包。没有对这 844 个做公告穷举。

**OSV / GHSA / 恶意包名，查询日 2026-11-09。**

对镜像里的精确版本调用 OSV `querybatch`。`pycryptodome` 4.0.0、`z3-solver` 5.3.1.0、`pyjwt` 2.15.1、`pwntools` 4.15.0、`requests` 2.34.2、`urllib3` 2.8.0、`paramiko` 5.0.0、`unicorn` 2.0.1.post1、`gmpy2` 2.3.2、`sympy` 1.14.0、`ROPgadget` 7.7、`@anthropic-ai/claude-code` 2.1.301、`@openai/codex` 0.42.0 返回 0 条，没有 `MAL-*`。

`cryptography` 49.0.0 返回 GHSA-g6cj-pr64-35w5，别名 CVE-2026-69247、PYSEC-2026-3552。OSV 写的影响范围是 `>=44.0.0`、`50.0.0` 修复。摘要是：PKCS#7 EnvelopedData 解密在错误返回和时间上可区分，应用若解密不可信输入并暴露结果，会形成针对内容密钥的 oracle。这里不写触发条件的操作细节。该包在发行版路径 `/usr/lib/python3/dist-packages`，是 apt 带进来的，不是 Dockerfile 的 pip 行。sapper 源码没有调用公告里点名的那几个解密函数。nuclei 3.3.7 的公告仍以原审计 1.4 节为准；本次只确认镜像里的二进制就是那一版。

nvwabase 没有可查询的依赖。OSV 对 PyPI/npm 上的名字 `nvwabase`、`fgc`、`nvwa` 返回空结果。PyPI 没有 `nvwabase`、`fgc`、`sapper`。同名但不是这个项目的包：PyPI `nuwa` 0.0.1（作者 pypiname）、PyPI `nvwa` 0.1b1（Linlin Jia）、npm `fgc`（fanguocai）、npm `nuwa`（Rayi 的 UI 工具）、npm `nvwa`（huntbao）、npm `sapper`（Svelte 的旧项目）。sapper 文档让人 `git clone`，没有让人 `pip install` / `npm install` 这些名字。

**来源可信度。**

- 镜像的 sapper 文件多数与后来公开的 `d4de077` 哈希相同，差异是文档、注释、钩子导入名、arena 调用名和 Codex 主机。它不是一套对不上的外来源码，也不是发布说明所写的那个找不到的提交。
- Claude Code 与 Codex 的安装日志和文件哈希都落在 `registry.npmjs.org` 的官方包上。
- nuclei 与官方 3.3.7 zip 一致。
- Kali 基础层的包清单和 bashrc 校验和符合层内的 `kali-defaults`，但是 2026-10-11 的官方层 digest 今天已经不在 Hub 的标签上，字节级「就是官方镜像」无法确认。
- nvwabase 与 sapper 是同一个 GitHub 账号下的公开仓库，提交未签名，没有发布工件。可信度止于「公开 git 历史就是这些文件」，没有签名或发布校验和可以再对一层。

构建残留，不是第二套供应链：第 6 层的 `/tmp/node-compile-cache`，第 6 层和第 21 层的 `/root/.npm` 日志，第 21 层的 npm 缓存（其中一份 78,350,954 字节，等于官方 `codex-0.42.0.tgz` 的大小），第 20 层的 `/root/.cache/pip`。claude-code 的安装命令执行了 cache clean，日志还在。codex 的安装命令没有 clean。这些路径里没有读到密钥。

## 4. 未确认项与原因

- 公开提交 `8e03190` 仍不存在。镜像里也没有这个字符串。不能把镜像逐文件对到该提交。已能对上的是当前树 `d4de077` 的大部分复制文件。
- 基础层 `52ed1a34df98…` 与 2026-10-11 官方 `kalilinux/kali-rolling` 是否字节相同。Hub 不再提供那天的标签。层内的发行版文件清单和 bashrc 校验和支持「这是一份 Kali 根文件系统」，不是字节级证明。
- GHCR `n1neman/sapper-v2`：匿名 token 请求返回 403，没有标签列表。
- 844 个 apt 包里，除了抽查版本和 `cryptography` 这一条，没有做完整 CVE 对照。`ropper` 不在镜像里，不存在「镜像里的 ropper 版本」可查。
- pip wheel 没有按 PyPI 的 sha256 再下载复算。版本来自已安装的 METADATA。
- pyc 没有反编译执行。结论限于字符串与同目录 `.py` 的对比。
- 没有启动容器，所以没有观察入口脚本、nuclei 或模型 CLI 的实际出站。静态入口和配置见第 2.1 节。
- nvwabase 的 `install.sh` 第 72 行和测试没有运行。阅读范围是 `6a55bf57` 这棵树，不含未推送的本地文件。
- 原审计列出的、本仓 `.gitignore` 排除的 `HANDOVER.md` 等私有文档，镜像 `/opt/sapper` 里也没有。它们仍不在本次范围内。

## 5. 方法与读取过的 URL

做了：

- 读取本仓 `d4de077` 的 Dockerfile、`.dockerignore`、入口脚本、`config.toml`、`lib/arena.py`。
- `curl` 下载 Release 资源到沙箱，`sha256sum`，`tar -xzf` 得到 OCI 布局。用 Python `tarfile` 列成员、抽取文本、计算层文件和选定成员的 SHA256。没有 `docker load`，没有执行 ELF、`.pyc` 或 shell。
- 只读克隆 `N1nEmAn/nvwabase` 到沙箱并 `git log`。没有运行 `install.sh`、`nvwa` 或测试。
- 下载三个 npm tarball，核对 registry 给出的 sha512，再比对包内文件与镜像文件的 SHA256。没有执行其中的二进制。

没有做：对任何靶机、竞赛平台或配置文件里的网关主机发请求；安装本仓或 nvwabase；运行 nuclei、claude、codex。

读取过的 URL：

- `https://github.com/N1nEmAn/sapper-v2/releases/download/v18-champion/sapper-v18-mixed.tar.gz`
- `https://api.github.com/repos/N1nEmAn/sapper-v2/releases/tags/v18-champion`
- `https://api.github.com/repos/N1nEmAn/sapper-v2/commits/8e03190`（422）
- `https://api.github.com/repos/N1nEmAn/nvwabase`，以及该仓库的 commits、git trees、tags、releases、contents/.github
- `https://github.com/N1nEmAn/nvwabase`（只读 git clone）
- `https://hub.docker.com/v2/repositories/library/sapper-agent/`（404）
- `https://hub.docker.com/v2/repositories/n1neman/sapper-agent/`（404）
- `https://hub.docker.com/v2/repositories/n1neman/sapper/`（404）
- `https://hub.docker.com/v2/repositories/kalilinux/kali-rolling/tags?page_size=25&ordering=last_updated`
- `https://ghcr.io/token?service=ghcr.io&scope=repository:n1neman/sapper-v2:pull`（403）
- `https://gitlab.com/api/v4/projects/kalilinux%2Fbuild-scripts%2Fkali-docker/repository/commits/4d1ddb93`
- `https://registry.npmjs.org/@anthropic-ai/claude-code/2.1.301`
- `https://registry.npmjs.org/@anthropic-ai/claude-code/-/claude-code-2.1.301.tgz`
- `https://registry.npmjs.org/@anthropic-ai/claude-code-linux-x64/2.1.301`
- `https://registry.npmjs.org/@anthropic-ai/claude-code-linux-x64/-/claude-code-linux-x64-2.1.301.tgz`
- `https://registry.npmjs.org/@openai/codex/0.42.0`
- `https://registry.npmjs.org/@openai/codex/-/codex-0.42.0.tgz`
- `https://registry.npmjs.org/` 上对 `nvwabase`、`fgc`、`nuwa`、`nvwa`、`sapper`、`sapper-v2` 的包元数据（部分 404）
- `https://pypi.org/pypi/{nvwabase,fgc,nuwa,nvwa,sapper,sapper-v2}/json`
- `https://api.osv.dev/v1/querybatch`
- `https://api.osv.dev/v1/vulns/GHSA-g6cj-pr64-35w5`
- `https://api.osv.dev/v1/vulns/PYSEC-2026-3552`

原审计已经引用、本次没有重下的：`https://github.com/projectdiscovery/nuclei/releases/download/v3.3.7/nuclei_3.3.7_checksums.txt`。zip 与镜像内二进制的关系用本地哈希接上，没有重新获取该 checksums 文件。
