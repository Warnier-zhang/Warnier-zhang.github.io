---
layout: post
title: Ubuntu 22.04 部署基于IPVS的高可用Kubernetes v1.27.6集群
---

本篇将逐一介绍Ubuntu 22.04 部署基于IPVS的高可用Kubernetes v1.27.6集群的详细步骤。

1. **Ubuntu系统准备**

   - 编辑`/etc/hosts`文件，新增如下内容：

     ```
     192.168.250.8 k8s-master1
     192.168.250.9 k8s-master2
     192.168.250.101 k8s-node1
     192.168.250.102 k8s-node2
     192.168.250.100 k8s-vip
     ```

   - 安装缺失的应用程序：

     ```
     apt install conntrack
     ```

   - 加载缺失的内核模块：

     新增`k8s.conf`，按照如下内容编辑改文件，并拷贝到`/etc/modules-load.d/`：

     ```
     modprobe -- br_netfilter
     ```

2. **安装containerd**

   containerd，即容器运行时，是CRI接口规范的实现，来源于Docker Inc，从v1.23开始，取代Docker成为Kubernetes集群默认的容器运行时。

   Docker和containerd的关系类似Java中的JDK和JRE。

   **安装方式**如下：

   ```
   tar Cxzvf /usr/local containerd-1.6.24-linux-amd64.tar.gz
   cp containerd.service /usr/lib/systemd/system/containerd.service
   systemctl daemon-reload
   systemctl enable --now containerd
   
   install -m 755 runc.amd64 /usr/local/sbin/runc
   
   mkdir -p /opt/cni/bin
   tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
   
   # nerdctl客户端
   tar Cxzvf /usr/local/bin nerdctl-1.5.0-linux-amd64.tar.gz
   ```

   与Docker不同，containerd引入了命名空间（namespace）概念，容器（container）和镜像（image）只在命名空间下可见，常见的命名空间如下：

   | 运行环境   | 命名空间 |
   | ---------- | -------- |
   | containerd | default  |
   | Docker     | moby     |
   | Kubernetes | k8s.io   |

   containerd最初自带的客户端是ctr，之后社区推出了更好的nerdctl，和docker、nerdctl命令相比，ctr功能简陋、使用繁琐，举个拉取`nginx:latest`的例子：

   ```
   docker image pull nginx:latest
   ctr image pull docker.io/library/nginx:latest
   nerdctl image pull nginx:latest
   ```

   此外，containerd也支持CRI接口规范默认的客户端crictl。但是，crictl只能查看镜像，不能拉取、重命名镜像，可用ctr命令替代：

   ```
   ctr -n k8s.io image pull docker.io/library/nginx:latest
   ```

3. **配置containerd**

   因为新版本Kubernetes使用`systemd`取代`cgroupfs`作为cgroup驱动程序，加上Kubernetes和containerd用到的`registry.k8s.io/pause`镜像的版本可能不一致，所以需要修改部分containerd配置参数。

   执行如下命令创建containerd配置文件：

   ```
   mkdir -p /etc/containerd
   containerd config default > /etc/containerd/config.toml
   ```

   并且，按照如下方式编辑该文件：

   ```
   [plugins."io.containerd.grpc.v1.cri"]
   	...
   	sandbox_image = "registry.k8s.io/pause:3.9"
   	...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
         ...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
           SystemdCgroup = true
           ...
   ```

4. **安装、配置HAProxy**

   ```
   apt install haproxy
   ```

   HAProxy的配置文件是`/etc/haproxy/haproxy.cfg`，可按如下方式修改：

   ```
   ...
   
   frontend k8s-master
       bind *:16443
       mode tcp
       option tcplog
       default_backend k8s-master
   
   backend k8s-master
       mode tcp
       option tcplog
       option tcp-check
       balance roundrobin
       default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
       server k8s-master1 192.168.250.8:6443  check
       server k8s-master2 192.168.250.9:6443  check
       
   ...
   ```

5. **安装、配置keepalived**

   ```
   apt install keepalived
   ```

   keepalived的配置文件是`/etc/keepalived/keepalived.conf`，可按如下方式修改：

   ```
   global_defs {
     notification_email {
     }
     router_id LVS_DEVEL
     vrrp_skip_check_adv_addr
     vrrp_garp_interval 0
     vrrp_gna_interval 0
   }
   
   vrrp_script haproxy_check {
     script "/bin/bash -c 'if [[ $(netstat -nlp | grep 16443) ]]; then exit 0; else exit 1; fi'"
     interval 2
     weight 2
   }
   
   vrrp_instance haproxy-vip {
     state MASTER     
     priority 100
     # 当前机器网卡
     interface enp0s31f6                
     virtual_router_id 60
     advert_int 1
     authentication {
       auth_type PASS
       auth_pass P@ssw0rd
     }
     # 当前机器IP
     unicast_src_ip 192.168.250.9
     # 其他机器IP
     unicast_peer {
       192.168.250.8
     }
     # 虚拟IP
     virtual_ipaddress {
       192.168.250.100/24          
     }
   
     track_script {
       haproxy_check
     }
   }
   ```

