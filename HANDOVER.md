# HANDOVER — Mooncake NoF 改造测试任务书

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
| 环境 | 12 节点 Kubernetes 集群，1 个 master；你已连上 master |
| 结果通道 | 本轮会话（不回写仓库） |

**公开仓库提醒**：本仓库是公开 fork。**禁止**把内网 IP、密码、token、kubeconfig 内容写进任何仓库文件。
节点 IP 等信息可以出现在会话回执里，但不要提交。

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
下一轮建议：
  - 可选
```

规则：

- 先给结论，再给证据。
- 只贴关键行；表格优先。
- 遇到需要变更集群、删除资源、重启节点的操作，立即报 `NEED-CONFIRM`，不要自行执行。
- 任何一步失败，不要跳过；记录 `BLOCKED` 并继续能做的只读检查。

---

## 2. 安全红线

**NOF-001 是只读任务。** 允许：

- `kubectl get / describe / version / cluster-info`
- `ssh <node> <只读命令>`：`lsblk`、`nvme list`、`nvme id-ctrl`、`nvme list-ns`、`lspci`、
  `findmnt`、`lsblk`、`cat /proc/meminfo`、`numactl -H`、`lsmod`、`readlink`、`ls`
- `ssh` 连通性测试（`true`，不执行任何变更）

禁止（本轮）：

- 创建 / 修改 / 删除任何 Kubernetes 资源（包括临时 DaemonSet、debug Pod）
- `kubectl apply / delete / edit / patch / scale`
- 导入 / 删除镜像（`ctr image import / rm`）
- 绑定 / 解绑 vfio 驱动、`setup.sh`、SPDK 启停
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
  - 在本轮口径中，「一个 NoF target = 一个 Pod」是待验证的部署假设，不是既定事实。

---

## 4. 任务索引

| 编号 | 标题 | 状态 | 下发日期 |
|---|---|---|---|
| NOF-001 | 环境摸底（只读） | 已下发 | 2026-11-17 |

---

## 5. 当前任务

### NOF-001 环境摸底（只读）

**上一轮结论**：无（首轮）。

**本轮目标**：用只读命令摸清 12 节点集群的底子，回答三个问题：

1. **现有 Mooncake 部署形态**：rank / recall 是否在跑？用的什么镜像？有几个 store Pod？
2. **每节点 SSD 情况**：几块 NVMe？系统盘是哪块？数据盘每块有几个 namespace？
   是否支持 SR-IOV？当前驱动是什么（内核 / vfio）？
3. **运维通道**：从 master 到 12 个节点，SSH 免密、`sudo -n`、`kubectl`、`ctr` 是否可用？

这三项直接决定后续「一块 SSD 能不能拆给多个 NoF target Pod」以及镜像分发/部署能否自动化。

#### 执行步骤

**A. Kubernetes 概览（在 master 上）**

```bash
kubectl version --short 2>/dev/null || kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns

# Mooncake / 压力测试相关资源
kubectl get pods -A -o wide | grep -iE 'mooncake|memory-pressure|etcd' || true
kubectl get ds,deploy,sts -A | grep -iE 'mooncake|memory-pressure' || true

