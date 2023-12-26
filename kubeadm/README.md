## 安装和配置先决条件

master 和 worker 节点都素需要配置， 容器运行时是运行 pod 的基础现在 k8s 主推的是 containered 以前多使用 docker engine（k8s 1.20 开始抛弃 docker 1.24 已经标识为弃用了）,另外还有 CRI-O、Mirantis 容器运行时。不太了解就暂时不使用了。
k8s 更新比较快 各个云平台厂商都对偶数版本的 k8s 支持比较好，所以这里使用 1.24.x 版本，最近版本为 1.28.x。  
[阿里云 ACK 版本发布](https://help.aliyun.com/zh/ack/product-overview/support-for-kubernetes-versions)：

| 版本号 | 版本说明                          | 状态   | 发布时间      | 过期时间      |
| ------ | --------------------------------- | ------ | ------------- | ------------- |
| 1.26   | ACK 发布 Kubernetes 1.26 版本说明 | 已发布 | 2023 年 04 月 | 2025 年 04 月 |
| 1.24   | ACK 发布 Kubernetes 1.24 版本说明 | 已发布 | 2022 年 09 月 | 2024 年 09 月 |
| 1.22   | ACK 发布 Kubernetes 1.22 版本说明 | 已发布 | 2021 年 12 月 | 2023 年 12 月 |

[腾讯云 TKE Kubernetes Revision 版本历史](https://cloud.tencent.com/document/product/457/9315):
|时间 |版本|
|----|----|
|2023-07-21 |v1.24.4-tke.9
|2023-07-21 |v1.22.5-tke.18
|2023-07-20 |v1.20.6-tke.37
[Amazon EKS Kubernetes 版本](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/kubernetes-versions.html)

- 1.27
- 1.26
- 1.25
- 1.24
- 1.23

### 每一台机器的系统要求

1. Debian 和 Red Hat 的 Linux 发行版
2. 硬件要求
   CPU 2 核心及以上，至少 2GB 内存
3. 确保 每台机器 product_uuid 和 mac 地址 不同  
   Kubernetes 使用这些值来唯一确定集群中的节点

   ```bash
   $ sudo cat /sys/class/dmi/id/product_uuid
   $ ip link
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
      link/ether 00:15:5d:03:02:24 brd ff:ff:ff:ff:ff:ff
   # 00:15:5d:03:02:24 是mac地址
   ```

   !!! hype-v 复制 Virtual Hard Disks\xxx.vhdx 文件到 新的地方，创建新的虚拟机选择 vhdx 就可以避免 product_uuid 和 mac 地址 重复

4. 禁用交换分区

   ```bash
   # 关闭交换空间
   $ sudo swapoff -a

   # 查看交换分区的状态
   $ sudo free -m
                  total        used        free      shared  buff/cache   available
   Mem:             428         148          72           3         207         126
   Swap:              0           0           0

   # 注销掉/swap.img,不然重启又启用交换空间了
   $ sudo vim /etc/fstab

   # /etc/fstab: static file system information.
   #
   # Use 'blkid' to print the universally unique identifier for a
   # device; this may be used with UUID= as a more robust way to name devices
   # that works even if disks are added and removed. See fstab(5).
   #
   # <file system> <mount point>   <type>  <options>       <dump>  <pass>
   # / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
   /dev/disk/by-id/dm-uuid-LVM-muwmB6Z1tM7ceJ9oOFlGKLC52ArQi97huPeNu6Dl7wPJI9Y1JFhF6ouoYKfypRNT / ext4 defaults 0 1
   # /boot was on /dev/sda2 during curtin installation
   /dev/disk/by-uuid/d37075c8-66bb-424b-bf3a-12de3d7ce0f4 /boot ext4 defaults 0 1
   # /boot/efi was on /dev/sda1 during curtin installation
   /dev/disk/by-uuid/727E-AA48 /boot/efi vfat defaults 0 1
   # /swap.img       none    swap    sw      0       0
   ```

5. 配置 hosts 这个存疑 配置上也好 localAPIEndpoint.advertiseAddress 我是用的是 ip 不是域名
   没有配置 hosts 的话 有可能造成 pod 状态停滞在 ContainerCreating 中

   ```bash
   $ sudo vim /etc/hosts
    ::1     ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters

    185.199.108.133 raw.githubusercontent.com
    140.82.113.6 api.github.com
    140.82.112.3 github.com

    172.29.240.10 cloud
    172.29.240.5  k8s-worker-1
    172.29.240.6  k8s-worker-2
   ```

6. 转发 IPv4 并让 iptables 看到桥接流量  
   我有考虑使用 ipvs 后来看到文中中说 ipvs 还是需要使用 iptables 转发部分流量，后来又看到有人说 cilium 可以避免使用 iptables。
   iptable 当 pod 超过 5000 的时候 性能会下降。

   执行下述指令：

   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter

   # 设置所需的 sysctl 参数，参数在重新启动后保持不变

   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF

   # 应用 sysctl 参数而不重新启动
   sudo sysctl --system
   ```

   通过运行以下指令确认 `br_netfilter` 和 `overlay`模块被加载：

   ```bash
   lsmod | grep br_netfilter
   lsmod | grep overlay
   ```

   通过运行以下指令确认 `net.bridge.bridge-nf-call-iptables`、`net.bridge.bridge-nf-call-ip6tables` 和 `net.ipv4.ip_forward` 系统变量在你的 `sysctl` 配置中被设置为 1：

   ```bash
   sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
   ```

## 安装运行 k8s 所需的 容器运行时 containerd

k8s 的所有东西都要运行在容器运行时中。
容器运行时 还有 CRI-O、 Docker Engine、Mirantis 容器运行时
另外可见[kubernetes 安装容器运行时的介绍](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E6%97%B6)

https://github.com/containerd/containerd/blob/main/docs/getting-started.md

1. 安装 containerd.io
   https://github.com/containerd/containerd/blob/main/docs/getting-started.md

   ```bash
   wget https://github.com/containerd/containerd/releases/download/v1.7.6/containerd-1.7.6-linux-amd64.tar.gz
   sudo tar Cxzvf /usr/local containerd-1.7.6-linux-amd64.tar.gz

   sudo mkdir -p /usr/local/lib/systemd/system
   ```

   https://raw.githubusercontent.com/containerd/containerd/main/containerd.service写入`/usr/local/lib/systemd/system/containerd.service`

   ```bash
   cat << EOF | sudo tee /usr/local/lib/systemd/system/containerd.service
   # Copyright The containerd Authors.
   #
   # Licensed under the Apache License, Version 2.0 (the "License");
   # you may not use this file except in compliance with the License.
   # You may obtain a copy of the License at
   #
   #     http://www.apache.org/licenses/LICENSE-2.0
   #
   # Unless required by applicable law or agreed to in writing, software
   # distributed under the License is distributed on an "AS IS" BASIS,
   # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   # See the License for the specific language governing permissions and
   # limitations under the License.

   [Unit]
   Description=containerd container runtime
   Documentation=https://containerd.io
   After=network.target local-fs.target

   [Service]
   #uncomment to fallback to legacy CRI plugin implementation with podsandbox support.
   #Environment="DISABLE_CRI_SANDBOXES=1"
   ExecStartPre=-/sbin/modprobe overlay
   ExecStart=/usr/local/bin/containerd

   Type=notify
   Delegate=yes
   KillMode=process
   Restart=always
   RestartSec=5

   # Having non-zero Limit*s causes performance problems due to accounting overhead
   # in the kernel. We recommend using cgroups to do container-local accounting.
   LimitNPROC=infinity
   LimitCORE=infinity

   # Comment TasksMax if your systemd version does not supports it.
   # Only systemd 226 and above support this version.
   TasksMax=infinity
   OOMScoreAdjust=-999

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

   ```bash
   systemctl daemon-reload
   systemctl enable --now containerd
   ```

2. 创建配置文件

   ```bash
   sudo mkdir /etc/containerd
   sudo sh -c 'containerd config default > /etc/containerd/config.toml'
   ```

3. 把 `cri`从`disabled_plugins`中移除
   确保 `cri ` 没有在 `disabled_plugins` 中

   ```toml
   # /etc/containerd/config.toml,
   disabled_plugins: []
   ```

4. 结合 `runc` 使用 `systemd` CGroup 驱动，在 `/etc/containerd/config.toml` 中设置：  
    先说结果，kubelet 的驱动必须要和容器运行时 containerd 的驱动统一。  
   在 Linux 上，控制组（CGroup）用于限制分配给进程的资源，控制组（CGroup）的驱动一般有两种 cgroupfs 和 systemd。kubelet 和底层容器运行时都需要对接控制组（CGroup）来强制执行 为 Pod 和容器管理资源 并为诸如 CPU、内存这类资源设置请求和限制。  
    kubelet 目前默认的 CGroup 驱动是 cgroupfs。(估计以后或许会换为 systemd 驱动。在 Kubernetes v1.28 中， Alpha 特性中 你可以开启`automatic`来会自动发现 cgroup 驱动，估计是读取了 linux 的启动配置文件后决定的)  
   当系统使用 systemd 初始化系统是 不建议使用 cgroupfs 驱动程序,systemd 需要系统上只有一个 CGroup 管理器。  
   CGroup 有 v1 和 v2 两个版本 想用 v2 的化必须使用 systemd cgroup 驱动。
   这里有[配置 CGroup 的更多说明](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)

   ```toml
   $ sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml

   # /etc/containerd/config.toml
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
   ...
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
       SystemdCgroup = true
   ```

   ***

   !!! 我安装的高于 v1.22 所以这里不需要配置, 可以跳过  
   在 kubernetesVersion: v1.22.0 及更高版本中，如果用户没有在 KubeletConfiguration 中设置 cgroupDriver 字段， kubeadm 会将它设置为默认值 systemd。  
    kubeadm-config.yaml 在 kubeadm init --config kubeadm-config.yaml 中使用到

   ```bash
   # kubeadm-config.yaml
   kind: ClusterConfiguration
   apiVersion: kubeadm.k8s.io/v1beta3
   kubernetesVersion: v1.21.0 # 低于v1.22.0
   ---
   kind: KubeletConfiguration
   apiVersion: kubelet.config.k8s.io/v1beta1
   cgroupDriver: systemd
   ```

   ***

5. 修改 pause 镜像地址  
    把`registry.k8s.io`替换为`registry.aliyuncs.com/google_containers`或者`registry.cn-hangzhou.aliyuncs.com/google_containers`
   /etc/containerd/config.toml 配置的代理最好和 init 时候使用的代理 一样 要不然 在查看 sudo crictl

   ```toml
   $ sudo sed -i 's/registry.k8s.io/registry.aliyuncs.com\/google_containers/' /etc/containerd/config.toml

   # /etc/containerd/config.toml
   [plugins]
   [plugins."io.containerd.grpc.v1.cri"]
       sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
   ```

   ***

   弊端：

   在 join 节点的时候 cni 网络插件（我是用的是 cilium）的时候会在 node 节点上安装 quay.io/cilium/cilium 另外还会安装 registry.aliyuncs.com/google_containers/pause:3.8（注意并不是 3.9）

   ```bash
   $ sudo crictl image ls
   IMAGE                                           TAG                 IMAGE ID            SIZE
   quay.io/cilium/cilium                           <none>              1d8f313eedac9       183MB
   registry.aliyuncs.com/google_containers/pause   3.8                 4873874c08efc       311kB
   ```

   问题在于我并不知道他很多时候会安装 registry.k8s.io/pause:3.8,从`kubectl describe pod cilium-fjsh9 -n kube-system`中可以看到 registry.k8s.io/pause:3.8 下载失败

   ```bash
   $ kubectl get pods --all-namespaces -o wide
   NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
   kube-system   cilium-fjsh9                       0/1     Init:0/6   0          40s     172.29.240.6    k8s-worker-2   <none>           <none>

   $ kubectl describe pod cilium-fjsh9 -n kube-system
   ...
   Events:
   Type     Reason                  Age   From               Message
   ----     ------                  ----  ----               -------
   Normal   Scheduled               56s   default-scheduler  Successfully assigned kube-system/cilium-fjsh9 to k8s-worker-2
   Warning  FailedCreatePodSandBox  25s   kubelet            Failed to create pod sandbox: rpc error: code = DeadlineExceeded desc = failed to get sandbox image "registry.k8s.io/pause:3.8": failed to pull image "registry.k8s.io/pause:3.8": failed to pull and unpack image "registry.k8s.io/pause:3.8": failed to resolve reference "registry.k8s.io/pause:3.8": failed to do request: Head "https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/pause/manifests/3.8": dial tcp 182.43.124.6:443: i/o timeout
   ```

   后来在 github 上看到配置 proxy 的方式，至少在`sudo crictl pull registry.k8s.io/pause:3.8`成功了

   ```bash
   $sudo systemctl status containerd.service
   ● containerd.service - containerd container runtime
       Loaded: loaded (/usr/local/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
       Active: active (running) since Mon 2023-10-16 11:25:26 UTC; 3min 15s ago
         Docs: https://containerd.io
     Main PID: 10499 (containerd)
         Tasks: 8
       Memory: 44.4M
           CPU: 464ms
       CGroup: /system.slice/containerd.service
               └─10499 /usr/local/bin/containerd
   $ sudo vim /usr/local/lib/systemd/system/containerd.service
   # !!! 在Environment中添加代理
   ExecStartPre=-/sbin/modprobe overlay
   ExecStart=/usr/local/bin/containerd
   Environment="HTTPS_PROXY=http://192.168.3.2:7890"

   $ sudo systemctl  daemon-reload
   $ sudo systemctl restart containerd.service

   $ sudo crictl pull registry.k8s.io/pause:3.8
   Image is up to date for sha256:4873874c08efc72e9729683a83ffbb7502ee729e9a5ac097723806ea7fa13517
   # 终于成功了 使用这个方法按理说可以 不用添加 aliyun 的代理的了， 在多次反复使用aliyun的代理的时候总是出现下载失败的情况
   # 我在一篇博客中看到有搜索阿里云镜像仓库中搜索 可用版本的的介绍，登录进阿里云的管理平台 菜单不见了，他们取消了这个功能。
   # 但url其实还是能用的，只是某些版本可能下载不下载了 全凭运气，比如说我按照k8s官网中操作 `sudo kubeadm image pull --image-repository=registry.aliyuncs.com/google_containers`下载最新版本(v1.28.X)的镜像的时候总是失败 中途还换过
   # `registry.cn-hangzhou.aliyuncs.com/google_containers` 也不行，在查看国内云服务商支持的k8s版本时候才发觉 大家的最高的版本其实是v1.26.X，就有些怀疑是不是 阿里云 没有提供最新版本的下载，尝试降级果然成功了，坑爹呀。
   $ sudo crictl image ls
   IMAGE                                           TAG                 IMAGE ID            SIZE
   quay.io/cilium/cilium                           <none>              1d8f313eedac9       183MB
   ```

   还有一个问题是 node 节点 join 的时候 不清楚为什么 有时会下载`registry.aliyuncs.com/google_containers/pause:3.8`有时会下载`registry.k8s.io/pause:3.8`版本号也比`containerd config default > /etc/containerd/config.toml`和`sudo kubeadm config images list` 打印的 3.9 低，猜测是 cilium 造成的。

6. 安装 runc  
   https://github.com/containernetworking/plugins/releases
   ```bash
   wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64
   sudo install -m 755 runc.amd64 /usr/local/sbin/runc
   ```
7. 安装 CNI plugins
   下载 `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` 文件从 https://github.com/containernetworking/plugins/releases ,然后解压到 `/opt/cni/bin`:

   ```bash
   $ wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
   $ sudo mkdir -p /opt/cni/bin
   $ sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
   ./
   ./macvlan
   ./static
   ./vlan
   ./portmap
   ./host-local
   ./vrf
   ./bridge
   ./tuning
   ./firewall
   ./host-device
   ./sbr
   ./loopback
   ./dhcp
   ./ptp
   ./ipvlan
   ./bandwidth
   ```

二进制文件是静态构建的，应该可以在任何 Linux 发行版上运行。

8. 重启服务
   ```bash
   sudo systemctl restart containerd
   ```

## 安装 kubeadm、kubelet 和 kubectl

https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

1. 你需要在每台机器上安装以下的软件包：

   - kubeadm：用来初始化集群的指令。
   - kubelet：在集群中的每个节点上用来启动 Pod 和容器等。
   - kubectl：用来与集群通信的命令行工具。

   kubeadm 不能帮你安装或者管理 kubelet 或 kubectl， 所以你需要确保它们与通过 kubeadm 安装的控制平面的版本相匹配,
   控制平面与 kubelet 之间可以存在一个次要版本的偏差（0.1），但 kubelet 的版本不可以超过 API 服务器的版本版本。例如，1.7.0 版本的 kubelet 可以完全兼容 1.8.0 版本的 API 服务器

   由于 k8s 每 2 年就有一个大版本被淘汰，需要了解一下[升级方法](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

2. 安装

   ```bash
   sudo apt-get update
   # apt-transport-https 可能是一个虚拟包（dummy package）；如果是的话，你可以跳过安装这个包
   sudo apt-get install -y apt-transport-https ca-certificates curl

   #下载用于 Kubernetes 软件包仓库的公共签名密钥。所有仓库都使用相同的签名密钥，因此你可以忽略URL中的版本：
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   # 添加 Kubernetes apt 仓库：
   # 此操作会覆盖 /etc/apt/sources.list.d/kubernetes.list 中现存的所有配置。
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

   # 更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl

   $ kubelet --version
   Kubernetes v1.26.9
   $ kubeadm version -o yaml
   clientVersion:
      buildDate: "2023-09-13T11:31:28Z"
      compiler: gc
      gitCommit: d1483fdf7a0578c83523bc1e2212a606a44fd71d
      gitTreeState: clean
      gitVersion: v1.26.9
      goVersion: go1.20.8
      major: "1"
      minor: "26"
      platform: linux/amd64
   $ kubectl version -o json
   {
   "clientVersion": {
      "major": "1",
      "minor": "26",
      "gitVersion": "v1.26.9",
      "gitCommit": "d1483fdf7a0578c83523bc1e2212a606a44fd71d",
      "gitTreeState": "clean",
      "buildDate": "2023-09-13T11:32:41Z",
      "goVersion": "go1.20.8",
      "compiler": "gc",
      "platform": "linux/amd64"
   },
   "kustomizeVersion": "v4.5.7"
   }

   ```

## 配置 init 配置文件

```bash
# 查看版本
$ sudo kubeadm config images list --config kubeadm-config.yaml
I1008 10:28:12.521459    1191 version.go:256] remote version is much newer: v1.28.2; falling back to: stable-1.26
registry.k8s.io/kube-apiserver:v1.26.9
registry.k8s.io/kube-controller-manager:v1.26.9
registry.k8s.io/kube-scheduler:v1.26.9
registry.k8s.io/kube-proxy:v1.26.9
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3

# kubeadm config print join-defaults > kubeadm-config.yaml
# kubectl -n kube-system edit configmap kube-proxy 可以查看生成的默认配置
# 修改 版本号 master ip 和name
$ kubeadm config print init-defaults > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.29.240.10
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: cloud
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers # k8s-gcr.m.daocloud.io
kind: ClusterConfiguration
kubernetesVersion: 1.26.9
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}


apiVersion: kubeadm.k8s.io/v1beta3
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: kube-apiserver:6443
    token: abcdef.0123456789abcdef
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: abcdef.0123456789abcdef
kind: JoinConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: cloud
  taints: null
```

## 初始化 集群

可选呀，也可以直接 init

```bash
# 获取1.26的最新版本 这里可以看到是1.26.9
$ sudo kubeadm config images list --config kubeadm-config.yaml
I1008 10:28:12.521459    1191 version.go:256] remote version is much newer: v1.28.2; falling back to: stable-1.26
registry.k8s.io/kube-apiserver:v1.26.9
registry.k8s.io/kube-controller-manager:v1.26.9
registry.k8s.io/kube-scheduler:v1.26.9
registry.k8s.io/kube-proxy:v1.26.9
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.6-0

# 可以不写--kubernetes-version=1.26.9 这里默认是 1.26.9
$ sudo kubeadm config images pull --config kubeadm-config.yaml

$ sudo crictl image list
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
ERRO[0000] validate service connection: validate CRI v1 image API for endpoint "unix:///var/run/dockershim.sock": rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial unix /var/run/dockershim.sock: connect: no such file or directory"
IMAGE                                                             TAG                 IMAGE ID            SIZE
registry.aliyuncs.com/google_containers/coredns                   v1.9.3              5185b96f0becf       14.8MB
registry.aliyuncs.com/google_containers/etcd                      3.5.6-0             fce326961ae2d       103MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.26.9             6c52a40ca2c2b       36.2MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.26.9             139e76b0f613c       32.9MB
registry.aliyuncs.com/google_containers/kube-proxy                v1.26.9             95433ef6ee1d5       21.8MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.26.9             93796172d009b       18MB
registry.aliyuncs.com/google_containers/pause                     3.9                 e6f1816883972       322kB
$ sudo crictl rmi rmi  <IMAGE ID>
```

```bash
# 初始化集群 或者使用 配置文件
# https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/
# https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md
$ sudo kubeadm init --kubernetes-version=1.26.9 \
--apiserver-advertise-address=172.29.240.10 \
--image-repository=registry.aliyuncs.com/google_containers \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all \
--proxy-mode=ipvs \
--skip-phases=addon/kube-proxy \ # 如果你想使用 Cilium 的 kube-proxy 替代品，kubeadm 需要跳过 kube-proxy 部署阶段，因此必须使用 --skip-phases=addon/kube-proxy 选项执行
$ sudo kubeadm init --config kubeadm-config.yaml

[init] Using Kubernetes version: v1.26.9
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [cloud kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.29.240.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [cloud localhost] and IPs [172.29.240.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [cloud localhost] and IPs [172.29.240.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 9.001898 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node cloud as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node cloud as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.29.240.10:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:f00471bc26e8749641ccb03baf80e96a1b8716995a1544db29e35081f0e1bd50

$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
abcdef.0123456789abcdef   23h         2023-10-14T13:04:04Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
f00471bc26e8749641ccb03baf80e96a1b8716995a1544db29e35081f0e1bd50

$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   132m
kube-node-lease   Active   132m
kube-public       Active   132m
kube-system       Active   132m

$ kubectl get nodes
NAME    STATUS     ROLES           AGE    VERSION
cloud   NotReady   control-plane   131m   v1.26.9

$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
kube-system   cilium-lz2kb                       1/1     Running   0          12m   172.29.240.10   cloud    <none>           <none>
kube-system   cilium-operator-5ccfbdf746-mjpfj   0/1     Pending   0          12m   <none>          <none>   <none>           <none>
kube-system   cilium-operator-5ccfbdf746-tg9c6   1/1     Running   0          12m   172.29.240.10   cloud    <none>           <none>
kube-system   coredns-5bbd96d687-dh5j2           1/1     Running   0          15m   10.0.0.164      cloud    <none>           <none>
kube-system   coredns-5bbd96d687-p7d65           1/1     Running   0          15m   10.0.0.170      cloud    <none>           <none>
kube-system   etcd-cloud                         1/1     Running   8          15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-apiserver-cloud               1/1     Running   8          16m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-controller-manager-cloud      1/1     Running   12         15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-proxy-rfbss                   1/1     Running   0          15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-scheduler-cloud               1/1     Running   12         15m   172.29.240.10   cloud    <none>           <none>
```

```bash

```

## 安装 Pod 网络附加组件（cilium）

我安装的是 cilium
https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/

```bash
# https://docs.cilium.io/en/stable/installation/k8s-install-helm/
$ helm repo add cilium https://helm.cilium.io/
$ helm install cilium cilium/cilium --version 1.14.2 --namespace kube-system
# 或者使用 cilium cli 安装
$ cilium install --version 1.14.2

$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
kube-system   cilium-lz2kb                       1/1     Running   0          12m   172.29.240.10   cloud    <none>           <none>
kube-system   cilium-operator-5ccfbdf746-mjpfj   0/1     Pending   0          12m   <none>          <none>   <none>           <none>
kube-system   cilium-operator-5ccfbdf746-tg9c6   1/1     Running   0          12m   172.29.240.10   cloud    <none>           <none>
kube-system   coredns-5bbd96d687-dh5j2           1/1     Running   0          15m   10.0.0.164      cloud    <none>           <none>
kube-system   coredns-5bbd96d687-p7d65           1/1     Running   0          15m   10.0.0.170      cloud    <none>           <none>
kube-system   etcd-cloud                         1/1     Running   8          15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-apiserver-cloud               1/1     Running   8          16m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-controller-manager-cloud      1/1     Running   12         15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-proxy-rfbss                   1/1     Running   0          15m   172.29.240.10   cloud    <none>           <none>
kube-system   kube-scheduler-cloud               1/1     Running   12         15m   172.29.240.10   cloud    <none>           <none>
```

## join worker 节点

在 master 节点 kubeadm init 后会打印出 join 的命令行，如果没有记录下来 可以使用下面的方式获取。

```bash
# 在node 节点上执行
sudo kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>

# 获取 token
$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
abcdef.0123456789abcdef   23h         2023-10-14T13:04:04Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token

# hash 获取
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
f00471bc26e8749641ccb03baf80e96a1b8716995a1544db29e35081f0e1bd50

# 最终结果
$ sudo kubeadm join 172.29.240.10:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:f00471bc26e8749641ccb03baf80e96a1b8716995a1544db29e35081f0e1bd50
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
# 重新生成 token
$ kubeadm token create --print-join-command
kubeadm join 172.29.240.10:6443 --token usvf4p.b10auzartkw5vakg --discovery-token-ca-cert-hash sha256:f00471bc26e8749641ccb03baf80e96a1b8716995a1544db29e35081f0e1bd50

# master节点上执行
$ kubectl -n kube-system get cm kubeadm-config -o yaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: registry.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    kubernetesVersion: v1.26.9
    networking:
      dnsDomain: cluster.local
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
kind: ConfigMap
metadata:
  creationTimestamp: "2023-10-13T13:04:02Z"
  name: kubeadm-config
  namespace: kube-system
  resourceVersion: "195"
  uid: e4a3d7f2-4f81-4cda-b4b8-03c55b441b7a
$ kubectl get nodes
NAME           STATUS     ROLES           AGE    VERSION
cloud          Ready      control-plane   30m    v1.26.9
k8s-worker-1   NotReady   <none>          5m9s   v1.26.9
$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                               READY   STATUS              RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   cilium-lz2kb                       1/1     Running             0          33m   172.29.240.10   cloud          <none>           <none>
kube-system   cilium-operator-5ccfbdf746-mjpfj   0/1     ContainerCreating   0          33m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   cilium-operator-5ccfbdf746-tg9c6   1/1     Running             0          33m   172.29.240.10   cloud          <none>           <none>
kube-system   cilium-tjxhq                       0/1     Init:0/6            0          11m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   coredns-5bbd96d687-dh5j2           1/1     Running             0          37m   10.0.0.164      cloud          <none>           <none>
kube-system   coredns-5bbd96d687-p7d65           1/1     Running             0          37m   10.0.0.170      cloud          <none>           <none>
kube-system   etcd-cloud                         1/1     Running             8          37m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-apiserver-cloud               1/1     Running             8          37m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-controller-manager-cloud      1/1     Running             12         37m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-proxy-k7b9p                   0/1     ContainerCreating   0          11m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   kube-proxy-rfbss                   1/1     Running             0          37m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-scheduler-cloud               1/1     Running             12         37m   172.29.240.10   cloud          <none>           <none>
# 添加后第二个节点的时候
$kubectl get pods --all-namespaces -o wide --watch
NAMESPACE     NAME                               READY   STATUS              RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   cilium-crz25                       0/1     Init:0/6            0          52s   172.29.240.6    k8s-worker-2   <none>           <none>
kube-system   cilium-lz2kb                       1/1     Running             0          36m   172.29.240.10   cloud          <none>           <none>
kube-system   cilium-operator-5ccfbdf746-mjpfj   0/1     ContainerCreating   0          36m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   cilium-operator-5ccfbdf746-tg9c6   1/1     Running             0          36m   172.29.240.10   cloud          <none>           <none>
kube-system   cilium-tjxhq                       0/1     Init:0/6            0          14m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   coredns-5bbd96d687-dh5j2           1/1     Running             0          39m   10.0.0.164      cloud          <none>           <none>
kube-system   coredns-5bbd96d687-p7d65           1/1     Running             0          39m   10.0.0.170      cloud          <none>           <none>
kube-system   etcd-cloud                         1/1     Running             8          39m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-apiserver-cloud               1/1     Running             8          39m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-controller-manager-cloud      1/1     Running             12         39m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-proxy-h8vwt                   0/1     ContainerCreating   0          52s   172.29.240.6    k8s-worker-2   <none>           <none>
kube-system   kube-proxy-k7b9p                   0/1     ContainerCreating   0          14m   172.29.240.5    k8s-worker-1   <none>           <none>
kube-system   kube-proxy-rfbss                   1/1     Running             0          39m   172.29.240.10   cloud          <none>           <none>
kube-system   kube-scheduler-cloud               1/1     Running             12         39m   172.29.240.10   cloud          <none>           <none>
```

## （可选）从控制平面节点以外的计算机控制集群

为了使 kubectl 在其他计算机（例如笔记本电脑）上与你的集群通信， 你需要将管理员 kubeconfig 文件从控制平面节点复制到工作站，如下所示：

```bash
scp root@172.29.240.10:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```

## 安装 Hubber
1. 安装hubber cli
```bash
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
# 注意不要转义{} 坑死人了
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-$HUBBLE_ARCH.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```
2. cilium 开启 hubber

```bash
# 开启hubble
cilium hubble enable
# 或者 使用helm 开启hubber
helm upgrade cilium cilium/cilium --version 1.14.2 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```

## 问题总结

### 1. 代理问题

- `registry.aliyuncs.com/google_containers`或者`registry.cn-hangzhou.aliyuncs.com/google_containers` 代理可能没有最新的版本，导致 `sudo kubeadm config images pull --config kubeadm-config.yaml`失败，

  解决版本：  
  降低版本 参考 <b>安装 kubeadm、kubelet 和 kubectl > 2. 安装</b> 部分将 1.26 降低 最高版本参考[阿里云 ACK 版本发布](https://help.aliyun.com/zh/ack/product-overview/support-for-kubernetes-versions)：

- worker 节点 join 后 会在 自动在 worker 节点上下载 cni 的所需镜像（cilium 需要`quay.io/cilium/cilium`） 以及`registry.aliyuncs.com/google_containers/pause:3.8`

  解决版办法：  
  为 worker 节点的 kubelet 添加代理环境变量

  ```bash
  $ sudo vim /usr/local/lib/systemd/system/containerd.service
  ...
  [Service]
  Environment="HTTPS_PROXY=http://192.168.3.2:7890"
  ...
  $ sudo systemctl daemon-reload
  $ sudo systemctl restart containerd
  ```

### 2. 安装疑惑

- 每台机器的环境配置

  每个机器都要 执行`每一台机器的系统要求`和 `安装容器运行时 containerd`部分的安装

- kubeadm、kubelet 和 kubectl 是否每个节点都要安装？
  至于`安装 kubeadm、kubelet 和 kubectl`部分， 只需要在控制平面节点执行即可，有些人说 kubectl worker 节点是不需要安装的，原因是 kubectl 是使用 Kubernetes API 与 Kubernetes 集群的控制面进行通信的命令行工具。worker 节点不需要控制集群那是控制平台节点的工作。

### 3. 命令行警报解决方法

- 执行 crictl 的时候总是会提示
  `WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. `

  解决方法：

  ```bash
  cat <<EOF | sudo tee /etc/crictl.yaml
  runtime-endpoint: unix:///var/run/containerd/containerd.sock
  image-endpoint: unix:///var/run/containerd/containerd.sock
  timeout: 10
  # debug: true
  debug: false
  EOF
  ```

  ***

  - crictl 是 kubernetes 集群的客户端工具，用于与 kubelet 通信。
  - ctr 是 containerd 的命令行工具，用于与 containerd 通信。
    crictr image 命令用于与 kubelet 通信。kubelet 是通过 crictl 与 containerd 通信的，所以需要配置 crictl 的默认连接。
    kubeadm config image pul

### 4. 流程注意问题：

- 先配置好 master 节点，安装上 cni 网络组件（如 cilium flannel calico ）再 从 worker 节点上执行 `sudo kubeadm join masterIP::6443 ...`

### 5. 配置注意项

- k8s 文档中有交代过在[关于 apiserver-advertise-address 和 ControlPlaneEndpoint 的注意事项](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#considerations-about-apiserver-advertise-address-and-controlplaneendpoint) 的配置问题

  大概讲的是 kubeadm 不支持将没有 --control-plane-endpoint 参数的单个控制平面集群转换为高可用性集群

  - kubeadm init 和 join 所需的 config 怎么修改？
  - [kubeadm init](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)
  - [kubeadm join](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-join/)
  - [kubeadm config](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-config/)
  - 在[配置 API](https://kubernetes.io/zh-cn/docs/reference/config-api/)中的[kubeadm 配置 (v1beta3)](https://kubernetes.io/zh-cn/docs/reference/config-api/kubeadm-config.v1beta3/)这个页面中可以看到
    有一下配置框架 具体有哪些选项都可以查到

    ```yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: InitConfiguration

    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration

    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration

    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration

    apiVersion: kubeadm.k8s.io/v1beta3
    kind: JoinConfiguration
    ```

    这也应征了 yaml 配置文件为什么这么配置 又为什么叫做配置 api 的原因了，以前总是很好奇 他们所说的 api 和 配置文件怎么关联的。
