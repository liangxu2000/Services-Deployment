# HANDOVER — Mooncake NoF 改造测试任务书

> ## ⛔ 最高优先级安全令（2026-11-17，环境负责人命令）
>
> **这套测试环境非常重要，绝对禁止任何危险操作。默认且优先：只读。**
>
> - 未经本任务书**逐项明确授权**，一律只读；未来任务同样适用，不存在“默认允许”。
> - 严禁：删除 / 修改 / 重启任何节点、Pod、namespace、DaemonSet、Deployment、StatefulSet、
>   镜像、磁盘、分区、挂载、驱动、网络、kubeconfig。
> - 严禁：`kubectl apply / delete / edit / patch / scale`、`ctr image import / export / rm`、
>   vfio 绑定 / 解绑、SPDK `setup.sh` / 启停、分区 / 格式化 / 挂载 / 卸载、
>   `systemctl restart`、重启节点。
> - **禁止在任何节点/容器创建临时文件（包括 `/tmp`）**；所有解析用纯管道
>   （`tar -xOf ... | strings` 等），不得落盘。
> - 任何非只读动作，必须先报 `NEED-CONFIRM`，等待明确书面授权；未授权一律不做。
> - **不确定是否安全 → 不做，先报告。**
> - 宁可 `BLOCKED`，不可冒险；违反本令的“完成”视为失败。

> 本文件是测试执行 Agent 的**唯一任务来源**。
> 每轮对话 = 一个任务（编号 `NOF-xxx`）。收到消息 `git pull && 任务 NOF-xxx` 后：
> 拉取本分支 → 找到同编号任务 → 执行 → 按固定格式在**会话里**回执。
>
> **权限约束（硬性）**：你对代码仓库只有只读权限。禁止 `git push`、禁止修改仓库文件、
> 禁止把执行结果写回仓库。所有结果只在本轮会话中输出。

---

## 0. 通道与版本

| 项 | 值 |
|---|---|
| 代码仓（fork） | `https://github.com/liangxu2000/Services-Deployment` |
| 分支 | `nof-ssd` |
| 任务书 | 仓库根目录 `HANDOVER.md`（本文件） |
| 首次拉取 | `git clone --branch nof-ssd https://github.com/liangxu2000/Services-Deployment.git` |
| 后续每轮 | `git pull` |
| 任务编号 | `NOF-001` 起递增；每轮只做消息中指定的编号，不要提前做后续任务 |
| 环境 | **2026-11-18 实测 9 节点**（master + worker02/03/05/06/07/08/09/10；worker01/04/11 不在）；最终节点集待用户确认 |
| 结果通道 | 本轮会话（不回写仓库） |

**角色分工（硬性）**：

| 角色 | 职责 |
|---|---|
| **Dev Agent**（本任务书作者） | **开发、思考、决策**：写任务书、改代码、分析回执、决定方案与下一步 |
| **Test Agent**（你） | **测试与环境事实**：按任务书执行、采集证据、报告事实/BLOCKED |

**角色边界（重要）**：

- 你**不做方案设计、不做架构取舍、不自行决定下一步、不执行未授权变更**。
- 你可以写“观察”（例如发现某个现象），但必须标注 **“仅供 Dev Agent 参考”**，不能把它当成结论或行动。
- 任务书与你的判断冲突时：按任务书执行；认为有风险就报 `NEED-CONFIRM`，不要自行改道。
- 决策、代码、编译方案、部署形态由 Dev Agent 负责；你只负责把它变成可验证的测试结果。

**Design note（2026-11-17，用户确认）**：数据盘 1/7/11 的不均分布是**临时状态**，
不作为方案约束；方案按最终均匀盘位设计。

---

## 1. 固定回执格式（每轮必须）

```
任务：NOF-xxx
状态：DONE | BLOCKED | NEED-CONFIRM
结论：一句话
证据：
  - 关键命令 / 关键输出行（不要贴几百行原始日志）
  - 失败项贴退出码 + 错误原文
待确认：
  - 没有就写「无」
观察（仅供 Dev Agent 参考，不是决策）：
  - 可选
```

规则：

- 先给结论，再给证据。
- 只贴关键行；表格优先。
- 遇到需要变更集群、删除资源、重启节点的操作，立即报 `NEED-CONFIRM`，不要自行执行。
- 任何一步失败，不要跳过；记录 `BLOCKED` 并继续能做的只读检查。
- **不得写临时文件**；解析用管道。

---

## 2. 安全红线

> **置顶安全令对所有任务永久有效。** 本节只是对当前任务的具体化，不构成对置顶令的放宽。

当前任务（NOF-004）是**只读任务**。允许：

- `kubectl get / describe / logs / version / cluster-info`
- `ctr -n k8s.io content ls / get`（`get` 只输出到 stdout，管道给 `tar` / `strings` / `jq`，**不落盘**）
- `tar -xOf <tar> <layer> | tar -xOf - <path> | strings`（纯 stdout 管道）
- `ssh <node>` 只读命令：`lsblk`、`nvme list`、`nvme id-ctrl`、`lspci`、`cat`、`readlink`、`ls`
- `crictl images / inspecti`

禁止（本轮）：

- `kubectl exec`
- `ctr image export / import / rm`；任何落盘的镜像导出/解包
- 创建 / 修改 / 删除任何 Kubernetes 资源
- `kubectl apply / delete / edit / patch / scale`
- 绑定 / 解绑 vfio 驱动、`setup.sh`、SPDK 启停
- 写 `sriov_numvfs`、写 hugepages、`modprobe / rmmod`
- 挂载 / 卸载 / 分区 / 格式化 / 删除磁盘数据
- 重启节点、重启服务、改 kubeconfig
- **任何临时文件（包括 `/tmp`）**
- 把任何结果写回仓库或 push

> 如果某项检查必须变更集群才能做，报 `BLOCKED`，等下一轮任务授权。

---

## 3. 背景与已确认决策

- `Services-Deployment` 是集群服务部署工具：`services/k8s-0..3` 对应
  **编译 → 分发镜像 → 起 rank/recall → 加压** 五个环节。
