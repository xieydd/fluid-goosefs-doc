## 快速入门使用 GooseFSRuntime


使用 GooseFSRuntime 流程简单，在准备好基本 k8s 和 COS 环境的条件下，您只需要耗费10分钟左右时间即可部署好需要的 GooseFSRuntime 环境，您可以按照下面的流程进行部署。

### 前提条件

- Git
- Kubernetes集群（version >= 1.14）, 并且支持CSI功能;为获得最佳实践，可以直接在腾讯云容器服务部署 [TKE](https://cloud.tencent.com/document/product/457/32187)
- kubectl（version >= 1.14）
- Helm（version >= 3.0）

接下来的文档假设您已经配置好上述所有环境。

对于`kubectl`的安装和配置，请参考[此处](https://kubernetes.io/docs/tasks/tools/install-kubectl/)。

对于Helm 3的安装和配置，请参考[此处](https://v3.helm.sh/docs/intro/install/)。

这里 `fluid.tgz` 为提供的本地安装包
```shell
$ helm install fluid fluid.tgz
NAME: fluid
LAST DEPLOYED: Mon Mar 29 11:21:46 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

> `helm install`命令的一般格式是`helm install <RELEASE_NAME> <SOURCE>`，在上面的命令中，第一个`fluid`指定了安装的release名字，>这可以自行更改，第二个`fluid.tgz`指定了helm chart所在路径。
### 1、创建命名空间
```shell
kubectl create ns fluid-system
```
### 2、下载 fluid-0.6.0.tgz

TODO
下载 [fluid-0.6.0.tgz]()
### 3、使用 Helm 安装 Fluid

```shell
helm install --set runtime.goosefs.enabled=true fluid fluid-0.6.0.tgz
```
### 4、查看 Fluid 的运行状态

```shell
$ kubectl get pod -n fluid-system
NAME                                         READY   STATUS    RESTARTS   AGE
csi-nodeplugin-fluid-2mfcr                   2/2     Running   0          108s
csi-nodeplugin-fluid-l7lv6                   2/2     Running   0          108s
dataset-controller-5465c4bbf9-5ds5p          1/1     Running   0          108s
goosefsruntime-controller-564f59bdd7-49tkc    1/1     Running   0          108s
```
其中 csi-nodeplugin-fluid-xx 的数量应该与k8s集群中节点node的数量相同。
### 5、GooseFS 缓存环境准备

#### 1、准备 COS 服务
您需要开通阿里云对象存储服务(Object Storage Service，简称COS)，按照使用文档进行 [bucket创建](https://cloud.tencent.com/document/product/436/13309)。


#### 2、准备测试样例数据
我们可以使用 Apache 镜像站点上的 Spark 相关资源作为演示中使用的远程文件。这个选择并没有任何特殊之处，你可以将这个远程文件修改为任意你喜欢的远程文件。

- 现在远程资源文件到本地
```shell
mkdir tmp
cd tmp
wget https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-2.4.8/spark-2.4.8-bin-hadoop2.7.tgz 
wget https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2.tgz 
```

- 上传本地文件到 COS 上

您可以使用腾讯云 COS 团队提供的客户端 [COSCMD](https://cloud.tencent.com/document/product/436/10976) 或者腾讯云 COS 团队提供的 [COS SDK](https://cloud.tencent.com/document/product/436/6474)，按照使用说明将本地下载的文件上传到远程 COS 的 bucket 上。

#### 3、创建 dataset 和 GooseFSRuntime

1. 创建一个 resource.yaml 文件里面包含两部分：
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
        fs.cos.accessKeyId: <COS_ACCESS_KEY_ID>
        fs.cos.accessKeySecret: <COS_ACCESS_KEY_SECRET>
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
        low: "0.8"
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



2. 执行命令，创建 GooseFSRuntime
```shell
kubectl create -f resource.yaml
```

3. 查看部署的 GooseFSRuntime 情况，显示都为 Ready 状态表示部署成功。
```shell
kubectl get goosefsruntime hadoop
NAME     MASTER PHASE   WORKER PHASE   FUSE PHASE   AGE
hadoop    Ready           Ready           Ready     62m
```

4. 查看 dataset 的情况，现实 Bound 状态表示 dataset 绑定成功。
```shell
$ kubectl get dataset hadoop
NAME     UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hadoop        511MiB       0.00B    180.00GiB              0.0%          Bound   1h
```


5. 查看 PV，PVC 创建情况，GooseFSRuntime 部署过程中会自动创建 PV 和 PVC
```shell
kubectl get pv,pvc
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/hadoop   100Gi      RWX            Retain           Bound    default/hadoop                           58m

NAME                           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/hadoop   Bound    hadoop   100Gi      RWX                           58m
```


### 6、创建应用容器体验加速效果
您可以通过创建应用容器来使用 GooseFS 加速服务，或者进行提交机器学习作业来进行体验相关功能。
接下来，我们创建一个应用容器 app.yaml 来使用该数据集，我们将多次访问同一数据，并比较访问时间来展示 GooseFS 的加速效果。

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


1. 使用kubectl完成创建应用
```shell
kubectl create -f app.yaml
```


2. 查看文件大小
```shell
$ kubectl exec -it demo-app -- bash
$ du -sh /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 
210M	/data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 
```

3. 进行文件的cp观察时间消耗了18s
```shell
$ time cp /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2 /dev/null

real	0m18.386s
user	0m0.002s
sys	  0m0.105s
```


4. 查看此时 dataset 的缓存情况，发现210MB的数据已经都缓存到了本地。
```shell
$ kubectl get dataset hadoop
NAME     UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hadoop   210.00MiB       210.00MiB    180.00GiB        100.0%           Bound   1h
```

5. 为了避免其他因素(比如page cache)对结果造成影响，我们将删除之前的容器，新建相同的应用，尝试访问同样的文件。由于此时文件已经被 GooseFS 缓存，可以看到第二次访问所需时间远小于第一次。
```shell
kubectl delete -f app.yaml && kubectl create -f app.yaml
```


6. 进行文件的拷贝观察时间，发现消耗48ms，整个拷贝的时间缩短了300倍
```shell
$ time cp /data/hadoop/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2  /dev/null

real	0m0.048s
user	0m0.001s
sys	  0m0.046s
```

### 7、环境清理
- 删除应用和应用容器
- 删除 GooseFSRuntime
```shell
kubectl delete goosefsruntime hadoop
```
- 删除 dataset
```shell
kubectl delete dataset hadoop
```
TODO
以上通过一个简单的例子完成 GooseFS on Fluid 的入门体验和理解，并最后进行环境的清理，更多 Fluid GooseFSRuntime 的功能使用具体[功能文章]()会进行详细介绍。
