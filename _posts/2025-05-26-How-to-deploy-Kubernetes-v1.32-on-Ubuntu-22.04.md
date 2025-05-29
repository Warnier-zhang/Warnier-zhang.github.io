---
layout: post
title: Ubuntu 22.04 部署高可用的Kubernetes v1.32.5集群
---

本篇将逐一介绍Ubuntu 22.04 部署高可用的Kubernetes v1.32.5集群的详细步骤。

#### 1、Ubuntu系统准备

- 修改中文时区

  ```
  timedatectl set-timezone Asia/Shanghai
  ```

- 修改主机名

  ```
  hostnamectl set-hostname k8s-master1
  ```

- 编辑`/etc/hosts`文件，新增如下内容：

  ```
  192.168.1.1 k8s-master1
  192.168.1.2 k8s-master2
  192.168.1.3 k8s-node1
  192.168.1.4 k8s-node2
  192.168.1.10 k8s-vip
  ```

- 安装Ubuntu系统缺失的依赖：

  ```
  apt install conntrack ca-certificates ipset ipvsadm -y
  ```

- 修改Ubuntu系统参数：

  - 关闭交换区

    编辑`/etc/fstab`文件，找到`swap`所在行并注释；或者执行`swapoff -a`命令来临时关闭交换区。
  
  - 启用IPV4转发
  
    编辑`/etc/sysctl.conf`文件，找到如下内容所在行并取消注释：
  
    ```
    ...
    # Uncomment the next line to enable packet forwarding for IPv4
    net.ipv4.ip_forward=1
    
    # Uncomment the next line to enable packet forwarding for IPv6
    #  Enabling this option disables Stateless Address Autoconfiguration
    #  based on Router Advertisements for this host
    net.ipv6.conf.all.forwarding=1
    
    ...
    ```
  
  - 新增`/etc/modules-load.d/k8s.conf`，并新增如下内容：
  
    ```
    overlay
    br_netfilter
    ```
  
  - **启用IPVS**
  
    新增`/etc/modules-load.d/ipvs.conf`文件，并新增如下内容：
  
    ```
    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    #nf_conntrack_ipv4
    nf_conntrack
    ```
  
    > 注意：若Linux内核版本为5.x.x，则使用`nf_conntrack`，否则使用`nf_conntrack_ipv4`。
  
    可通过`ipvsadm -Ln`命令查看IPVS状态。

#### 2、安装containerd

containerd，即容器运行时，是CRI接口规范的实现，来源于Docker Inc，从v1.23开始，取代Docker成为Kubernetes集群默认的容器运行时。

Docker和containerd的关系类似Java中的JDK和JRE。

##### 安装containerd

```
tar Cxzvf /usr/local containerd-2.0.5-linux-amd64.tar.gz
cp containerd.service /usr/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd

install -m 755 runc.amd64 /usr/local/sbin/runc

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz

tar Cxzvf /usr/local/bin nerdctl-2.1.2-linux-amd64.tar.gz
```

客户端命令行程序，除了containerd最初自带的**<u>ctr</u>**和支持CRI接口规范的**<u>crictl</u>**之外，**containerd官方**又推出了兼容Docker语法的**<u>nerdctl</u>**。

##### 配置containerd

新建`/etc/containerd/`目录，并执行`containerd config default > /etc/containerd/config.toml`命令创建默认的配置文件。**配置containerd之后需要重启**！！!

- 配置`cgroup`的驱动程序为`systemd`：

  ```
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes]
      ...
      [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
      ...
          [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
              ...
              ShimCgroup = ''
              SystemdCgroup = true
  ```

- 配置**国内镜像源**：

  containerd **v2.x**和containerd **v1.x**配置镜像的的方式略有不同。

  在**containerd v2.x**之下，以配置`registry.k8s.io`镜像为例，详细步骤如下：
  
  - 首先，在配置文件`config.toml`中，找到如下内容所在行并修改，之后新建`/etc/containerd/certs.d`目录：
  
    ```
    [plugins.'io.containerd.cri.v1.images'.registry]
    	config_path = '/etc/containerd/certs.d'
    ```

  - 然后，在`certs.d`目录下新建`registry.k8s.io`目录，再在`registry.k8s.io`目录下新增`hosts.toml`文件，最后，按如下内容编辑`hosts.toml`文件：
  
    ```
    server = "https://registry.k8s.io"
    
    [host."https://registry.cn-hangzhou.aliyuncs.com/google_containers"]
      capabilities = ["pull", "resolve"]
    [host."https://k8s.nju.edu.cn"]
      capabilities = ["pull", "resolve"]
    ```

  - 配置**私服**

    配置**私服**和配置**镜像**的方式有点类似，以私服地址：192.168.1.11:5000为例：

    首先，在`certs.d`目录下新建`192.168.1.11_5000_`目录，再在`192.168.1.11_5000_`目录下新增`hosts.toml`文件，接下来，按如下内容编辑`hosts.toml`文件：

    ```
    server = "https://192.168.1.11:5000"
    
    [host."http://192.168.1.11:5000"]
      capabilities = ["pull", "resolve", "push"]
      skip_verify = true
    ```

    > 注意：
    >
    > 若私服的协议是**<u>HTTP</u>**，则需要设置`skip_verify = true`，否则会出现`failed to do request: http: server gave HTTP response to HTTPS client`问题。