- 本次目标：新增 **NoF 版** 的 rank / recall / 背压服务，并实现合并版。
- Mooncake 源码固定用 `LeeJace993/Mooncake` 的 **`mooncake-vesegment`** 分支。
- 名词：
  - **NoF target**：对外提供 NVMe-oF namespace 的角色（存储提供方）。
  - **initiator**：连接 target 读写的一方；本项目的 `mooncake_client` 是 initiator。

**3.1 已确认的设计决策（2026-11-17，用户确认）**

1. 节点数：**不硬编码**；部署时用 `kubectl get nodes` / DaemonSet 动态发现。
   2026-11-18 重建后实测 12 节点；最终节点集待用户确认（2 盘节点可能移除）。
2. 感知范围：管理端与测试 Agent **只连 controller/master**；集群内部由 Kubernetes 管理。
   只有镜像分发（k8s-1）按现有脚本需要 controller → worker 的 SSH。
3. 实现范围：`k8s-4` rank、`k8s-4` recall、`k8s-5` pressure、`k8s-6` combined
   **全部要实现**；顺序：**k8s-4 rank/recall → k8s-5 → k8s-6**。
4. **k8s-6**：rank 和 recall **共用一个 master**。
5. 分盘：以**整盘为最小单位**，不做 VF、不做每盘 1/3。
   **bring-up 阶段每个 target Pod 只分 1 块整盘**（先打通，后续再扩多块）：
   - k8s-4：每节点 3 个 target（rank 1 + recall 2），rank 用 1 块、recall 两个 target 各用 1 块；
     需要每节点至少 3 块数据盘（2 盘节点组 k8s 时移除）。
   - k8s-6：每节点 1 个 target，bring-up 也先只给它 1 块盘；后续可改为多块/全部。
   - 实现口径：变量 `DISKS_PER_TARGET`（bring-up 默认 1）+ `DISK_INDEX` 映射；
     系统盘/已挂载盘永不选取；数据盘按 BDF/序列号稳定排序后取第 N 块。
   - 这是**功能 bring-up 口径**，不代表最终容量/带宽；性能测试前再扩多块盘。

**3.2 当前 NoF 镜像（2026-11-18，同事构建）**

- 镜像：`mooncake-runtime-nof-vsegment:202611180847.cb1d25e7`（commit `cb1d25e7`）；
  tar 6,626,733,568 B；构建日志显示在 **node1**；同事称已分发到 10 个节点并打 `latest`。
- **但 2026-11-18 集群重建后，12 个节点上都没有该镜像**（A0 实测）；
  镜像 tar 仍在 master 上：`/home/yin/downloads/...` 与
  `/home/y00988740/Services-Deployment/k8s-1-distribute-image/...`。
  **需要等最终节点集确定后重新 import/分发（写操作，待授权）。**
- NoF 服务部署时使用 `MOONCAKE_IMAGE=mooncake-runtime-nof-vsegment:latest`（重新打 tag 后）。
- **node1 的角色**：node1 是同事的**构建/镜像源节点**，**独立于 k8s worker 集群**。
  k8s-0 在 node1 编镜像/导 tar；k8s-1 在 controller 上用 `--remote` 从 node1 拉 tar，
  再分发到 k8s 节点。node1 上还放了 `target_nvme_takeover.sh`、
  `vseg_scripts` / `vsegment_cluster.sh`、`spdk-cpu` 等源材料。
- 待与同事对齐：同事进度写的是「k8s-2 部署 rank/recall（NoF 参数改造）」，
  我们的计划是新增 `k8s-4`；部署前必须确认命名/命名空间。

**3.3 NoF target 架构（2026-11-18，NOF-005 确认，来自同事文档/产物）**

- **target 路线（已确认）**：外部 SPDK `nvmf_tgt` **注入到 store Pod 内**
  （`/home/mooncake/spdk/bin/nvmf_tgt`），配 `target_nvme_takeover.sh`
  （vfio-pci **noiommu** 接管 + RPC 一键脚本，同目录有 `scripts/rpc.py` + `python/spdk/`）；
  再执行 `nof_register <nqn> 1 <traddr> <trsvcid> URMA <master_addr> <size_bytes>` 注册。
  `SPDK_TARGET_ENABLED=false` 可跳过（纯 client 镜像）。
- **不是二选一**：target 侧走外部 `nvmf_tgt`，client/注册侧走上游 `mooncake_client` + `nof_register`
  （后者静态链 SPDK）；两条并行。
- **设备/权限**：store Pod `privileged: true`；hostPath `/sys -> /sys`；`/dev` 由 privileged 带；
  不需要硬件 IOMMU（vfio-pci `enable_unsafe_noiommu_mode`）；hugepages 由 takeover 脚本按 iobuf 池自动上调；
  每节点固定一块**已确认空闲**的 3.5T 数据盘（`/dev/nvmeXn1` 或 BDF），必须排除 `mounted>0`。
- **端口**：每 Pod 独立端口，计划 **4420–4499**，避免同节点多 target 冲突；NQN 每 target 唯一。
- **验收（唯一判据）**：master 日志出现 `MountNoFSegment_slow rc[0]`；
  `nof_register` 退出码 0 是**假成功**。
- **已知坑**：`MountNoFSegment rc=-101`、`nvmf_urma_accept handshake failed rc=-5`（URMA fabric 层，可重试）；
  store 探针不能用 `tcpSocket 17704`（那是 client 端口），应查 `nvmf_tgt` 存活 + 4420 监听，或不设探针靠 master 日志。
- **NoF 参数（同事文档 §7.1）**：
  - master 追加：`--cvm_segments_vsegment_exclusive=true --vsegment_required_medium=nof
    --vsegment_io_alignment=4096 --vsegment_member_count=1 --vsegment_member_extent_size=67108864`；
  - store/client env：`MC_STORE_PARTITION_SLOT_COUNT=1024`、
    `MOONCAKE_VSEGMENT_METRICS_REFRESH_INTERVAL_MS=1000000`、`SPDK_URMA_DEV_NAME`、
    `SPDK_URMA_CAPSULE_TRANSPORT=urma`、`MC_NOF_SUBMIT_CHUNK_BYTES`、`MC_NOF_INFLIGHT_BYTES_LIMIT`、
    `MC_NOF_WORKERS`、`MC_URMA_*`、`URMA_RPC_*`；
  - `cluster_id` 建议独立（如 `mooncake_cluster_nof`），避免与同事集群冲突。
