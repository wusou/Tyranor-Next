# Ethornell 本地兼容补丁

补丁必须按下列顺序应用到 provenance 中固定的 `source_revision`，当前基线为 `a6c925e566f6bc1f7d60ab31afaec1d04dbfa8ab`，已确认可从批准 fork 按该提交获取。库、ABI header、来源 inventory 与补丁哈希以 `engine/src/main/jniLibs/arm64-v8a/libethornell.provenance.json` 为准。

## 补丁顺序与边界

1. `0001-user-data-double-bgi-resource.patch`：收敛一层重复 `bgi-resource` 前缀，将 `UserData/**`、顶层 `BGI.gdb` 和精确四位数字 `BGI####.cad` 路由到 `bgi-user/`；其它资源写入与路径逃逸仍拒绝。
2. `0002-game-local-savedata.patch`：允许经过文件描述符与 inode 校验的 `<resource>/savedata` 直接子目录作为 user-root；该子树不参与普通资源读取、存在性检查、枚举和资源扫描。其它重叠根及符号链接仍拒绝，分离的 user-root 继续供 JNI 回归夹具使用。
3. `0003-public-storage-ofd-writer-lock.patch`：当 `flock(LOCK_EX | LOCK_NB)` 仅返回 `ENOSYS`、`EOPNOTSUPP` 或 `ENOTSUP` 时，在 Android/Linux 尝试全文件非阻塞 `F_OFD_SETLK` 写锁。忙锁、权限错误与其他失败原样拒绝；OFD 失败也拒绝。OFD 锁随宿主持有的文件描述符关闭而释放。
4. `0005-amachoco-graph-startup-compat.patch`：仅对经资源指纹识别的 Amachoco ABI 恢复 `Graph91:00` 旧版空操作行为，避免启动时因上游未知调用处理收紧而退出；该原生调用的完整语义仍未核实。
5. `0006-bgi-version-compat.patch` 通过 IPL 与标题双指纹，为核实过的 Cinderella 与 Pure HD 版本选择 HoukagoFd ABI、原版 16 项显示模式、128 个音效通道和主线程内存规格。PE 未检出显示表时保留资源指纹配置。保留图形文本归一化中越界整数的原有载荷，更新上游指令数量回归，并增加有界的 CBG_MT 分块图片解码及原创回归夹具。Sys80:30/31/CF 与 ModernFused 位图指针入口共用图片格式识别，MT 输入按所属分配范围解码，静态输入的正计数不再截断压缩数据，全局输入限制在全局区末端。
6. `0007-bgi-audio-recovery.patch` 让 Android CPAL 后端合并 Oboe 关闭前后的重复错误回调，避免 Kira 单槽错误队列溢出，保留混音器重建输出，失败时每 250ms 重试。声音控制入口清理已结束的静态与流式播放句柄，防止渐变轮询塞满已停止消费的命令队列。回归覆盖终态控制与重新播放。
7. `0008-bgi-render-scheduler-performance.patch` 修复现代 BP 内联复制经 `VmMemory::DerefMut` 将整个 12 MiB 局部窗口标脏的问题，仅登记复制目标。线程交接交换并复用大小缓冲区，避免复制后反复扩缩容。脏区间超过 16 段时合并最小相邻间隙，维持高位稀疏写入的覆盖。图形路径为整数采样和整数单位仿射提供像素等价的快速分支，双线性插值以整数权重保留旧浮点公式的结果。

历史补丁 `0004-virtual-user-file-reads.patch` 仅用于旧基线；新版上游已实现虚拟根与存档读取映射，不再应用该补丁。

```bash
cd /path/to/ethornell-rs
git checkout --detach a6c925e566f6bc1f7d60ab31afaec1d04dbfa8ab
git apply /path/to/TyranorNext/tools/ethornell/patches/0001-user-data-double-bgi-resource.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0002-game-local-savedata.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0003-public-storage-ofd-writer-lock.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0005-amachoco-graph-startup-compat.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0006-bgi-version-compat.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0007-bgi-audio-recovery.patch
git apply /path/to/TyranorNext/tools/ethornell/patches/0008-bgi-render-scheduler-performance.patch
cargo test -p ethornell-app --locked
cargo test -p ethornell-image --locked
cargo test -p ethornell-script --locked
cargo test -p ethornell-vm --locked
```

