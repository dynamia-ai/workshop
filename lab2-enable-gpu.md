HAMi 管理的节点是需要手动开启，在 GPU 节点上注入 label 来开启 HAMi 的功能：

```yaml
kubectl label nodes {{__NODE_NAME__}} gpu="on"
```

在开启 GPU 的节点请检查该节点的 device-plugin 是否正常启动：

```bash
~ kubectl get po -n kube-system | grep hami-device-plugin
hami-device-plugin-gnstr                                          2/2     Running                 0                 4s
hami-device-plugin-rllff                                          2/2     Running                 0                 5s
hami-device-plugin-w27lk                                          2/2     Running                 0                 5s
```

device-plugin 所有的 pod 都 running 以后，我们查看卡的信息有没有被注册到节点：

```yaml
~ ❯ kubectl get node {{__NODE_NAME__}} -oyaml
kind: Node
metadata:
  annotations:
    ....
    hami.io/node-handshake: Requesting_2025-09-08 02:41:04
    hami.io/node-nvidia-register: 'GPU-8b644fb4-b7e8-56df-3fda-912e3a5c643c,10,23034,100,NVIDIA-NVIDIA
      L4,0,true,0:'
    ....
```

hami.io/node-nvidia-register 表示设备的信息，它的格式：
```
{device UUID},{device split count},{device memory limit},{device core limit},{device type},{device numa},{healthy},{index}
```

* device UUID:  设备的唯一标识
* device split count：设备切分的数量，10 表示设备被虚拟化 10 个设备
* device memory limit：设备的显存上限，单位为 M
* device core limit：设备的算力上线，单位为 %
* device type：设备的类型
* device numa：设备是否开启 numa
* healthy：设备的健康状态
* index：设备的序列号

hami.io/node-handshake：节点锁时间