- **镜像**：用 **`202611181103.cb1d25e7`**（6,643,408,896 B，含 `nvmf_tgt` + takeover + rpc.py + python/spdk）；
  `202611180847.cb1d25e7` 不含 target，不能用于 target Pod。**1103 尚未分发**；当前集群重建后无任何镜像。
- **k8s-2/3**：同事**完全未改**（k8s-2 12/12 文件与上游逐字节相同）；`k8s-4/5/6` 不存在。
- **节点口径**：文档写 10、`cluster.json` 写 `k8s-controller-16nodes`、实际 12；后续统一，不硬编码。

**3.4 NOF-006 提取的实现规格（2026-11-18）**

**target 启动（takeover 脚本）**

- 本机副本：`target_nvme_takeover.sh` 36,054 B，MD5 `19893fb9…`；
  与 node1 版本行号不一致（node1 引用 §513-569，本机 490-558），**实现前必须核对 node1 版**。
- 关键参数：
  - `-d <nvmeXn1>` 目标盘（逗号/重复）；默认只分析；`-r` 还原；`-Z` Null bdev 自测（1TiB，不碰盘）。
  - `-p urma` **必须显式给**（默认 tcp）；`-L /usr/lib64`（默认是 GDR 路径）；`-N <urma_dev>`（默认 `udmac0d1e2`）。
  - `-i <fabric_ip>`、`-m <core mask>`、`-I 8192,8192,1024,8388608`、`-O 8388608`、`-C 64`。
  - `-R <strip_kb>` raid0；bring-up 单盘不用。
- `nvmf_tgt`：`$SPDK_DIR/build/bin/nvmf_tgt` + `scripts/rpc.py` 必须存在；
  `nohup ... > /tmp/nvmf_tgt.log &`，PID `/var/tmp/nvmf_tgt.pid`；就绪用 `rpc_get_methods` 30s。
- vfio-pci noiommu：`modprobe vfio vfio_pci`；`enable_unsafe_noiommu_mode=1`；
  `driver_override=vfio-pci`；unbind nvme / bind vfio-pci；必须 root；BDF 每次现查。
- hugepages：基线 2048（4GiB@2MB）；`-I` 时按池上调；写 `/proc/sys/vm/nr_hugepages`；Pod 必须 privileged。
- **端口/NQN 写死**：`LISTEN_PORT=4420`、`NQN=nqn.2026-01.io.spdk:urma-gpu-test`、
  `SUBSYS_SN=URMAGPU0001`，**无选项**。
  → rank 1 target/节点可用默认；**recall 2 target/节点必然冲突**，需加 `-P/-Q` 或每 Pod 改脚本副本。
- **镜像路径 bug**：STEP 5.6 放在 `/home/mooncake/spdk/bin/nvmf_tgt`，脚本校验
  `$SPDK_DIR/build/bin/nvmf_tgt`；启动脚本需建 `build/bin/nvmf_tgt` 符号链（容器内可写层），
  或改用匹配布局。

**store Pod 进程**

- 顺序：takeover → `nvmf_tgt` → `nof_register <nqn> 1 <traddr> <trsvcid> URMA <master_addr> <size_bytes>`。
- **`mooncake_client` 不在 NoF store 链路里**（文档明确从 `exec mooncake_client` 改为上面三步）；
  client/压测侧才用 `mooncake_client`。
- `nof_register` 的 `<master_addr>` 形态、`<size_bytes>` 计算、NQN 一致性：**文档未给**，
  需 node1 `vseg_scripts` / `vsegment_cluster.sh`。

**k8s 改造**

- namespace：`mooncake-nof-rank` / `mooncake-nof-recall`。
- master flags：`--cvm_segments_vsegment_exclusive=true --vsegment_auto_plan=true
  --vsegment_required_medium=nof --vsegment_io_alignment=4096 --vsegment_member_count=1
  --vsegment_member_extent_size=67108864 --vsegment_stripe_size=2097152 --vsegment_reserved_ratio=0`。
- env：`MC_STORE_PARTITION_SLOT_COUNT=1024`、`MOONCAKE_VSEGMENT_METRICS_REFRESH_INTERVAL_MS=1000000`、
  `SPDK_URMA_DEV_NAME`、`SPDK_URMA_CAPSULE_TRANSPORT=urma`；client 侧 `MC_NOF_SUBMIT_CHUNK_BYTES`、
  `MC_NOF_INFLIGHT_BYTES_LIMIT`、`MC_NOF_WORKERS=4`、`MC_URMA_*`、`URMA_RPC_*`。
- **禁用** `MC_NOF_REPLICA_NUM` / `MC_MEM_REPLICA_NUM`（vsegment + medium=nof 下会打架）。
- `cluster_id=mooncake_cluster_nof`。
- YAML：`privileged: true`；hostPath `/sys`；UB=1 + memory；每节点一块空闲 3.5T（排除 mounted>0）；
  探针不用 17704。
- 验收：master 日志 `MountNoFSegment_slow rc[0]`；`nof_register` exit 0 是假成功；
  `ss -ltn :4420`；`ldd nof_register` 无 not found。
- 镜像：**1103**（6,643,408,896 B，含 target）未分发；0847 不含 target。
- **`-Z` Null bdev** 是首 PoC 最安全路径：不碰盘，先验证 `nvmf_tgt` + `nof_register` + master 挂载。
- **master 磁盘风险（2026-11-18 NOF-008 核对）**：master 根盘 2.9T 已用 99%（49G 可用），
  containerd 数据 12G；`evictionHard` 不监控 nodefs，不会报 DiskPressure；imageGC 默认 85% 可能回收。
  **不要往 master 导镜像/落大文件**；k8s-4 的 etcd/master/target 一律调度到 worker。

---