补丁顺序必须与 provenance 的 `local_patches` 一致，每份补丁后的源码哈希应与下一份补丁的输入对应。多文件补丁的 `target_source_file_sha256` 记录各目标文件的结果，source inventory 覆盖全部构建输入、依赖锁文件及新增的 `crates/ethornell-image/src/cbg/mt.rs`。

锁回归应分别覆盖不支持操作与锁占用错误、同进程独立文件描述符及跨进程互斥、关闭和进程异常退出后的释放；公共共享存储必须单独测试，不能以应用专属外部目录替代。

## 构建与校验

使用 NDK `28.0.13004108`、API 26、锁文件及独立构建目录。Windows 本次使用 NDK clang 交叉编译，其他平台可使用 cargo-ndk。精确参数见 provenance 的 `build_command`，链接必须保留 16KB 对齐、RELRO、NOW 与 `--no-undefined`。将产物 `libethornell_mobile.so` 经 NDK `llvm-strip --strip-unneeded` 后复制为本项目的 `libethornell.so`，同步更新库哈希、补丁清单、来源 inventory 和构建信息。

```bash
python3 tools/ethornell/verify_provenance.py --local-static \
  --ndk /path/to/android-sdk/ndk/28.0.13004108 --host-tag darwin-x86_64 \
  --report /tmp/ethornell-local-static.json
```

静态校验仅覆盖来源记录、ELF/ABI、依赖与包内库字节，不代表设备或游戏兼容性。带本地补丁的 `dirty=true` 产物不能通过要求精确已发布源码的严格来源审计，`release_ready` 保持 false。原始验证日志与商业游戏截图保存在本地临时目录，摘要记录设备、产物身份与验收范围。



## 启动兼容验证

2026-11-16 复验运行库 SHA-256 为 `bd384cdbce8c31469650bd02c26ccb68f0357911d320c96c3695e8a38c9771ca`，26,738,304 字节。固定上游提交加五份补丁，来源清单为 408 项，整条补丁链重放后的文件逐字节匹配构建源码。Android ARM64 / API 26 构建、ELF/ABI、16KB 对齐及 APK 内库字节校验通过，`release_ready=false`。

测试设备为 iQOO 12（vivo V2307A，Android 15 / API 35，ARM64，4KB 页）。测试 APK 基于原版 Tyranor Next 1.65 替换运行库资产并以隔离包 `com.tyranor.test` 安装。第一轮启用测试 APK 的 debuggable 属性以读取解压库 SHA-256 与宿主 `/proc/<pid>/maps`，确认实际映射到 `files/engine_runtime/ethornell/libethornell.so`，哈希与本次产物一致。之后使用相同库的非 debuggable 测试 APK 运行原生探针。

| 样本 | 本轮观察 |
| --- | --- |
| 放课后的灰姑娘女孩 | 冷启动到标题，Start 后进入街景与室内对白，点击推进 |
| ピュア×コネクト フルHDリマスター | 冷启动到标题，New Game、默认姓名确认后进入对白，点击推进 |
| 一夜之后体验版 | 冷启动后进入开场动态场景与ナナ对白，用户确认测试成功 |

