# 排障总表：鸿蒙内嵌 Node.js 的全部已知问题

> **这份文件按"你看到的报错"组织。** 直接把你的报错原文对照下表即可。
> 每条都给出：真因 → 修法 → **怎么验证真的修好了**。
>
> 机器可读版本：[`symptoms.json`](symptoms.json)（22 条，含证据强度标注）
> 平台：HarmonyOS / OpenHarmony · arm64 · API 26 · 实测 Node v24.2.0

---

## 一、启动阶段（Node 还没跑起来）

### 1.1 `dlopen` 成功但立刻崩，V8 报 `Check failed: AllowHeapAllocationInRelease`

**真因**：你的 `libnode.so` 是 **PIE**（`ET_DYN` 但带 `PT_INTERP`）。
`dlopen` 一个 PIE 会把 local-exec 的 `%fs` TLS 别名到宿主进程的 TLS 块 ⇒
V8 的 `thread_local current_per_thread_assert_data` 读到垃圾 ⇒
release 版 CHECK 在 `Isolate::Initialize` **第一次堆分配就崩**。

**修法**：换 `--shared` 构建的真共享库。

```bash
llvm-readelf -h libnode.so | grep -E 'Type|Machine'   # 期望 Type: DYN
llvm-readelf -l libnode.so | grep -c INTERP           # 期望 0  ← 关键
```

> 网上下到的某些 `libnode.so` 其实是 PIE，症状就是"dlopen 成功但一进去就崩"。

### 1.2 `dlopen` 失败，但 ArkTS 侧**只拿到 `undefined`**（不报错）

**真因**：漏了 `libc++_shared.so`。`libnode.so` 的 `DT_NEEDED` 声明了它，
但它没一起打进 HAP。

**修法**：从 NDK 取 `native/llvm/lib/aarch64-linux-ohos/libc++_shared.so`，
与 `libnode.so` 放同一 ABI 目录。

```bash
llvm-readelf -d libnode.so | grep NEEDED   # 对照 entry/libs/<abi>/ 目录内容
```

### 1.3 `node::Start` 里 **5 毫秒内段错误**（无堆栈）

**真因**：`argc` 把 `argv` 末尾的 `nullptr` 哨兵算进去了 ⇒ Node 拿到 `NULL` 参数。

**修法**：`argc = argv.size() - 1`。

**验证**：把 `argv[i]` 逐个打印，看到 `argv[N]=(null)` 就是多算了一个。

### 1.4 libuv 在 loop init 就崩（V8 还没介入）

**真因**：libuv 初始化时**无条件**探测 `io_uring_setup(425)`，鸿蒙沙箱 seccomp 会 trap 它。

**修法**：装 SIGSYS handler，把 trap 转成 **正好 `-1`**（不是 `-ENOSYS`，libuv 的守卫才认）；
同时 `setenv("UV_USE_IO_URING","0",1)`，**且必须在 libuv 初始化之前**。

### 1.5 启动即 ANR：`APP_INPUT_BLOCK`

**真因**：`dlopen`/`node::Start` 跑在 ArkTS **UI 线程**。

**修法**：在 **detached 线程**里做。

### 1.6 `internal/modules/*` 全部 `MODULE_NOT_FOUND`

**真因**：启动参数缺 **`--expose-internals`**。
`require('internal/...')` 只认 **`process.execArgv`**（不是 `NODE_OPTIONS`，它不在白名单里）。

**修法**：加 `--expose-internals`，并用约 20 行**纯 JS 桩**替代原生 `require-builtin` 插件。

**验证**（对照实测）：

| 实验 | 结果 |
|---|---|
| 带 flag + 纯 JS 桩 | **13/13 通过** |
| 不带 flag | **5/5 `MODULE_NOT_FOUND`** |

> **注意**：`process.execArgv` 在启动时固定。用 `process.execArgv.push()` **改不了**，
> 必须让它出现在**启动参数**里。在 Electron 宿主下尤其要确认它真的传进去了。

---

## 二、载荷与文件（Node 跑起来了，但读不到东西）

### 2.1 解包出来的文件 Node 报 `MODULE_NOT_FOUND`
（**同目录、同进程、同 uid 也读不到**）

**真因**：同一沙箱目录下**「谁写的文件谁能读」并不成立**：

| 写入者 | Node 能否读 |
|---|---|
| ArkTS `fs.writeSync` | ✅ 能 |
| 鸿蒙 `zlib.decompressFile` | ❌ **不能**（`requireStack: []`，但 ArkTS 侧 `statSync` 看到文件有 11237 字节） |

