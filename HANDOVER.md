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

**Design note（2026-11-17，来自用户）**：数据盘 1/7/11 的不均分布是**临时状态**，
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

---

## 2. 安全红线

> **置顶安全令对所有任务永久有效。** 本节只是对当前任务的具体化，不构成对置顶令的放宽。

当前任务（NOF-003）是**只读任务**。允许：

- `kubectl get / describe / logs / version / cluster-info`
- `ctr -n k8s.io content ls / get`（`get` 只输出到 stdout，管道给 `tar` / `jq`，**不落盘**）
- `ssh <node>` 只读命令：`lsblk`、`nvme list`、`nvme id-ctrl`、`nvme id-ns`、`nvme list-ns`、
  `lspci`、`findmnt`、`cat /proc/meminfo`、`cat /proc/cmdline`、`numactl -H`、`lsmod`、
  `readlink`、`ls`、sysfs 读取
- `crictl images / inspecti`

禁止（本轮）：

- `kubectl exec`（本轮不授权；先用 content store 只读方案）
- `ctr image export / import / rm`；任何落盘的镜像导出/解包
- 创建 / 修改 / 删除任何 Kubernetes 资源（包括临时 DaemonSet、debug Pod）
- `kubectl apply / delete / edit / patch / scale`
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
| NOF-002 | 基线故障 + 镜像/ SR-IOV 只读深挖 | **DONE** | 2026-11-17 |
| NOF-003 | 镜像文件级只读查验 + 探针/虚拟化能力核对 | **已下发** | 2026-11-17 |

---

## 5. 已完成任务摘要

### NOF-001 环境摸底（DONE，2026-11-17）

- 集群：**10 节点**（master + 9 worker；worker01/04 不存在），k8s v1.19.9。
- 磁盘：每盘 **`nn=128`**；`sriov_totalvfs=32`（master 一块 7T 除外）；
  `sriov_numvfs=0`；驱动全为内核 `nvme`；hugepages 2M/512M/16G 全 0；
  宿主机无 SPDK、内核 `nvmet` 未加载。
- 通道：master → 10 节点 root 免密 + `sudo -n` 全通；
  `kubectl`、`ctr`、`crictl`、`nvme-cli`、`jq`、`urma_admin` 可用。
- 数据盘分布（**临时状态，不作为方案约束**）：worker02/06/07/08/09/10 各 7 块；
  worker05 11 块；worker03/11 各 1 块；master 7 块（含一块无 SR-IOV）。
- UB：每节点 `unifiedbus.com/ub_net_device=96`；`urma_admin show` 全部 ACTIVE。

### NOF-002 基线故障 + 镜像/SR-IOV 深挖（DONE，2026-11-17）

**A. CrashLoopBackOff（已自愈）**

- 直接原因：启动期 etcd ClusterIP `10.111.215.14:17779` TCP 不可达约 70s →
  startup probe 连续 7 次失败 → kubelet SIGKILL（exit 137）；**非 OOM**。
- 时间线：20:56:41 容器启动 → 20:57:52 startup probe 判死 → 20:57:53 etcd 恢复 →
  20:57:54 监听 17704；本次存活约 101s。同镜像在 worker02/03 上 1.1s 即监听。
- 现状：rank DS 已 **10/10 READY**；该 Pod 自 20:58:22 起未再重启。

**B. 镜像（元数据层面）**

- `mooncake-runtime-w00921547:latest`，imagePullPolicy `Never`；
  `crictl inspecti` id `sha256:7ca496f3d215...`，size 5,834,332,609 B，arm64，Entrypoint `/bin/sh`。
- history 11 层：KIWI openEuler 基础层 + 5× `while true; do sleep 1000; done` + 5× `/bin/bash` +
  末层 `sleep 3600`；**history 内无 apt/pip/编译命令文本**。
- 关键词 `spdk` / `nvme_urma` / `nof_register` / `nof` / `urma` 在元数据中 **0 命中**。
- **文件级未确认**（本轮 BLOCKED）：需要 content store 只读查 layer（NOF-003 方案）。

**C. SR-IOV / NVMe 能力（worker02，nvme0n1 / BDF 0000:a1:00.0）**

- `Initial VFs: 32, Total VFs: 32, NumVFs: 0`；`VF offset: 128, stride: 1`；
  `VF Device ID: 3755`；PF 与 VF 同 DID；`sriov_drivers_autoprobe=1`；driver `nvme`。
- `nvme id-ctrl`：`nn=128`、`oncs=0x56`、`ctratt=0x281`、`oaes=0x2100`、
  **`vwci=0`**、`nmic=0x1`、`nsfeat=0x10`、`nlbaf=4 / flbas=0x10`（lbaf0 512B）。