6. **启用IPVS**

   首先，安装必备的应用程序：

   ```
   apt install ipset ipvsadm
   ```

   然后，加载IPVS相关的模块：

   ```
   # 按照如下内容来编辑ipvs.conf文件
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack
   
   cp ipvs.conf /etc/modules-load.d/ipvs.conf
   ```

   最后，可通过`lsmod | grep -e ip_vs -e nf_conntrack` 命令验证是否启用成功。

7. **安装kubeadm、kubelet、kubectl**

   - 关闭交换区

     ```
     # 临时关闭交换区
     swapoff -a
     # 永久关闭交换区
     编辑/etc/fsta文件，找到swap所在行并注释
     # 查看交换区状态
     free -m
     ```

   - 安装crictl

     ```
     tar Cxzvf /usr/local/bin crictl-v1.27.0-linux-amd64.tar.gz
     crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
     ```

   - 安装kubeadm

     ```
     install -m 755 kubeadm /usr/local/bin/kubeadm
     ```

   - 安装kubelet

     ```
     install -m 755 kubelet /usr/local/bin/kubelet
     cp kubelet.service /etc/systemd/system/kubelet.service
     mkdir -p /etc/systemd/system/kubelet.service.d
     cp 10-kubeadm.conf /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
     systemctl daemon-reload
     systemctl enable --now kubelet
     ```

   - 安装kubectl

     > 注意：不需要在Node节点上安装kubectl！

     ```
     install -m 755 kubectl /usr/local/bin/kubectl
     ```

8. **初始化Kubernetes集群**

   考虑到`registry.k8s.io`在国内受限，可提前准备好相关的镜像，执行如下命令即可：

   ```
   kubeadm config images pull --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
   
   crictl images
   
   # 使用ctr重命名镜像
   ctr -n k8s.io image tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.10.1 registry.k8s.io/coredns/coredns:v1.10.1
   ...
   ```

   - 高可用

     - 使用`kubeadm init`命令行

       ```
       kubeadm init --kubernetes-version=v1.27.6 --control-plane-endpoint "k8s-vip:16443" --apiserver-advertise-address=192.168.250.9 --upload-certs  --pod-network-cidr=192.168.0.0/16  --service-cidr=10.96.0.0/12    
       ```

     - 使用`kubeadm-config.yaml`

       执行如下命令生成`kubeadm-config.yaml`文件，

       ```
       kubeadm config print init-defaults --component-configs KubeletConfiguration --component-configs KubeProxyConfiguration > kubeadm-config.yaml
       ```

       并且，按照如下内容修改该文件：

       ```
       apiVersion: kubeadm.k8s.io/v1beta3
       ...
       kind: InitConfiguration
       localAPIEndpoint:
         # 对应 --apiserver-advertise-address=192.168.250.9
         advertiseAddress: 192.168.250.9
         bindPort: 6443
       nodeRegistration:
         # 对应 --cri-socket=unix:///var/run/containerd/containerd.sock
         criSocket: unix:///var/run/containerd/containerd.sock
         imagePullPolicy: IfNotPresent
         # 主机名
         name: k8s-master1
       ...
       ---
       apiVersion: kubeadm.k8s.io/v1beta3
       ...
       # 对应 --control-plane-endpoint "k8s-vip:16443"
       controlPlaneEndpoint: "k8s-vip:16443"
       ...
       # 可改为registry.cn-hangzhou.aliyuncs.com/google_containers
       imageRepository: registry.k8s.io
       kind: ClusterConfiguration
       # 对应 --kubernetes-version=1.27.6
       kubernetesVersion: 1.27.6
       ...
         # 对应 --pod-network-cidr=192.168.0.0/16
         podSubnet: 192.168.0.0/16
         # 对应 --service-cidr=10.96.0.0/12
         serviceSubnet: 10.96.0.0/12
       ...
       ---
       apiVersion: kubelet.config.k8s.io/v1beta1
       ...
       # 配置cgroup驱动程序
       cgroupDriver: systemd
       ...
       kind: KubeletConfiguration
       ...
       ---
       apiVersion: kubeproxy.config.k8s.io/v1alpha1
       ...
       kind: KubeProxyConfiguration
       # 启用IPVS
       mode: "ipvs"
       ...
       ```

       最后，执行`kubeadm init --config kubeadm-config.yaml --upload-certs`命令即可。

   - 单机版

     ```
     kubeadm init --kubernetes-version=v1.27.6 --apiserver-advertise-address=192.168.250.9 --pod-network-cidr=192.168.0.0/16 --service-cidr=10.96.0.0/12
     ```

   > 注意：--pod-network-cidr参数的值需要与选配的Calico、Flannel等CNI网络插件保持一致！