## 4. 任务索引

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| NOF-001 | 环境摸底（只读） | **DONE** | 2026-11-17 |
| NOF-002 | 基线故障 + 镜像/ SR-IOV 只读深挖 | **DONE** | 2026-11-17 |
| NOF-003 | 新 NoF 镜像只读查验 + 探针/虚拟化能力核对 | **DONE** | 2026-11-18 |
| NOF-004 | NoF target 路线只读识别 + 新集群基线记录 | **已取消（NOF-005 已回答路线：外部 nvmf_tgt）** | 2026-11-18 |
| NOF-005 | 本地 k8s_deploy 产物只读盘点 | **DONE** | 2026-11-18 |
| NOF-006 | 从同事文档提取 target takeover / k8s-2 改造规格（只读） | **DONE** | 2026-11-18 |
| NOF-007 | 前置探查 + node1 规格核对（只读） | **DONE** | 2026-11-18 |
| NOF-008 | 分发 1156 到缺失节点（写） | **待用户授权** | 2026-11-18 |
| NOF-009 | 单节点 -Z PoC（写） | **待用户授权** | 2026-11-18 |

---

## 5. 已完成任务摘要

### NOF-001 环境摸底（DONE，2026-11-17）

- 旧集群 10 节点；每盘 `nn=128`、`sriov_totalvfs=32`、`sriov_numvfs=0`；
  驱动内核 `nvme`、无 vfio；hugepages 全 0；宿主机无 SPDK、内核 `nvmet` 未加载。
- 通道：master → 节点 root 免密 + `sudo -n` 全通；kubectl/ctr/crictl/nvme-cli/jq/urma_admin 可用。
- **2026-11-18 集群已重建**：当前 12 节点，旧的 rank/recall/pressure 资源与镜像全部消失。

### NOF-002 基线故障 + 镜像/SR-IOV 深挖（DONE，2026-11-17）

- CrashLoopBackOff：启动期 etcd ClusterIP TCP 不可达 ~70s → startup probe 7 次失败 →
  SIGKILL(137)；已自愈；非 OOM。（该集群已被重建，现值不可复现。）
- 旧镜像元数据无 SPDK/NoF/URMA 痕迹；仓库编译脚本无 `nof_register`、无 `USE_NOF`。
- SR-IOV：32 VF、offset 128/stride 1、VF DID 3755；`sriov_numvfs=0`；
  无 SR-IOV device plugin；hugepages allocatable 全 0；`/dev/vfio` 无 group；cmdline 含 `iommu=pt`。
- **字段解释修正（2026-11-18，NOF-003）**：
  - `vwci=0` 是 **Volatile Write Cache Present**（易失写缓存），**与虚拟化无关**；
  - 虚拟化管理看 **OACS bit 1**：`oacs=0x2be` bit1=1 → **支持 Virtualization Management**；
  - `mnan=0`（无上限）；`ctratt=0x281`；`nsetidmax=0` / `endgidmax=0` → **NVM Sets 未启用**。
  - 因此之前「`vwci=0` 堵死 VF / 每盘 1/3」的结论**作废**；VF 路线仍需厂商文档/实测，
    但**没有被该字段排除**。当前 bring-up 仍按整盘 1 块/target，不依赖 VF。

### NOF-003 新 NoF 镜像只读查验（DONE，2026-11-18）

**A0 / 集群状态**

- 集群重建：节点 creationTimestamp 2026-11-18T03:21:30Z 起，实测 **12 节点**
  （master + worker01…worker11；worker01/04 回来了）。
- **12/12 节点没有任何 mooncake 镜像**（`mooncake-runtime-nof-vsegment` 与 `latest` 均不存在）；
  master 上 `ctr images ls` 仅 calico×3 + ub-network-plugin；无 6.6GB layer。
- 当前无任何 mooncake namespace / DaemonSet / Deployment；Pod 总数 26。
- 镜像 tar 仍在 master：
  `/home/yin/downloads/mooncake-runtime-nof-vsegment-202611180847.cb1d25e7.tar`
  与 `/home/y00988740/Services-Deployment/k8s-1-distribute-image/...`，
  大小 6,626,733,568 B（与同事日志一致）。

**B / 镜像文件级（通过已有 tar 纯管道查验，未解包落盘）**

- RepoTag：`mooncake-runtime-nof-vsegment:202611180847.cb1d25e7`；
  manifest `sha256:7c8303fa…`，config `sha256:ae8c4cd1…`；arm64/linux；11 层未压缩 OCI layer。
- 有：`nof_register`（182,496,920 B）、`nof_unregister`（180,130,792 B）、
  `mooncake_client`（508,792,120 B）、`mooncake_master`（291,709,088 B）、
  `mooncake_kvtest`（510,628,408 B）、`stress_cluster_bench`（492,333,352 B）。
- **无**：SPDK 头（`/usr/local/include/spdk/nvme_urma.h`）、SPDK lib（`libspdk_*`/`libspdk_nvme.a`）、
  `nvmf_tgt` 可执行文件、SPDK RPC（`rpc.py`/`spdk_rpc.py`）。
- **SPDK 静态链入 `nof_register`/`nof_unregister`**：
  `$SPDK v26.09-pre git sha1 e12b7cb21`；含 nvmf RPC decoders、DPDK EAL
  `-A/--pci-allowed`、`-B/--pci-blocked`；vsegment 命中 10,408、urma 命中 16,683。
- 镜像内有 Python `spdk_tgt_create.py`（35,093 B，site-packages），
  设计是 **SSH 到远端 `path:/home/spdk` 拉起 SPDK**，支持 `--transport-type URMA`、`--urma-env`。
- **无内建 target/NoF 启动脚本**；层内 `.sh` 仅 `check-mooncake-ldd.sh`、`config-network.sh`。
- config：Entrypoint `/bin/sh`，Cmd `sleep 3600`；history 末层仍是 sleep，未记录构建命令；
  created 2026-11-18T00:47:17Z。
- 结论（事实层）：镜像**自带 NoF 注册/客户端二进制**，但**不含外部 SPDK target 运行时**；
  存在两条路线：
  - **A：`nof_register` 内建 SPDK**（静态链入）——是否能创建 target/namespace，待查 CLI；
  - **B：外部 SPDK `nvmf_tgt` + `spdk_tgt_create.py`**——需要宿主机 `/home/spdk`，当前宿主机没有。