启动范围内仍有动态场景卡顿。音频恢复和渲染性能分别由后续独立补丁处理。原版三个样本的错误位置与日志见 [Issue #175](https://github.com/Weiss-UltimateSavior/Tyranor-Next/issues/175)。

本轮回归结果如下。

- `cargo +1.96.0 test -p ethornell-app -p ethornell-image -p ethornell-script -p ethornell-vm --lib --target x86_64-pc-windows-gnu --locked` 分别通过 532、26、29、359 项，共 946 项，11 项按既有条件忽略。
- 六个新增 VM 回归覆盖 Sys80:30 归档读取、Sys80:31 解码后切片、Sys80:CF 内存解码和 ModernFused 堆/静态代码区短计数位图注册及全局区跨界拒绝，断言完整图片描述头、像素和返回值。恢复旧分派条件后四项全部失败，修正后全部通过。静态短计数与全局跨界两项也分别确认在撤销对应修正后失败。
- 仓库 `EthornellReleaseInstrumentation` 源码单独编译为框架探针，使用仓库 C++ probe 和生产 ABI header。SJIS、GBK 各 3/3，通过非 debuggable 目标身份、RTLD_NOW / 15 导出 / ABI / build_info，以及生产 JNI create/poll/destroy、无效 token 与请求校验。编译专用 API stub 未进入测试 DEX，目标生产类来自目标 APK。该探针只覆盖加载和会话创建范围。
- `python -m unittest discover -s tools/ethornell/tests` 在 UTF-8 与 `core.autocrlf=false` 环境下为 78 通过、1 跳过、1 失败。失败为既有 Windows 文本夹具写入 CRLF 后与 LF 常量比较 header SHA-256，当前改动未修改该工具或夹具。
- 相同来源工具与测试在主工作树的 WSL 环境中 80/80 通过。工具文件与本 PR 完全一致。Windows 下的单项换行失败保留为平台差异记录。
- 两份独立只读审核核对分派逻辑、旧 CBG/DSC 回归风险、补丁重放与来源哈希，均无阻断发现。

完整当前源码 APK、AndroidX 设备测试、Surface/暂停/存档全套回归、API 26 / 16KB 设备及长稳尚未完成。本机 Gradle 在补齐 JDK 17 后仍无法识别已解压的 `android-37.0` 平台，故使用上述原有框架探针单独复验，未把历史设备通过数计入本轮。Android CLI 审查工具和适用技能在本会话未发现，设备操作使用 ADB。

本地证据的 SHA-256 如下，原始商业游戏截图保留在本地。下列三张截图属于先行修正库 `84376203…`，最终 `bd384cdb…` 的复验另记。

| 证据 | SHA-256 |
| --- | --- |
| `cinderella-body.png` | `cef211390924ea731615548a319dd617064d0733d08eb62b8b5e618918fdfd09` |
| `pure-body.png` | `0718fc368661561af3a84631bb8677299255f58e9eb53cca3dda8c79f568aeb2` |
| `onenight-title2.png` | `adaccc146c7889f136b4e4c1ef4d7472f463028bfd4381bfddce9e613dc38afb` |
| `probe-sjis.log` | `96370c7fe858ab8afe8dea0e5d0b5e4509ec53ec10a4353cc0199f061486f67c` |
| `probe-gbk.log` | `96370c7fe858ab8afe8dea0e5d0b5e4509ec53ec10a4353cc0199f061486f67c` |

### 最终静态输入修正复验

最终库 `bd384cdbce8c31469650bd02c26ccb68f0357911d320c96c3695e8a38c9771ca` 在同一手机复查 Cinderella 的室内对白与 Pure HD 姓名确认后的对白。两次均读取宿主 maps 与实际解压文件 SHA-256，确认映射当前库。原有框架探针 SJIS、GBK 各 3/3，静态门禁通过。《一夜之后》体验版也从标题进入列车开场动态画面与对白框，当前库哈希及映射同样核对。

| 最终证据 | SHA-256 |
| --- | --- |
| `cinderella-final-body.png` | `907d13d2d8f59e011e7f2a5eae3b1eb473f25277b3deca4da09d59a5408e619e` |
| `pure-final-body.png` | `44e25a9dff8e86f4cc6e81b401caf49588c6c43e108c15f2966fe45968e7e122` |
| `cinderella-final-library.txt` | `4426119fa7cd3ee574b91023ee940d81b82910cb54dca82ce4edef11ad6b741e` |
| `pure-final-library.txt` | `4426119fa7cd3ee574b91023ee940d81b82910cb54dca82ce4edef11ad6b741e` |
| `probe-final-sjis.log` | `96370c7fe858ab8afe8dea0e5d0b5e4509ec53ec10a4353cc0199f061486f67c` |
| `probe-final-gbk.log` | `96370c7fe858ab8afe8dea0e5d0b5e4509ec53ec10a4353cc0199f061486f67c` |
| `onenight-final-name.png` | `1b3867115cf2f3a00b433e4a255bcbbf717db75c1c12da0b14889e80e14b55a8` |
| `onenight-final-body.png` | `d837e7f96e08f02043575989dbd3ca78b8095e68c90808d817493989903643a7` |
| `onenight-final-library.txt` | `4426119fa7cd3ee574b91023ee940d81b82910cb54dca82ce4edef11ad6b741e` |

## 音频恢复验证

2026-11-16 同步启动审核修正后，当前音频阶段运行库为 `7c9296d0f3623ebfb0aa5f62547720c54b3be9203ef9c80f370ab6cecf349e78`，26,760,800 字节。六份补丁与 409 项来源清单同步更新，ARM64 / API 26 锁定依赖构建和 ELF/ABI 静态校验通过。该阶段新产物的独立真机音频恢复仍待复验。

历史候选 `772ba49fefe70a7408accf446b57f5c0d2d9b226666047f254efbb6b58def4f0` 已在 iQOO 12（Android 15 / API 35）通过三个启动样本与普通 PURExCONNECT 的标题到正文流程，《一夜之后》体验版越过原队列满位置。静态和流式终态回归移除修复时失败，恢复后通过，原音频模块 21 项通过、2 项忽略。AAudio 实际断连恢复记录属于先行候选 `0fa9e6be…`。完整源码 APK、长时间听感及跨设备验证仍待完成。

## 2026-11-16 性能阶段同步验证

当前运行库为 `1203e7c36f9da9e12688b9363bc6b237507ad5cf241de12135a6ca9b298f00de`，26,765,920 字节，包含启动审核修正、音频恢复与性能七份补丁，409 项来源清单已同步。ARM64 / API 26 构建及 APK/ELF/ABI 静态校验通过，宿主回归 980 通过、13 忽略、0 失败，分别为 App 537、音频 21、图片 26、脚本 29、VM 367。

已将该组合产物装回同一台 iQOO 12 的隔离包 `com.tyranor.test`。原有框架探针 SJIS、GBK 各 3/3，通过范围与前述加载/创建探针相同。本次组合产物尚未重新测量动态 CG 性能或完成全部样本游戏流程，历史计时见下节。严格来源仍为 `release_ready=false`。

## 2026-11-15 性能候选验证

该轮运行库 SHA-256 为 `dd1b2e94b13bf78ada765a6710c7dfce216783dce93a5b50d9828909df9dd2f0`，26,766,176 字节。按上述顺序在固定上游提交重放七份补丁后，409 项源码哈希均与构建时记录一致，沿用原产物。生产产物已移除计时与原始错误输出插桩。设备为 iQOO 12（vivo V2307A，Android 15 / API 35）。测试 APK 基于 1.65 替换运行库资产。最终整数插值库在《一夜之后》完成标题、姓名确认及正文进入，进程存活且未见原生失败日志。前一零权重候选还复查了 Cinderella 和 Pure HD 的标题及正文，最终库尚待复查这两个样本。

在《一夜之后》体验版以临时计时库测量每 60 帧的处理耗时。基线 `b39d76e5…` 已包含整数图形快速分支，仍使用旧线程交接；零权重优化阶段的计时库为 `b2af41a1…`，最终整数插值计时库为 `a291a19d…`。基线静态标题每帧约 25–26ms，其中线程交接约 24ms。大小窗口交替时，最高 12 MiB 缓冲反复扩缩容，保存缓冲的再次复制及容量处理每 60 帧约 1.1s。内联复制经 `DerefMut` 造成全窗口标脏的调用位置实测为 `modern_bp.rs` 的 `copy_within`。

零权重优化阶段计时中，静态阶段单帧处理约 6–11ms。重动态开场的连续采样窗口由基线每帧约 359–398ms 降至约 263–268ms，仍有明显卡顿。该场景目标读取效果的 CPU 合成约 139–146ms，其中仿射合成约 73–76ms；多层背景另占约 49–51ms，GPU 提交约 1ms。上述数字是引擎内部处理时间，实际显示帧率和输入到显示延迟须单独测量。场景时间随交互和帧处理推进，采样窗口未逐帧锁定。后续将双线性 1/16 权重改为与旧浮点公式精确等价的整数累加，最终计时中的仿射合成为约 67–69ms，重动态连续窗口仍约 260–275ms。

纯源码宿主回归合计 974 通过、13 忽略、0 失败，分别为 App 537、VM 361、音频 21、图片 26、脚本 29。新增用例覆盖缓冲复用、脏区间溢出、重叠与跨区域复制、清零隔离、仿射边界及旧采样公式对照，经过两个独立 Agent 审核。来源工具在 WSL 下 80 通过。生产库及测试 APK 的 `--local-static`、16KB zipalign、签名校验通过，严格来源与逐字节重建验收仍待完成。
