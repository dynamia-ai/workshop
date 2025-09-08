考核目标：

1. 安装 HAMi 后开启 GPU 的节点，并成功导入 license，保证 HAMi 的功能正常。

2. 创建一个只调度到 UUID 为（GPU-8b644fb4-b7e8-56df-3fda-562j3c5a694b）卡的 pod，名为 exam-02，分配 100M 的显存，10% 的算力和 1 个 vgpu。

3. 创建一个不调度到 UUID（GPU-8b644fb4-b7e8-56df-3fda-912e3a5c643c）卡的 pod，名为 exam-03，分配 100M 的显存，10% 的算力和 1 个 vgpu。

4. 创建一个指定卡类型为（NVIDIA-Tesla P4）的 pod，名为 exam-04，分配 100M 的显存，10% 的算力和 1 个 vgpu。

5. 创建一个节点调度策略为 binpack 且不调度到 UUID 为 GPU-8b644fb4-b7e8-56df-3fda-912e3a5c643c 的卡上，名为 exam-05 的 deployment，副本为 3，并分配 100M 的显存，1 个 vgpu。

6. 创建一个卡级别调度策略为 spread 且不调度到卡类型为 `NVIDIA-NVIDIA L4` 名为 exam-06 的 deployment，副本为 3，并分配 100M 的显存，1 个 vgpu。

7. 创建一个显存为 200M， 算力为 10%， 1 个 vgpu 名为 exam-07 的 pod，它的调度优先级为 0 且节点调度策略为 binpack。

8. 在 default 命令空间创建一个限制显存为 5G，算力大小为 200，gpu 数量为 20 的 resourceQuota。

9. 创建一个 pod 名为 exam-08 使用 vllm 运行模型 Qwen/Qwen3-0.6B，并分配 4 个 G 的显存，1 个 vgpu。

10. 将所有节点的显存超配设置为 1.8，将显卡切分数量设置为 12。

11. 将调度器默认节点调度策略设置为 `spread`。
