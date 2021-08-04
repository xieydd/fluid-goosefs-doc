在 Fluid 中默认的数据源 mount 路径是/<ns>/dir，中间默认会有一层 ns 的路径。如果您希望支持如果只有一个数据源的情况下，可以将该数据源 mount 到 GooseFS 协议的根目录下，这样的好处是用户的程序完全无需改变，则可以使用该功能。

本文档将向你简单地展示上述特性
## 前提条件
TODO
在运行该示例之前，请参考[安装文档]()完成安装，并检查 Fluid 各组件正常运行：


```shell
$ kubectl get pod -n fluid-system
goosefsruntime-controller-5b64fdbbb-84pc6   1/1     Running   0          8h
csi-nodeplugin-fluid-fwgjh                  2/2     Running   0          8h
csi-nodeplugin-fluid-ll8bq                  2/2     Running   0          8h
dataset-controller-5b7848dbbb-n44dj         1/1     Running   0          8h
```
通常来说，你会看到一个名为`dataset-controller`的 Pod、一个名为 `goosefsruntime-controller` 的 Pod 和多个名为`csi-nodeplugin`的 Pod 正在运行。其中，`csi-nodeplugin`这些 Pod 的数量取决于你的 Kubernetes 集群中结点的数量。

## 新建工作环境
```shell
$ mkdir <any-path>/co-locality
$ cd <any-path>/co-locality
```

## 示例

**检查待创建的Dataset资源对象**
```shell
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/stable/
      name: hbase
      path: /
```
TODO
> mountPoint 这里为了方便用户进行实验使用的是 Web UFS, 使用 COS 作为 UFS 可见 []()

在该`Dataset`资源对象的`spec`属性中，我们定义了一个 `path` 的子属性，该子属性要求数据源 mount 到根目录下

**创建Dataset资源对象**
```shell
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
```

**检查待创建的GooseFSRuntime资源对象**
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
```
该配置文件片段中，包含了许多与 GooseFS 相关的配置信息，这些信息将被 Fluid 用来启动一个 GooseFS 实例。上述配置片段中的`spec.replicas`属性被设置为1,这表明 Fluid 将会启动一个包含1个 GooseFS Master和1个 GooseFS Worker 的 GooseFS 实例


**创建GooseFSRuntime资源并查看状态**
**
```shell
$ kubectl get pod -o wide
NAME                 READY   STATUS    RESTARTS   AGE    IP              NODE                       NOMINATED NODE   READINESS GATES
hbase-fuse-42csf     1/1     Running   0          104s   192.168.1.146   192.168.1.146   <none>           <none>
hbase-master-0       2/2     Running   0          3m3s   192.168.1.147   192.168.1.146   <none>           <none>
hbase-worker-l62m4   2/2     Running   0          104s   192.168.1.146   192.168.1.146   <none>           <none>
```


**创建 fuse 节点上查看 mount 信息路径**
```shell
$ kubectl exec -ti hbase-fuse-42csf bash
$ ls /runtime-mnt/goosefs/<namespace>/<DatasetName>/goosefs-fuse/
drwxrwx--x 2 message   99  4096 Apr 23 16:39 /user
drwxr-xr-x 2 message root  4096 Apr 23 16:38 /data
```