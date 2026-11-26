# 供应链（sapper-v2 @ 7b75e6f1）

对象：<https://github.com/N1nEmAn/sapper-v2/tree/7b75e6f1f5d5b4ae0c55878b0dd0994d9b08f1fb>。  
查询时刻：**2026-11-26 09:38–09:39 香港时间**（UTC+8）。OSV 用 `api.osv.dev`，包元数据用 pypi.org 与 registry.npmjs.org，Nuclei 校验和用 GitHub Release 上的官方 `nuclei_3.3.7_checksums.txt`。未安装依赖，未拉取镜像层。

## 结论

已核对的直接依赖名称，以及仓内已提交的 Nuclei 压缩包，**未见恶意替换**。不能把这句话扩到「构建时才会解析的未锁定版本」或「没拉取的镜像层」。Nuclei 压缩包与官方 v3.3.7 字节一致，但该版本有后续公开漏洞，单列在恶意判断之后。

## 清单

| 类别 | 本树里有什么 | 版本是否固定 | 校验和 | 来源 |
|---|---|---|---|---|
| Python 应用代码 | `lib/*.py`，仅标准库 | 不涉及第三方包 | — | 本仓 |
| pip（只出现在 Dockerfile） | `requests` `pyjwt` `pwntools` `pycryptodome` `sympy` `ROPgadget` `ropper` `gmpy2` `z3-solver` 不锁定；`unicorn==2.0.1.post1` 锁定 | 仅 unicorn | 无 lockfile，无 hash | 未写 `--index-url`，按默认 PyPI |
| npm | `@anthropic-ai/claude-code` 不锁定，且 `--allow-scripts`（`docker/Dockerfile` 第 71 行）；`@openai/codex@0.42.0` 锁定版本，同样允许脚本，失败被 `\|\| echo` 吞掉（`docker/Dockerfile.commander` 第 32–33 行） | 仅 codex 的版本号 | 无 integrity | 未写 registry，按默认 npm |
| 锁文件 | 无 requirements、package-lock、poetry、go.sum、Cargo、composer | — | — | — |
| 基础镜像 | `kalilinux/kali-rolling:latest`（`docker/Dockerfile` 第 11 行） | tag 浮动 | **无 digest** | Docker Hub 官方库名；层未拉取 |
| 下游镜像 | `sapper-agent:latest`、`sapper-agent:tmux`（本地 tag，`Dockerfile.tmux` 第 7 行、`Dockerfile.commander` 第 7 行） | 本地标签 | 无 | 需先在本机构建，不是独立上游 |
| apt | 源改为 `http://mirrors.tuna.tsinghua.edu.cn/kali`（第 26–28 行），包名不锁定 | 否 | 未见额外 pin | 清华 TUNA，不是 `http.kali.org`。签名是否在构建时按 Kali keyring 校验：**未构建，未核实** |
| 预置二进制 | `docker/nuclei.zip` 约 29MB | 以 git blob 固定；Dockerfile 未写版本号 | 见下节，与官方 v3.3.7 zip 一致 | ProjectDiscovery 发布物（字节级） |
| 子模块 | 无 `.gitmodules` | — | — | — |
| compose | 无 | — | — | — |
| `curl` / `wget` | 入口只用 `curl` 看平台 HTTP 状态码，不把响应送进 shell。`install.sh` 不下载。未见 `curl \| sh` | — | — | 平台 URL 来自环境变量 |
| 安装钩子 | 本仓无 setup.py / postinstall。镜像构建明确打开 npm 脚本，为的是让 claude-code 的 postinstall 再拉原生二进制（Dockerfile 第 69–71 行注释） | 该二进制无版本、无校验和 | 未锁定 | npm 上游，构建时才发生 |
| 谱系仓 | README 指向 `N1nEmAn/nvwabase` | 不是包依赖 | — | 本单未审 |

`docker/build.sh` 与 `docker/export.sh` 只做 `docker build` / `docker save`，不拉取额外脚本。

## 依赖是否恶意

结论：**对下面已核对的对象，没有看到仿冒包名、被掉包的压缩包或 MAL 记录。未锁定、未安装的解析结果保持「未核实」，不写成安全，也不写成恶意。**