**C / 其他**

- Live DS 探针：BLOCKED（`mooncake-rank` 不存在）。
- NVMe 字段修正见 NOF-002。
- 透明披露：本轮曾写 `/tmp/_l.txt` 一次并立即删除；后续严禁任何落盘临时文件。

### NOF-005 本地 k8s_deploy 产物只读盘点（DONE，2026-11-18）

- 目录：164 文件 / 2,549,698 B；顶层 7 文件 + `Services-Deployment-main/`。
- 核心文档：`k8s-nof-vsegment.md`（与 `_v1.md` MD5 相同）、
  `k8s-nof-vsegment-progress-20261118.md`、`kvtest-nof-vsegment-adaptation.md`。
- 脚本：`build-mooncake-runtime-image.sh`（NoF 版，STEP 5.5 注入 4 个系统库、STEP 5.6 注入 SPDK target、
  离线 pip）、`probe-nof-prereq.sh` + `remote-probe-body.sh`（纯只读前置探查）。
- 同事 `Services-Deployment-main/` **只改了 k8s-0 和 k8s-1**；k8s-2/3 与上游逐字节相同；无 k8s-4/5/6。
- 关键规格见 3.3（target 路线、设备/权限、NoF 参数、验收判据、镜像 tag）。
- 敏感信息：未发现明文密码/token；IP/路径已 redacted。
- BLOCKED：node1 上的 tar、spdk-cpu、`target_nvme_takeover.sh` 不在本地产物内；
  `data/tasks.json` 无运行日志/退出码；脚本未执行（语法/行为未验证）。

### NOF-006 从同事文档提取规格（DONE，2026-11-18）

- 已提取：takeover 参数、进程顺序、k8s-2 改造、验收判据、镜像 1103、探查脚本；详见 3.4。
- **关键阻塞**：
  1. 本机 takeover 副本与 node1 版本不一致，端口/NQN 是否已参数化未知；
  2. `nof_register` 的 `<master_addr>` 形态、`<size_bytes>` 计算、NQN 一致性无来源；
  3. node1 `vseg_scripts` / `vsegment_cluster.sh` 不可达；
  4. 镜像 1103 路径 bug（`bin/` vs `build/bin/`）。
- **观察**：`-Z` Null bdev 适合首 PoC；端口/NQN 写死使 recall 2 target/节点不可行；
  hugepages 按池上调；多盘时 `member_count` 取值文档未回答。

### NOF-007 前置探查 + node1 核对（DONE，2026-11-18）

- 节点集：**9 节点**（master, worker02/03/05/06/07/08/09/10）；worker01/04/11 不在。
- 盘：worker02/06/07/08/09/10 各 7 块空闲 3.5T；worker05 11 块；**worker03 仅 1 块**；
  master 5 块 3.5T + 2 块 7T。盘符已重排，部署时必须现查。
- 9/9 节点：vfio/vfio_pci/vfio_iommu_type1 未加载；hugepages=0；无 nvmf_tgt 残留；
  liburma 全套在；URMA 设备每节点不同（`bonding_dev_*`，11–53 个）。
- node1：root 免密可达；takeover 脚本 MD5 `c3bc2308…`（36,334 B），常量与本地副本一致
  （NQN/4420 等写死，无 `-P/-Q`）。
- **端口/NQN 已有参数化方案**：`/home/yin/Vsegment/vseg_scripts/run_vesegment_target.sh`
  用 awk 打补丁到 `/tmp` 副本，端口 4420–4499，带端口占用/进程存活护栏；k8s-4 可照搬。
- **`nof_register` 真实调用**：`nof_register <nqn> 1 <traddr> <trsvcid> URMA <master_ip:port> <capacity>`；
  capacity 默认 **100 GiB**（107374182400），跨轮次须一致；master 是 `ip:port`，不是 `etcd://`。
- **master 真实参数**：除 vsegment flags 外，还需 `--enable_ha --enable_oplog --ha_backend_type=etcd
  --submaster_count=1 --rpc_port 50058 --metrics_port 9008 --default_kv_lease_ttl/soft_pin_ttl`。
- **镜像实际是 1156**：`202611181156.9b86170a`（6,646,642,688 B，含 `nvmf_tgt` + takeover + rpc.py +
  python/spdk）；已分发到 worker02/03/05/06/07/08/10（7 台，digest 一致，latest 指向它），
  **master/worker09 缺失**。1103 不是最终分发版本；0847 不含 target。
- **`preflight_check.sh` 是破坏性的**（kill master/清 etcd/`rm -rf /var/lib/etcd`/还原盘），**禁止运行**；
  `probe-nof-prereq.sh` 的 ssh 缺 `-n`，复用需补。

---

## 6. 当前任务

### NOF-008 分发 1156 到 worker09（待用户授权，不要执行）

**决策（Dev Agent）**：**只补 worker09，不碰 master**。master 根盘 99% 满（49G 可用），
导入 6.6G content + 约 6.6G 解包 ≈ 13G，且 imageGC 默认 85% 可能回收；master 是控制面，不冒这个险。
k8s-4 的 etcd/master/target Pod 一律调度到 worker（7 台 worker 已有镜像；补 worker09 后 8 台 worker 都有）。

- 源 tar：controller `/home/yin/downloads/mooncake-runtime-nof-vsegment-202611181156.9b86170a.tar`
  （6,646,642,688 B，sha256 `58a718b8…`）。
- 步骤（逐条可停）：
  1. 记录 worker09 导入前 `crictl images` 快照；
  2. `scp` tar 到 worker09 的 `/home/yin/downloads/`（**唯一授权的临时文件**，6.6G）；
  3. worker09 `sha256sum` 校验 `58a718b8…`；
  4. `ctr -n k8s.io images import <tar>`（不用 `--no-unpack`；worker09 剩 525G，正常解包）；
  5. `ctr -n k8s.io images tag --force docker.io/library/mooncake-runtime-nof-vsegment:202611181156.9b86170a docker.io/library/mooncake-runtime-nof-vsegment:latest`；
  6. `crictl inspecti` 校验 tag 与 latest 的 id = `sha256:7dda528a…`；
  7. 删除 worker09 上的 tar（**授权的清理**；回退需重新 scp）。
