## 前提条件

- [Fluid](https://github.com/fluid-cloudnative/fluid)(version >= 0.6.0)
  // TODO
- 请参考[安装文档](../Introduction/goosefs_fluid_install-安装文档.md)完成安装

## 创建 Dataset 和 GooseFSRuntime

创建一个 resource.yaml 文件里面包含两部分：

- 首先包含数据集及 ufs 的 dataset 信息，创建一个 Dataset CRD 对象，其中描述了数据集的来源，如例子中的 test-bucket。
- 接下来需要创建一个 GooseFSRuntime，相当于启动一个 GooseFS 的集群来提供缓存服务。

```yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hadoop
spec:
  mounts:
    - mountPoint: cosn://test-bucket/
      options:
        fs.cosn.userinfo.secretId: <COS_ACCESS_KEY_ID>
        fs.cosn.userinfo.secretKey: <COS_ACCESS_KEY_SECRET>
        fs.cosn.bucket.region: <COS_REGION>
        fs.cosn.impl: org.apache.hadoop.fs.CosFileSystem
        fs.AbstractFileSystem.cosn.impl: org.apache.hadoop.fs.CosN
        fs.cos.app.id: <COS_APP_ID>
  name: hadoop
---
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hadoop
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: HDD
        path: /mnt/disk1
        quota: 100G
        high: "0.9"
        low: "0.2"
```
// TODO
当然为了 AK 等信息安全性，建议使用 secret 来保存相关密钥信息，secret 使用请参考[使用参数加密](./goosefs_fluid_encryptOption-使用参数加密.md)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
stringData:
  fs.cosn.userinfo.secretId: <COS_ACCESS_KEY_ID>
  fs.cosn.userinfo.secretKey: <COS_ACCESS_KEY_SECRET>
---
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hadoop
spec:
  mounts:
    - mountPoint: cosn://yourbucket/
      options:
        fs.cosn.bucket.region: <COS_REGION>
        fs.cosn.impl: org.apache.hadoop.fs.CosFileSystem
        fs.AbstractFileSystem.cosn.impl: org.apache.hadoop.fs.CosN
        fs.cos.app.id: <COS_APP_ID>
      name: hadoop
      encryptOptions:
        - name: fs.cosn.userinfo.secretId
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: fs.cosn.userinfo.secretId
        - name: fs.cosn.userinfo.secretKey
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: fs.cosn.userinfo.secretKey
---
apiVersion: data.fluid.io/v1alpha1
kind: GooseFSRuntime
metadata:
  name: hadoop
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: SSD
        path: /mnt/disk1
        quota: 100G
        high: "0.9"
        low: "0.2"
```

Dataset:
- mountPoint：表示挂载UFS的路径，路径中不需要包含 endpoint 信息。
- options, 在 options 需要指定桶的必要信息，具体可参考 [腾讯云 COS](https://cloud.tencent.com/document/product/436/7751):
  - fs.cos.accessKeyId/fs.cos.accessKeySecret：cos bucket的 AK 信息，有权限访问该 bucket。
GooseFSRuntime, 更多 API 可参考 [api_doc.md](https://github.com/fluid-cloudnative/fluid/blob/master/docs/en/dev/api_doc.md):
- replicas：表示创建 GooseFS 集群节点的数量。
- mediumtype： GooseFS 支持 HDD/SSD/MEM 三种类型缓存介质，提供多级缓存配置。
- path：存储路径。
- quota：缓存最大容量。
- high：水位上限大小 / low： 水位下限大小。

创建 GooseFSRuntime

```shell
$ kubectl create -f resource.yaml
```

查看部署的 GooseFSRuntime 情况，显示都为 Ready 状态表示部署成功。

```shell
$ kubectl get goosefsruntime hadoop
NAME     MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
hadoop    Ready           Ready           Ready     62m
```

查看 dataset 的情况，现实 Bound 状态表示 dataset 绑定成功。

```shell
$ kubectl get dataset hadoop
NAME     UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hadoop       210.00MiB       0.00B    180.00GiB              0.0%          Bound   1h
```

查看 PV，PVC 创建情况，GooseFSRuntime 部署过程中会自动创建 PV 和 PVC

```shell
$ kubectl get pv,pvc
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/hadoop   100Gi      RWX            Retain           Bound    default/hadoop                           58m

NAME                           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/hadoop   Bound    hadoop   100Gi      RWX                           58m
```

## 检查服务是否正常

1、登陆到 master/worker pod 上

```shell
$ kubectl get pod
NAME                              READY   STATUS      RESTARTS   AGE
hadoop-fuse-svz4s         1/1     Running     0          23h
hadoop-master-0           1/1     Running     0          23h
hadoop-worker-2fpbk       1/1     Running     0          23h
```

```shell
$ kubectl exec -ti hadoop-goosefs-master-0 bash
goosefs fs ls /hadoop
```

观察是否可以正常 list 文件

2、登陆到 fuse pod上

```shell
$ kubectl exec -ti hadoop-goosefs-fuse-svz4s bash
cd /runtime-mnt/goosefs/<namespace>/<DatasetName>/goosefs-fuse/<DatasetName>
```

观察是否可以正常 list 文件

## 创建应用容器体验加速效果

您可以通过创建应用容器来使用 GooseFS 加速服务，或者进行提交机器学习作业来进行体验相关功能。接下来，我们创建一个应用容器 app.yaml 来使用该数据集，我们将多次访问同一数据，并比较访问时间来展示 GooseFSRuntime 的加速效果。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-app
spec:
  containers:
    - name: demo
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hadoop
  volumes:
    - name: hadoop
      persistentVolumeClaim:
        claimName: hadoop
```

使用 kubectl 完成创建

```shell
$ kubectl create -f app.yaml
```

查看文件大小

```shell
$ kubectl exec -it demo-app -- bash
$ du -sh /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 
210M	/data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 
```

进行文件的 cp 观察时间消耗了 18s

```shell
$ time cp /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 /dev/null

real	0m18.386s
user	0m0.002s
sys	  0m0.105s
```

查看此时 dataset 的缓存情况，发现 210MB 的数据已经都缓存到了本地。

```shell
$ kubectl get dataset hadoop
NAME     UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hadoop   210.00MiB       210.00MiB    180.00GiB        100.0%           Bound   1h
```

为了避免其他因素(比如 page cache )对结果造成影响，我们将删除之前的容器，新建相同的应用，尝试访问同样的文件。由于此时文件已经被 GooseFS 缓存，可以看到第二次访问所需时间远小于第一次。

```shell
$ kubectl delete -f app.yaml && kubectl create -f app.yaml
```

进行文件的拷贝观察时间，发现消耗 48ms，整个拷贝的时间缩短了 300 倍

```shell
$ time cp /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2  /dev/null

real	0m0.048s
user	0m0.001s
sys	  0m0.046s
```

## 清理环境

```shell
$ kubectl delete -f resource.yaml
```
