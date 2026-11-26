# 验收矩阵（2026-11-26 重审）

对象：<https://github.com/N1nEmAn/sapper-v2> @ `7b75e6f1f5d5b4ae0c55878b0dd0994d9b08f1fb`。  
起点表的需求行都保留。证据以本环境读到的树和 2026-11-26 香港时间的只读查询为准。

| ID | 需求 | 检查项 | 证据 | 结论 | 未覆盖 |
|---|---|---|---|---|---|
| R1 | 对指定仓、固定基准 SHA 做只读静态重审 | 上游 URL、默认分支、SHA、提交时间（香港时间）；只读源码与清单 | 2026-11-26 09:34 香港时间 `git fetch` 上游 `master`，HEAD = `7b75e6f1f5d5b4ae0c55878b0dd0994d9b08f1fb`。作者 `Wanli Chen <1332037591@qq.com>`，`2026-11-11T20:18:45+08:00`（即 2026-11-11 20:18:45 香港时间，等同 API `2026-11-11T12:18:45Z`）。派工附注 09:29 的 ls-remote 与此次一致 | 已按该树阅读。默认分支是 `master` | SHA 已检出。私库未建、也未读 |
| R2 | 代码安全：故意漏洞与部署者／宿主机风险分开；高／中／低 | 危险执行、默认凭据、硬编码密钥、外传、隔离、暴露面、容器／宿主挂载 | F-01 高：`docker/entrypoint.sh` 34–74、`docker/commander/entrypoint.sh` 46–58 与 105、`docker/commander/worker.sh` 200–207 与 395–410、`lib/sapper.py` 1455–1456 与 2919–2927。F-02 中：`docker/local-sim.sh` 28。F-05 低：`lib/arena.py` 42，`lib/sapper.py` 2041 与 2068。F-06 低：`docker/commander/watchdog.sh` 145–150，`lib/sapper.py` 1420–1422。F-07 低：`lib/sapper.py` 3647–3649 与 4550。知识库在 `kb/entries/`。未见硬编码密钥 | 无自埋关卡。真实高风险是入口强制跳过权限且本仓不强制隔离。高 1、中 3、低 3（中含供应链两条，见 R4） | 未运行容器，不能确认平台侧挂载、seccomp、出网或 sudoers。遥测环境变量是否生效未核实 |
| R3 | 架构，含与学习目标相称的功能阅读 | 组件、信任边界、数据与权限流；关键流程能否被绕过 | README「六种建制」对应 `lib/sapper.py`、`lib/arena.py`、`docker/commander/`。题面写入事实图：`lib/sapper.py` 1938–1968。指挥官仍以无确认 CLI 启动：`docker/commander/entrypoint.sh` 105。hook 只做文本标注：`lib/sapper-hook.py` 157–162 | 编排与执行分文件，不分成强制的权限域。README「指挥官不打」可被该进程的工具权限绕过 | 未把 README 成绩与战报做成宣称全表。未逐条读完 56 个知识条目。`nvwabase` 未审 |
| R4 | 供应链，且单列「依赖是否恶意」 | 包与锁、镜像 tag/digest、子模块、curl/wget、校验和、仿冒名、安装钩子、非官方源、OSV/GHSA | 无 lockfile、无 compose、无 `.gitmodules`。`install.sh` 无管道安装。`docker/Dockerfile` 11、26–28、64–78。`docker/nuclei.zip` SHA-256 与 Nuclei v3.3.7 官方 `nuclei_3.3.7_linux_amd64.zip` 一致（2026-11-26 09:38 香港时间下载官方 checksum 对比）。OSV 查询 2026-11-26 09:39 香港时间：`unicorn==2.0.1.post1`、`@openai/codex@0.42.0` 无条目；nuclei v3 模块 3.3.7 有后续 GHSA。PyPI/npm 元数据同刻解析到正牌项目页 | 已核对对象未见恶意替换。未锁定依赖与未拉取镜像层写「未核实」。Nuclei 是官方旧版本，有已知漏洞，不是掉包。详见 `audit/supply-chain.md` | 未安装，传递依赖未展开。Kali apt 未逐包查。镜像层未拉取。未锁定包的「解析后版本」未查询 OSV |
| R5 | 精练中文报告，供日后压缩 | 四个路径；每节先结论 | `audit/RERUN-20261126.md`、`audit/matrix.md`、`audit/findings.json`、`audit/supply-chain.md` | 已写。未改总报告、飞书或 Page | 无本环境写不出的附件。旧报告原文未取回，故未附对照摘录 |
| R6 | 对照旧基准 `d4de0770ae1fba706f00b7e29197ce739adbea94`（2026-11-09） | compare；高／中结论标仍成立／已变化／未覆盖 | `merge-base` 等于旧基准。ahead 14。净差异 `README.md` +24、`README.zh-CN.md` +14。产品脚本、Dockerfile、`nuclei.zip` 不在差异中 | `bypassPermissions` = High **仍成立**。预构建镜像层 **仍未覆盖**。`nuclei.zip` 的「未拆开」**已变化**为本轮已核对官方校验和（源码未改，覆盖变了） | 旧报告全文未读（飞书未授权；本树无 `audits/sapper-v2-SECURITY_AUDIT.md`） |
| R7 | 不新建私库、不改产品代码、不把静态阅读写成已运行 | 交付物与措辞 | 分支 `cursor/rerun-20261126-sapper-v2` 仅新增 `audit/`。PR 基线为 fork 默认分支。描述写明上游 URL 与本单 SHA | 遵守。未写「已验证可部署」或「pstack 已启用」 | — |
| R8 | （补行）容器入口与宿主网络这条部署者路径单独留痕 | `docker/local-sim.sh` 是否把攻击镜像放进宿主网络命名空间 | 第 6–8、28 行：注释与 `docker run --network host`。靶场脚本路径 `/tmp/mock_bench.py` 不在树内 | 中。只在有人执行该脚本时成立。未执行 | 脚本指向的 mock 靶场源码未读 |