- 回退：`ctr -n k8s.io images rm` 两个 tag；不影响 w00921547 与既有 Pod。
- **不要碰 master；不要 import 到 master。**
- **用户未明确授权前，不要执行。**

### NOF-009 单节点 -Z PoC（待用户授权，不要执行）

- 目标：在 **worker02** 用 1156 起最小栈（临时 namespace `mooncake-nof-poc`：etcd + master + 1 个 target Pod）：
  - etcd/master/target 都用 `nodeSelector kubernetes.io/hostname=worker02`，确保镜像已在；
  - 建 `build/bin/nvmf_tgt` 符号链；
  - takeover `-Z -p urma -L /usr/lib64 -N <urma_dev> -i <pod_ip> -m 0x3 -y`；
  - `nof_register <nqn> 1 <pod_ip> <port> URMA <master_ip:port> 107374182400`；
  - 验收：master 日志 `MountNoFSegment_slow rc[0]`。
- **不碰任何真实盘**；hugepages 由 takeover 上调（写 `/proc/sys`），Pod 需 privileged。
- 如果 Pod IP 作为 `-i` 失败：改用 `hostNetwork: true` + 节点 fabric IP（`dnsPolicy: ClusterFirstWithHostNet`）。
- 回退：删除临时 namespace/Pod；`nr_hugepages` 恢复基线 0；释放 vfio。
- **用户未明确授权前，不要执行。**

---

### NOF-007 前置探查 + node1 规格核对（DONE，本轮不执行）

**上一轮结论**：NOF-006 提取了 target 规格；关键阻塞是 takeover 版本、端口/NQN 写死、
`nof_register` 参数、镜像路径不匹配。

**本轮目标**：

1. 在 controller 上运行同事的只读前置探查 `probe-nof-prereq.sh`（+ `remote-probe-body.sh`），
   得到当前每节点空闲 3.5T 盘/BDF、URMA 设备名、hugepages、vfio、SPDK 残留、fuser；
2. 如能只读访问 node1，核对 `target_nvme_takeover.sh` 的 MD5/端口/NQN/参数化，
   并读取 `vseg_scripts` / `vsegment_cluster.sh` 中 `nof_register` 的真实调用；
3. 记录 1103 tar 位置与当前节点集。

**授权边界**：只读。允许运行 `probe-nof-prereq.sh` / `remote-probe-body.sh`（其 body 全只读）、
`ssh node1` 只读命令（**用户已授权；node1 地址由本轮消息给出，不写公开仓库**）；
禁止写盘、绑定 vfio、改 hugepages、`kubectl apply`、任何临时文件。

#### A. 运行前置探查

```bash
# 在 controller 上、脚本所在目录
export PROBE_B64=$(base64 -w0 remote-probe-body.sh)
bash probe-nof-prereq.sh
```

如果脚本会写临时文件，先检查；如会写，改用只读命令手动采集，不要落盘。

目标输出（每节点）：

- 空闲 3.5T 数据盘：`mounted=0`、`holders` 空、`sub_parts=0`；
- BDF：必要时补 `readlink -f /sys/block/<dev>/device`；
- URMA 设备名（`urma_admin show`）；
- hugepages、vfio 模块/`/dev/vfio`、SPDK 残留、fuser。

#### B. node1 只读核对（用户已确认可连；地址由本轮消息给出，不写公开仓库）

- 先 `ssh <node1> true` 确认连通；不通报 `BLOCKED`。
- `md5sum /home/yin/nof/spdk/target_nvme_takeover.sh`；
- 读关键行：`LISTEN_PORT`、`NQN`、`SUBSYS_SN`、getopts 是否有 `-P/-Q`；
- 读 `vseg_scripts` / `vsegment_cluster.sh` 中 `nof_register` 的调用：
  `<master_addr>` 形态（`ip:port` / `etcd://`）、`<size_bytes>` 计算、NQN 是否与 takeover 一致；
- 记录 1103 tar 路径/大小。
- 若不可达：`BLOCKED`，列出需要的文件/命令，不要尝试密码登录。

#### C. 记录

- 当前节点数/名称；是否仍有 2 盘节点；磁盘编号是否在重建后变化；
- 回执：每节点盘/BDF 表、URMA 设备、hugepages、vfio、SPDK 残留；
  node1 版本差异；`nof_register` 真实参数；1103 tar 位置；BLOCKED。

---

### NOF-006 从同事文档提取 target takeover / k8s-2 改造规格（DONE，本轮不执行）

**上一轮结论**：NOF-005 确认 target 路线为「外部 `nvmf_tgt` 注入 Pod + `target_nvme_takeover.sh` +
`nof_register`」；k8s-2/3 未改、k8s-4/5/6 不存在；镜像 1103 含 target 但未分发。

**本轮目标**：只读同事的 `k8s-nof-vsegment.md` / `k8s-nof-vsegment-progress-20261118.md` /
`kvtest-nof-vsegment-adaptation.md`，提取可以直接写进 k8s-4 的实现规格。**不要执行任何脚本。**

**授权**：只读本地文件（`Get-Content` / `cat` / `Select-String` / `grep`）。
禁止执行、修改、解压、落盘临时文件；敏感值标 `redacted`。

**需要提取的内容**：

1. `target_nvme_takeover.sh`：
   - 完整调用方式/参数（磁盘 BDF 或 `/dev/nvmeXn1`、hugepages、RPC 端口、NQN、traddr/trsvcid）；
   - `nvmf_tgt` 启动命令/路径（`/home/mooncake/spdk/bin/nvmf_tgt`）；
   - `SPDK_TARGET_ENABLED` 的用法；
   - 4420–4499 端口如何分配；NQN 命名规则。
2. store Pod 内的进程顺序和命令：
   - takeover → nvmf_tgt → nof_register → mooncake_client？每一步的完整命令（敏感值 redacted）；
   - `nof_register <nqn> 1 <traddr> <trsvcid> URMA <master_addr> <size_bytes>` 的实际参数来源。
