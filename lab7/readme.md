# XV6 风格文件系统实现文档 (Lab 7: Filesystem)

## 1. 总体架构

本项目实现了一个典型的类 Unix 分层文件系统。为了支持持久化存储和并发访问，系统从下至上构建了六层抽象：

1. **Buffer Cache 层 (bio.c)**：通过内存缓存磁盘块，减少 I/O 损耗，并同步多个进程对同一块磁盘数据的访问。
2. **Logging 层 (log.c)**：实现事务机制。所有对磁盘元数据的修改先记入日志，只有在事务提交时才写入实际位置，确保在系统崩溃后能通过 `recover_from_log` 恢复一致性。
3. **Inode 层 (fs.c)**：提供未命名的文件抽象。每个 Inode 包含文件类型、链接数、大小及数据块索引（`addrs`）。
4. **Directory 层 (dir.c)**：将文件名映射到 Inode 编号。目录本质上是一种特殊的 Inode 记录文件。
5. **Path 层 (dir.c)**：解析如 `/usr/bin/test` 这样的递归路径名。
6. **File Descriptor 层 (file.c)**：最高层的抽象。通过全局文件表 `ftable` 记录每个打开文件的偏移量和权限，支持管道（Pipe）、设备和普通文件。

---

## 2. 核心模块详解

### 2.1 磁盘缓存与并发控制 (`bio.c`)

* **LRU 算法**：`bcache` 维护一个由 `head` 指向的双向循环链表。新使用的块通过 `brelse` 放入头部，最久未使用的块在 `bget` 时从尾部回收。
* **双重锁机制**：
* `bcache.lock`：保护链表结构，防止在分配缓存块时发生竞争。
* `b->lock` (sleeplock)：保护数据内容。由于磁盘 I/O 较慢，使用休眠锁（sleeplock）允许其他进程在当前进程等待 I/O 时继续执行。



### 2.2 日志系统与原子性 (`log.c`)

* **事务保护**：代码中使用 `begin_op()` 和 `end_op()` 包裹写操作。
* **写流程**：
1. `log_write()`：将脏块编号记录在内存中的 `logheader`，并增加缓存块引用计数（pinned）。
2. `commit()`：
* `write_log()`：将修改过的数据块从内存写入磁盘日志区。
* `write_head()`：更新磁盘上的日志头（设置事务已提交标志）。
* `install_trans()`：将数据从日志区拷贝到真正的磁盘位置。





### 2.3 索引节点与布局 (`fs.c` & `mkfs.py`)

* **磁盘布局**：由 `mkfs.py` 脚本定义，包含：
* `boot block` | `super block` | `log` | `inodes` | `bitmap` | `data blocks`


* **bmap (数据块映射)**：支持直接索引。当文件增大时，`inode_bmap` 负责从位图中分配新块并更新 Inode。

---

## 3. main.c 测试逻辑详解

在 `main.c` 中，针对文件系统功能的完整性、并发性和健壮性设计了多维度测试。

### 3.1 `test_filesystem_integrity` (完整性测试)

* **目的**：验证基本读写逻辑。
* **逻辑**：创建文件后写入特定模式的字符串，关闭后重新打开。通过 `readi` 读取内容并与原始数据对比，确保存储和索引逻辑无误。

### 3.2 `test_concurrent_access` (高并发测试)

* **目的**：测试文件系统在多核/多进程下的同步机制。
* **逻辑**：
* 使用 `fork()` 创建 `LAB7_CONCURRENT_WORKERS` 个子进程。
* 每个进程创建独立的文件名（如 `file_0`, `file_1`），并在循环内进行写-读-删除操作。
* **考察点**：这会触发大量的 `balloc` (分配块) 和 `ialloc` (分配 Inode) 竞争，验证全局锁和位图锁是否能防止数据破坏。



### 3.3 `test_crash_recovery` (故障恢复模拟)

* **目的**：验证日志系统的有效性。
* **逻辑**：通过手动调用日志内部接口（模拟在 `write_head` 之后、`install_trans` 之前断电的情况），然后重启系统执行 `log_init` 中的 `recover_from_log`。如果日志头显示有未安装的事务，系统应能自动将数据重写到正确位置。

### 3.4 `test_filesystem_performance` (性能测试)

* **目的**：量化缓存效果。
* **关键指标**：
* `disk_read_count` / `disk_write_count`：记录实际触发底层 `virtio_disk_rw` 的次数。
* **分析**：如果大量重复读取相同块而 `disk_read_count` 没有显著增加，说明 `bio.c` 的缓存命中率良好。



---

## 4. 如何部署与运行

1. **准备环境**：确保 Python 3 可用，用于运行 `mkfs.py`。
2. **生成镜像**：
```bash
python3 mkfs.py fs.img

```


此操作会初始化超级块、创建根目录 `/` 并分配位图。
3. **编译运行**：
编译内核并挂载 `fs.img`。
4. **观察输出**：
内核启动后会打印 `[fs] init start`，并自动运行 `run_lab7_filesystem_tests`。如果所有子项显示 `PASSED`，则表示实现成功。

---

## 5. 实现重点回顾

* **Inode 引用计数与链接数**：区分 `ip->ref`（内存中 Inode 的引用）和 `ip->nlink`（磁盘上目录指向该 Inode 的次数）。
* **休眠锁应用**：在执行磁盘 I/O 时必须释放 `bcache.lock` 以免阻塞整个系统，但必须持有块自身的 `sleeplock`。
* **目录查找**：通过 `dirlookup` 遍历目录文件的数据块，逐个匹配 `struct dirent` 中的文件名。