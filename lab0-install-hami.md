# 离线安装

HAMi 安装前需要提前安装 gpu-operator 相关组件，并确保 gpu-operator 相关组件能正常工作，
> 注意：gpu-operator 的 device-plugin 是不需要安装的。helm 安装可以通过 `--set devicePlugin.enabled=false` 参数来设置。

1. HAMi 商业版离线版解压后，先将 HAMi 的镜像导入到集群：

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
