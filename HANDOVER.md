# HANDOVER — Mooncake NoF 改造测试任务书

> ## ⛔ 最高优先级安全令（2026-11-17，环境负责人命令）
>
> **这套 12 节点测试环境非常重要，绝对禁止任何危险操作。默认且优先：只读。**
>
> - 未经本任务书**逐项明确授权**，一律只读；未来任务同样适用，不存在“默认允许”。
> - 严禁：删除 / 修改 / 重启任何节点、Pod、namespace、DaemonSet、Deployment、StatefulSet、
>   镜像、磁盘、分区、挂载、驱动、网络、kubeconfig。
> - 严禁：`kubectl apply / delete / edit / patch / scale`、`ctr image import / rm`、
>   vfio 绑定 / 解绑、SPDK `setup.sh` / 启停、分区 / 格式化 / 挂载 / 卸载、
>   `systemctl restart`、重启节点。
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
| 环境 | 实测 10 个 Kubernetes 节点（master + 9 worker；原口径 12，worker01/04 不存在）；你已连上 master |
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

---

## 2. 安全红线

> **置顶安全令对所有任务永久有效。** 本节只是对当前任务的具体化，不构成对置顶令的放宽。

当前任务（NOF-002）是**只读任务**。允许：

- `kubectl get / describe / logs / version / cluster-info`
- `ssh <node>` 只读命令：`lsblk`、`nvme list`、`nvme id-ctrl`、`nvme id-ns`、`nvme list-ns`、
  `lspci`、`findmnt`、`cat /proc/meminfo`、`cat /proc/cmdline`、`numactl -H`、`lsmod`、
  `readlink`、`ls`、`sysfs` 读取
- `ctr -n k8s.io image ls / info`、`crictl images / inspecti`
- SSH 连通性测试（`true`，不执行任何变更）

禁止（本轮）：

- `kubectl exec`（本任务不做容器内检查；需要时由后续任务单独授权）
- 创建 / 修改 / 删除任何 Kubernetes 资源（包括临时 DaemonSet、debug Pod）
- `kubectl apply / delete / edit / patch / scale`
- 导入 / 导出 / 删除镜像（`ctr image import / export / rm`）
- 绑定 / 解绑 vfio 驱动、`setup.sh`、SPDK 启停
- 写 `sriov_numvfs`、写 hugepages、`modprobe / rmmod`、加载内核模块
- 挂载 / 卸载 / 分区 / 格式化 / 删除磁盘数据
- 重启节点、重启服务、改 kubeconfig
- 把任何结果写回仓库或 push

> 如果某项检查必须变更集群才能做，报 `BLOCKED`，等下一轮任务授权。

---

## 3. 背景（30 秒）

- `Services-Deployment` 是集群服务部署工具：`services/k8s-0..3` 对应
  **编译 → 分发镜像 → 起 rank/recall → 加压** 五个环节。
- 本次目标：新增 **NoF 版** 的 rank / recall / 背压服务（`k8s-4`、`k8s-5`），
  并评估是否需要合并 rank+recall 的 `k8s-6`。
- Mooncake 源码固定用 `LeeJace993/Mooncake` 的 **`mooncake-vesegment`** 分支
  （自定义 NoF / vsegment / URMA）。
- 名词：
  - **NoF target**：对外提供 NVMe-oF namespace 的角色（存储提供方）。
  - **initiator**：连接 target 读写的一方；本项目的 `mooncake_client` 是 initiator。
  - 「一个 NoF target = 一个 Pod」是待验证的部署假设，不是既定事实。

---

## 4. 任务索引

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| NOF-001 | 环境摸底（只读） | **DONE** | 2026-11-17 |
| NOF-002 | 基线故障 + 镜像/ SR-IOV 只读深挖 | **已下发** | 2026-11-17 |

---

## 5. 已完成任务摘要

### NOF-001 环境摸底（DONE，2026-11-17）

**实测事实**（以本轮回执为准，仓库不保存 IP 列表）：

- 集群：**10 节点**（master + 9 worker；worker01/04 不存在），k8s v1.19.9。
- 现有部署：只有 `mooncake-rank`；etcd 1/1、master Deployment 1/1、`mooncake-store-0`
  DaemonSet desired 10 / ready 9；master 节点上的 store Pod `CrashLoopBackOff`
  （exit 1，restart 20）；**没有 recall**。
- 镜像：`mooncake-runtime-w00921547:latest` 与 `:202611171516.95d8ef07`，
  digest `sha256:003bde03...`，5.4 GiB，arm64。
