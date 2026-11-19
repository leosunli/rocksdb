# RocksDB 数据恢复能力分析

本文基于 RocksDB 源码提交 `0df5500d3`，分析 RocksDB 在进程崩溃、机器掉电、WAL 损坏、SST/Blob 文件丢失、MANIFEST 损坏以及备份恢复等场景下的数据恢复能力。

这里的“无损”需要区分两个参照点：

- 相对于已经可靠持久化的数据是否无损；
- 相对于应用已经收到成功返回的所有写入是否无损。

当写入使用 `sync=false` 时，这两个集合并不相同。应用已经收到成功返回的写入仍可能只存在于操作系统页缓存中，机器掉电后允许丢失。

## 1. 总体结论

RocksDB 的恢复能力可以分为四个层次：

1. **正常崩溃恢复**：读取 MANIFEST 重建 LSM 状态，再重放 WAL。只要所需文件完整，通常可以无损恢复到持久化边界。
2. **一致性回退**：WAL 或 SST 不完整时，回退到一个可用的历史时间点。通常会丢失最近数据，但优先维持逻辑一致性。
3. **备份或检查点恢复**：从 Backup 或 Checkpoint 恢复到创建时的数据库状态。相对于该备份时间点通常无损。
4. **抢救式修复**：使用 `best_efforts_recovery`、BlobDB repair 或 `RepairDB()` 尽量恢复仍可读取的数据。这些模式明确允许丢失数据。

RocksDB 不是带数据冗余或纠删码的数据修复系统。单份 SST 或 Blob 文件的内容已经物理丢失或严重损坏时，RocksDB 通常不能重建原值，只能回退、跳过文件，或者依赖独立备份和存储副本。

## 2. 恢复能力矩阵

| 故障场景 | 恢复方式 | 是否有损 | 一致性说明 |
|---|---|---:|---|
| 进程崩溃，WAL、MANIFEST 和 SST 完整 | 正常 `DB::Open()` 和 WAL 重放 | 通常无损 | 恢复到持久化边界 |
| 操作系统崩溃，但 WAL 已进入内核页缓存 | WAL 重放 | 通常无损 | 取决于文件系统和操作系统状态 |
| 机器掉电，写入使用 `sync=false` | 恢复已经落盘的 WAL/SST | 可能丢最近写入 | 恢复后的数据库应保持一致 |
| WAL 最后一条记录未写完整 | 容忍尾部截断或回退到损坏前 | 通常只丢未完成记录 | 一致 |
| WAL 中间损坏 | PIT 停止回放，或 skip 模式跳过坏记录 | 有损 | PIT 保持时间点语义；skip 不保证 |
| CURRENT 丢失 | BER 枚举 MANIFEST 并重建 CURRENT | 可能回退 | 取决于可用 MANIFEST |
| 最新 MANIFEST 损坏 | BER 尝试更老的 MANIFEST | 通常可能有损 | 每个 CF 恢复到可用版本 |
| SST 文件丢失或被截短 | BER 回退或丢弃最近 L0 后缀 | 有损 | 尽量保持有效时间点 |
| SST 内部随机损坏 | 默认报错；可尝试 BER 或 `RepairDB()` | 通常有损或失败 | BER 不是通用坏块修复器 |
| Blob 文件丢失 | BlobDB repair 把不可读值按删除处理 | 明确有损 | 不保证时间点一致 |
| 所有 MANIFEST 丢失，但还有 SST/WAL | `RepairDB()` 扫描文件并重建 MANIFEST | 明确有损 | 不保证原始快照一致 |
| 有完整 Backup 或 Checkpoint | Restore 或直接打开 Checkpoint | 相对于备份点无损 | 恢复到备份时间点 |
| SST/Blob 已删除且没有其他副本 | RocksDB 自身无法重建 | 不可恢复 | 需要外部副本 |
| 加密密钥丢失或底层存储整体损坏 | RocksDB 无法解决 | 不可恢复 | 需要外部灾备 |