- **仿冒名。** 2026-11-26 09:39 香港时间查询 PyPI：`requests`、`pyjwt`、`unicorn`、`pwntools`、`pycryptodome`、`sympy`、`ROPgadget`、`ropper`、`gmpy2`、`z3-solver` 均存在，`yanked=false`，项目页指向各自正牌站点（例如 ROPgadget → `JonathanSalwan/ROPgadget`，sympy → sympy.org，z3-solver → `Z3Prover/z3`）。npm：`@anthropic-ai/claude-code` 当时 latest 为 `2.1.332`，描述为 Anthropic 官方 CLI；`@openai/codex@0.42.0` 存在，仓库字段为 `github.com/openai/codex`，未被标记 deprecated。这些是名称与登记信息，不是安装后文件的哈希。
- **锁定版本的 OSV。** 同一时刻 `querybatch`：`PyPI unicorn 2.0.1.post1` 0 条；`npm @openai/codex 0.42.0` 0 条。返回里没有 `MAL-` 编号。这只说明这两个版本在 OSV 上没有漏洞或恶意条目，不能外推到未锁定的包。
- **Nuclei 压缩包。** `sha256sum docker/nuclei.zip` = `725ef892fcffd1b03ad4f0874942fc4b623c0419b6b6c6c91fe4a5a65671f77c`。官方 <https://github.com/projectdiscovery/nuclei/releases/download/v3.3.7/nuclei_3.3.7_checksums.txt>（发布 `2024-12-02T12:14:59Z`，即 2024-12-02 20:14:59 香港时间）中 `nuclei_3.3.7_linux_amd64.zip` 为同一哈希。解包内容是 LICENSE、多语言 README 和一个 stripped 的 linux amd64 ELF，没有多余可执行脚本，README 中无 `<script>`。**这是官方发布物，不是第三方改包。** Dockerfile 仍未把版本和校验和写进构建文件，以后若有人换 zip，审查时不会自动失败。
- **知识库不是恶意依赖。** `kb/entries/*.json` 是本仓自己的参赛手法文本，随 git 分发，不从第三方包安装。
- **未核实（不得写成无恶意）。** 未锁定的 pip/npm 在未来构建中解析到的具体版本与传递依赖；`@anthropic-ai/claude-code` postinstall 拉到的原生二进制；Kali `latest` 镜像层与 TUNA 上的 apt 包；`gmpy2` / `z3-solver` / `ropper` 构建失败时被跳过的那部分。镜像层一次都没有拉取。

## 已知漏洞（不是恶意）

`github.com/projectdiscovery/nuclei/v3` 版本 `3.3.7` 在 2026-11-26 09:39 香港时间的 OSV 查询中，修复版本晚于 3.3.7 的条目有：

| 编号 | 影响（OSV 摘要） | 修复于 |
|---|---|---|
| GHSA-29rg-wmcw-hpf4 / CVE-2026-41646 | 模块加载绕过导致本地读文件 | 3.8.0 |
| GHSA-jm34-66cf-qpvr / CVE-2026-41282、CVE-2026-41645 | 响应派生表达式泄露环境变量 | 3.8.0 |
| GHSA-jpf4-98qj-qr67 / CVE-2026-76802 | DAST 代码签名绕过导致命令执行 | 3.10.0 |
| GHSA-jpvm-9frm-hjcq / CVE-2026-76805 | DAST/Fuzz 路径泄露环境变量 | 3.10.0 |
| GHSA-qgw5-7j4f-fg97 / CVE-2026-76804 | workflow 文件协议绕过导致本地读文件 | 3.10.0 |
| GHSA-vxg7-f2jj-jmqm / CVE-2026-76819 | Goja 引擎导致代码执行 | 3.10.0 |
| GHSA-xhmx-w2j4-rw3q / CVE-2026-76803 | MySQL 客户端沙箱绕过导致本地读文件 | 3.10.0 |

这些条目的前提都是处理不可信模板或响应。本仓知识库让 agent 对靶标使用 nuclei（例如 `kb/entries/kb053.json`）。未复现。修复版本早于 3.3.7 的 2024 年签名绕过（如 CVE-2024-43405，修复于 3.3.2）**不**算进「3.3.7 仍受影响」。

其余未锁定的 PyPI/npm 包没有按「假设装到 latest」去记 CVE，避免把历史漏洞安到一次尚未发生的构建上。

## 未覆盖

- 传递依赖、安装脚本实际下载的二进制、镜像层、apt 包版本与签名。
- 未查询 Docker Hub 上 `kali-rolling:latest` 在查询当日的 digest（Dockerfile 本身没钉 digest，查到的浮动 digest 也不能代表本仓某次构建）。
- `nvwabase` 不在本单。
