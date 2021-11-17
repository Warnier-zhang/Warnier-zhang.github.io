---
layout: post
title: Kubernetes集成Ceph做持久化存储
---

## Ceph

### 创建存储池（pool）

```
ceph osd pool create amos 256 256
```

#### 设置Pool的存储类型

```
ceph osd pool application enable amos rbd
```

### 创建客户端（client.XXX）

客户端也就是用户，是由CephX管理的。用户的命名遵循`type.id`规则，类型`type`的可选值有`mon`、`osd`、`client`，`client`是最常用的。

`ceph -s`相当于`ceph -s --conf /etc/ceph/ceph.conf --name client.admin --keyring /etc/ceph/ceph.client.admin.keyring`，即以`admin`的身份登录到Ceph集群。

```
创建用户amos，并生成秘钥
ceph auth get-or-create client.amos mon 'allow r' osd 'allow rwx pool=amos' -o ceph.client.amos.keyring

查看用户amos的秘钥
cat ceph.client.amos.keyring

使用Base64算法加密输出用户amos的秘钥
ceph auth get-key client.amos | base64
```

## Kubernetes

### 创建秘钥（Secret）

```
apiVersion: v1
kind: Secret
metadata:
  name: ceph-client-amos-secret
  namespace: kube-system
  labels:
    name: ceph-client-amos-secret
    project: amos
data:
  # ceph auth get-key client.amos | base64
  key: 

```

### 创建存储类（StorageClass）

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: amos-cloud-storage-sc
  labels:
    name: amos-cloud-storage-sc
    project: amos
provisioner: ceph.com/rbd
parameters:
  monitors: 172.16.35.154:6789
  pool: amos
  adminId: admin
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: kube-system
  userId: amos
  userSecretName: ceph-client-amos-secret
  userSecretNamespace: kube-system
  imageFormat: "2"
  imageFeatures: layering
reclaimPolicy: Delete
volumeBindingMode: Immediate

```

### 创建持久卷声明（PersistentVolumeClaim）

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: amos-busybox-pvc
  labels:
    name: amos-busybox-pvc
    project: amos
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: amos-cloud-storage-sc
```

在PVC（PersistentVolumeClaim）绑定池之后，会自动创建对应的块（起到数据隔离的作用，名称：kubernetes-dynamic-pvc-XXX），在PO运行过程中产生的（文件）数据会保存到这个块上；

### 部署Pod

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: amos-busybox-deploy
  labels:
    name: amos-busybox-deploy
    project: amos
spec:
  selector:
    matchLabels:
      name: amos-busybox-po
  template:
    metadata:
      labels:
        name: amos-busybox-po
        project: amos
    spec:
      restartPolicy: Always
      containers:
      - image: busybox:1.28.3
        name: amos-busybox-cntr
        command:
          - sleep
          - "3600"
        volumeMounts:
        - name: busybox-data
          mountPath: /busybox
      volumes:
      - name: busybox-data
        persistentVolumeClaim:
          claimName: amos-busybox-pvc
```

### 查看Pod运行时产生的数据

```
kubectl get po -l 'project=amos'
```

![][1]

```
kubectl exec -it amos-busybox-deploy-f5d495b49-v4mxp -- sh

cd /busybox && echo 'hello, busybox' > hello.txt
```

查询RBD块

```
rbd ls -p amos
```

![查询RBD块][2]

映射RBD块成Block设备，并挂载到文件系统中

```
映射RBD块
rbd map -p amos --image kubernetes-dynamic-pvc-d09cf830-477e-11ec-acd8-b6d799f2c737

挂载RBD模拟磁盘
mount /dev/rbd2 /mnt/ceph-amos/
```

查看Pod运行时产生的数据

![查看Pod运行时产生的数据][3]

[1]: ../images/2021/11/17/6.png
[2]: ../images/2021/11/17/7.png
[3]: ../images/2021/11/17/8.png