- **per-VF namespace 隔离能力：无法从只读信息判定**；`vwci=0` 表示不支持标准
  Virtualization Management Command，需厂商文档或使能 VF 后实测。
- 集群无 SR-IOV device plugin；只有 `ub-network-device-plugin`。
  10 节点 allocatable 只有 `unifiedbus.com/ub_net_device=96`；无 `sriov*` / `intel.com/*` / `nvidia.com/gpu`。
- hugepages 2Mi/512Mi/16Gi 的 capacity / allocatable 全为 0。
- `/dev/vfio` 只有控制节点，无 IOMMU group 设备节点；cmdline 含 `iommu=pt`。

**D. 现有 DaemonSet 基线（mooncake-rank/mooncake-store-0）**

- image `latest` / `Never`；command `k8s-run_mooncake_store_realclient_ha.sh`；
- resources（requests=limits）：cpu 8、memory 16Gi、`unifiedbus.com/ub_net_device` 1；
- securityContext：`privileged: false`，capabilities `SYS_ADMIN`；
- env：POD_NAME/POD_IP/NODE_NAME、`MOONCAKE_NAMESPACE=mooncake-rank`、`STORE_SLOT=0`；
- volumes：host-ping、mooncake-logs、startup-scripts(configMap)；
- 探针：startup tcpSocket 17704；**仓库 YAML 写 fT=30 / p=10s / t=3s，
  但实际日志 7 次失败即判死，live DS 的 fT 可能不是 30**（NOF-003 核对）。

**E. 仓库侧**

- `build-mooncake.sh`：`BIN_TARGETS` 无 `nof_register`/`nof_unregister`；
  cmake 无 `USE_NOF` / `USE_NOF_URMA`；build target 无 nof；
  `stress_clutser_bench` 拼写与 `stress_cluster_bench` 不一致。
- `build-mooncake-runtime-image.sh`：产物只涉及 `bin/lib/dist`，无 spdk/urma 目录。
- `check-mooncake-ldd.sh`：ELF 清单无 nof 二进制。
- 全仓 grep `nof_register`：无。
- store client：`--enable_offload=false`。
- master 上无编译容器（本机不是编译机）。

**F. 环境变更（待用户确认）**

- 检查期间新增：`mooncake-recall`（etcd + master + store-0/store-1 两套 DS，各 10 ready）
  与 `mooncake-memory-pressure`（numa0–3 四套 DS，各 10 ready）；rank DS 由 9/10 → 10/10；
  Pod 总数 34 → 176。**是否为新基线，待用户确认；NOF-003 只记录，不停任何东西。**

**Dev Agent 结论**：CrashLoop 是 etcd 启动窗口的瞬态问题，非 NoF 阻塞；镜像几乎肯定不含 NoF，
需要重新编译；SR-IOV 硬件有能力但 `vwci=0` 使 per-VF 隔离存疑，且无 device plugin、hugepages=0；
下一步先做镜像文件级只读查验，再定 k8s-4 / k8s-6 方案。

---

## 6. 当前任务

### NOF-003 镜像文件级只读查验 + 探针/虚拟化能力核对

**上一轮结论**：CrashLoop 是 etcd 启动窗口瞬态（已自愈）；镜像元数据无 NoF/SPDK 痕迹；
SR-IOV 32 VF 但 `vwci=0`、无 device plugin、hugepages=0；环境新增 recall + memory-pressure。

**本轮目标**：不运行/不导出镜像、不 `exec`，用 containerd content store 只读确认镜像里到底有没有
`nof_register` / SPDK / URMA；核对 live DS 探针；补齐 NVMe 虚拟化字段；记录新基线。

**授权边界**：只读。允许 `ctr -n k8s.io content ls/get`（`get` 只输出到 stdout 并管道给 `tar`/`jq`，
**不落盘**）、`kubectl get/describe/logs`、`ssh` 只读命令、`nvme id-ctrl`。
禁止：`kubectl exec`、`ctr image export/import/rm`、创建/删除资源、写 `sriov_numvfs`、
改 hugepages、绑 vfio、`modprobe/rmmod`、重启任何东西。

#### A. 镜像文件级查验（不落盘）

目标：确认镜像内是否存在：

- `/home/Mooncake/outputs/bin/nof_register`
- `/home/Mooncake/outputs/bin/nof_unregister`
- `/home/Mooncake/outputs/bin/mooncake_client`
- `/usr/local/include/spdk/nvme_urma.h`
- `/usr/local/lib/libspdk_nvme.a`（或 `libspdk_*`）

方法（按顺序尝试；任何一步不支持就记录并换下一种）：