## 3. 正常崩溃恢复

RocksDB 的持久化状态主要由以下部分构成：

- SST/Blob：已经通过 flush 或 compaction 持久化的数据；
- MANIFEST：记录 Column Family、SST/Blob 文件集合、LSM 版本和序列号等元数据；
- WAL：保护尚未持久化到 SST 的 memtable 写入；
- CURRENT：指向当前 MANIFEST。

正常打开数据库时，大致执行以下流程：

```text
读取 CURRENT
    -> 打开 MANIFEST，重建各 Column Family 的 Version
    -> 找出 MANIFEST 尚未覆盖的 WAL
    -> 按 WAL 编号顺序重放 WriteBatch 到 memtable
    -> 必要时把恢复出的 memtable flush 为 L0 SST
```

源码按编号排序 WAL 并调用 `RecoverLogFiles()`：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L907)。WAL 中恢复出的 WriteBatch 会重新插入 memtable：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L1698)。

WAL 恢复路径会执行 checksum 校验。即使部分 paranoid 检查关闭，代码仍然要求 WAL reader 做 checksum，避免损坏内容生成异常序列号或其他错误状态：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L1540)。

### 3.1 `sync` 对持久性的影响

`WriteOptions::sync` 的源码契约为：

- `sync=false`：进程崩溃但机器不重启时通常不会丢失；机器掉电可能丢失最近写入；
- `sync=true`：写入返回前调用 `WritableFile::Sync()`，语义近似 `write()` 后执行 `fdatasync()`。