**修法**：**让最终使用者也当写入者** —— 载荷打成单个 `tar.gz`，
由端上脚本用 **Node 自己的 `zlib` + `fs`** 解包。

**验证**：解包后立刻 `fs.readFileSync(entryAbs)`，成功即证明。

### 2.2 入口"已加载"之后进程立刻退出（无任何报错）

**真因**：入口若是包 CLI，最后一行常是 `if (import.meta.main) runCli()`。
用 **`import()`** 加载时 `import.meta.main` **恒为 `false`** ⇒ `runCli()` 永不执行。

**修法**：显式调用入口导出的 `runCli()`（或 `main()`），并把 `process.argv` 摆成它期望的形状。

### 2.3 `EPERM: opendir '/storage/Users/currentUser'`

**真因**：`os.homedir()` 读 `$HOME`，而鸿蒙 App 读不到沙箱外的用户主目录。
框架的文件选择器默认从 homedir 开始。

**修法**：宿主壳里 `setenv("HOME", <沙箱内可读目录>)`（用 dataDir 的父目录）。

---

## 三、运行期 syscall 差异

### 3.1 `flock is not supported on openharmony-arm64`

**真因**：鸿蒙不提供 `flock(2)`。Node 生态大量用 lockfile + flock 做单实例互斥。

**修法**：**单进程场景视为"总是获取成功"。**
先问"这个锁在鸿蒙的进程模型下还要不要"，再决定替换还是退化 ——
**别急着找一个 flock 的替代品**。

### 3.2 `EACCES: permission denied, link '...tmp' -> '...final'`

**真因**：鸿蒙不提供 `link(2)`。Node 生态常用「写临时文件 + link 发布」实现
**原子写入 + 独占创建**（`link` 在目标已存在时报 `EEXIST`）。

**修法**：

```js
// 需独占语义 → open(dst,'wx') 保住 EEXIST
const fh = await open(dst, 'wx');          // 目标存在则抛 EEXIST，与 link 语义一致
// 已确认目标不存在 → rename 更省事
await rename(tmp, dst);
```

> ⚠ **不能简单退化成"写两次"** —— 那会把"独占"变成"后者覆盖前者"。

### 3.3 `spawnSync` 返回 `EACCES`（而 `/bin/sh` 明明存在）

**真因**：**seccomp 禁止 app 执行任何外部程序**。
`/bin/sh` 与 `/system/bin/sh` 都存在 ⇒ **"检查文件是否存在"这个判据会误导你**。

**修法**：用**纯 JS 复刻该 CLI 与调用方之间的输出契约**（不是复刻整个工具）。

**判据要改成**：`spawnSync('/bin/sh', ['-c','echo x'])` 执行一下看返回 `EACCES`。

---

## 四、设备能力（`801` / `9800005` / `9900002`）

> **规律**：鸿蒙很多能力是**「SDK 里有声明、具体设备未实现」**。
> 看声明文件只能证明它**存在**，不能证明它**可用** —— **必须真机试一次**。

| 报错 | 含义 | 该怎么办 |
|---|---|---|
| `Capability not support. (code=801)` | 该 API **本设备未实现** | 换设备或换路；**别改代码** |
| `9800005 ... TASK_KEEPING type` | 该长时任务类型**本机不支持** | 换类型或放弃无限常驻 |
| `9900002 ... not allow after the preset time of entering background` | **进后台超时后才申请** | **必须在进后台【之前】申请** |

**普通应用在鸿蒙上无法无限常驻后台。** `requestSuspendDelay` 可用，
把切后台后的可用时间从约 90 秒延长到 **3~4 分钟**（成本极低，值得保留）。

---

## 五、构建与打包

### 5.1 `arkts-no-any-unknown`（编译期）

**真因**：用 `file:` 依赖 + types 包时，ets-loader 会把 `import ... from 'x.so'`
编译成**包内相对模块**，运行时解析成 `undefined`。

**修法**：用**环境声明**（`declare module 'x.so'`）提供类型；
或在 `oh-package.json5` 加 `file:` 依赖并**跑一次 `ohpm install`**。

**验证**：`ark_disasm modules.abc out.pa` 后 grep `requireNapi`。

### 5.2 HAP 里**没有 `libs/` 目录**

**真因**：CMake 没设输出目录，native 产物落在 `build/` 下。

**修法**：

```cmake
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
    ${CMAKE_CURRENT_SOURCE_DIR}/../../../libs/${OHOS_ARCH})
```

### 5.3 `Schema validate failed`（想开 `compressNativeLibs` 找不到位置）

**真因**：它是 **`module.json5` 的 module 级**属性。
写进 `build-profile.json5` 的 `buildOption` / `buildOption.nativeLib` 都会被拒绝，
而 **schema 只报"哪个键不对"，不报"该写到哪"**。

