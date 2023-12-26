### 线上使用 k3s 的问题

1. 关于腾讯云之类服务器和 racknerd 组合集群时

   111.230.70.135 为公网 ip 10.0.8.16 为内网 ip

   ```sh
   INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" INSTALL_K3S_EXEC="--node-external-ip 111.230.70.135 --advertise-address 111.230.70.135 --node-ip 10.0.8.16" ./install.sh
   ```

   racknerd 那边加入集群式需要注意
   104.168.30.145 为我的 racknerd 的 ip， 我不知道它的内网 ip 是什么 racknerd 的控制台不像腾讯云阿里云一样在控制台告诉内网 ip 和公网 ip

   ```sh
    INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.26.10+k3s2" K3S_URL=https://111.230.70.135:6443 K3S_TOKEN=K104b052cc848bdec68c91593f1a831b846608c7d660fcb6f97288df69618dcc1ba::server:a32af900f460605676bc028bcab7bba6 INSTALL_K3S_EXEC="--node-external-ip 104.168.30.145" ./install.sh
   ```
