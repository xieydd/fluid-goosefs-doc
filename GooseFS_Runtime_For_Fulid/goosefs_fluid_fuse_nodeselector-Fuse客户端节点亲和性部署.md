在 Fluid 中可以通过 nodeselector 来指定 fuse 节点的部署，比如选择部署 fuse 和 worker 节点不同的节点上，进行存储和计算分离

本文档将向你简单地展示上述特性

## 前提条件

// TODO 
在运行该示例之前，请参考[安装文档](../Introduction/goosefs_fluid_install-安装文档.md)完成安装，并检查 Fluid 各组件正常运行：

```shell
$ kubectl get pod -n fluid-system
goosefsruntime-controller-5b64fdbbb-84pc6   1/1     Running   0          8h
csi-nodeplugin-fluid-fwgjh                  2/2     Running   0          8h
csi-nodeplugin-fluid-ll8bq                  2/2     Running   0          8h
dataset-controller-5b7848dbbb-n44dj         1/1     Running   0          8h
goosefsruntime-controller-654fb74447-cldsv    1/1     Running   0          8h
```

通常来说，你会看到一个名为 `dataset-controller` 的 Pod、一个名为 `goosefsruntime-controller` 的 Pod 和多个名为 `csi-nodeplugin`
的Pod正在运行。其中，`csi-nodeplugin` 这些 Pod 的数量取决于你的 Kubernetes 集群中结点的数量。

## 新建工作环境

```shell
$ mkdir <any-path>/co-locality
$ cd <any-path>/co-locality
```

## 示例

**查看全部结点**

```shell
$ kubectl get nodes
NAME                       STATUS   ROLES    AGE     VERSION
cn-beijing.192.168.1.146   Ready    <none>   7d14h   v1.16.9-aliyun.1
cn-beijing.192.168.1.147   Ready    <none>   7d14h   v1.16.9-aliyun.1
```

**使用标签标识结点**

```shell
$ kubectl label nodes cn-beijing.192.168.1.146 hbase-cache=true
```

在接下来的步骤中，我们将使用 `NodeSelector` 来管理集群中存放数据的位置，所以在这里标记期望的结点

**再次查看结点**

```shell
$ kubectl get node -L hbase-cache
NAME                       STATUS   ROLES    AGE     VERSION            HBASE-CACHE
ap-beijing.192.168.1.146   Ready    <none>   7d14h   v1.16.9-aliyun.1   true
ap-beijing.192.168.1.147   Ready    <none>   7d14h   v1.16.9-aliyun.1
```

目前，在全部2个结点中，仅有一个结点添加了 `hbase-cache=true` 的标签，接下来，我们希望数据缓存仅会被放置在该结点之上

**检查待创建的 Dataset 资源对象**

```shell
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: cosn://test-bucket/
      options:
        fs.cos.accessKeyId: <COS_ACCESS_KEY_ID>
        fs.cos.accessKeySecret: <COS_ACCESS_KEY_SECRET>
        fs.cosn.bucket.region: <COS_REGION>
        fs.cosn.impl: org.apache.hadoop.fs.CosFileSystem
        fs.AbstractFileSystem.cosn.impl: org.apache.hadoop.fs.CosN
        fs.cos.app.id: <COS_APP_ID>
      name: hbase
```

**创建 Dataset 资源对象**

```shell
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
```

**检查待创建的 GooseFSRuntime 资源对象**

```shell
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hbase
spec:
  replicas: 1
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
  fuse:
    nodeSelector:
      hbase-cache: true
```

该配置文件片段中，包含了许多与 GooseFS 相关的配置信息，这些信息将被 Fluid 用来启动一个 GooseFS 实例。上述配置片段中的 `spec.replicas` 属性被设置为 1，这表明 Fluid 将会启动一个包含 1 个 GooseFS 
Master 和 1 个 GooseFS Worker 的 GooseFS 集群

**创建 GooseFSRuntime 资源并查看状态**

```shell
$ kubectl create -f runtime.yaml
goosefsruntime.data.fluid.io/hbase created

$ kubectl get pod -o wide
NAME                 READY   STATUS    RESTARTS   AGE    IP              NODE                       NOMINATED NODE   READINESS GATES
hbase-fuse-42csf     1/1     Running   0          104s   192.168.1.146   ap-beijing.192.168.1.146   <none>           <none>
hbase-master-0       2/2     Running   0          3m3s   192.168.1.147  ap-beijing.192.168.1.146   <none>           <none>
hbase-worker-l62m4   2/2     Running   0          104s   192.168.1.146   ap-beijing.192.168.1.146   <none>           <none>
```

在此处可以看到，fuse 成功启动并且运行在具有指定标签（即 `hbase-cache=true` ）的结点之上。
