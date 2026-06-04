---
layout: post
title: Kubernetes集群证书过期更新
---

Kubernetes**客户端证书**的有效期是**1年**，过期之后，就无法访问和管理Kubernetes集群。更新过期证书的方法和步骤如下：

> 注意：在**每台Master**节点上都要执行一遍！对于高可用的Kubernetes集群，推荐最后更新**VIP** Master节点！
>

1. 检查证书是否过期

   ```
   kubeadm certs check-expiration
   ```

2. 备份Kubernetes集群配置

   ```
   cp -rf /etc/kubernetes /etc/kubernetes.bak
   ```

3. 手动更新证书

   ```
   kubeadm certs renew all
   ```

4. 重启kubelet

   ```
   systemctl restart kubelet
   ```

5. 配置kubectl

   ```
   cp -i /etc/kubernetes/admin.conf /root/.kube/config
   ```

6. 删除过期的容器：

   - Docker：
   
     ```
     docker ps |grep etcd|grep -v pause|awk '{print $1}'|xargs -i docker restart {}
     docker ps |grep kube-apiserver|grep -v pause|awk '{print $1}'|xargs -i docker restart {}
     docker ps |grep kube-controller-manager|grep -v pause|awk '{print $1}'|xargs -i docker restart {}
     docker ps |grep kube-scheduler|grep -v pause|awk '{print $1}'|xargs -i docker restart {}
     ```
   
   - containerd：
   
     ```
     nerdctl ps |grep etcd | grep -v pause|awk '{print $1}'|xargs -i nerdctl restart {}
     nerdctl ps |grep kube-apiserver|grep -v pause|awk '{print $1}'|xargs -i nerdctl restart {}
     nerdctl ps |grep kube-controller-manager|grep -v pause|awk '{print $1}'|xargs -i nerdctl restart {}
     nerdctl ps |grep kube-scheduler|grep -v pause|awk '{print $1}'|xargs -i nerdctl restart {}
     ```