3. k8s-2 改造清单 §7.1–7.5：
   - master 追加 flags（vsegment / medium / alignment / member）；
   - store env（`MC_STORE_PARTITION_SLOT_COUNT`、`SPDK_URMA_*`、`MC_NOF_*`、`URMA_RPC_*`）；
   - resources / privileged / hostPath / probes 的变更；
   - namespace 命名（`mooncake-nof-rank` / `mooncake-nof-recall`）。
4. 验收与坑：
   - 唯一判据 `MountNoFSegment_slow rc[0]` 的上下文；
   - `rc=-101` / `handshake failed rc=-5` 的处理/重试；
   - 探针为什么不能用 17704，应该用什么。
5. 镜像 1103：
   - tar 路径/大小/分发状态；`SPDK_TARGET_ENABLED=false` 的纯 client 用法。
6. 前置探查脚本：
   - `probe-nof-prereq.sh` 的只读命令清单，是否可直接在最终集群上跑；
   - 输出字段（盘/BDF/mounted/holders/vfio/hugepages/liburma/SPDK 残留/fuser）。

**输出格式**：

- 一份「实现规格摘要」，按上面 6 节列命令/参数/路径（IP/密钥 redacted）；
- 每条注明来源文档与章节/行号；
- 不要贴整份文档；不要执行；
- BLOCKED 项。

---

### NOF-005 本地 k8s_deploy 产物只读盘点（DONE，本轮不执行）

**背景**：同事在测试 Agent 所在机器留下了一份 k8s 部署产物目录；
完整路径由本轮会话消息给出（HANDOVER 在公开仓库，路径不写死）。
如果消息里没有路径 → 报 `BLOCKED`，不要猜。

**授权边界**：只读本地文件。允许 `ls / dir / Get-ChildItem / find`、`cat / type / Get-Content`、
`file`、`stat`。禁止：执行任何脚本/二进制；`kubectl apply`；修改/删除/移动/重命名文件；
解压 tar/镜像；把文件内容写回仓库；把密码/token/内网 IP 原样贴出（标 `redacted`）。

**步骤**：

#### A. 目录清单（Windows PowerShell 或 Git Bash 二选一）

```powershell
Get-ChildItem -LiteralPath '<LOCAL_ARTIFACT_DIR>' -Recurse -File |
  Select-Object FullName, Length, LastWriteTime | Format-Table -AutoSize
```

```bash
find '<LOCAL_ARTIFACT_DIR>' -type f -printf '%p\t%s\t%TY-%Tm-%Td %TH:%TM\n' | sort
```

如果路径不存在：列出上一级目录，报 `BLOCKED`。

#### B. 分类阅读（只读文本；二进制/大文件只列名字和大小）

- `.yaml/.yml/.json`：k8s manifest；记录 kind/name/namespace、image、env、volumes、
  resources、securityContext、device mounts、hugepages、nodeSelector。
- `.sh/.py/.ps1/.bat`：只读文本；记录用途、参数、关键命令；**不要执行**。
- `.md/.txt/README`：总结工作流、前置条件、步骤。
- `.tar/.img/.bin/.exe` 等：只列名字/大小，不读取/解压/执行。

#### C. 重点找（与 NoF 相关）

- NoF target 的启动/注册方式：`nof_register`、`spdk_tgt_create.py`、`nvmf_tgt`、NQN、
  `traddr/trsvcid/trtype`（是否 URMA）、`/home/spdk`。
- `MC_NOF_*`、`MC_MEM_*`、`vsegment`、`enable_offload`、`GLOBAL_SEGMENT_SIZE`。
- 镜像名/tag（是否 `mooncake-runtime-nof-vsegment`）。
- 是 k8s-2 的修改，还是 k8s-4/5/6 的新服务；namespace 命名。
- 设备挂载：vfio、hostPath、`/dev/nvme*`、hugepages、privileged。
- 是否包含 README/步骤文档，能直接说明同事的意图。

#### D. 输出

1. 文件清单表：路径、类型、大小、一句话用途。
2. 关键发现：按 C 的重点逐条列。
3. 与当前仓库 `services/k8s-2` 的差异摘要（如果能对照；不要修改仓库）。
4. 敏感信息只标 `redacted`。
5. `BLOCKED` 项。

> 再次强调：**不执行、不修改、不解压、不落盘**。

---

### NOF-004 NoF target 路线只读识别 + 新集群基线记录（已取消，不执行）

> NOF-005 已确认 target 路线；NOF-004 取消。保留原文仅供追溯。

**上一轮结论**：集群重建为 12 节点、无 mooncake 镜像/资源；新镜像自带走 NoF 注册/客户端二进制，
SPDK 静态链入 `nof_register`，但镜像内没有 `nvmf_tgt`/SPDK 头/lib；`vwci` 解释已修正。

**本轮目标**：不做任何变更、不落盘，只读回答：

1. `nof_register` 到底是「只注册」还是「自带 target 创建能力」？它的 CLI/用法是什么？
2. 镜像里的 `spdk_tgt_create.py` / `mooncake_ssd_register.py` 期望的工作流是什么？
3. 新集群的 12 节点基线（节点、namespace、资源、镜像）是什么？
4. 最终节点集是否已稳定（12 还是还会移走 2 盘节点）？

**授权边界**：只读。允许 `kubectl get/describe/logs`、`ctr content ls/get`、
`tar -xOf | tar -xOf | strings/jq`（纯管道）、`ssh` 只读命令、`crictl images/inspecti`。
禁止：`kubectl exec`、`ctr image import/export/rm`、创建/删除资源、写 sriov/hugepages/vfio、
`modprobe`、重启任何东西、**任何临时文件（包括 `/tmp`）**。

#### A. 新集群基线（只读）

```bash
kubectl get nodes -o wide
kubectl get ns
kubectl get pods -A -o wide
kubectl get ds,deploy,sts -A -o wide
crictl images | grep -i mooncake || true
```

记录：节点数、节点名、master/worker、是否还有 2 盘节点、有无 mooncake 资源/镜像。

#### B. `nof_register` / `nof_unregister` CLI 字符串（不落盘、不执行）

从 master 上已有的 tar 中，定位包含二进制的 layer，然后纯管道提取 `strings`：

