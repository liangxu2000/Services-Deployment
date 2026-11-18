# Mooncake NoF 改造 — 当前状态说明

> 更新：2026-11-18
> 读者：项目同事、测试同学、管理者
> 目的：用最短篇幅说明「我们在做什么、现在到哪、卡在哪、下一步谁做什么」

---

## 一句话

我们要把 Mooncake 的缓存仓库从「只用内存」扩展成「也能用 SSD」。
**现在 target（SSD 出货口）已经能跑通；唯一卡点是 Mooncake master（登记处）里一个 2 行代码的 bug，等同事重编一版镜像。**

---

## 背景（30 秒）

- **Mooncake** 是一个分布式 KV 缓存；原来的 store 节点贡献的是 **DRAM（内存）**。
- **NoF = NVMe over Fabrics**：把 SSD 通过网络暴露成远端盘。
  - **target**：提供盘的一方（仓库出货口）
  - **initiator**：使用盘的一方（来取货的车）
- 我们的部署工具 **Services-Deployment** 负责在 k8s 上把 etcd、master、target 拉起来。
- 目标：先新增 `k8s-4`（NoF 版 rank/recall），最后做 `k8s-6`（rank+recall 合并）。

---

## 现在到哪一步了

| 阶段 | 状态 | 说明 |
|---|---|---|
| 方案确认 | ✅ | 整盘分配；每个 target 先 1 块盘；先 rank，后 recall；k8s-6 最后 |
| 镜像 | ✅ | 同事已做出带 SPDK target 的镜像（1156），PoC 节点上已有 |
| target 启动 | ✅ | `nvmf_tgt` + 假盘 Null0 + URMA 监听 4420，全部正常 |
| `nof_register` | ✅（命令层） | 4/4 次 exit 0；但它退出 0 **不代表成功** |
| master 注册 | ❌ | master 在「自己初始化 SPDK」时失败，报 `opts_size=0` |
| 根因定位 | ✅ | master 的 `spdk_wrapper.cpp` 少初始化一个结构体（2 行修复） |
| 修复/重编 | ⏳ | 等同事应用 patch、重编 master 镜像 |
| 重跑 PoC | ⏳ | 新镜像出来后重跑一次 `-Z` PoC |
| k8s-4 / k8s-5 / k8s-6 | ⏳ | PoC 通过后再做 |

---

## 这次 PoC 证明了什么

在一个 worker 节点上，用**假盘**（Null bdev，不碰真 SSD）跑起了：

- etcd
- master（前台登记处）
- target（出货口）：`nvmf_tgt` + 假盘 + URMA 监听
- `nof_register`（登记命令）

结果：

- **target 侧全部正常**；
- `nof_register` 每次都返回 0（但这是「假成功」）；
- **master 每次都在自己的 SPDK 初始化阶段失败，根本没有连到 target**。

---

## 唯一卡点：master 的 2 行代码 bug

文件：`mooncake-store/src/spdk/spdk_wrapper.cpp`（第 190 行附近）

原因：`struct spdk_env_opts opts;` 没有初始化；SPDK v26.09 的 `spdk_env_opts_init()`
不会自动补 `opts_size`；运行时 `opts_size=0`，SPDK 直接报错，后面的 DPDK/EAL 初始化
跟着失败，最终 `MountNoFSegment rc[-101]`。

修复：

```diff
-    struct spdk_env_opts opts;
+    struct spdk_env_opts opts = {};
     spdk_env_opts_init(&opts);
+    opts.opts_size = sizeof(opts);
```

已排除：

- 不是动态 `libspdk` 版本/ABI 不匹配（master 静态链入 SPDK，NEEDED 里没有 `libspdk`）；
- 不是头文件/库不一致（MD5 一致，三个二进制 SPDK 版本一致）；
- 不是 master 没重编（mtime / build-id / 符号都是新的）。

---

## 下一步：谁做什么

### 同事（关键路径）

1. 把上面 **2 行 patch** 打到 Mooncake fork；
2. 重编 `mooncake_master`，重打 runtime 镜像（新 tag；**保留 1156 用于回退**）；
3. 把新镜像 tag 告诉我们。

### 测试 Agent

1. 新镜像出来后，用同一套 `-Z` PoC 重跑；
2. 验收标准：master 日志出现 **`MountNoFSegment_slow rc[0]`**；
3. 仍然不碰真盘、不碰 master 节点。

### Dev Agent（我）

1. 维护任务书和 patch；
2. PoC 通过后，开始写 `k8s-4` rank / recall 的服务目录；
3. recall 的端口/NQN 用同事已有的补丁方案。

### 协调人（你）

1. 把这份说明和 2 行 patch 转给同事；
2. 决定什么时候授权分发新镜像 / 重跑 PoC。

---

## 安全红线（必须遵守）

- 测试环境**默认只读**；所有写操作都必须单独授权。
- **不要往 master 节点导镜像/写大文件**：根盘已 99% 满。
- **不要往 worker09 导镜像**：根盘 93%，ImageGC 已僵持。
- **hugepages 不能按 2MiB 算**：这台机器默认 **512MiB 页**，写 2048 页会变成 **1TiB**；
  下次 PoC 用 **8 × 512MiB = 4GiB**，或 target 用 `--no-huge`。
- PoC 用**假盘**（Null bdev），不碰真实 SSD。
- 本次 PoC 的所有临时资源已删除，hugepages 已恢复 0，其他命名空间未受影响。

### 当前环境备注

- 集群当前 **9 个节点**；
- worker02 主机失联，暂不使用；
- master 根盘 99%、worker09 根盘 93%（ImageGC 僵持），两者都不导镜像；
- worker03 只有 1 块空闲盘（rank 可跑，recall 要排除）；
- 下一次 PoC 使用 **worker10**（镜像已在、磁盘/内存充足）。

---

## 名词表

| 名词 | 大白话 |
|---|---|
| NoF | NVMe over Fabrics，把 SSD 通过网络当远端盘用 |
| target | 提供盘的一方（仓库出货口） |
| initiator | 使用盘的一方（来取货的车） |
| master | Mooncake 的登记处/调度台 |
| `nof_register` | 把 target 登记到 master 的命令 |
| Null bdev | 假盘，用来做不碰真盘的演练 |
| hugepages | DPDK/SPDK 用的大页内存；这台机器默认 512MiB 一页 |
| `opts_size` | SPDK 结构体的尺寸字段；没填对，SPDK 拒绝初始化 |

---

## 相关文件

- 任务书：`HANDOVER.md`（每轮 `NOF-xxx` 任务）
- 本说明：`NOF-STATUS.md`
- 代码仓：fork 的 `nof-ssd` 分支

---

## 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-18 | 初版：target 通；master 定位到 2 行 bug；等同事重编镜像 |