9. **加入其他的Master、Node节点**

   在上述步骤初始化完成之后，会出现这样一段提示：

   ```
   Your Kubernetes control-plane has initialized successfully!
   
   To start using your cluster, you need to run the following as a regular user:
   
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   Alternatively, if you are the root user, you can run:
   
     export KUBECONFIG=/etc/kubernetes/admin.conf
   
   You should now deploy a pod network to the cluster.
   Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/
   
   You can now join any number of the control-plane node running the following command on each as root:
   
     kubeadm join k8s-vip:16443 --token lqp8i6.t6obdyuv0rnt7dxc \
           --discovery-token-ca-cert-hash sha256:c4f00fab1ac096899e13acfb1df9a3649f42fed1a7c1f5407b077fdbb21c5660 \
           --control-plane --certificate-key abac0f0aaf5ecdbbb60a6ff57c30eeb481dbf2c5857b7af2142a7e9b9b958f4b
   
   Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
   As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
   "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
   
   Then you can join any number of worker nodes by running the following on each as root:
   
   kubeadm join k8s-vip:16443 --token lqp8i6.t6obdyuv0rnt7dxc \
           --discovery-token-ca-cert-hash sha256:c4f00fab1ac096899e13acfb1df9a3649f42fed1a7c1f5407b077fdbb21c5660 
   ```

   在其他Master节点的机器上，执行如下命令加入新的Maser：

   ```
   kubeadm join k8s-vip:16443 --token lqp8i6.t6obdyuv0rnt7dxc \
           --discovery-token-ca-cert-hash sha256:c4f00fab1ac096899e13acfb1df9a3649f42fed1a7c1f5407b077fdbb21c5660 \
           --control-plane --certificate-key abac0f0aaf5ecdbbb60a6ff57c30eeb481dbf2c5857b7af2142a7e9b9b958f4b
   ```

   在其他Node节点的机器上，执行如下命令加入新的Node：

   ```
   kubeadm join k8s-vip:16443 --token lqp8i6.t6obdyuv0rnt7dxc \
           --discovery-token-ca-cert-hash sha256:c4f00fab1ac096899e13acfb1df9a3649f42fed1a7c1f5407b077fdbb21c5660 
   ```

10. **配置kubectl连接Kubernetes集群**

    在Master节点的机器上，执行如下命令，即可使kubectl生效：

    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    export KUBECONFIG=/etc/kubernetes/admin.conf
    ```

11. **安装CNI网络插件——Calico**

    ```
    kubectl apply -f calico.yaml
    ```

    > 注意：
    >
    > （1）Calico默认的pod CIDR是192.168.0.0/16，如果和Kubernetes集群不一致，就要配置CALICO_IPV4POOL_CIDR环境变量；
    >
    > （2）如果遇到“Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/bird/bird.ctl: connect: no such file or directory”异常，就要配置IP_AUTODETECTION_METHOD环境变量：
    >
    > ```
    > - name: IP_AUTODETECTION_METHOD
    >   value: "interface=enp.*"
    > ```

12. **安装Helm**

    ```
    install -m 755 helm /usr/local/bin/helm
    ```

13. **去掉Master节点无法部署Pod限制**

    ```
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    # kubectl taint nodes --all node-role.kubernetes.io/control-plane="":NoSchedule
    ```

14. **安装metrics-server**

    ```
    # ctr -n k8s.io image pull registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.4
    # ctr -n k8s.io image tag registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.4 registry.k8s.io/metrics-server/metrics-server:v0.6.4
    kubectl apply -f metrics-server.yaml
    ```

    > 注意：
    >
    > 如果遇到“Failed to scrape node" err="Get \"https://192.168.250.102:10250/metrics/resource\": x509: cannot validate certificate for 192.168.250.102 because it doesn't contain any IP SANs" node="k8s-node1”异常，就要配置容器的启动参数：
    >
    > ```
    > ...
    > spec:
    >   containers:
    >     - args:
    > 	  ...
    >       - --kubelet-insecure-tls
    >       image: registry.k8s.io/metrics-server/metrics-server:v0.6.4
    > ...
    > ```

15. **卸载Kubernetes集群**

    ```
    kubeadm reset
    # kubectl delete node k8s-node1
    
    # 如果基于iptables部署kube-proxy，就要清理iptables。
    iptables -F
    iptables -X
    ```

16. **常用的Kubernetes排错方法**

    ```
    lsof -i:6443
    netstat -tunlp | grep 6443
    
    systemctl status containerd
    systemctl status kubelet
    journalctl -xefu kubelet
    ```