参见 [`include/rocksdb/options.h`](../include/rocksdb/options.h#L2894)。

所以正常恢复的“无损”边界是：

- 对已经同步落盘的写入，正常崩溃恢复应当无损；
- 对只存在于 memtable 或尚未同步的 WAL，掉电时允许丢失；
- `sync=false` 的成功返回不能被解释为抗机器掉电的持久性承诺。

### 3.2 禁用或手动刷新 WAL

设置 `WriteOptions::disableWAL=true` 后，写入不会先进入 WAL，崩溃时尚未 flush 的 memtable 数据可能全部丢失：[`include/rocksdb/options.h`](../include/rocksdb/options.h#L2913)。

设置 `DBOptions::manual_wal_flush=true` 后，RocksDB 不再在每次写入后自动把用户态 WAL 缓冲写入文件，需要应用显式调用 `FlushWAL()`：[`include/rocksdb/options.h`](../include/rocksdb/options.h#L1615)。

## 4. WAL 损坏恢复模式

RocksDB 提供四种 `WALRecoveryMode`：[`include/rocksdb/options.h`](../include/rocksdb/options.h#L413)。

### 4.1 `kAbsoluteConsistency`

发现任何 WAL 损坏就拒绝打开数据库。

- 不静默丢数据；
- 可用性最低；
- 本质上是检测异常后失败，而不是有损抢救。

适合不能接受隐式数据损失的系统。

### 4.2 `kTolerateCorruptedTailRecords`

允许 WAL 最后一条记录不完整，以及预分配 WAL 尾部存在零字节。这主要覆盖进程在写 WAL 过程中崩溃的场景。

- 完整记录仍然恢复；
- 最后一条未完整写入的记录可能丢失；
- 如果发现 WAL 中间损坏，仍然返回错误。

### 4.3 `kPointInTimeRecovery`

这是当前选项声明的默认模式：[`include/rocksdb/options.h`](../include/rocksdb/options.h#L1533)。

发现 WAL 不一致时，RocksDB 停止继续回放，保留损坏点之前的状态。损坏点之后即使还有可读记录，也不会继续应用。普通 I/O error 仍然返回失败，而不是当作可忽略 corruption：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L1802)。

这个模式具有以下特点：

- 可能丢弃损坏位置之后的所有写入，因此是有损恢复；
- 不跨过 WAL 空洞任意拼接状态；
- 恢复结果尽量对应损坏前的一个真实时间点；
- 如果部分 Column Family 已经 flush 到损坏点之后，跨 CF 无法构造一致状态时，打开可能失败。

跨 CF 检查见 [`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L1876)。

### 4.4 `kSkipAnyCorruptedRecords`

该模式忽略损坏记录并继续读取后续 WAL。源码会对这种模式直接忽略相关错误：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L1802)。

这是最激进的 WAL 抢救模式：

- 明确有损；
- 不保证恢复到一个真实存在过的时间点；
- 丢失 Delete 可能导致旧值重新出现；
- 丢失事务或 WriteBatch 的关键记录可能破坏业务状态机语义。

它只适合记录相互独立、业务愿意接受部分缺失的抢救场景。

## 5. Best-efforts recovery

`DBOptions::best_efforts_recovery=true` 主要面向以下故障：

- SST/Blob 文件缺失；
- 文件被截短；
- 数据库目录是不完整的文件系统物理拷贝；
- CURRENT 缺失；
- 最新 MANIFEST 不可用；
- SST 被另一个同大小文件替换，并且 MANIFEST 记录了 SST unique ID。

完整 API 契约见 [`include/rocksdb/options.h`](../include/rocksdb/options.h#L1735)。

### 5.1 恢复原理

BER 不只依赖 CURRENT 指向的最新 MANIFEST，而是：

1. 枚举可用 MANIFEST；
2. 从最新向更老的 MANIFEST 尝试；
3. 顺序回放 MANIFEST 中的 VersionEdit；
4. 验证候选版本引用的 SST/Blob 是否可用；
5. 保存最近一个文件完整的有效版本；
6. 在允许的情况下，丢弃最近缺失的 L0 文件后缀；
7. 重建 CURRENT 和 MANIFEST。

MANIFEST 选择和尝试逻辑见 [`db/version_set.cc`](../db/version_set.cc#L7470)。

候选版本从有效变成无效时，代码回滚导致文件缺失的 VersionEdit，保存编辑之前的版本，然后继续扫描 MANIFEST：[`db/version_edit_handler.cc`](../db/version_edit_handler.cc#L881)。

### 5.2 数据损失边界

BER 明确可能有损，而且不重放 WAL。进入 BER 路径后不会枚举 WAL：[`db/db_impl/db_impl_open.cc`](../db/db_impl/db_impl_open.cc#L815)。

因此即使 WAL 文件仍然完整，只存在于 WAL 或 memtable 中的最新数据也不会被恢复。结果可能是：

- 每个 Column Family 的较早有效时间点；
- 丢掉最近 L0 文件后缀后的版本；
- 极端情况下恢复成空数据库。

### 5.3 跨 Column Family 一致性

普通情况下，不同 Column Family 可能回退到不同时间点，不能保证跨 CF 一致。

使用 `atomic_flush` 时，MANIFEST 中的 AtomicGroup 会按照全有或全无方式恢复。代码缓存相关 CF 的候选版本，只有全部可用才一起安装：[`db/version_edit_handler.cc`](../db/version_edit_handler.cc#L762)。

### 5.4 局限

- BER 更适合文件丢失、截短和不完整复制；
- 它不是 SST 内部随机坏块的通用修复器；
- 至少需要一份有效 MANIFEST 才能恢复出非空状态；
- 所有 MANIFEST 都不可用时，需要转向 `RepairDB()`。

## 6. BlobDB 文件缺失恢复

当前源码提供 `best_efforts_blob_db_repair`，用于 integrated BlobDB 中 SST 仍存在，但其引用的外部 Blob 文件缺失或无法打开的情况：[`include/rocksdb/options.h`](../include/rocksdb/options.h#L1784)。

该模式不能重建丢失的 Blob 内容，而是：

- 保留仍可读取的数据；
- 把缺失 Blob 文件记录为 dropped；
- 引用缺失 Blob 的标量值读取为 NotFound；
- 后续 compaction 把丢失值转换成同序列号的删除记录，防止旧值重新出现；
- wide-column entity 可以保留仍然可用的列。

它明确是有损恢复，并且不保证恢复到一致时间点。修复结果会记录到新 MANIFEST，并可能在后续不启用该选项的启动中继续生效。

## 7. `RepairDB()` 抢救式修复

公共 API 明确警告 `RepairDB()` 可能丢失数据：[`include/rocksdb/db.h`](../include/rocksdb/db.h#L3076)。

### 7.1 修复流程

`RepairDB()` 的主要步骤是：

1. 获取数据库 LOCK，要求离线修复；
2. 枚举 SST、WAL 和 MANIFEST；
3. 把旧 MANIFEST 移入 `lost/`；
4. 创建新的空 MANIFEST；
5. 扫描每个 SST，提取 key 范围、序列号和 CF 信息；
6. 无法打开或扫描的 SST 移入 `lost/`；
7. 扫描 WAL，把可读 WriteBatch 转换成新 SST；
8. 跳过 WAL 中损坏的记录；
9. 把成功读取的 SST 作为 L0 文件加入新 MANIFEST。

总体流程见 [`db/repair.cc`](../db/repair.cc#L203)。

WAL corruption 会被记录为 dropped bytes，然后继续扫描：[`db/repair.cc`](../db/repair.cc#L363)。无法扫描的 SST 会被忽略并归档：[`db/repair.cc`](../db/repair.cc#L527)。修复成功日志也明确打印 `Some data may have been lost`：[`db/repair.cc`](../db/repair.cc#L239)。

### 7.2 为什么必须视为有损

原 MANIFEST 不只是文件列表，还描述数据库版本演进关系。RepairDB 丢弃这些历史，只根据目录中仍能读取的文件重组数据库，因此可能出现：

- 损坏 SST 整个被丢弃；
- 损坏 WAL record 被跳过；
- 丢失 Delete 或 RangeDelete 后，旧值重新可见；
- 不同时间生成的文件组合成一个从未真实存在过的状态；
- 原 level 布局丢失，恢复文件先进入 L0；
- 未提供描述符的 Column Family 被跳过；
- 老格式 SST 缺少 CF 信息时被归入 default CF。

RepairDB 的目标是尽量恢复 key/value，而不是恢复最近一致快照。

### 7.3 正确使用方式

1. 先完整复制损坏数据库目录；
2. 在副本上运行 `RepairDB()`；
3. 对恢复结果执行 checksum 和业务级校验；
4. 导出确认正确的数据并建立新数据库；
5. 不要把唯一原始副本直接作为试验对象反复 repair。

## 8. Checkpoint 恢复

Checkpoint 创建一个可以直接打开的数据库快照：

- 同一文件系统优先 hard link SST/Blob；
- 跨文件系统时复制文件；
- MANIFEST 等可变元数据总是复制；
- 默认触发所有 Column Family flush；
- 不 flush 时，需要把相关 WAL 包含在 Checkpoint 中。

接口契约见 [`include/rocksdb/utilities/checkpoint.h`](../include/rocksdb/utilities/checkpoint.h#L142)。

相对于 Checkpoint 创建时间点，恢复通常无损，但需要注意：

- 创建之后的写入不在快照中；
- 使用非默认 `log_size_for_flush` 且写入禁用了 WAL 时，未 flush 数据不会进入 Checkpoint；
- hard link 只节省空间，不构成独立故障域；同一磁盘整体损坏时，原库和 Checkpoint 可能同时丢失。

## 9. BackupEngine 恢复

BackupEngine 支持：

- 多版本备份；
- SST/Blob 文件跨备份共享；
- 增量复制新文件；
- 恢复指定 BackupID；
- 恢复最新非损坏备份；
- 校验文件存在性和大小；
- 可选 checksum 验证。

文件共享和增量机制见 [`include/rocksdb/utilities/backup_engine.h`](../include/rocksdb/utilities/backup_engine.h#L37)，恢复接口见 [`include/rocksdb/utilities/backup_engine.h`](../include/rocksdb/utilities/backup_engine.h#L503)，备份校验接口见 [`include/rocksdb/utilities/backup_engine.h`](../include/rocksdb/utilities/backup_engine.h#L528)。

当 `BackupEngineOptions::sync=true` 时，接口契约保证机器崩溃或重启后，已完成的备份和恢复保持一致：[`include/rocksdb/utilities/backup_engine.h`](../include/rocksdb/utilities/backup_engine.h#L60)。

Backup 恢复是否有损取决于参照点：

- 相对于选定 BackupID，正常情况下无损；
- 相对于故障发生前的最新数据库状态，会丢失备份之后的写入，RPO 等于备份间隔；
- 如果业务写入使用 `disableWAL=true`，备份时必须使用 `flush_before_backup=true`，否则未 flush 的 memtable 数据不会进入备份。

## 10. 校验能力不等于恢复能力

RocksDB 提供 WAL fragment CRC、SST block checksum、SST whole-file checksum、SST unique ID、`DB::VerifyChecksum()`、`DB::VerifyFileChecksums()` 和 `BackupEngine::VerifyBackup()` 等校验机制。

数据库校验接口见 [`include/rocksdb/db.h`](../include/rocksdb/db.h#L2826)。

这些机制可以：

- 尽早发现静默损坏；
- 防止把损坏 WAL 当作合法 WriteBatch；
- 防止错误 SST 被误认为 MANIFEST 中引用的文件；
- 判断备份文件是否完整。

但 checksum 只能检测内容错误，不能反推出原始内容。真正恢复仍需要其他正确副本，例如：

- Backup；
- Checkpoint；
- 文件系统或云盘快照；
- 对象存储版本；
- 数据库副本节点；
- RAID 或纠删码存储。

## 11. 生产环境建议

如果业务要求较低的 RPO 和数据损失风险，建议：

- 关键写入保持 WAL 开启，不使用 `disableWAL=true`；
- 需要抗掉电确认语义时使用 `WriteOptions::sync=true`，或建立明确的 `SyncWAL()` 策略；
- 定期创建 Backup，并放置在独立故障域；
- 把 Checkpoint 用作快速时间点快照，而不是唯一异地灾备；
- 周期性调用 `VerifyBackup(..., true)`；
- 周期性校验 SST checksum，尽早发现潜在介质损坏；
- 日常启动使用严格或 PIT 恢复，不把 skip、BER 或 RepairDB 当作普通启动配置；
- 只有正常恢复和备份恢复都不可行时，才在数据库副本上尝试 `RepairDB()`；
- 多 CF 强一致业务使用 atomic flush 或事务，并单独验证备份的跨 CF 一致性。

推荐的故障处理顺序如下：

```text
正常 DB::Open() + WAL 回放
        -> 失败
从最近有效 Backup 或 Checkpoint 恢复
        -> 无可用备份
使用 best_efforts_recovery 回退到有效版本
        -> 无有效 MANIFEST
复制故障现场后运行 RepairDB
        ->
执行 checksum、业务校验，导出数据并重建新库
```

## 12. 最终判断

- **正常崩溃恢复**：相对于已经可靠持久化的写入，可以视为无损。
- **完整 Backup/Checkpoint 恢复**：相对于备份创建时间点，可以视为无损。
- **Point-in-time WAL recovery**：保持一致性，但可能丢失损坏点之后的写入。
- **Best-efforts recovery**：回退到仍然可用的文件版本，明确可能丢失最近数据，并且不重放 WAL。
- **BlobDB repair**：把无法读取的 Blob 值按删除处理，明确有损。
- **Skip-any-corrupted-records**：跳过坏 WAL 记录后继续，明确有损且不保证时间点一致。
- **RepairDB**：从仍可读取的 SST/WAL 中尽量抢救数据，明确有损，不保证恢复成历史上真实存在过的一致状态。