#### 3、安装kubeadm、kubelet、kubectl

首先，执行`curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`命令；

然后，新增`/etc/apt/sources.list.d/kubernetes.list`文件，并按如下内容编辑该文件：

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.32/deb/ /
```

最后，执行如下脚本安装：

```
apt install kubeadm kubelet kubectl -y
```

#### 4、安装、配置HAProxy

```
apt install haproxy -y
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
    server k8s-master1 192.168.1.1:6443  check
    server k8s-master2 192.168.1.2:6443  check
    
...
```

#### 5、安装、配置keepalived

```
apt install keepalived -y
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
  interface ens18                
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass P@ssw0rd
  }
  # 当前机器IP
  unicast_src_ip 192.168.1.1
  # 其他机器IP
  unicast_peer {
    192.168.1.2
  }
  # 虚拟IP
  virtual_ipaddress {
    192.168.1.10/24          
  }

  track_script {
    haproxy_check
  }
}
```

#### 6、初始化Kubernetes集群

- 高可用

  - 使用`kubeadm init`命令行

    ```
    kubeadm init --kubernetes-version=v1.32.0 --control-plane-endpoint "k8s-vip:16443" --apiserver-advertise-address=192.168.1.1 --upload-certs  --pod-network-cidr=192.168.0.0/16  --service-cidr=10.96.0.0/12  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers  
    ```

  - 使用`kubeadm-config.yaml`

    执行`kubeadm config print init-defaults --component-configs KubeletConfiguration --component-configs KubeProxyConfiguration > kubeadm-config.yaml`命令生成`kubeadm-config.yaml`文件，并按照如下内容修改该文件：

    ```
    apiVersion: kubeadm.k8s.io/v1beta4
    ...
    kind: InitConfiguration
    localAPIEndpoint:
      # 对应 --apiserver-advertise-address=192.168.1.1
      advertiseAddress: 192.168.1.1
      bindPort: 6443
    nodeRegistration:
      # 对应 --cri-socket=unix:///run/containerd/containerd.sock
      criSocket: unix:///run/containerd/containerd.sock
      imagePullPolicy: IfNotPresent
      imagePullSerial: true
      # 主机名
      name: k8s-master1
      taints: null
    ...
    ---
    apiServer: {}
    apiVersion: kubeadm.k8s.io/v1beta4
    ...
    clusterName: kubernetes
    # 对应 --control-plane-endpoint "k8s-vip:16443"，如没有需新增
    controlPlaneEndpoint: "k8s-vip:16443"
    controllerManager: {}
    dns: {}
    encryptionAlgorithm: RSA-2048
    etcd:
      local:
        dataDir: /var/lib/etcd
    # 对应 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers      
    imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    # 对应 --kubernetes-version=1.32.0
    kubernetesVersion: 1.32.0
    networking:
      dnsDomain: cluster.local
      # 对应 --pod-network-cidr=192.168.0.0/16，如没有需新增
      podSubnet: 192.168.0.0/16
      # 对应 --service-cidr=10.96.0.0/12
      serviceSubnet: 10.96.0.0/12
    proxy: {}
    scheduler: {}
    ---
    ...
    kind: KubeProxyConfiguration
    ...
    # 启用IPVS
    mode: "ipvs"
    ...
    ```
    
    最后，执行`kubeadm init --config kubeadm-config.yaml --upload-certs`命令即可。

- 单机版

  ```
  kubeadm init --kubernetes-version=v1.32.0 --apiserver-advertise-address=192.168.1.1 --pod-network-cidr=192.168.0.0/16 --service-cidr=10.96.0.0/12
  ```

> 注意：--pod-network-cidr参数的值需要与选配的Calico（默认192.168.0.0/16）、Flannel（默认10.244.0.0/16）等CNI网络插件保持一致！

#### 7、加入新的Master、Node节点

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

You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join k8s-vip:16443 --token 98hzyr.569w5qqmywa41uq8 \
        --discovery-token-ca-cert-hash sha256:fa9234b899cda31ca15b85dafc89b0bfe000591c6fc10730212acfe0f2c0fcdd \
        --control-plane --certificate-key 631b26d370cb4a86e081fdaad4235201697b33f52c40d4c1ca1e56c99fb6b108

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-vip:16443 --token 98hzyr.569w5qqmywa41uq8 \
        --discovery-token-ca-cert-hash sha256:fa9234b899cda31ca15b85dafc89b0bfe000591c6fc10730212acfe0f2c0fcdd

```

