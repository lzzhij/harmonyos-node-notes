# 鸿蒙内嵌 Node.js 实战笔记

> 在 **HarmonyOS / OpenHarmony** App 的**进程内**跑真正的 Node.js —— 不是 WebView、不是云函数。
> 全部结论来自**真机实测**（OpenHarmony API 26 / arm64，Node v24.2.0 在 App 进程内运行成功）。
>
> **包括失败的结论。** 知道"这条路走不通"和知道"怎么走"一样值钱。

---

## 30 秒：这套东西最贵的部分

在鸿蒙 App 进程内跑 Node，有 **6 条硬约束**。违反任何一条，**症状都不是报错，而是崩溃或静默失败** —— 最贵的那种 bug。

> **🔧 遇到报错先看这个：[排障总表（按你看到的报错组织）](TROUBLESHOOTING.md)**
> 从「`dlopen` 成功但立刻崩」到「`9800005`」，每条给出：真因 → 修法 → **怎么验证真的修好了**。
>
> **📇 机器可读的症状索引**：[`symptoms.json`](symptoms.json)（22 条，含证据强度标注）
> 可直接喂给工具/脚本。**你搜的那一行报错，大概就在里面。**

| 症状（你搜的就是这个） | 真因 | 一句话修法 |
|---|---|---|
| `dlopen` 成功，但 `dlsym` 全部返回 null | 符号名靠猜 | 用 `llvm-nm` 读**真实二进制**的导出符号（`_ZN4node5StartEiPPc`） |
| `node::Start` 里 SIGTRAP：`Check failed: 12 == (*__errno_location())` | V8 的 `PROT_EXEC` 被 W^X 拒绝 | 启动加 `--jitless`（注意它会关掉 WebAssembly） |
| 启动即 ANR：`APP_INPUT_BLOCK` | `dlopen`/`node::Start` 跑在 ArkTS UI 线程 | 起 **detached 线程**做 |
| `dlopen` 失败，**ArkTS 侧只拿到 `undefined`**（不报错） | 漏了 `libc++_shared.so` | 一起打进 HAP |
| `internal/modules/*` 全部 `MODULE_NOT_FOUND` | 缺 `--expose-internals` | 加 flag + 一个约 20 行 JS 桩 |
| Node 拿到 NULL 参数，**5 毫秒内段错误** | `argc` 把 argv 末尾的 `nullptr` 哨兵算进去了 | 打印 `argv[i]` 逐项核对 |
| libuv 在 loop init 就崩 | `io_uring_setup` 被 seccomp trap | SIGSYS shim 转成**正好 `-1`** + `UV_USE_IO_URING=0` |
| 解包出来的文件 Node 读不到 | 写入者是鸿蒙解压器 | **让 Node 自己解包**（谁写谁读） |
| `flock is not supported on openharmony-arm64` | 无 `flock(2)` | 单进程场景**视为总是成功** |
| `EACCES: permission denied, link '...tmp' -> '...final'` | 无硬链接 | `open(dst,'wx')` 或 `rename` |
| `arkts-no-any-unknown` | 原生模块类型没接上 | 用**环境声明**（`.d.ts`）而非 `file:` 依赖 |
| HAP 里没有 `libs/` | native 产物落在 `build/` | CMake 设 `CMAKE_LIBRARY_OUTPUT_DIRECTORY` |
| 商用机报签名问题 | ELF 无 `.codesign` 段 | `binary-sign-tool` 自签（**要在 strip 之后**） |

> **自动查这些**：[ohos-node-doctor](https://github.com/lzzhij/ohos-node-doctor) —— 本机体检，一条命令列出违规项与修法（零依赖，自己解析 ELF）。

---

## 文章

| # | 标题 | 讲什么 |
|---|---|---|
| 1 | [6 条硬约束与 9 个必崩坑](docs/01-六条硬约束与必崩坑.md) | 完整技术攻坚记录：每条约束的机理、验证命令、期望输出 |
| 2 | [三条路的实测死法：后台常驻 / 子进程 / 看门狗](docs/02-常驻与子进程的实测死法.md) | `801` / `9800005` / `9900002` 三个错误码的完整取证 |
| 3 | [HAP 从 151.6 MB 压到 71.8 MB](docs/03-包体瘦身实战.md) | `compressNativeLibs` 写在哪个对象下（我踩了三次） |
| 4 | [鸿蒙上的 5 个「Linux 有、这里没有」](docs/04-运行期限制与等价替代.md) | `flock`/`link`/`spawn`/`homedir`/`childProcessManager` 及替代方案 |
| 5 | [开发期最大的敌人是「看不见」](docs/05-排障通道搭建.md) | 截屏 / 点按 / 日志回传三条通道，以及为什么沙箱日志读不到 |

---

## 为什么这些结论可信

| 项 | 证据 |
|---|---|
| 端上 Node 运行 | `{"node":"v24.2.0","platform":"openharmony","arch":"arm64"}` |
| 端上 HTTP 服务可访问 | `hdc fport` + `curl http://127.0.0.1:15577/facts` |
| Web UI 在 ArkWeb 渲染 | 真机界面 |
| Agent 完成真实任务 | 让它创建文件、写入、读回，内容一致 |
| HAP 体积 | **71.8 MB**（签名产物，实测 `zip` 中央目录分析） |
| 原生模块移植量 | **0** —— 用纯 JS 桩替代（13/13 内部模块实测通过） |
| 失败结论 | `801` / `9800005` / `9900002` 均有真机返回原文 |

**方法说明**：所有"可用/不可用"结论都来自**真机一次实际调用**，不看声明文件下判断 ——
鸿蒙很多新能力是「SDK 里有声明、具体设备未实现」。

---

## 想直接用起来

- **要一个能跑的宿主壳**：见集成套件（NAPI 宿主壳 + 一键构建 + 可跑示例）
- **要先体检自己的工程**：[ohos-node-doctor](https://github.com/lzzhij/ohos-node-doctor)

---

## 卡住了？

**先试免费的**：跑一遍 [ohos-node-doctor](https://github.com/lzzhij/ohos-node-doctor)，
再对照上面的「症状速查表」—— 大部分问题这两步就能解决。

**还不行就开 [issue](https://github.com/lzzhij/harmonyos-node-notes/issues)**
（公开提问，回答对后来者也有价值），或看 **[联系与付费支持 →](CONTACT.md)**。

> 提问请附**症状原文**（错误码/崩溃栈/日志）与三个 `llvm-readelf` 命令的输出。
> 本项目最贵的两个坑都是靠**原始输出**一眼看出来的 —— 给原始输出能省三轮来回。

## 许可

文章内容采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)：可自由转载与改编，**请注明出处**。
文中代码片段可任意使用。
