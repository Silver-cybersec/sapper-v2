# 供应链补查 v4（只读）

基准仍是上游 <https://github.com/N1nEmAn/sapper-v2> `7b75e6f1f5d5b4ae0c55878b0dd0994d9b08f1fb`。长清单与 Nuclei 校验和见 `audit/supply-chain.md`，这里不重复。未安装依赖，未拉镜像层，未跑脚本或 CI。查不到的写未核实，不写成安全。

查询窗口：**2026-11-26 11:32 香港时间**（UTC+8）。OSV `https://api.osv.dev/v1/querybatch`、PyPI/npm 索引、`https://hub.docker.com/v2/repositories/kalilinux/kali-rolling/tags/latest`、`https://api.github.com/repos/N1nEmAn/sapper-v2/actions/workflows` 的批量返回时刻为 **2026-11-26 11:32:19 香港时间**。

| # | 项 | 结论 |
|---|---|---|
| 1 | 依赖清单与 lockfile（含间接） | 未核实 |
| 2 | 版本钉、校验和、git／URL 直装 | 已确认 |
| 3 | 仿冒名、安装钩子、非官方源 | 已确认 |
| 4 | 容器镜像来源、tag、digest | 已确认 |
| 5 | 子模块、下载脚本、curl\|bash、运行时远程拉取 | 已确认 |
| 6 | CI 工作流 | 未发现问题 |
| 7 | 恶意包库与已知漏洞 | 已确认 |
| 8 | 主要依赖维护状态 | 未发现问题 |

「已确认」指文件或查询结果能对上的风险或缺口，不是指已确认恶意替换。

## 1. 清单与 lockfile — 未核实

直接依赖名称已列在 `audit/supply-chain.md` 清单表。本树无 `requirements*.txt`、`package-lock.json`、`poetry.lock`、`go.sum`、`Cargo.lock`、`composer.lock`，也没有 apt 版本钉。间接依赖因此不能从树内展开。未安装，故传递依赖未核实。

## 2. 版本钉、校验和、直装 — 已确认

多数直接依赖没有版本钉，也没有完整性校验：`docker/Dockerfile:71` 的 `@anthropic-ai/claude-code` 不锁版本；同文件 `76-78` 除 `unicorn==2.0.1.post1` 外均不锁；`81-82` 的 `gmpy2`、`z3-solver` 不锁且失败可跳过。`docker/Dockerfile.commander:32-33` 把 `@openai/codex` 钉在 `0.42.0`，无 integrity，安装失败被忽略。`docker/nuclei.zip` 的 SHA-256 与官方 v3.3.7 linux/amd64 zip 一致（证据与哈希见 `audit/supply-chain.md`），但 `docker/Dockerfile:64-66` 未把版本和校验和写进构建指令。

树内 pip/npm 行没有 `git+`、`github:` 或 URL 直装。这一子项未发现问题。

## 3. 仿冒名、钩子、非官方源 — 已确认

`docker/Dockerfile:69-71` 打开 npm `--allow-scripts`。2026-11-26 11:32:19 香港时间 npm 索引上 `@anthropic-ai/claude-code@2.1.332`（当时 latest）的 `postinstall` 为 `node install.cjs`。Dockerfile 不锁版本，所以构建当时实际执行的脚本正文未核实；「允许执行安装脚本」这一条是文件里写明的。`@openai/codex@0.42.0` 与当时 latest `0.172.0` 的索引页都没有 install/postinstall 脚本。

`docker/Dockerfile:26-28` 把 apt 源换成 `http://mirrors.tuna.tsinghua.edu.cn/kali`，不是 `http.kali.org`。签名是否在构建时按 Kali keyring 校验：未构建，未核实。这是非官方镜像源，不把它写成已确认恶意。

包名：同上刻 PyPI/npm 索引，`requests`、`pyjwt`、`unicorn`、`pwntools`、`pycryptodome`、`sympy`、`ROPgadget`、`ropper`、`gmpy2`、`z3-solver`、`@anthropic-ai/claude-code`、`@openai/codex` 均指向既有项目页，未见仿冒名。OSV 按这些名称查询，返回中无 `MAL-` 编号。未锁定版本的字节未核实。

## 4. 镜像 — 已确认

`docker/Dockerfile:11` 为 `kalilinux/kali-rolling:latest`，无 digest。`docker/Dockerfile.tmux:7`、`docker/Dockerfile.commander:7` 使用本地 tag `sapper-agent:latest` / `sapper-agent:tmux`，无 digest。

Docker Hub 标签元数据（不是层）：2026-11-26 11:32:19 香港时间 `latest` 的 `last_updated` 为 `2026-11-22T04:16:03Z`（2026-11-22 12:16 香港时间）；amd64 清单 digest 为 `sha256:55f4755c54407ec7104d2287e869b3420d788a680ab5d1da70fa798a54c46d02`。该 digest 没有写进 Dockerfile，层未拉取，不能当作本仓某次构建已审核。

## 5. 子模块与远程拉取 — 已确认

无 `.gitmodules`。`install.sh` 不下载。入口的 `curl` 只取平台 HTTP 状态码（`docker/entrypoint.sh:42-46`，tmux/commander 入口同样），没有 `curl|bash`。

