# 离线安装

## 前提条件
* NVIDIA 驱动版本 >= 440
* nvidia-docker 版本 > 2.0
* 默认运行时配置为 NVIDIA 运行时
* Kubernetes 版本 >= 1.18
* glibc >= 2.17 & glibc < 2.30
* kernel 版本 >= 3.10
* helm 版本 > 3.0

## 准备 GPU 节点

HAMi 安装前需先部署 gpu-operator 核心组件（无需安装 device-plugin），确保 GPU 节点能正常识别显卡资源。

### 部署 gpu-operator（排除 device-plugin）

```yaml
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator \
    --set devicePlugin.enabled=false \
    --set dcgmExporter.serviceMonitor.enabled=true
```

> helm 安装可以通过 `--set devicePlugin.enabled=false` 参数来禁用 device-plugin。

### 验证 gpu-operator 部署状态

```yaml
# 检查 gpu-operator 核心组件运行状态
kubectl get pods -n gpu-operator
# 预期输出：gpu-operator 相关 pod 均为 Running 状态（device-plugin 无相关 pod）
```

## 获取 HAMi 商业版离线包

HAMi 商业版仅支持离线安装，需从相关同事处获取完整离线包，包含以下两部分：

1. 镜像包：hami_commercial_image.tar（HAMi 商业版核心镜像）、certgen.tar（证书生成工具镜像）
2. 安装包：hami.tgz（HAMi Helm Chart 包）

## 安装

1. 将 HAMi 镜像导入集群所有节点（或镜像仓库，确保集群可拉取），执行以下命令：

```bash
ctr -n k8s.io i import hami_commercial_image.tar \
ctr -n k8s.io i import certgen.tar
```

2. 使用 helm 进行安装，安装完以后确保所有的组件能正常运行。

```bash
helm install hami ./hami.tgz -n hami-system --create-namespace --debug
```

3. 运行一下命令检查是否 HAMi 是否安装成功，HAMi 一共有两个组件：hami-scheduler 和 device-plugin.

```bash
kubectl get po -n hami-system
```
