## 离线安装 Rancher server

rancher 可以安装在 k3s 集群中 也可以安装在 docker 中。安装在 docker 上 rancher 不能使用集群实现高可用。  
据官网的交代是 rancher 部署集群中一般创建 3 个 pod 以及以上（后面提及到的 rancher 安装脚本在不指定个数的时候也会自动安装 3 个 rancher pod），并且在 rancher server 集群上面再增加一个负载均衡器。  
在线安装的方式因为网络的问题很容易出错，即便我使用外网部署的 harbor 做加速但还是很容易出错，或许出错的原因是网络问题，也或许是我操作的问题，总之离线安装的方式是更稳定的方式，所以还是离线安装吧。

离线安装具体看官网 开始使用>安装和升级>其他安装方式>[离线 Helm cli 安装](https://ranchermanager.docs.rancher.com/zh/pages-for-subheaders/air-gapped-helm-cli-install)

另外还有 开始使用> 快速入门> 指南部署 > [RancherHelm CLI 快速入门](https://ranchermanager.docs.rancher.com/zh/getting-started/quick-start-guides/deploy-rancher-manager/helm-cli)
rancher 的文档也是很散 好多注意事项都是分散的

| 说明                | 域名                  | IP             |
| ------------------- | --------------------- | -------------- |
| rancher server      | rancher.dashboard.com | 192.168.56.101 |
| 集群控制节点        |                       | 192.168.56.102 |
| 集群控制 work1 节点 |                       | 192.168.56.103 |
| 集群控制 work2 节点 |                       | 192.168.56.104 |
| harbor              | reg.kougen.buzz       |                |

### 1. 配置 Harbor

0. 安装在其他文档中

1. 准备离线镜像(体积过大，建议在本地搭建)

   离线镜像非常非常大 估计有几十个 G 大我云上的空间 11G 已经被它占满了还只下载到列表的一半。所以我一般都会跳过这一步，除非要在本地搭建，这样安装 rancher 的时候速度会更快。
   具体看 开始使用> 安装和升级>其他安装方式>离线 Helm CLI 安装>[2. 收集镜像并发布到私有仓库](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/other-installation-methods/air-gapped-helm-cli-install/publish-images)
   我这里就不写了，我没有用这种办法。

2. 配置镜像地址

   具体看 k3s 的文档 安装> [私有镜像仓库配置](https://docs.k3s.io/zh/installation/private-registry)
   你可以将 Containerd 配置为连接到私有镜像仓库，并在节点上使用私有镜像仓库拉取私有镜像。

   启动时，K3s 会检查 /etc/rancher/k3s/ 中是否存在 registries.yaml 文件，并指示 containerd 使用该文件中定义的镜像仓库。如果你想使用私有的镜像仓库，你需要在每个使用镜像仓库的节点上以 root 身份创建这个文件。
   安装后 你可以使用 `sudo circtl image ls` 来查看镜像。

   ```sh
   sudo mkdir -p /etc/rancher/k3s/
   sudo touch /etc/rancher/k3s/registries.yaml
   sudo vim /etc/rancher/k3s/registries.yaml
   ```

   ```yaml
   # /etc/rancher/k3s/registries.yaml
   ---
   mirrors:
   docker.io:
     endpoint:
       - "https://reg.kougen.buzz"
   quay.io:
     endpoint:
       - "https://reg.kougen.buzz"
   configs:
   reg.kougen.buzz:
     auth:
       username: admin
       password: 910624Sai
   ```

### 2. 安装 Rancher 的宿主环境

k3s 和 docker 二选一 docker 不支持高可用

- 准备 k3s 集群  
   下载版本需要看 Rancher[支持的 k3s 版本](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-7-9)
  现在 最新版的 Rancher 是 v2.7.9，支持 k3s 版本是 v1.26 https://github.com/k3s-io/k3s/releases 下找到的最高发行版是 v1.27.7+k3s2。
  我在 github 上看到 Rancher v2.8.0 支持 k3s 版本是 v1.27.7 (Default)、v1.26.10、 v1.25.15

  > 还是推荐偶数版本号

  1. 离线或者在线 二选一

     - 选项 A：离线安装 k3s

       ```sh
       wget https://github.com/k3s-io/k3s/releases/download/v1.26.10%2Bk3s2/k3s
       wget https://github.com/k3s-io/k3s/releases/download/v1.26.10%2Bk3s2/k3s-airgap-images-amd64.tar

       wget https://github.com/k3s-io/k3s/releases/download/v1.27.7%2Bk3s2/k3s
       wget https://github.com/k3s-io/k3s/releases/download/v1.27.7%2Bk3s2/k3s-airgap-images-amd64.tar
       ```

       ```sh
       # agent 节点不需要config.yaml 他在安装的时候 默认读取环境变量 而不是文件 读不到环境变量的话默认安装的是 k3s server 而不是 k3s agent
       sudo mkdir -p /var/lib/rancher/k3s/agent/images/
       sudo cp ./k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
       sudo mkdir -p /etc/rancher/k3s
       sudo cp k3s /usr/local/bin
       sudo chmod +x /usr/local/bin/k3s
       sudo cp registries.yaml /etc/rancher/k3s/
       chmod +x install.sh
       sudo cp config.yaml /etc/rancher/k3s/

       # 安装master节点
       # config.yaml
       # write-kubeconfig-mode: "0644"
       # cluster-init: true
       # 我有看到 有添加 server 或者 agent 参数 但是我发现并没有用
       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" ./install.sh
       # --flannel-backend=none --disable-network-policy 想禁用flannel的话就把它添加上

       # 10.0.8.16 是腾讯云的内网ip 111.230.70.135是公网ip
       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" INSTALL_K3S_EXEC="--disable traefik --disable metrics-server --disable servicelb --disable coredns --node-external-ip 111.230.70.135 --advertise-address 111.230.70.135 --node-ip 10.0.8.16" ./install.sh

       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" INSTALL_K3S_EXEC="--advertise-address 111.230.70.135 --node-ip 10.0.8.16" ./install.sh

       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" K3S_URL=https://111.230.70.135:6443 K3S_TOKEN=K104b052cc848bdec68c91593f1a831b846608c7d660fcb6f97288df69618dcc1ba::server:a32af900f460605676bc028bcab7bba6 INSTALL_K3S_EXEC="--node-external-ip 104.168.30.145" ./install.sh


       # 安装另外的master节点 需要提前修改config.yaml文件 修改token 从`sudo cat /var/lib/rancher/k3s/server/node-token`查看 ip指定为第一创建的master节点的ip
       # config.yaml
       # agent节点没有/etc/rancher/k3s/k3s.yaml  write-kubeconfig-mode在agent上没有意义而且会报错
       # write-kubeconfig-mode: "0644"
       # server: https://cluster-init节点的ip:6443
       # token: 集群token
       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" ./install.sh

       # 添加agent节点
       # 一定要写上K3S_URL和K3S_TOKEN在config.yaml中标识会让它认为是master节点即使使用agent参数也不行
       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" K3S_URL=https://<init master node>:6443 K3S_TOKEN=集群token ./install.sh

       INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" K3S_URL=https://111.230.70.135:6443 K3S_TOKEN=K104b052cc848bdec68c91593f1a831b846608c7d660fcb6f97288df69618dcc1ba::server:a32af900f460605676bc028bcab7bba6 INSTALL_K3S_EXEC="--node-external-ip 104.168.30.145" ./install.sh
       ```

       ```sh
       # 查看集群token 在第一个初始化集群的节点上执行
       sudo cat /var/lib/rancher/k3s/server/node-token
       ```

     - 选项 B：在线安装 k3s

       ```sh
       # 使用国内的镜像下载其实也不慢的
       curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_VERSION=v1.26.10+k3s2 sh -s - server --cluster-init
       # 安装普通k3s集群 命令是这样的 没有server --cluster-init参数
       #[配置选项](https://docs.k3s.io/zh/installation/configuration）
       curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_VERSION=v1.26.10+k3s2 sh
       ```

  2. 配置 KUBECONFIG 环境变量

     给当前用户访问权限要不然总需要 sudo kubectl 不想加 sudo。
     如果向远程访问 rancher 的 k3s 需要把这个`config`复制到主机上 export KUBECONFIG 指定目录 还需要把里面的 127.0.0.1:6443 的 ip 改为集群的 ip

     ```sh
     sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
     export KUBECONFIG=~/.kube/config
     sudo chmod 600 ~/.kube/config
     ```

- 准备 docker 连带安装 Rancher

  1.  安装 docker  
      直接参见[docker 官网](https://docs.docker.com/engine/install/)
  2.  在 docker 中安装 Rancher  
       本文主要还是使用 k3s 集群 安装 Rancher，这就就先交代了。 官方文档中只提及到了 CATTLE_SYSTEM_DEFAULT_REGISTRY 指定离线仓库，我不知道怎么设置/etc/rancher/k3s/registries.yaml 文件
      具体可以 rancher 文档 开始使用>安装和升级>其他安装方式>离线 Helm CLI >[安装 Docker 安装命令](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/other-installation-methods/air-gapped-helm-cli-install/docker-install-commands)

      - 选项 A：使用 Rancher 默认的自签名证书

      ```sh
      docker run -d --restart=unless-stopped \
        -p 80:80 -p 443:443 \
        -e CATTLE_SYSTEM_DEFAULT_REGISTRY=reg.kougen.buzz \ # 设置在 Rancher 中使用的默认私有镜像仓库
        -e CATTLE_SYSTEM_CATALOG=bundled \ # 使用打包的 Rancher System Chart 不知道做什么的
        --privileged \
        reg.kougen.buzz/docker.io/rancher/rancher:v2.8.0
      ```

      - 选项 B：使用你自己的证书 - 自签名
        > 先决条件
        >
        > 1. 证书必须是 pem 文件
        > 2. FULL_CHAIN 包含所有所有中间证书，自己的证书在最前面，后面跟着中间证书, 不包含没有根证书。如需查看示例，请参见[证书故障排除](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/other-installation-methods/rancher-on-a-single-node-with-docker/certificate-troubleshooting)。

      ```sh
      docker run -d --restart=unless-stopped \
        -p 80:80 -p 443:443 \
        -v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem \
        -v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem \
        -v /<CERT_DIRECTORY>/<CA_CERTS.pem>:/etc/rancher/ssl/cacerts.pem \
        -e CATTLE_SYSTEM_DEFAULT_REGISTRY=reg.kougen.buzz \ # 设置在 Rancher 中使用的默认私有镜像仓库
        -e CATTLE_SYSTEM_CATALOG=bundled \ # 使用打包的 Rancher System Chart
        --privileged \
        reg.kougen.buzz/docker.io/rancher/rancher:<RANCHER_VERSION_TAG>
      ```

      - 选项 C：使用你自己的证书 - 可信 CA 签名的证书 我用的 一般是 Let's Encrypt 的
        > 先决条件
        >
        > 1. 证书必须是 pem 文件
        > 2. FULL_CHAIN 包含所有所有中间证书，自己的证书在最前面，后面跟着中间证书, 不包含没有根证书。如需查看示例，请参见[证书故障排除](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/other-installation-methods/rancher-on-a-single-node-with-docker/certificate-troubleshooting)。
        > 3. --no-cacerts 禁用 Rancher 生成的默认 CA 证书
        ```sh
        docker run -d --restart=unless-stopped \
          -p 80:80 -p 443:443 \
          --no-cacerts \
          -v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem \
          -v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem \
          -e CATTLE_SYSTEM_DEFAULT_REGISTRY=reg.kougen.buzz \ # 设置在 Rancher 中使用的默认私有镜像仓库
          -e CATTLE_SYSTEM_CATALOG=bundled \ # 使用打包的 Rancher System Chart
          --privileged \
          reg.kougen.buzz/docker.io/rancher/rancher:<RANCHER_VERSION_TAG>
        ```

### 3. 添加 Helm Chart 仓库

1. [安装 Helm](https://helm.sh/zh/docs/intro/install/)
2. 安装 Rancher 的 Chart 的 Helm Chart 仓库
   ```sh
   helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
   ```
3. 最新的 Rancher Chart 的.tgz 文件
   ```sh
   helm fetch rancher-latest/rancher --version=v2.8.0
   ```

### 4.安装 rancher

#### chart 选项说明

具体配置见 开始使用>安装和升级>安装参考>[Rancher Helm Chart 选项](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/installation-references/helm-chart-options)

| 配置                                                           | Chart 选项                 | 描述                                                                          | 选项值 加粗标注为默认值              |
| -------------------------------------------------------------- | -------------------------- | ----------------------------------------------------------------------------- | ------------------------------------ |
| Rancher 生成的自签名证书 需要 cert-manager                     | ingress.tls.source=rancher | Ingress 的证书来源 使用 Rancher 生成的 CA 签发的自签名证书。需要 cert-manager | rancher(默认值), letsEncrypt, secret |
| 使用已有的证书 安装 rancher 需要在 cattle-system 中创建 Secret | ingress.tls.source=secret  | 通过创建 Kubernetes Secret 保存你自己的证书文件。不需要 cert-manager          | rancher(默认值), letsEncrypt, secret |
| 外部 TLS 终止 ingress 相关配置将无效                           | tls=external               | 在 Rancher 集群（Ingress）外部的 L7 负载均衡器上终止 SSL/TLS                  | ingress(默认值), external            |
| 使用的是私有 CA 签名的证书                                     | privateCA=true             |                                                                               |

#### 安装说明

> Rancher Server 默认设计为安全的，并且需要 SSL/TLS 配置，除非使用 tls=external。
> 以下 3 种安装方式 都是围绕着 ssl 展开的  
> ingress.tls.source=rancher 需要 cert-manager 支持，证书都是生成由 cert-manager 生成。  
> ingress.tls.source=secret 需要自己创建证书，证书是使用 kubernetes 中的 secret 保存的。
> ingress.tls.source=letsEncrypt Rancher 将通过 Let's Encrypt 的 ACME 协议自动申请证书，并在 Ingress 中配置 TLS。比较适合再外网单独部署。
> tls=external 将关闭 Rancher 的 ssl，需要外部程序提供 ssl。比如自己部署 ingress 照文章中说的这个是的 ingress 还担当着负载均衡的功能。

- (可选)1. rancher 结合 cert-manager

  1. 安装 cert-manager

     ```sh
     # 在可以连接互联网的系统中，将 cert-manager 仓库添加到 Helm：
     helm repo add jetstack https://charts.jetstack.io
     helm repo update

     # 为 cert-manager 创建命名空间：
     kubectl create namespace cert-manager

     # 从 Helm Chart 仓库中获取最新可用的 cert-manager Chart：目录下会多出一个cert-manager-v1.13.2.tgz文件
     helm fetch jetstack/cert-manager

     # 为 cert-manager 下载所需的 CRD 文件
     curl -L -o cert-manager-crd.yaml https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.crds.yaml

     # 创建 cert-manager CustomResourceDefinition (CRD)。
     kubectl apply -f cert-manager-crd.yaml

     # 安装 cert-manager
     helm install cert-manager ./cert-manager-v1.13.2.tgz  \
       --namespace cert-manager \
       --set image.repository=reg.kougen.buzz/quay.io/jetstack/cert-manager-controller \
       --set webhook.image.repository=reg.kougen.buzz/quay.io/jetstack/cert-manager-webhook \
       --set cainjector.image.repository=reg.kougen.buzz/quay.io/jetstack/cert-manager-cainjector \
       --set startupapicheck.image.repository=reg.kougen.buzz/quay.io/jetstack/cert-manager-ctl
     ```

  卸载 cert-manager

  ```sh
  helm uninstall cert-manager -n cert-manager
  ```

  2. Heml 安装 rancher

     ```sh
     kubectl create namespace cattle-system
     helm install rancher ./rancher-2.8.0.tgz \
       --namespace cattle-system \
       --set hostname='rancher.dashboard.com' \
       --set replicas=1 \
       --set ingress.tls.source=rancher \
       --set useBundledSystemChart=true
       --set bootstrapPassword='!QAZxsw21234' \
       --set rancherImage=reg.kougen.buzz/docker.io/rancher/rancher \
       --set systemDefaultRegistry=reg.kougen.buzz/docker.io \
       --set certmanager.version=v1.13.2 \
     ```

     > 排错
     >
     > - 问题 1  
     >   Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: resource mapping not found for name: "rancher" namespace: "" from "": no matches for kind "Issuer" in version "cert-manager.io/v1alpha2"  
     >   解决版办法：  
     >   不要加 --set certmanager.version=v1.13.2

  3. 导出 ca 证书供 浏览器和其他地方

     ```sh
     kubectl get secret tls-rancher -n cattle-system -o jsonpath="{.data.tls\.crt}" | base64 --decode > rancher.crt
     ```

     下载到 windows 上然后安装证书浏览器就可以安全访问了。

- (可选)2. rancher 结合 自签证书
  privateCA=true 需要和 ingress.tls.source=secret 结合使用。

  1. 创建 cattle-system

     ```sh
     kubectl create namespace cattle-system
     ```

  2. 为 ingress.tls.source=secret 参数准备工作

     如果指定了 ingress.tls.source=secret 需要配置好 TLS Secret

     我们使用证书和密钥将 cattle-system 命名空间中的 tls-rancher-ingress 密文配置好后，Kubernetes 会为 Rancher 创建对象和服务。  
     将服务器证书和所需的所有中间证书合并到名为 tls.crt 的文件中。将证书密钥复制到名为 tls.key 的文件中。

     例如，acme.sh 在 fullchain.cer 文件中提供服务器证书和 CA 链。 请将 fullchain.cer 命名为 tls.crt，将证书密钥文件命名为 tls.key。

     我这里使用 `mkcert` 生辰的证书 因为是离线环境，如果线上的可能会选择 acme.sh

     ```sh
     # 初始化 生成ca证书 可以看到
     $ mkcert -install
     $ ls $(mkcert -CAROOT)
     rootCA-key.pem  rootCA.pem
     $ echo $(mkcert -CAROOT)
     /home/ubuntu/.local/share/mkcert
     ```

  3. 为 privateCA=true 参数的准备工作

     使用 kubectl 创建 tls 类型的密文。

     ```sh
     kubectl -n cattle-system create secret tls tls-rancher-ingress \
     --cert=tls.crt \
     --key=tls.key
     ```

     关于是不是 ca 证书 官方文档中交代了 2 处配置方式：

     1. 开始使用>安装和升级>资源>[添加 TLS 密文](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/resources/add-tls-secrets#%E4%BD%BF%E7%94%A8%E7%A7%81%E6%9C%89-ca-%E7%AD%BE%E5%90%8D%E8%AF%81%E4%B9%A6)
        使用 kubectl 创建 tls 类型的密文。

        ```sh
        kubectl -n cattle-system create secret generic tls-ca \
          --from-file=cacerts.pem=$(mkcert -CAROOT)/rootCA.pem
        ```

     2. 开始使用>安装和升级>资源>[自定义 CA 根证书](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/resources/custom-ca-root-certificates)

        如果你在内部生产环境使用 Rancher，且不打算公开暴露应用，你可以使用使用私有 CA 颁发的证书。

        Rancher 可能会访问配置了自定义/内部 CA 根证书（也称为自签名证书）的服务。如果 Rancher 无法验证服务的证书，则会显示错误信息 x509: certificate signed by unknown authority。

        如需验证证书，你需要把 CA 根证书添加到 Rancher。由于 Rancher 是用 Go 编写的，我们可以使用环境变量 `SSL_CERT_DIR` 指向容器中 CA 根证书所在的目录。启动 Rancher 容器时，可以使用 Docker 卷选项（-v host-source-directory:container-destination-directory）来挂载 CA 根证书目录。

  4. 创建 Rancher

     ```sh
     kubectl create namespace cattle-system
     helm install rancher ./rancher-2.8.0.tgz \
       --namespace cattle-system \
       --set hostname='rancher.dashboard.com' \
       --set replicas=1 \
       --set ingress.tls.source=secret \
       --set privateCA=true \
       --set bootstrapPassword='!QAZxsw21234' \
       --set rancherImage=reg.kougen.buzz/docker.io/rancher/rancher \
       --set systemDefaultRegistry=reg.kougen.buzz/docker.io \
       --set useBundledSystemChart=true
     ```

- (可选)3. rancher 自身放弃 ssl 使用外部 ssl
  > 出错了在导入集群的时候出现
  >
  > Certificate chain is not complete, please check if all needed intermediate certificates are included in the server certificate (in the correct order) and if the cacerts setting in Rancher either contains the correct CA certificate (in the case of using self signed certificates) or is empty (in the case of using a certificate signed by a recognized CA). Certificate information is displayed above. error: Get \"https://rancher.dashboard.com\": x509: certificate signed by unknown authority
  ````
  ```sh
  kubectl create namespace cattle-system
  helm install rancher ./rancher-2.8.0.tgz \
    --namespace cattle-system \
    --set hostname='rancher.dashboard.com' \
    --set replicas=1 \
    --set privateCA=true \
    --set tls=external \
    --set bootstrapPassword='!QAZxsw21234' \
    --set rancherImage=reg.kougen.buzz/docker.io/rancher/rancher \
    --set systemDefaultRegistry=reg.kougen.buzz/docker.io \
    --set useBundledSystemChart=true
  ````

### 使用 traefik 为 rancher 做负载均衡

```sh
kubectl -n cattle-system create secret tls tls-rancher-ingress \
     --cert=tls.crt \
     --key=tls.key
kubectl -n kube-system create secret tls tls-cert \
  --cert=tls.crt \
  --key=tls.key
kubectl -n kube-system create secret generic tls-ca \
  --from-file=cacerts.pem=$(mkcert -CAROOT)/rootCA.pem
```

默认使用 k3s 安装的 traefik 要想另外使用 helm 安装 traefik 的话会出现 crd 已经安装的错误。我想法是要不然在 k3s 安装时跳过安装 traefik 然后自己使用 helm 再安装最新版的 traefik，要不就用 k3s 自带的 traefik，如果非要安装的话 需要先确认 k3s 安装的 traefik 版本，然后使用 helm 安装指定版本，很大可能需要先删除之前安装过的 treafik crd，crd 是全局安装的会出现冲突。

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: kube-system

spec:
  defaultCertificate:
    secretName: tls-cert
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: rancher-dashboard
  namespace: kube-system
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`rancher.dashboard.com`)
      kind: Rule
      services:
        - name: rancher
          namespace: cattle-system
          port: 80
        - name: rancher
          namespace: cattle-system
          port: 443
      #middlewares:
      # - name: redirect-to-https
#
#  tls:
#    secretName: tls-cert
#---
#apiVersion: traefik.io/v1alpha1
#kind: Middleware
#metadata:
#  name: redirect-to-https
#  namespace: kube-system
#spec:
#  redirectScheme:
#    scheme: https
#    port: "443"
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.dashboard.com`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
#
#  tls:
#    secretName: tls-cert
```

重新部署

```sh
kubectl rollout restart deployment -n kube-system traefik
# 如果还是不行的话 删除pod就好了
kubectl delete pod -n kube-system traefik-xxxxxxxxx
```

### 创建 nginx 为 rancher 做负载均衡

我没试验 搜罗来的数据没有验证

````yaml
# traefik-values.yaml
deployment:
  replicas: 1

service:
  enabled: true
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: traefik-redirectscheme@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: web-secure
    traefik.ingress.kubernetes.io/router.middlewares: default-headers@kubernetescrd

ports:
  web:
    enabled: true
    expose: true
  websecure:
    enabled: true
    expose: true

api:
  insecure: true

tls:
  enabled: true
  secretName: tls-rancher-ingress·
```

```sh
# 查看traefik版本
kubectl get deployment -n kube-system traefik -o jsonpath='{.spec.template.spec.containers[0].image}'
# kubectl get deployment -n kube-system traefik -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d':' -f2
# 删除traefik crd
kubectl delete crd ingressroutes.traefik.containo.
````

```sh
# helm repo add traefik https://helm.traefik.io/traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
# 导出默认 values 添加
# tls:
# enabled: true
# secretName: tls-ca
helm show values traefik/traefik > traefik-values.yaml
helm install traefik traefik/traefik --namespace traefik -f traefik-values.yaml
helm upgrade traefik traefik/traefik -f values.yaml --namespace traefik
```

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: rancher-tls
  issuerRef:
    name: letsencrypt-prod # 根据您的证书颁发机构进行调整
    kind: ClusterIssuer
  commonName: rancher.example.com # 修改为您的域名
  dnsNames:
    - rancher.example.com # 修改为您的域名
```

```
worker_processes 4;
worker_rlimit_nofile 40000;


events {
worker_connections 8192;
}

http {
upstream rancher {
server IP_NODE_1:80;
server IP_NODE_2:80;
server IP_NODE_3:80;
}

    map $http_upgrade $connection_upgrade {
        default Upgrade;
        ''      close;
    }

    server {
        listen 443 ssl http2;
        server_name FQDN;
        ssl_certificate /certs/fullchain.pem;
        ssl_certificate_key /certs/privkey.pem;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Port $server_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://rancher;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            # 此项允许执行的 shell 窗口保持开启，最长可达15分钟。不使用此参数的话，默认1分钟后自动关闭。
            proxy_read_timeout 900s;
            proxy_buffering off;
        }
    }

    server {
        listen 80;
        server_name FQDN;
        return 301 https://$server_name$request_uri;
    }

}

```

#### 关于卸载 Rancher 和 卸载 k3s

官方文档有谈到卸载 常见问题>[卸载 Rancher](https://ranchermanager.docs.rancher.com/zh/faq/rancher-is-no-longer-needed)

```sh
# 卸载Rancher
# 其实卸载不干净 尤其是 cattle-system根本删除不了 以下方法 其实也不适用，我最后的办法都是 k3s-uninstall 重装都比则会个省心省时间
sudo k3s-uninstall.sh
sudo rm -rf /var/lib/rancher
sudo rm -rf /etc/rancher
helm uninstall rancher -n cattle-system --delete-all

git clone https://github.com/rancher/rancher-cleanup.git
kubectl create -f rancher-cleanup/deploy/rancher-cleanup.yaml

kubectl delete secrets --all -n cattle-system
kubectl delete clusterrole --all -n cattle-system
kubectl delete crd --all -n cattle-system
kubectl delete clusterrolebinding --all -n cattle-system
```

--set tls=external 跳过 tls 认证。[外部负载均衡器的 TLS 终止](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/installation-references/helm-chart-options#%E5%A4%96%E9%83%A8-tls-%E7%BB%88%E6%AD%A2)中有具体说明

### (可选) 添加域名解析 192.168.56.101 rancher.dashboard.com

由于我的 rancher 部署在本地，需要在集群中添加域名解析，否则将集群加入到 rancher 时集群无法解析 rancher.dashboard.com。

在集群中运行

```sh
sudo kubectl edit cm coredns -n kube-system
```

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        hosts /etc/coredns/NodeHosts {
          ttl 60
          reload 15s
          fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
        import /etc/coredns/custom/*.override
    }
    import /etc/coredns/custom/*.server
  NodeHosts: |
    10.0.2.3 node1
    192.168.56.101 rancher.dashboard.com
kind: ConfigMap
metadata:
  annotations:
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/4yQwWrzMBCEX0Xs2fEf20nsX9BDybH02lMva2kdq1Z2g6SkBJN3L8IUCiVtbyNGOzvfzoAn90IhOmHQcKmgAIsJQc+wl0CD8wQaSr1t1PzKSilFIUiIix4JfRoXHQjtdZHTuafAlCgq488xUSi9wK2AybEFD
    objectset.rio.cattle.io/id: ""
    objectset.rio.cattle.io/owner-gvk: k3s.cattle.io/v1, Kind=Addon
    objectset.rio.cattle.io/owner-name: coredns
    objectset.rio.cattle.io/owner-namespace: kube-system
  creationTimestamp: "2023-12-02T00:59:34Z"
  labels:
    objectset.rio.cattle.io/hash: bce283298811743a0386ab510f2f67ef74240c57
  name: coredns
  namespace: kube-system
  resourceVersion: "14097"
  uid: d3ca3bda-2f56-4fdf-9932-092c9d8373a9
```

```sh
# 测试是否生效
$ kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 -- nslookup rancher.dashboard.com

Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      rancher.dashboard.com
Address 1: 192.168.56.101 rancher.dashboard.com
pod "busybox" deleted
```

```sh
kubectl run -i --restart=Never --rm test-${RANDOM} --image=ubuntu --overrides='{"kind":"Pod", "apiVersion":"v1", "spec": {"dnsPolicy":"Default"}}' -- sh -c 'cat /etc/resolv.conf'

kubectl get configmap -n kube-system coredns -o json | sed -e 's_loadbalance_log\\n loadbalance_g' | kubectl apply -f -
```

由于我使用是自签证书在集群与 rancher 交流的时候 还需要做 443 加密的 还需要把自签证明 安装到 集群中

```sh
 sudo scp ubuntu@192.168.56.101:~/ssl/cacerts.pem /usr/local/share/ca-certificates/
 sudo update-ca-certificates
```

#### 排错

1. 错误

```sh
kubectl cluster-info dump
# 可以看到类似
 Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: CustomResourceDefinition "ingressroutes.traefik.containo.us" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "traefik-crd"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "kube-system"
```

是因为 之前我手动 helm install traefik 在执行过`helm uninstall traefik -n traefik`但是没有删除`traefik-crd`这个 CRD。删除不了，我卸载 k3s 了

```sh
sudo k3s-uninstall.sh
sudo rm -rf /var/lib/rancher
sudo rm -rf /etc/rancher
```
