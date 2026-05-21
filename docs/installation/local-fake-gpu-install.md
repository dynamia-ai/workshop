# 实验 1A: mac 本地 Fake GPU 安装 HAMi

本实验将在 macOS 上使用 Docker Desktop、kind 和 [run-ai/fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator) 搭建一个纯本地 Kubernetes 集群，然后在线安装 HAMi。

这个实验不需要真实 NVIDIA GPU，适合用于课堂预习、讲解 HAMi 组件组成、验证 GPU Pod 调度流程，以及在个人电脑上快速熟悉 HAMi 的基础使用方式。

## 你将得到什么

完成本实验后，你会得到一个本地 Kubernetes 集群：

- fake-gpu-operator 在 CPU 节点上模拟 `nvidia.com/gpu` 资源
- HAMi scheduler、admission webhook 等控制面组件正常运行
- 普通 Pod 可以通过 `nvidia.com/gpu` 申请模拟 GPU
- 可以观察 fake GPU 资源从节点发现、Pod 申请、调度到运行的完整链路

> 注意：fake GPU 不能代表真实 GPU 的显存隔离、算力隔离、CUDA 运行时和驱动能力。本实验用于理解 HAMi 组成和基础调度链路；涉及真实显存切分、`nvidia.com/gpumem`、`nvidia.com/gpucores`、CUDA 程序运行和性能隔离时，仍需要真实 NVIDIA GPU 环境。

## 安装全景图

整个本地安装过程分 5 步：

```mermaid
flowchart LR
    Step1["步骤1<br/>安装本地工具"] --> Step2["步骤2<br/>创建 kind 集群"]
    Step2 --> Step3["步骤3<br/>安装 fake-gpu-operator"]
    Step3 --> Step4["步骤4<br/>安装 HAMi"]
    Step4 --> Step5["步骤5<br/>运行模拟 GPU 工作负载"]
```

| 步骤 | 目的 | 解决什么问题 |
| ------ | ------ | ------------- |
| 安装本地工具 | 准备 Docker、kubectl、Helm、kind | 在 mac 上创建和管理本地 Kubernetes |
| 创建 kind 集群 | 启动一个单节点 Kubernetes | 给 HAMi 和 fake-gpu-operator 提供运行环境 |
| 安装 fake-gpu-operator | 模拟 NVIDIA GPU 资源 | 让无 GPU 节点也能上报 `nvidia.com/gpu` |
| 安装 HAMi | 部署 HAMi 控制面 | 观察 HAMi scheduler、webhook 等组件 |
| 运行模拟 GPU 工作负载 | 验证调度链路 | 体验 Pod 申请 GPU 后被调度运行 |

## 前提条件

- macOS，Intel 或 Apple Silicon 均可
- 已安装 Docker Desktop，并确保 Docker Desktop 正在运行
- 能访问 GitHub、GHCR、Docker Hub 和 HAMi Helm 仓库
- 本机至少 4 CPU、8 GB 内存可用于实验

安装命令行工具：

```bash
brew install kubectl helm kind
```

验证工具：

```bash
docker version
kubectl version --client
helm version
kind version
```

## 步骤 1: 创建 kind 集群

### 目的

kind 使用 Docker 容器模拟 Kubernetes 节点。macOS 本身不能直接运行 Linux kubelet，kind 会在 Docker Desktop 中启动一个 Linux 节点容器。

### 操作

创建集群配置文件：

```bash
cat > kind-hami-fake-gpu.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
EOF
```

创建集群：

```bash
kind create cluster --name hami-fake-gpu --config kind-hami-fake-gpu.yaml
```

验证节点状态：

```bash
kubectl get nodes -o wide
```

预期输出中 `STATUS` 为 `Ready`：

```plaintext
NAME                          STATUS   ROLES           AGE   VERSION
hami-fake-gpu-control-plane   Ready    control-plane   1m    v1.31.x
```

## 步骤 2: 安装 fake-gpu-operator

### 目的

fake-gpu-operator 会在没有 NVIDIA GPU 的节点上模拟 GPU 资源，并把节点容量写成 `nvidia.com/gpu`。这一步替代了真实 GPU 环境中的 NVIDIA GPU Operator、驱动、device-plugin 和 DCGM 指标采集链路。

### 操作

创建命名空间并开启 privileged Pod Security：

```bash
kubectl create namespace gpu-operator
kubectl label namespace gpu-operator pod-security.kubernetes.io/enforce=privileged
```

给 kind 节点打 fake GPU 节点池标签：

```bash
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node ${NODE_NAME} run.ai/simulated-gpu-node-pool=default
```

安装 fake-gpu-operator：

```bash
export FAKE_GPU_OPERATOR_VERSION=0.0.80

helm upgrade -i gpu-operator \
    oci://ghcr.io/run-ai/fake-gpu-operator/fake-gpu-operator \
    --namespace gpu-operator \
    --create-namespace \
    --version ${FAKE_GPU_OPERATOR_VERSION}
```

> `0.0.80` 是 fake-gpu-operator 在 2026-04-12 发布的稳定版本。后续实验时可以从 fake-gpu-operator 的 GitHub Releases 页面选择更新版本。官方 README 使用 OCI Helm Chart：`oci://ghcr.io/run-ai/fake-gpu-operator/fake-gpu-operator`。

等待组件运行：