- 磁盘：每盘 **`nn=128`**（支持 128 namespace），当前每盘只用 1 个 namespace；
  **`sriov_totalvfs=32`**（master 一块 7T 盘除外），但 **`sriov_numvfs=0`** 全网未使能；
  驱动统一为内核 `nvme`，无 vfio 绑定。
- hugepages：2M / 512M / 16G **全部为 0**；SPDK（`nvme_urma.h`、`libspdk_nvme.a`）
  宿主机**缺失**；内核 `nvmet` 未加载、`nvmetcli` 缺失。
- 通道：master → 10 节点 **root 免密 + `sudo -n` 全通**；
  `kubectl`、`ctr`、`crictl`、`nvme-cli`、`jq`、`urma_admin` 均可用。
- 数据盘分布：worker02/06/07/08/09/10 各 7 块；worker05 11 块；
  **worker03/11 各仅 1 块**；master 7 块（含一块无 SR-IOV）。
- UB：每节点 `unifiedbus.com/ub_net_device=96`；`urma_admin show` 全部 ACTIVE。
- 缺失镜像：`memory-pressure:latest` 在 master 与 worker09 不存在，其余 8 节点存在。

**Dev Agent 结论**：硬件具备拆分能力（nn=128 + SR-IOV 32 VF），但 VF、namespace、
hugepages、SPDK 均未就绪；数据盘数量不均（1/7/11），整盘 2/2/3 无法在所有节点成立。
方案未定，先做 NOF-002 的只读深挖。

---

## 6. 当前任务

### NOF-002 只读深挖：基线故障、镜像内容、SR-IOV 细节

**上一轮结论**：10 节点；硬件可拆分（nn=128 / SR-IOV 32 VF），但 VF/hugepages/SPDK
均未就绪；现有 rank 有一个 store Pod CrashLoopBackOff；没有 recall。

**本轮目标**：不做任何变更，补齐决定方案所需的三类事实：

1. 现有 store `CrashLoopBackOff` 的直接原因；
2. runtime 镜像里是否已带 SPDK / NoF / URMA（决定编译与镜像改造范围）；
3. SR-IOV 的 per-VF 隔离能力、集群是否有 SR-IOV 设备插件、hugepages 预留方式。

**授权边界**：只读。允许 `kubectl get/describe/logs`、`ctr image info`、`crictl inspecti`、
`ssh` 只读命令、`lspci` / `nvme id-ctrl` / `cat`。
禁止：`kubectl exec`、创建/删除资源、导出/运行镜像、写 `sriov_numvfs`、改 hugepages、
绑定 vfio、`modprobe/rmmod`、重启任何东西。
如果某项必须写操作才能确认 → 报 `BLOCKED`，不要执行。

#### A. 基线故障（只读日志）

```bash
NS=mooncake-rank
POD=mooncake-store-0-cl7xg   # 以实际 Running/异常 Pod 名为准

kubectl -n "$NS" get pods -o wide
kubectl -n "$NS" describe pod "$POD"
kubectl -n "$NS" logs "$POD" --tail=120 --timestamps
kubectl -n "$NS" logs "$POD" --previous --tail=120 --timestamps
kubectl -n "$NS" get ds mooncake-store-0 -o yaml
```

只读；不要 `exec`、不要删 Pod、不要重启。

#### B. 镜像内容（只读元数据）

```bash
# 实际镜像引用
kubectl -n mooncake-rank get ds mooncake-store-0 \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# 尝试两种元数据检查（不支持就记录 BLOCKED，不要导出/运行镜像）
ctr -n k8s.io image info docker.io/library/mooncake-runtime-w00921547:latest
crictl inspecti docker.io/library/mooncake-runtime-w00921547:latest
```

在输出里搜索（只做文本检索）：`spdk`、`nvme_urma`、`nof`、`urma`、`pip install`、`apt`。
若历史/配置里看不到具体文件 → 报 `BLOCKED: 需要导出/运行镜像才能确认`，**不要自行 export/run**。

#### C. SR-IOV / hugepages 只读检查

选 **worker02**（8 盘、7 数据盘、BDF 规律清晰），不动任何配置：