构建期远程拉取是已确认的：npm `--allow-scripts` 会再取 Claude Code 的原生二进制（`docker/Dockerfile:69-71`）；apt/pip 从第 3、4 项的源安装。`nuclei` 使用仓内 zip，不在构建时从 GitHub 下载（`docker/Dockerfile:61-66`）。

运行时：入口要求模型网关与答题 API（`docker/entrypoint.sh:13-19`）；DeepSeek 基址默认 `https://api.deepseek.com/anthropic`（`docker/commander/worker.sh:43`）。这是 API 调用，不是在仓库里写明的模型权重或插件市场下载。未见独立的插件安装器。具体会拉下什么二进制，未安装，未核实。

## 6. CI — 未发现问题

本 SHA 工作树没有 `.github/`。2026-11-26 11:32 香港时间 `GET /repos/N1nEmAn/sapper-v2/actions/workflows` 的 `total_count` 为 0。因此树内没有 Action 版本钉、没有工作流里的密钥引用，也没有 `pull_request_target` 或 `workflow_run`。

GitHub 仓库 Settings 中的 secret 与 Actions 权限不在 git 树里，未打开设置页，这一面未核实。

## 7. 公开漏洞库

### 依赖是否恶意 — 未发现问题（仅限已点名的包名）

2026-11-26 11:32:19 香港时间对第 3 项所列 PyPI/npm 名称做 OSV 查询，结果中没有 `MAL-` 编号。`docker/nuclei.zip` 与官方发布物哈希一致，不是替换包（见 `audit/supply-chain.md`）。未锁定版本、apt 包、镜像层、postinstall 实际下载的二进制未核实，不写成无恶意。

### 已知漏洞 — 已确认

只把版本钉死、且 OSV 按该版本返回的条目算进来。同一时刻：

| 依赖 | 版本 | 结果 |
|---|---|---|
| PyPI `unicorn` | `2.0.1.post1` | 0 条。名称历史中的 PYSEC-2021-868 修复于 `2.0.0rc1`，不记入该钉版本 |
| npm `@openai/codex` | `0.42.0` | 0 条 |
| Go `github.com/projectdiscovery/nuclei/v3` | `v3.3.7` | 7 组仍影响该版本的 GHSA，修复版本均晚于 3.3.7。编号与影响见 `audit/supply-chain.md`「已知漏洞」表，此处不重列 |

其余 pip/npm 与全部 apt 包没有锁定安装版本，不把「假设装到 latest」的 CVE 记到本树。那些名称在 OSV 上有历史条目（例如 `pyjwt` 名称查询 48 条、`@anthropic-ai/claude-code` 30 条），影响的是未在本仓钉住的版本，未核实。

## 8. 维护状态 — 未发现问题

均未看到归档或弃用标记。时间为上游原始时间换算的香港时间，查询日 2026-11-26。

| 依赖 | 最近发版或推送 | 归档／弃用 |
|---|---|---|
| PyPI requests 2.34.2 | 2026-05-15 03:25 | 未 yanked |
| PyPI pyjwt 2.15.1 | 2026-09-29 02:40 | 未 yanked |
| PyPI unicorn 2.1.4（仓内钉的是 2.0.1.post1） | 2025-09-10 01:10；GitHub `unicorn-engine/unicorn` 推送 2026-11-10 21:58 | 仓库未归档 |
| PyPI pwntools 4.15.1 | 2026-11-18 03:07；`Gallopsled/pwntools` 推送 2026-11-26 01:17 | 未归档 |
| PyPI pycryptodome 4.6.0 | 2026-11-22 22:23 | 未 yanked |
| PyPI sympy 1.14.0 | 2025-04-28 02:04 | 未 yanked |
| PyPI ROPgadget 7.7 | 2025-10-15 18:34 | 未 yanked |
| PyPI ropper 1.13.13 | 2025-03-01 05:48 | 未 yanked。间隔最长，未再查其 Git 是否归档 |
| PyPI gmpy2 2.3.2 | 2026-10-04 09:56 | 未 yanked |
| PyPI z3-solver 5.4.1.0 | 2026-11-11 10:07 | 未 yanked |
| npm `@anthropic-ai/claude-code` 2.1.332 | 2026-11-25 06:06；`anthropics/claude-code` 推送 2026-11-26 11:28 | 未归档、未 deprecated |
| npm `@openai/codex` latest 0.172.0；仓内钉 0.42.0 | latest 2026-11-24 08:09；0.42.0 发布于 2025-09-27 02:43。`openai/codex` 推送 2026-11-26 11:31 | 未归档。0.42.0 索引页未标 deprecated。钉旧版是 `docker/Dockerfile.commander:28-29` 的注释选择，不是上游弃坑 |
| Nuclei 仓内 v3.3.7；上游已到 v3.11.1（前次发布列表） | `projectdiscovery/nuclei` 推送 2026-11-26 02:23 | 未归档。旧版本的已知漏洞见第 7 项 |
| `kalilinux/kali-rolling:latest` | 标签元数据更新 2026-11-22 12:16 | 标签状态 active。层未拉取 |

## 本轮相对旧文

没有新的恶意替换，也没有新的漏洞编号。已确认项仍是旧文中的未钉版本、`--allow-scripts`、浮动基础镜像，以及 Nuclei 3.3.7 的既有 GHSA。本轮补上的是否决项：树内无 CI 工作流、上述包名无 `MAL-`、抽查的上游仓库未归档。
