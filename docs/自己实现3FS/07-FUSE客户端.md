# FUSE 客户端

## 这个模块解决什么问题

FUSE 客户端的核心价值是：

- 让应用几乎零改造地使用你的分布式文件系统。
- 把文件系统操作转成 meta / storage RPC。

如果目标是尽快获得可用系统，FUSE 通常是最合理的第一版客户端形态。

## MVP 怎么实现

### 最小目标

先支持：

- mount / unmount
- `lookup`
- `getattr`
- `readdir`
- `mkdir`
- `create`
- `open`
- `read`
- `write`
- `flush`
- `release`
- `unlink`
- `rename`

### 客户端内部最小组成

- 一个 meta client
- 一个 storage client
- 一个 inode/handle 缓存
- 最简单的写缓冲或直接写路径

## FUSE 读写主路径应该怎么设计

### 读

1. `open` 时向 meta 取 inode 与 layout。
2. `read` 时按偏移切分成 chunk 访问。
3. 直接向 storage 拉数据。

### 写

1. `write` 根据偏移定位 chunk。
2. 向 storage 提交写。
3. 在适当时机通过 `flush/fsync/close` 回 meta 更新长度和时间戳。

## 建议从一开始就做的客户端状态

- `RcInode`
  缓存 inode 和动态属性。
- `FileHandle`
  保存 open 状态和 session 信息。
- `dirty inode set`
  标记需要后台 sync 的文件。

因为真实文件系统里，客户端不可能每次都完全无状态。

## 为什么真实工程会更复杂

### FUSE 的性能瓶颈很快就会出现

典型问题包括：

- 内核和用户态之间的数据拷贝。
- FUSE 队列锁争用。
- 小 I/O 随机读性能差。
- 同文件高并发写限制。

因此 FUSE 非常适合做功能入口，但不适合做终极性能方案。

### 客户端还会承担很多附加责任

例如：

- 周期性 sync。
- 写回缓冲。
- attribute cache。
- 失效通知。

如果这些全靠服务端驱动，客户端会变得迟钝且低效。

## 向 3FS 风格演进时要补什么

### 演进 1：把 FUSE 客户端做成完整运行时

不要让 FUSE 只是 syscall 的薄转发层。更好的方向是：

- 客户端本地维护 inode 状态。
- 管理 periodic sync。
- 维护共享 buffer 和高性能 I/O 资源。

### 演进 2：把数据路径和 FUSE 控制路径分离

FUSE 可以继续负责：

- open/close/stat/list 这类语义性操作。

而更高性能的数据读写路径可以走独立接口。

## 自己重做时推荐的取舍

- 第一版先求正确，不要过早做复杂 writeback。
- 如果没有强需求，先把 page cache / direct I/O 策略做简单。
- `close` 和 `fsync` 的长度更新语义要认真定义。

## 常见坑

### 坑 1：把所有状态都只存内核 fd

这样后面很难支持 session、后台 sync、原生 I/O 注册等能力。

### 坑 2：把 FUSE 读写都强行同步走 meta

这会极大放大元数据服务压力，也违背了 3FS 风格架构的初衷。
