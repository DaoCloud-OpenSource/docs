# [1432-volume-health-monitor](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/1432-volume-health-monitor)

[TOC]

添加对volume状态健康监测的monitor以便在问题发生时(无法正常读写volume)及时根据监控的报错进行问题定位和修复


### 相关核心组件

- External Controller
  - 以sidecar的形式和csi driver一并部署
  - 对driver发起相关的rpc调用获取挂载卷的健康信息
  - 支持可配置参数开启controller对节点故障的监测
- Kubelet
  - kubelet已经通过NodeGetVolumeStats获取volume状态信息，将对其进行扩展支持健康检测
  - kubelet将会把异常的日志信息记录到pod的事件上
  - 若之后的core仓库有针对本地存储的csi实现，其i也应该实现监控相关的接口
  - 对外暴露volume健康信息的监控指标
  - 

### feature gate

[CSIVolumeHealth](https://github.com/kubernetes/kubernetes/blob/a1e8a5bf39d48719dfbcf49ea09223ee04840502/pkg/features/kube_features.go#L702)



### CSI变化

- ListVolumes 需要返回挂载卷的健康信息
- driver 需要实现 GetVolume 以支持对单个 volume 的健康监测（可选），若不支持将回退调用 ListVolumes
- 对现有的NodeGetVolumeStats功能进行扩展
  - 使用中的pv会有node agent 调用 NodeGetVolumeStats 检测是否正常挂载
  - 调用 NodeGetVolumeStats 检测挂载卷是否可用

如果 csi 实现了VOLUME_CONDITION，需要在ListVolumesResponse携带挂载卷健康信息

如果csi实现了GET_VOLUME，需要在GetVolumeResponse返回volume的大致信息，如果同时实现了VOLUME_CONDITION，则需要将挂载卷的健康信息也包含其中

如果csi实现了VOLUME_CONDITION。需要在NodeGetVolumeStats中包含挂载卷的健康信息。



### 相关PR

[feature: add CSIVolumeHealth feature and gate](https://github.com/kubernetes/kubernetes/pull/99284/files#) #99284 merged

[add volume kubelet_volume_stats_health_abnormal to kubelet](https://github.com/kubernetes/kubernetes/pull/105585/files#) #105585 open