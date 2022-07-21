---
layout: post
title: 如何使用Docker部署minikube
---

minikube可以被部署到虚拟机中，也可以部署成一个容器。本文将介绍如何使用Docker部署minikube。

## 前提条件

安装并配置如下的软件：

- WSL 2；
- Docker Desktop for Windows；

## 安装

1. 把docker设置成默认的驱动：

   ```
   minikube config set driver docker
   ```

2. 以管理员身份运行CMD，并执行如下脚本：

   ```
   minikube start --driver=docker --image-mirror-country=cn
   ```
   
   > 如果出现`* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default`，就说明安装成功！

## 常见问题

1. `error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster`

   此时，**Docker Desktop已创建好minikube容器**，在容器内部执行`journalctl -xeu kubelet`或`journalctl -r -n 500 -u kubelet`命令查看详细的错误日志，如下：

   ```
   Jul 20 08:47:02 minikube kubelet[3433]: E0720 08:47:02.968896    3433 pod_workers.go:951] "Error syncing pod, skipping" err="failed to \"CreatePodSandbox\" for \"kube-apiserver-minikube_kub
   e-system(fa1cd85f643eb6467cafb6b166ba85da)\" with CreatePodSandboxError: \"Failed to create sandbox for pod \\\"kube-apiserver-minikube_kube-system(fa1cd85f643eb6467cafb6b166ba85da)\\\": rp
   c error: code = Unknown desc = failed pulling image \\\"k8s.gcr.io/pause:3.6\\\": Error response from daemon: Get \\\"https://k8s.gcr.io/v2/\\\": net/http: request canceled while waiting fo
   r connection (Client.Timeout exceeded while awaiting headers)\"" pod="kube-system/kube-apiserver-minikube" podUID=fa1cd85f643eb6467cafb6b166ba85da
   Jul 20 08:47:02 minikube kubelet[3433]: E0720 08:47:02.968838    3433 kuberuntime_manager.go:815] "CreatePodSandbox for pod failed" err="rpc error: code = Unknown desc = failed pulling imag
   e \"k8s.gcr.io/pause:3.6\": Error response from daemon: Get \"https://k8s.gcr.io/v2/\": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting heade
   rs)" pod="kube-system/kube-apiserver-minikube"
   ```

   由此可见，原因是**拉取`k8s.gcr.io/pause:3.6`失败**。国内的通病！

   **解决方法**：

   1. 首先，在容器内部执行如下命令：

      ```
      docker image pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6
      docker image tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
      systemctl stop kubelet
      systemctl start kubelet
      ```

   2. 然后，在CMD中重启minikube：

      ```
      minikube stop
      minikube start
      ```
2. 