**修法**：写在 `entry/src/main/module.json5` 的 `module` 级：

```json5
{ "module": { "compressNativeLibs": true, "extractNativeLibs": true } }
```

**效果**：HAP **151.6 MB → 71.8 MB**（`libnode.so` 120.9 → 43.0 MB）。

**验证**（必须查 HAP，不能只看大小）：

```bash
python -c "import zipfile; z=zipfile.ZipFile('x.hap'); print([(i.filename, i.compress_type) for i in z.infolist() if i.filename.endswith('.so')])"
# compress_type 应为 8（DEFLATE），不是 0（STORED）
```

> **方法论**：遇到 `Schema validate failed`，去
> `toolchains/configcheck/configSchema_rich.json` 或 `toolchains/modulecheck/module.json`
> 里**搜这个键，看它出现在哪个对象下** —— 比反复试位置快得多。

### 5.4 `.so` 没签名，商用机上装上也用不了

鸿蒙商用版对 ELF 做**代码签名校验**，没有 `.codesign` section 直接不可用。

```bash
binary-sign-tool sign --in-file libnode.so --out-file libnode.signed.so --selfSign 1
```

**⚠ 要在 `strip` 之后签**（先签后 strip 会把签名抹掉）。

---

## 六、观测与发布（"看不见"和"推不上去"）

### 6.1 `hdc` 读沙箱日志报 `permission denied`（连 debug 签名也不行）

**这是开发期最大的坑** —— 直觉是"日志写在文件里，拉出来看"，**拉不出来**。

**四条通道**：

| 通道 | 命令 |
|---|---|
| 看屏幕 | `hdc shell snapshot_display -f /data/local/tmp/s.jpeg` + `hdc file recv` |
| 点屏幕 | `hdc shell uitest uiInput click <x> <y>` |
| 日志回传 | `hdc rport tcp:19999` + 端上 POST 到 `127.0.0.1:19999` |
| 端到端 | `hdc fport tcp:15577 tcp:5577` + `curl` |

**三个"假信号"**：

1. **hilog 里没有你的日志 ≠ 代码没执行**（应用侧 INFO 常不进 dump 缓冲）
2. **`VmSize` 很大 ≠ V8 起来了**（ArkTS/ArkWeb 自己就占约 40 GB 虚拟地址空间）
3. **点击"看起来生效了" ≠ 生效** —— 判据是**前后两次 `dumpLayout` 的差异**

### 6.2 `git push` 长时间无进展，但 `git ls-remote` 只要 1.5 秒

**真因**：上行不稳；且全局 `url.<proxy>.insteadOf` 会把 `origin` 的读取结果
也改写成代理地址。

**修法**：改用 **GitHub Contents API**（普通 HTTPS POST）逐文件上传。
⚠ 但 **Contents API 拒绝写入 `.github/` 下的路径**（CI 工作流会静默丢失）⇒ 用 **Git Data API**。

### 6.3 CI 报告失败，但最新几次其实是成功的

**真因**：逐个文件上传会产生多个提交，**早期提交缺少 CI 需要的文件** ⇒ 必然失败并产生通知。

**修法**：判定 CI 健康要看**最近若干次运行**，而不是未读通知。

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/actions/runs?per_page=10 \
  | grep -o '"conclusion":"[a-z]*"'
```

---

## 一张判据表：**别用"看起来能用"当判据**

| 判据 | ❌ 会被骗 | ✅ 应该这样验 |
|---|---|---|
| 文件存在 | `existsSync('/bin/sh')` → true，但不可执行 | **执行一下**看 `EACCES` |
| 路径可用 | `accessSync` 对 0 字节文件也返回 true | 判 `isFile() && size > 阈值` |
| API 可用 | 声明文件里有 = 存在 ≠ 可用 | **真机调一次**（801 的教训） |
| 写入成功 | 写完了，但读的人读不到 | **让最终使用者去读一遍**（谁写谁读） |
| 压缩生效 | HAP 变小了 | **读 zip 中央目录**看 `compress_type` |
| 输入正确 | "已输入 30 个字符" | **截图看实际落了什么字** |

**这六条我都踩过。** 共同点：**"能构建/能写入/能返回"都不等于"能运行/能读取/能用"。**

---

## 还有问题？

- 先跑 [ohos-node-doctor](https://github.com/lzzhij/ohos-node-doctor)（本机体检，自动查出上述违规项）
- 开 [issue](https://github.com/lzzhij/harmonyos-node-notes/issues)（**请附报错原文**与 `llvm-readelf` 输出）
- 或见 [联系与付费支持](CONTACT.md)