```bash
# 以 nvme0n1 为例；先确认 BDF
bdf=$(basename "$(readlink -f /sys/block/nvme0n1/device)")
echo "$bdf"

# PCIe SR-IOV capability（完整输出，重点 VF Device ID / NumVFs / VF Offset / Stride）
lspci -vv -s "$bdf" | grep -i -A25 'Single Root I/O Virtualization'

# NVMe 控制器/namespace 能力（完整输出；重点 nn / oncs / vwci / vcmic / mnan 等）
nvme id-ctrl /dev/nvme0n1
nvme id-ns /dev/nvme0n1 -n 1

# sysfs 现状（只读）
cat "/sys/bus/pci/devices/$bdf/sriov_totalvfs"
cat "/sys/bus/pci/devices/$bdf/sriov_numvfs"
cat "/sys/bus/pci/devices/$bdf/sriov_vf_device" 2>/dev/null || true
readlink "/sys/bus/pci/devices/$bdf/driver"

# hugepages / 内核参数
cat /proc/cmdline
grep -i huge /proc/meminfo
cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages 2>/dev/null || true

# 集群侧：是否有 SR-IOV device plugin / 扩展资源
kubectl get ds -A | grep -iE 'sriov|device-plugin|intel|mellanox' || true
kubectl get nodes -o json | jq -r '
  .items[] |
  [ .metadata.name,
    ( [.status.allocatable | to_entries[]
       | select(.key | test("sriov|intel.com|nvidia.com|unifiedbus|hugepages"))
       | (.key + "=" + .value)] | join(",") )
  ] | @tsv'
```

只读；不要写 `sriov_numvfs`、不要写 hugepages、不要 `modprobe/rmmod`。

#### D. 现有 DaemonSet 基线

`kubectl -n mooncake-rank get ds mooncake-store-0 -o yaml` 中记录：

- `resources.requests/limits`（CPU / 内存 / UB 设备）；
- 环境变量（尤其 `GLOBAL_SEGMENT_SIZE`、`MC_*`、`STORE_SLOT`）；
- `volumeMounts/volumes`（日志、脚本、设备）；
- `securityContext`、`imagePullPolicy`、`nodeSelector/tolerations`；
- 探针端口与超时。

目的：NoF 版要在此基础上加设备 / hugepages / 权限，先留基线。

#### E. 仓库侧事实（只读代码）

从本仓库确认（不要构建、不要进编译机改东西）：

- `services/k8s-0-mooncake-build-image/build-mooncake.sh` 的 `BIN_TARGETS` 是否包含
  `nof_register` / `nof_unregister`；
- `build-mooncake-runtime-image.sh` 的产物目录是否只含 `bin/lib/dist`；
- `check-mooncake-ldd.sh` 的 ELF 清单是否包含新二进制；
- `services/k8s-2-mooncake-deployment-rank/scripts/k8s-run_mooncake_store_realclient_ha.sh`
  当前的 client 启动参数与 `--enable_offload` 值。

如果测试 Agent 另有编译机访问权限，可额外只读报告 `docker ps` 里的编译容器是否存在；
**不要编译、不要进容器改文件。**

#### 回执要求

1. **CrashLoopBackOff 原因**：从日志摘 3–5 行关键错误 + exit code + 发生频率。
2. **镜像结论**：有没有 SPDK / NoF / URMA 的痕迹；没有就写 `BLOCKED` 及“需要什么授权才能确认”。
3. **SR-IOV 关键参数**：`sriov_totalvfs` / `sriov_numvfs`、VF Device ID、VF Offset/Stride；
   是否支持 per-VF namespace 隔离（看不出来就写“需厂商文档/实测”）。
4. **集群侧**：有没有 SR-IOV device plugin；各节点 hugepages allocatable 现状。
5. **现有 DS 基线**：resources / env / volumes / securityContext / imagePullPolicy。
6. **仓库侧**：当前编译脚本是否已含 NoF 目标（直接引用文件行号）。
7. **BLOCKED 项**清单；不要给方案决策。

---

## 7. 后续任务（占位，未下发）

> 不要在 NOF-002 中执行以下任务，等对应任务号下发。

- `NOF-003`：编译 NoF 版本（基于 `mooncake-vesegment`，`USE_NOF/USE_NOF_URMA`）——是否执行、在哪执行，由 Dev Agent 决定
- `NOF-004`：构建 runtime 镜像并校验 `nof_register` / SPDK 依赖
- `NOF-005`：镜像分发到实际节点
- `NOF-006`：部署 NoF rank / recall，验证向 master 注册
- `NOF-007`：背压与基线性能测试

---

## 8. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 初始版本；下发 NOF-001 环境摸底 |
| 2026-11-17 | **置顶最高优先级安全令：默认且优先只读，任何变更须逐项书面授权**；明确 Dev Agent / Test Agent 分工 |
| 2026-11-17 | NOF-001 完成（实测 10 节点）；下发 NOF-002 只读深挖；强化角色边界：Test Agent 只执行/取证/报事实，方案与决策归 Dev Agent |