```bash
TAR=/home/yin/downloads/mooncake-runtime-nof-vsegment-202611180847.cb1d25e7.tar
LAYER=<包含 home/Mooncake/outputs/.../bin/nof_register 的 layer>

tar -xOf "$TAR" "$LAYER" \
  | tar -xOf - home/Mooncake/outputs/mooncake-cb1d25e7-aarch64-linux/bin/nof_register \
  | strings | grep -E 'Usage|usage|--[a-z-]+|nqn|nsid|traddr|trsvcid|trtype|master|register|target|subsystem|namespace|listen|pci|spdk|urma' \
  | head -200
```

同样看 `nof_unregister`。

目标（只报事实）：

- 是否有 `target`/`subsystem`/`namespace`/`listen`/`pci` 相关参数；
- 是否只有 `nqn/nsid/traddr/trsvcid/trtype/master` 这类**注册参数**；
- 是否能看出它自己创建 target，还是只连接外部 target 注册。

#### C. 镜像内脚本（纯管道读取）

```bash
# 先找脚本路径（用 tar -tf 列表）
tar -xOf "$TAR" "$LAYER" | tar -tf - | grep -E 'spdk_tgt_create|mooncake_ssd_register|\.sh$'

# 读 spdk_tgt_create.py 的 argparse/关键路径（不落盘）
tar -xOf "$TAR" "$LAYER" \
  | tar -xOf - usr/local/lib64/python3.11/site-packages/mooncake/spdk_tgt_create.py \
  | grep -nE 'add_argument|spdk_target_info|nvmf_tgt|rpc.py|transport|urma|pci' \
  | head -120
```

目标（只报事实）：外部 SPDK 路线需要哪些路径/参数；镜像内是否带 `mooncake_ssd_register.py`。

#### D. 最终节点集

- 只记录当前节点清单和磁盘数量（`lsblk -d` 只读即可，不必逐节点全量扫描；重点确认 2 盘节点是否还在）。
- 不等待用户；把事实报回来即可。

#### 回执要求

1. 新集群基线表（节点、角色、mooncake 资源/镜像有无）。
2. `nof_register` CLI 关键字符串：是否含 target/subsystem/namespace/listen/pci 参数；
   结论为「自带 target 创建」/「仅注册」/「无法判断」之一（附证据行）。
3. `spdk_tgt_create.py` 期望的工作流：外部 SPDK 路径、`nvmf_tgt`、RPC、URMA 参数。
4. 当前节点数、是否还有 2 盘节点。
5. BLOCKED 项；不要给方案决策。

---

## 7. 后续任务（占位，未下发）

> 不要在 NOF-004 中执行以下任务，等对应任务号下发。

- `NOF-008` / `NOF-009`：见「当前任务」，**待用户授权**。
- `NOF-010`：`k8s-4` NoF rank 部署（每节点 1 个 target，bring-up 先 1 块整盘；
  建议排除 master，worker03 只有 1 块盘可跑 rank）
- `NOF-011`：`k8s-4` NoF recall 部署（每节点 2 个 target；端口/NQN 用
  `run_vesegment_target.sh` 的 awk 补丁方案；**worker03 只有 1 块空闲盘，需排除**）
- `NOF-012`：`k8s-5` NoF 加压（复用 k8s-3，切到 NoF namespace）
- `NOF-013`：`k8s-6` combined（每节点 1 个 target，bring-up 先 1 块盘，rank+recall 共用一个 master）

---

## 8. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 初始版本；下发 NOF-001 环境摸底 |
| 2026-11-17 | **置顶最高优先级安全令：默认且优先只读，任何变更须逐项书面授权**；明确 Dev Agent / Test Agent 分工 |
| 2026-11-17 | NOF-001 完成（实测 10 节点）；下发 NOF-002 只读深挖；强化角色边界 |
| 2026-11-17 | NOF-002 完成；下发 NOF-003；数据盘不均视为临时状态 |
| 2026-11-17 | 用户确认设计决策：10 节点、管理端只感知 master、k8s-4/5/6 全部实现、k8s-6 单 master、整盘分配 |
| 2026-11-17 | **bring-up 简化**：每个 target Pod 先只分 1 块整盘；后续再扩多块；`DISKS_PER_TARGET` 可配 |
| 2026-11-18 | 同事产出 NoF-vsegment 镜像；NOF-003 改为查验新镜像 |
| 2026-11-18 | NOF-003 完成：集群重建为 12 节点、镜像未导入；镜像含 `nof_register`/客户端、SPDK 静态链入 `nof_register`、无 `nvmf_tgt`；**修正 `vwci` 误读（OACS bit1=1 支持 VMC）**；下发 NOF-004 只读识别 target 路线；新增“禁止任何临时文件” |
| 2026-11-18 | 用户提供同事本机 `k8s_deploy` 产物目录；NOF-004 暂停，下发 NOF-005 只读盘点（完整路径不进公开仓库，由会话消息提供） |
| 2026-11-18 | NOF-005 完成：确认 target 路线为外部 `nvmf_tgt` 注入 Pod + `target_nvme_takeover.sh`（vfio-pci noiommu）+ `nof_register`；镜像 1103 含 target 但未分发；k8s-2/3 未改；k8s-4/5/6 不存在。NOF-004 取消，下发 NOF-006 只读提取实现规格 |
| 2026-11-18 | NOF-006 完成：提取 takeover/进程顺序/k8s-2 规格；发现镜像路径 bug（`bin/` vs `build/bin/`）、端口/NQN 写死、`nof_register` 参数缺口；下发 NOF-007 只读前置探查 + node1 规格核对 |
| 2026-11-18 | 用户确认 node1（构建/镜像源节点）可由测试 Agent 只读访问；地址仅在会话消息中给出，不写公开仓库 |
| 2026-11-18 | NOF-007 完成：实际 9 节点；盘/BDF/URMA 实测；node1 端口/NQN 已有 awk 补丁方案；`nof_register` 参数确认（master ip:port、capacity 100GiB）；实际镜像 1156 已分发 7 台、master/worker09 缺失；`preflight_check.sh` 破坏性禁止运行。NOF-008/009 进入待授权 |