```bash
# 1) 列出 content store（找 manifest / config / layers）
ctr -n k8s.io content ls

# 2) 从 crictl inspecti 拿到 image 的 config digest / layer digest
crictl inspecti docker.io/library/mooncake-runtime-w00921547:latest

# 3) 读 image config（不落盘）
ctr -n k8s.io content get <CONFIG_DIGEST> | jq '{architecture, os, config, history}'

# 4) 读最上层 layer 的 tar 列表（不落盘；按实际压缩格式调整）
ctr -n k8s.io content get <LAYER_DIGEST> | tar -tzf - \
  | grep -E 'nof|spdk|nvme_urma|outputs/bin|mooncake_client|check-mooncake' || true
```

注意：

- 如果 `tar -tzf` 报格式错误：用 `ctr ... | file -` 判断是 gzip / zstd / 未压缩；
  zstd 用 `tar --zstd -tf -`，未压缩用 `tar -tf -`。
- 只列文件，不 `tar -x` 解包到磁盘。
- 如果 content store 不支持、权限不足或太慢 → 报 `BLOCKED: 需要授权其他只读方式`，
  **不要**自行 `ctr image export` 或 `kubectl exec`。

#### B. Live DS 探针核对

```bash
kubectl -n mooncake-rank get ds mooncake-store-0 \
  -o jsonpath='{.spec.template.spec.containers[0].startupProbe}{"\n"}'
kubectl -n mooncake-rank get ds mooncake-store-0 \
  -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}{"\n"}'
kubectl -n mooncake-rank get ds mooncake-store-0 \
  -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}{"\n"}'
```

只读；回答：live 的 `failureThreshold` 是否与仓库 YAML 的 30 不一致？
若不一致，贴出实际值（这解释为何 7 次失败就重启）。

#### C. NVMe 虚拟化能力补齐

```bash
nvme id-ctrl /dev/nvme0n1 | grep -E '^(nn|mnan|vwci|vcmic|nmis|ctratt|oaes|oncs|nmic|nsfeat)'
```

只读；确认 `vwci` / `vcmic` / `mnan` 等字段。**不要**尝试创建 namespace 或使能 VF。

#### D. 新基线记录（不停任何东西）

```bash
kubectl get ns
kubectl get ds,deploy,sts -A -o wide | grep -i mooncake || true
kubectl get pods -A -o wide | grep -i mooncake | wc -l
kubectl -n mooncake-recall get ds,deploy,sts -o wide 2>/dev/null || true
kubectl -n mooncake-memory-pressure get ds -o wide 2>/dev/null || true
```

记录 `mooncake-recall` / `mooncake-memory-pressure` 的 `creationTimestamp`、labels、
是否带 `mooncake.openai.com/managed-by` 之类的标记；**不要停止/修改**。

#### 回执要求

1. 镜像文件级结论：`nof_register` / `nof_unregister` / `mooncake_client` / SPDK 头/lib
   各自 **有 / 无 / 无法确认**（附 layer 路径证据）。
2. live DS 三个探针的实际 `failureThreshold / periodSeconds / timeoutSeconds`，与仓库 YAML 对比。
3. `vwci` / `vcmic` / `mnan` 实际值；`vwci=0` 是否意味着不支持标准 VMC。
4. 新基线：recall / memory-pressure 的创建时间、DS 数、Pod 数；是否像 Services-Deployment 创建的。
5. BLOCKED 项；不要给方案决策。

---

## 7. 后续任务（占位，未下发）

> 不要在 NOF-003 中执行以下任务，等对应任务号下发。

- `NOF-004`：编译 NoF 版本（基于 `mooncake-vesegment`，`USE_NOF/USE_NOF_URMA`）——执行位置与方式由 Dev Agent 决定
- `NOF-005`：构建 runtime 镜像并校验 `nof_register` / SPDK 依赖
- `NOF-006`：镜像分发到实际节点
- `NOF-007`：部署 NoF rank / recall，验证向 master 注册
- `NOF-008`：背压与基线性能测试

---

## 8. 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 初始版本；下发 NOF-001 环境摸底 |
| 2026-11-17 | **置顶最高优先级安全令：默认且优先只读，任何变更须逐项书面授权**；明确 Dev Agent / Test Agent 分工 |
| 2026-11-17 | NOF-001 完成（实测 10 节点）；下发 NOF-002 只读深挖；强化角色边界 |
| 2026-11-17 | NOF-002 完成（CrashLoop 瞬态自愈；镜像元数据无 NoF；SR-IOV `vwci=0`；环境新增 recall+pressure）；下发 NOF-003 镜像文件级只读查验；数据盘不均视为临时状态，不作为方案约束 |