# 每节点可分配资源：UB 设备、GPU、CPU、内存
kubectl get nodes -o json | jq -r '
  .items[] |
  [ .metadata.name,
    (.status.allocatable["unifiedbus.com/ub_net_device"] // "-"),
    (.status.allocatable["nvidia.com/gpu"] // "-"),
    (.status.allocatable.cpu // "-"),
    (.status.allocatable.memory // "-")
  ] | @tsv'
```

> 若 `jq` 不存在，记录 `BLOCKED: jq missing`，改用
> `kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory`
> 并说明无法直接读取 UB/GPU 扩展资源。

**B. master → 节点连通性（只做测试，不执行变更）**

```bash
# 取节点 IP（InternalIP）
kubectl get nodes -o wide --no-headers | awk '{print $1, $6}'

# 对每个节点依次尝试：root 免密、当前用户、sudo -n
ssh -o BatchMode=yes -o ConnectTimeout=5 root@<NODE_IP> true
ssh -o BatchMode=yes -o ConnectTimeout=5 <CURRENT_USER>@<NODE_IP> true
ssh -o BatchMode=yes -o ConnectTimeout=5 <USER>@<NODE_IP> 'sudo -n true'
```

记录每个节点的结果：`root-ok / user-ok / sudo-ok / FAIL`。**不要尝试密码登录。**

**C. 单节点磁盘详查（先选 1 个有代表性的节点）**

优先选一个满足「有 UB 设备 + 有多块 NVMe」的节点；不要动任何盘。

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
findmnt -rn -o TARGET,SOURCE,FSTYPE
nvme list 2>/dev/null || echo "nvme-cli missing"

# 对每块 NVMe 盘（示例：/dev/nvme0n1）
for dev in /dev/nvme*n1; do
  echo "===== $dev ====="
  nvme id-ctrl "$dev" 2>/dev/null | grep -E '^nn|^mn|^sn' || true
  nvme list-ns "$dev" 2>/dev/null || true
  bdf=$(basename "$(readlink -f "/sys/block/$(basename "$dev")/device")")
  echo "BDF=$bdf"
  readlink "/sys/bus/pci/devices/$bdf/driver" || echo "driver: none"
  cat "/sys/bus/pci/devices/$bdf/sriov_totalvfs" 2>/dev/null || echo "sriov_totalvfs: none"
  lspci -vv -s "$bdf" 2>/dev/null | grep -i -A3 'SR-IOV' || true
done

# NUMA / 内存 / hugepages
lscpu | grep -iE 'numa|model name'
numactl -H 2>/dev/null || true
grep -i huge /proc/meminfo
cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages 2>/dev/null || true
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages 2>/dev/null || true

# vfio 现状
ls -l /dev/vfio 2>/dev/null || echo "no /dev/vfio"

# UB / URMA
command -v urma_admin && urma_admin show || echo "urma_admin missing"
lsmod | grep -iE 'urma|udma' || true

# SPDK（是否已安装 NoF/URMA 版本）
test -f /usr/local/include/spdk/nvme_urma.h && echo "spdk-urma-header: OK" || echo "spdk-urma-header: MISSING"
ls -l /usr/local/lib/libspdk_nvme.a 2>/dev/null || echo "libspdk_nvme.a: MISSING"
```

**D. 12 节点精简汇总（若 B 的 SSH 可用）**

对全部 12 节点执行精简版：

```bash
lsblk -dno NAME,SIZE,TYPE,MODEL | grep -i nvme
nvme list 2>/dev/null | grep -i nvme || true
grep -i huge /proc/meminfo
cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages 2>/dev/null || true
```

不要执行任何写操作。

**E. 现有镜像与部署**

```bash
# containerd 镜像（二选一，按节点实际工具）
ctr -n k8s.io image ls 2>/dev/null | grep -iE 'mooncake|memory-pressure|pause' || true
crictl images 2>/dev/null | grep -iE 'mooncake|memory-pressure|pause' || true

# 运行中的 Mooncake Pod（只看状态，不拉日志）
kubectl get pods -A -o wide | grep -i mooncake || true
```

#### 回执要求

按第 1 节格式输出，并额外包含：

1. **三个问题的结论**（各一句话）：
   - 现有部署：是否已有 rank/recall？几个 store Pod？镜像 tag？
   - 磁盘：每节点几块 NVMe？数据盘几块？每盘 `nn`（namespace 数）是多少？SR-IOV 有/无？
   - 通道：SSH / sudo / kubectl / ctr 各是否可用？
2. **12 节点汇总表**：

| 节点 | 角色 | SSH(root/user/sudo) | NVMe总数 | 数据盘数 | 每盘 ns 数 | SR-IOV | 当前驱动 | hugepages | UB设备 |
|---|---|---|---|---|---|---|---|---|---|

3. **明确结论**：
   - 若所有数据盘 `nn=1` 且无 SR-IOV → 直接给结论：
     「一块 SSD 无法在硬件层拆给多个 Pod；只能整盘分配（2/2/3）或采用逻辑切片（1 个 target + 多个 initiator 注册区间）。」
   - 若有盘 `nn>1` 或 SR-IOV 可用 → 列出盘符、BDF、可选 VF 数。
4. **BLOCKED 项**清单（如无 SSH、无 sudo、无 nvme-cli、无 jq 等）。
5. 不要贴完整 IP 列表到仓库；回执里可以写。

---

## 6. 后续任务（占位，未下发）

> 不要在 NOF-001 中执行以下任务，等对应任务号下发。

- `NOF-002`：编译 NoF 版本（基于 `mooncake-vesegment`，`USE_NOF/URMA`）
- `NOF-003`：构建 runtime 镜像并校验 `nof_register` / SPDK 依赖
- `NOF-004`：镜像分发到 12 节点
- `NOF-005`：部署 NoF rank / recall，验证向 master 注册
- `NOF-006`：背压与基线性能测试

---

## 7. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 初始版本；下发 NOF-001 环境摸底 |