在其他Master节点的机器上，执行如下命令加入新的Maser：

```
kubeadm join k8s-vip:16443 --token 98hzyr.569w5qqmywa41uq8 \
        --discovery-token-ca-cert-hash sha256:fa9234b899cda31ca15b85dafc89b0bfe000591c6fc10730212acfe0f2c0fcdd \
        --control-plane --certificate-key 631b26d370cb4a86e081fdaad4235201697b33f52c40d4c1ca1e56c99fb6b108
```

在其他Node节点的机器上，执行如下命令加入新的Node：

```
kubeadm join k8s-vip:16443 --token 98hzyr.569w5qqmywa41uq8 \
        --discovery-token-ca-cert-hash sha256:fa9234b899cda31ca15b85dafc89b0bfe000591c6fc10730212acfe0f2c0fcdd
```

#### 8、配置kubectl连接Kubernetes集群

在Master节点上，执行如下命令即可使kubectl生效：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
或
export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### 9/1、安装CNI网络插件——Calico（2选1）

首先，提前下载好[calico.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml)文件，Calico默认的`pod CIDR`是`192.168.0.0/16`，如果和Kubernetes集群不一致，就要配置该文件中的`CALICO_IPV4POOL_CIDR`环境变量；

最后，执行`kubectl apply -f calico.yaml`命令来安装Calico。

> 注意：
>
> 1. 如果遇到“Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/bird/bird.ctl: connect: no such file or directory”异常，就要配置IP_AUTODETECTION_METHOD环境变量：
>
>    ```
>    - name: IP_AUTODETECTION_METHOD
>      value: "interface=ens18"
>    ```
>
> 2. 如果主机是**<u>国产</u>**的**<u>海光</u>**CPU，并且虚机启用了**<u>Host CPU</u>**模式，Calico就会出现兼容性问题，此时**<u>可选用Flannel</u>**！！！

> 备注：**如何卸载Calico？**
>
> 1. 执行`kubectl delete -f calico.yml`命令删除所有相关的Pod；
>
> 2. 在**所有节点**上，执行`rm -rf /etc/cni/net.d/*`命令清除Calico缓存；
>
> 3. 重启所有节点上的`kubelet`；

#### 9/2、安装CNI网络插件——Flannel（2选1）

首先，提前下载好[kube-flannel.yaml](https://github.com/flannel-io/flannel/releases/download/v0.26.7/kube-flannel.yml)文件，并按照如下方式修改配置参数：

- **（可选）**找到`namespace: kube-flannel`所在行，并替换为`namespace: kube-system`；

- 配置`pod CIDR`

  Flannel默认的`pod CIDR`是`10.244.0.0/16`，如果和Kubernetes集群不一致，就要修改`net-conf.json`的内容：

  ```
  net-conf.json: |
  {
      # --pod-network-cidr=192.168.0.0/16
      "Network": "192.168.0.0/16",
      "EnableNFTables": false,
      "Backend": {
          "Type": "vxlan"
      }
  }
  ```

- **<u>配置VXLAN端口号</u>**

  > 注意：若Kubernetes集群是基于**<u>深信服超融合云主机</u>**搭建的，则需要配置VXLAN端口号，否则会出现**<u>Pod与Pod之间无法通信、域名无法解析</u>**等问题！！！

  ```
  net-conf.json: |
  {
      "Network": "192.168.0.0/16",
      "EnableNFTables": false,
      "Backend": {
          "Type": "vxlan"
          # Port的默认值为8472
          "Port": 8475
      }
  }
  ```

最后，执行`kubectl apply -f kube-flannel.yml`命令来安装Flannel。

> 备注：**如何卸载Flannel？**
>
> 1. 执行`kubectl delete -f kube-flannel.yml`命令删除所有相关的Pod；
>
> 2. 在**所有节点**上，执行如下命令清除Flannel缓存、网卡：
>
>    ```
>   # 删除网卡
>    ifconfig cni0 down
>    ip link delete cni0
>    ifconfig flannel.1 down
>    ip link delete flannel.1
> 
>    # 清除缓存
>       rm -rf /var/lib/cni/*
>    rm -rf /etc/cni/net.d/*
>    ```
> 
> 3. 重启所有节点上的`kubelet`；

#### 10、卸载Kubernetes集群

```
kubeadm reset
# kubectl delete node k8s-node1
```

#### 11、FAQ

- `Unable to connect to the server: tls: failed to verify certificate: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")`

  由于keepalived创建的VIP可能飘移在其他的Maser节点上，在加入该Master节点之前，通过`k8s-vip:16443`访问Kubernetes API Server可能会报上述错误。

#### 12、常用的Kubernetes排错方法

```
lsof -i:6443
netstat -tunlp | grep 6443

journalctl -xefu containerd
journalctl -xefu kubelet
```
