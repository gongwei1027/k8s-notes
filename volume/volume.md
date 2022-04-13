# Volume

[volume介绍](https://kubernetes.io/zh/docs/concepts/storage/volumes/)`https://kubernetes.io/zh/docs/concepts/storage/volumes/`


## 持久卷的完整生命周期
由于 k8s 目前已经陆续的将 in-tree volumePlugin 全部迁移到 csi，以下所有介绍都是基于 csi 。

demo.ymal 创建并添加给 pod 使用一个持久卷。
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: demo
    image: centos:centos8
    command:
      - sleep
      - "3600"
    volumeMounts:
      - name: pvc-test
        mountPath: /var/usr/share/demo-pvc
  volumes:
    - name: pvc-test
      persistentVolumeClaim:
        claimName: demo-pvc
        readOnly: false

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: demo-sc

# pvc 对应的 storageClass 属性，不需要每次都创建，只需部署 csi 时创建即可，列出仅仅是为了好描述
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: demo-sc
provisioner: demo.csi.cn
parameters:
  fsType: ext4
reclaimPolicy: Delete
VolumeBindingMond: Immediate
```
`demp.yaml` 这是一种比较常见的使用持久卷的方式，下面将以 `demo.yaml` 为例描述整个卷的生命周期。

### 1. pvc 和 pod 资源声明
pvc 和 pod 资源请求创建没有先后顺序
#### 1.1 pv，pvc 资源创建
1. pvc 资源创建会提交到 apiServer， apiServer  接收请求并将数据写入到 etcd 中。
2. pv Controller watch 到 pvc 资源有新建。触发 pvc 创建的处理，检查系统中是否存在匹配的 pv， 如果不存在则创建对应的 pv 资源。并指定 pv 所创建的 provisioner。`pvc -> storageClass -> provisioner` 
3. provisioner watch 到 pv 资源有新建，且该 pv 所指定的 provisoner 与自身匹配。触发 pv 创建处理流程，通过 grpc 调用 csi controller plugin 去创建对应的实体 volume。完成创建后，更新 status 并与 pvc 进行 bind。
> 至此整个 pv pvc 创建成功，可以被 pod 进行消费了。
#### 1.2 pod 资源创建
1. pod 资源创建被提交到 apiServer， apiServer 接收请求并将数据写入到 etcd 中。
2. sheduler watch 到有新的 pod 创建，开始进行 pod 调度，通过算法和一些约束给 pod 指定一个可用的 node。将该 pod 与 node 进行 bind，并将该信息通过 apiServer 更新到 etcd 中。
3. 该 kubelet watch 到有新的pod创建，并且该 pod 绑定了该kubelet 所在的节点，kubelet 将走创建 pod 的流程。
   1. 调用 cni 接口创建 pod 网络。
   2. 调用 cri 接口创建 container。
   3. 调用 csi 进行 volume 的挂载。
#### 1.3 kubelet 调用 csi 进行 volume 的挂载