```bash
kubectl get pods -n gpu-operator
```

查看节点是否出现模拟 GPU：

```bash
kubectl get node ${NODE_NAME} \
    -o custom-columns=NAME:.metadata.name,GPU:.status.capacity.nvidia\\.com/gpu
```

预期输出：

```plaintext
NAME                          GPU
hami-fake-gpu-control-plane   2
```

如果 `GPU` 为空，检查节点标签是否正确：

```bash
kubectl get node ${NODE_NAME} --show-labels | grep run.ai/simulated-gpu-node-pool
```

## 步骤 3: 安装 HAMi

### 目的

本步骤安装 HAMi 的控制面组件，用于观察 HAMi 在 Kubernetes 中的组成：

- `hami-scheduler`：调度增强组件
- admission webhook：自动改写 GPU Pod 的调度器配置
- Helm release：统一管理 HAMi 相关 Kubernetes 资源

在 fake GPU 环境中，GPU 资源由 fake-gpu-operator 提供。为了避免两个 device-plugin 同时注册 `nvidia.com/gpu`，本实验不让 HAMi device-plugin 接管 fake 节点。

### 操作

添加 HAMi Helm 仓库：

```bash
helm repo add hami-charts https://project-hami.github.io/HAMi/
helm repo update
```

安装 HAMi：

```bash
helm install hami hami-charts/hami \
    -n kube-system \
    --set devicePlugin.enabled=false
```

验证 HAMi 组件：

```bash
kubectl get pods -n kube-system | grep hami
```

预期至少看到 `hami-scheduler` 处于 `Running`：

```plaintext
hami-scheduler-xxxxxxxxxx-xxxxx   1/1   Running   0   1m
```

查看 Helm Release：

```bash
helm list -A
```

预期看到：

```plaintext
NAME           NAMESPACE      STATUS
gpu-operator   gpu-operator   deployed
hami           kube-system    deployed
```

## 步骤 4: 运行模拟 GPU 工作负载

### 目的

验证 Kubernetes 可以把申请 `nvidia.com/gpu` 的 Pod 调度到 fake GPU 节点。fake-gpu-operator 会为 GPU Pod 注入模拟 `nvidia-smi` 工具，便于观察 GPU 可见性。

由于本实验没有启用 HAMi device-plugin，HAMi 不会写入真实环境中的 `hami.io/node-nvidia-register` 节点注册信息。因此测试 Pod 会显式绕过 HAMi webhook，使用 Kubernetes 默认调度器和 fake-gpu-operator 提供的模拟 GPU 资源。

### 操作

创建测试 Pod：

```bash
cat > fake-gpu-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: fake-gpu-pod
  labels:
    hami.io/webhook: ignore
  annotations:
    run.ai/simulated-gpu-utilization: "10-30"
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: ubuntu:22.04
      command: ["bash", "-lc", "sleep 3600"]
      resources:
        limits:
          nvidia.com/gpu: 1
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
EOF

kubectl apply -f fake-gpu-pod.yaml
```

等待 Pod 运行：

```bash
kubectl get pod fake-gpu-pod -o wide
```

预期输出中 `STATUS` 为 `Running`，`NODE` 为 kind 节点：

```plaintext
NAME           READY   STATUS    NODE
fake-gpu-pod   1/1     Running   hami-fake-gpu-control-plane
```

查看 Pod 的 GPU 资源申请：

```bash
kubectl describe pod fake-gpu-pod | grep -A6 Limits
```

预期看到：

```plaintext
Limits:
  nvidia.com/gpu:  1
```

执行模拟 `nvidia-smi`：

```bash
kubectl exec -it fake-gpu-pod -- nvidia-smi
```

如果输出中能看到模拟 GPU 信息，说明 fake-gpu-operator 注入成功。

## 步骤 5: 观察 HAMi 和 fake GPU 的边界

### HAMi 在本实验中负责什么

执行：

```bash
kubectl get deploy,svc,cm,sa -n kube-system | grep hami
```

你会看到 HAMi 的控制面资源。它们说明 HAMi 已经作为 Kubernetes 调度增强组件安装进集群。

### fake-gpu-operator 在本实验中负责什么

执行：

```bash
kubectl get daemonset,deploy,pod -n gpu-operator
kubectl describe node ${NODE_NAME} | grep -A5 "Capacity"
```

你会看到 fake-gpu-operator 负责模拟设备发现和 `nvidia.com/gpu` 节点容量。

### 这个实验不能验证什么

以下能力需要真实 NVIDIA GPU 环境：

- HAMi device-plugin 真实注册 GPU 并写入 `hami.io/node-nvidia-register`
- `nvidia.com/gpumem` 显存切分
- `nvidia.com/gpucores` 算力比例限制
- CUDA 程序真实运行
- 显存超配、显存分析、显存覆盖
- DCGM 真实 GPU 指标

如果要继续完整学习这些能力，请使用 [实验 1: 在线安装 HAMi](online-install.md) 的真实 GPU 环境。

## 清理环境

删除测试 Pod：

```bash
kubectl delete pod fake-gpu-pod
```

删除本地集群：

```bash
kind delete cluster --name hami-fake-gpu
```

## 下一步

完成本实验后，建议继续阅读 [HAMi 集群架构](../concepts/hami-architecture.md)，重点理解 scheduler、device-plugin、webhook 和 GPU Operator 的职责边界。
