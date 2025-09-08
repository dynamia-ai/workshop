HAMi 商业版安装完后，必须要导入 License 后才能激活功能。

# 获取 License 信息

* ESN: 当前集群的唯一标识，kube-system 命名空间的 UID
* 型号获取：设备型号可以从节点的 annotation 上获取，例如（NVIDIA-Tesla P4）：

```yaml
hami.io/node-nvidia-register: 'GPU-61951b89-82be-83fe-3068-2fea68026576,10,7680,100,NVIDIA-Tesla P4,0,true,0,hami-core:GPU-c080157b-3ab1-179d-d6dd-483adab68726,10,8192,100,NVIDIA-Tesla P4,0,true,1,hami-core:'
```

提供以上信息，生成 License 后，需要导入到集群，目前 License 是以 secret 的形式存在的下面是 License 的 secret 形式：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{__LICENSE_SECRET_NAME__}}
  namespace: {{__HAMI_NAMESPACE__}}
  labels:
    hami.io/license: "true"
data:
  license: {{__LICENSE__}}
```

* secret 的命名空间必须要跟 HAMi 保持一致
* - 必须添加 label：hami.io/license: "true"

> secret 在导入时需要将 license 进行 base64 编码

# License 激活

在导入 License 后，HAMi 会自动识别 License 并对比 esn，如果当前集群的 esn 和 License 申请时所填的一致，License 会自动激活。在 license 导入后，会自动添加 label。可以查看节点的详情：

[](./image/license-active.png)
