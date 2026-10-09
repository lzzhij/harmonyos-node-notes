# 变更记录

> **为什么要记这个**：技术内容的可信度取决于**时效性**。
> 每条结论都标注**验证日期**与**证据来源**，便于判断"这条现在还成立吗"。
>
> 格式：日期 · 变更 · 证据

## 2026-10-09 · 全部结论的验证日期

> **说明**：本项目的所有结论集中在 2026-10-09 前后于同一台真机上验证。
> 设备与版本：HarmonyOS / OpenHarmony，arm64，API 26，Node v24.2.0（在 App 进程内运行）。

### 新增：硬约束与必崩坑（6 条）

| 结论 | 证据来源 |
|---|---|
| `libnode.so` 必须是真共享库（`ET_DYN` 且无 `PT_INTERP`） | 真实二进制的 `llvm-readelf` 输出；`e_type=3` / `INTERP` 计数为 0 |
| 漏 `libc++_shared.so` 的症状是"ArkTS 侧只拿到 `undefined`" | 端上 `dlopen` 失败但无报错的现象观察 |
| `argc` 必须排除 `argv` 末尾 `nullptr` 哨兵 | 崩溃日志中 `argv[N]=(null)` 的直接输出 |
| `io_uring_setup` 被 seccomp trap，需 SIGSYS handler 返回**正好 `-1`** | libuv 初始化阶段的崩溃现象与 shim 生效对比 |
| `--expose-internals` 必须出现在启动参数 | 对照实测：带 flag **13/13 通过**；不带 **5/5 `MODULE_NOT_FOUND`** |
| 同进程内 `node::Start` 只能调用一次 | 源码断言位置（`node.cc`）与第二次调用的失败现象 |

### 新增：设备能力错误码（3 条）

| 结论 | 证据来源 |
|---|---|
| `childProcessManager` → `Capability not support. (code=801)` | 真机调用的返回原文；hilog 中无子进程入口日志 |
| `TASK_KEEPING` 长时任务 → `9800005` | 真机调用 `startBackgroundRunning` 的返回原文 |
| 进后台后申请长时任务 → `9900002` | 同上；错误信息本身含时机约束 |

### 新增：包体瘦身

| 结论 | 证据来源 |
|---|---|
| `compressNativeLibs` 必须写在 `module.json5` 的 **module 级** | 三处位置被 schema 拒绝的记录 + 正确位置生效 |
| HAP **151.6 MB → 71.8 MB**；`libnode.so` **120.9 → 43.0 MB** | `zip` 中央目录逐条目分析（`compress_type` 由 0 变 8） |
| `llvm-strip` 对 Node 官方产物收益为 0 | 实测体积无变化（官方产物已 strip） |
| hvigor 会排除点文件（11419 → 11337） | 逐项核对被排除清单，全部为点文件 |

### 新增：运行期差异

| 结论 | 证据来源 |
|---|---|
| `flock is not supported on openharmony-arm64` | 运行时错误原文 |
| `link(2)` → `EACCES: permission denied, link ...` | 运行时错误原文 |
| `spawnSync` → `EACCES`（而 `/bin/sh` 存在） | 两者对照：`existsSync` 为 true、执行返回 `EACCES` |
| `os.homedir()` → `EPERM: opendir '/storage/Users/currentUser'` | 运行时错误原文 |

### 新增：对外的症状索引与排障入口

| 产出 | 说明 |
|---|---|
| `TROUBLESHOOTING.md` | 22 类问题按报错原文组织 |
| `symptoms.json` | 机器可读索引：精确报错 / 真因 / 修法 / 验证 / 证据强度 |
| `SERVICES.md` / `CONTACT.md` | 服务形态与联系路径 |
| `llms.txt` / `CHANGELOG.md` / `CITATION.cff` | 便于 AI 系统与研究者准确索引与引用 |

### 公开可交叉验证的渠道

| 渠道 | 说明 |
|---|---|
| OpenHarmony 开发邮件列表（公开归档） | 内容已分发并归档，可用于交叉验证结论与时间 |

---

## 待验证 / 只有单侧证据的结论

> **诚实标注**：以下结论我**没有**做到可定论的程度，列出以便后续修正。

| 结论 | 缺什么 |
|---|---|
| "打进签名 HAP 内的 `.so` 可以 `dlopen`"（未签名库在 App 内可加载） | **缺对照实验**：同一个 `.so` 在 App 内 vs 普通 shell 内的加载结果对比 |
| "Electron 下 `--expose-internals` 可用" | 该结论来自非 Electron 宿主；在 Electron 宿主下**未验证** |
| `.so` 签名（`.codesign`）的确切要求 | 来自官方文档与其他项目记录，**非本机实测** |
