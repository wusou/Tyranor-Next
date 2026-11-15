# Ethornell 本地兼容补丁

补丁必须按下列顺序应用到 provenance 中固定的 `source_revision`，当前基线为 `a6c925e566f6bc1f7d60ab31afaec1d04dbfa8ab`，已确认可从批准 fork 按该提交获取。库、ABI header、来源 inventory 与补丁哈希以 `engine/src/main/jniLibs/arm64-v8a/libethornell.provenance.json` 为准。

## 补丁顺序与边界

1. `0001-user-data-double-bgi-resource.patch`：收敛一层重复 `bgi-resource` 前缀，将 `UserData/**`、顶层 `BGI.gdb` 和精确四位数字 `BGI####.cad` 路由到 `bgi-user/`；其它资源写入与路径逃逸仍拒绝。
2. `0002-game-local-savedata.patch`：允许经过文件描述符与 inode 校验的 `<resource>/savedata` 直接子目录作为 user-root；该子树不参与普通资源读取、存在性检查、枚举和资源扫描。其它重叠根及符号链接仍拒绝，分离的 user-root 继续供 JNI 回归夹具使用。
3. `0003-public-storage-ofd-writer-lock.patch`：当 `flock(LOCK_EX | LOCK_NB)` 仅返回 `ENOSYS`、`EOPNOTSUPP` 或 `ENOTSUP` 时，在 Android/Linux 尝试全文件非阻塞 `F_OFD_SETLK` 写锁。忙锁、权限错误与其他失败原样拒绝；OFD 失败也拒绝。OFD 锁随宿主持有的文件描述符关闭而释放。
4. `0005-amachoco-graph-startup-compat.patch`：仅对经资源指纹识别的 Amachoco ABI 恢复 `Graph91:00` 旧版空操作行为，避免启动时因上游未知调用处理收紧而退出；该原生调用的完整语义仍未核实。
5. `0006-bgi-version-compat.patch` 通过 IPL 与标题双指纹，为核实过的 Cinderella 与 Pure HD 版本选择 HoukagoFd ABI、原版 16 项显示模式、128 个音效通道和主线程内存规格。PE 未检出显示表时保留资源指纹配置。保留图形文本归一化中越界整数的原有载荷，更新上游指令数量回归，并增加有界的 CBG_MT 分块图片解码及原创回归夹具。
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
cargo test -p ethornell-audio --locked
```

补丁顺序必须与 provenance 的 `local_patches` 一致，每份补丁后的源码哈希应与下一份补丁的输入对应。多文件补丁的 `target_source_file_sha256` 记录各目标文件的结果，source inventory 覆盖全部构建输入、依赖锁文件及新增的 `crates/ethornell-image/src/cbg/mt.rs` 和 `crates/ethornell-audio/src/android.rs`。

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

本阶段运行库 SHA-256 为 `71785654916edc5b5a6d6f197fe776f3777b3f6177d3d0eaed4bcb2f22fa8a61`，26,738,496 字节。固定上游提交加五份补丁，来源清单为 408 项。Android ARM64 构建和测试 APK 内实际库字节校验通过。测试 APK 基于 Tyranor Next 1.65 替换运行库资产。

在 iQOO 12（vivo V2307A，Android 15 / API 35，ARM64）上已用原版 1.65 复现三个样本的启动错误。此前包含相同启动补丁和后续音频修复的候选 `772ba49f…` 在《放课后的灰姑娘女孩》《ピュア×コネクト フルHDリマスター》《一夜之后》体验版通过标题到正文流程，普通版 PURExCONNECT 通过标题、姓名确认和对白。此前图片模块 26 项回归通过。该真机记录属于历史候选，本阶段新构建的库尚待真机复验。

原版报错位置及 CBG_MT 回归输出见 [Issue #175](https://github.com/Weiss-UltimateSavior/Tyranor-Next/issues/175)。完整当前源码 APK 与严格来源验收尚待验证。

## 音频恢复验证

本阶段运行库 SHA-256 为 `772ba49fefe70a7408accf446b57f5c0d2d9b226666047f254efbb6b58def4f0`，26,760,992 字节，来源清单为 409 项。Android ARM64 构建和测试 APK 内实际库字节校验通过。

在 iQOO 12（vivo V2307A，Android 15 / API 35）上，历史候选 `0fa9e6be…` 的 Full HD 样本两次触发 AAudio 断连，输出依次重建并继续对白。候选 `772ba49f…` 在《一夜之后》越过原队列满位置。已结束静态与流式句柄的宿主回归移除修复后均失败，恢复修复后通过。本阶段重建的运行库与候选 `772ba49f…` 字节一致。两次系统断连恢复的直接证据来自先行候选 `0fa9e6be…`，本库此前的两次锁屏未再次触发系统断连。详见 [Issue #176](https://github.com/Weiss-UltimateSavior/Tyranor-Next/issues/176)。

## 2026-11-15 性能候选验证

当前运行库 SHA-256 为 `dd1b2e94b13bf78ada765a6710c7dfce216783dce93a5b50d9828909df9dd2f0`，26,766,176 字节。按上述顺序在固定上游提交重放七份补丁后，409 项源码哈希均与构建时记录一致，沿用原产物。生产产物已移除计时与原始错误输出插桩。设备为 iQOO 12（vivo V2307A，Android 15 / API 35）。测试 APK 基于 1.65 替换运行库资产。最终整数插值库在《一夜之后》完成标题、姓名确认及正文进入，进程存活且未见原生失败日志。前一零权重候选还复查了 Cinderella 和 Pure HD 的标题及正文，最终库尚待复查这两个样本。

在《一夜之后》体验版以临时计时库测量每 60 帧的处理耗时。基线 `b39d76e5…` 已包含整数图形快速分支，仍使用旧线程交接；零权重优化阶段的计时库为 `b2af41a1…`，最终整数插值计时库为 `a291a19d…`。基线静态标题每帧约 25–26ms，其中线程交接约 24ms。大小窗口交替时，最高 12 MiB 缓冲反复扩缩容，保存缓冲的再次复制及容量处理每 60 帧约 1.1s。内联复制经 `DerefMut` 造成全窗口标脏的调用位置实测为 `modern_bp.rs` 的 `copy_within`。

零权重优化阶段计时中，静态阶段单帧处理约 6–11ms。重动态开场的连续采样窗口由基线每帧约 359–398ms 降至约 263–268ms，仍有明显卡顿。该场景目标读取效果的 CPU 合成约 139–146ms，其中仿射合成约 73–76ms；多层背景另占约 49–51ms，GPU 提交约 1ms。上述数字是引擎内部处理时间，实际显示帧率和输入到显示延迟须单独测量。场景时间随交互和帧处理推进，采样窗口未逐帧锁定。后续将双线性 1/16 权重改为与旧浮点公式精确等价的整数累加，最终计时中的仿射合成为约 67–69ms，重动态连续窗口仍约 260–275ms。

纯源码宿主回归合计 974 通过、13 忽略、0 失败，分别为 App 537、VM 361、音频 21、图片 26、脚本 29。新增用例覆盖缓冲复用、脏区间溢出、重叠与跨区域复制、清零隔离、仿射边界及旧采样公式对照，经过两个独立 Agent 审核。来源工具在 WSL 下 80 通过。生产库及测试 APK 的 `--local-static`、16KB zipalign、签名校验通过，严格来源与逐字节重建验收仍待完成。
