## 安装 Harbor

### 1. [下载最新版 Harbor](https://github.com/goharbor/harbor/releases/download/v2.7.4/harbor-offline-installer-v2.7.4.tgz)

| 名称                                                                                                                                           | 大小      | 日期       |            |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ---------- | ---------- |
| [harbor-offline-installer-v2.7.4.tgz](https://github.com/goharbor/harbor/releases/download/v2.7.4/harbor-offline-installer-v2.7.4.tgz)         | 721 MB    | 3 days ago |
| [harbor-offline-installer-v2.7.4.tgz.asc](https://github.com/goharbor/harbor/releases/download/v2.7.4/harbor-offline-installer-v2.7.4.tgz.asc) | 833 Bytes | 3 days ago |
| [harbor-online-installer-v2.7.4.tgz](https://github.com/goharbor/harbor/releases/download/v2.7.4/harbor-online-installer-v2.7.4.tgz)           | 10.8 KB   | 3 days ago |
| [harbor-online-installer-v2.7.4.tgz.asc](https://github.com/goharbor/harbor/releases/download/v2.7.4/harbor-online-installer-v2.7.4.tgz.asc)   | 833 Bytes | 3 days ago |
| (md5sum)[https://github.com/goharbor/harbor/releases/download/v2.7.4/md5sum]                                                                   | 286 Bytes |            | 3 days ago |
| (Source code (zip))[https://github.com/goharbor/harbor/archive/refs/tags/v2.7.4.zip]                                                           |           |            | 4 days ago |
| (Source code (tar.gz))[https://github.com/goharbor/harbor/archive/refs/tags/v2.7.4.tar.gz]                                                     |           |            | 4 days ago |

https://github.com/goharbor/harbor/releases

### 2.（可选）下载相应的 \*.asc 文件以验证该包是否为正版

1.  获取 \*.asc 文件的公钥
    ```sh
    gpg --keyserver hkps://keyserver.ubuntu.com --receive-keys 644FF454C0B4115C
    ```
2.  通过运行以下命令之一验证该包是否为正版

    - 验证在线安装包

      ```sh
      gpg -v --keyserver hkps://keyserver.ubuntu.com --verify harbor-online-installer-v2.7.4.tgz.asc
      ```

    - 验证离线安装包

      ```sh
      gpg -v --keyserver hkps://keyserver.ubuntu.com --verify harbor-offline-installer-v2.7.4.tgz.asc
      ```

### 3. 解压

    `sh
    tar xzvf harbor-online-installer-v2.7.4.tgz
    # 或者
    tar xzvf harbor-offline-installer-v2.7.4.tgz
    `

### 4.[(可选）配置私有证书](https://goharbor.io/docs/2.0.0/install-config/configure-https/)

#### 生成证书颁发机构证书

1. 生成 CA 证书私钥

   ```sh
   openssl genrsa -out ca.key 4096
   ```

2. 生成 CA 证书。
   调整 `-subj` 选项中的值以反映您的组织。如果您使用 FQDN 连接 Harbor 主机，则必须将其指定为公用名 (`CN`) 属性。

   ```sh
   openssl req -x509 -new -nodes -sha512 -days 3650 \
   -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
   -key ca.key \
   -out ca.crt
   ```

#### 生成服务器证书

证书通常包含一个 `.crt` 文件和一个 `.key` 文件，例如 `yourdomain.com.crt` 和 `yourdomain.com.key`。

1. 生成私钥。
   ```sh
   openssl genrsa -out yourdomain.com.key 4096
   ```
2. 生成证书签名请求 (CSR)。

   调整 -subj 选项中的值以反映您的组织。如果您使用 FQDN 连接 Harbor 主机，则必须将其指定为公用名 (CN) 属性，并在密钥和 CSR 文件名中使用它。

   ```sh
    openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key \
    -out yourdomain.com.csr
   ```

3. 生成 x509 v3 证书。

   无论您是使用 FQDN 还是 IP 地址连接到 Harbor 主机，都必须创建此文件，以便可以为 Harbor 主机生成符合主题备用名称 (SAN) 和 x509 v3 的证书扩展要求。替换 DNS 条目以反映您的域。

   ```sh
    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = @alt_names

    [alt_names]
    DNS.1=yourdomain.com
    DNS.2=yourdomain
    EOF
   ```

4. 使用 `v3.ext` 文件为您的 Harbor 主机生成证书。

   将 CRS 和 CRT 文件名中的 `yourdomain.com` 替换为 Harbor 主机名。

   ```sh
   openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in yourdomain.com.csr \
    -out yourdomain.com.crt
   ```

#### 向 Harbor 和 Docker 提供证书

生成 `ca.crt`、`yourdomain.com.crt` 和 `yourdomain.com.key` 文件后，您必须将它们提供给 Harbor 和 Docker，并重新配置 Harbor 才能使用它们。

1. 将服务器证书和密钥复制到 Harbor 主机上的 certficates 文件夹中。

   ```sh
   cp yourdomain.com.crt /data/cert/
   cp yourdomain.com.key /data/cert/
   ```

2. 将 yourdomain.com.crt 转换为 yourdomain.com.cert，供 Docker 使用

   Docker 守护程序将 .crt 文件解释为 CA 证书，将 .cert 文件解释为客户端证书。

   ```
   openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert
   ```

3. 将服务器证书、密钥和 CA 文件复制到 Harbor 主机上的 Docker 证书文件夹中。

   ```sh
   cp yourdomain.com.cert /etc/docker/certs.d/yourdomain.com/
   cp yourdomain.com.key /etc/docker/certs.d/yourdomain.com/
   cp ca.crt /etc/docker/certs.d/yourdomain.com/
   ```

   如果您将默认 nginx 端口 443 映射到其他端口，请创建文件夹 /etc/docker/certs.d/yourdomain.com:port 或 /etc/docker/certs.d/harbor_IP:port。

4. 重新启动 Docker
   ```sh
   systemctl restart docker
   ```

以下示例说明了使用自定义证书的配置。

````
/etc/docker/certs.d/
└── yourdomain.com:port
    ├── yourdomain.com.cert  <-- Server certificate signed by CA
    ├── yourdomain.com.key   <-- Server key signed by CA
    └── ca.crt               <-- Certificate authority that signed the registry certificate
    ```
````

### HTTPS 连接故障排除

如果还有其他中间证书需要合并在

```sh
cat intermediate-certificate.pem >> yourdomain.com.crt
```

当 Docker 守护进程在某些操作系统上运行时，您可能需要在操作系统级别信任证书。

- Ubuntu:

```sh
cp yourdomain.com.crt /usr/local/share/ca-certificates/yourdomain.com.crt
update-ca-certificates
```

- CentOS:

```sh
cp yourdomain.com.crt /etc/pki/ca-trust/source/anchors/yourdomain.com.crt
update-ca-trust
```

### 部署或重新配置 Harbor

如果您尚未部署 Harbor，请参阅[配置 Harbor YML 文件](https://goharbor.io/docs/2.0.0/install-config/configure-yml-file/)以获取有关如何通过在`harbor.yml` 中指定`hostname `和 `https` 属性来配置 Harbor 以使用证书的信息。

如果您已经使用 HTTP 部署了 Harbor，并且想要将其重新配置为使用 HTTPS，请执行以下步骤。

1. 运行`prepare`脚本以启用 HTTPS

   Harbor 使用 `nginx` 实例作为所有服务的反向代理。您可以使用`prepare`脚本将 `nginx`配置为使用 HTTPS。准备工作位于 Harbor 安装程序包中，与 `install.sh` 脚本处于同一目录。

   ```sh
   ./prepare
   ```

   prepare 脚本会根据`harbor.yml`生成配置文件 例如 例如在 common 下会有生成的 nginx 配置文件，他会在/data 目录下创建 harbor 所需的文件 如证书和 database 文件 上传上去的image都保存在这里。

   - 题外话

     由于一台机器很多时候会绑定多个域名或者他们的二级三级域名给443和80端口，但是 docker 抛出端口是不能冲突。harbor 使用的 nginx 向外绑定了 443 的话，不巧我主机还运行了 kubespary 的静态服务器就没法用了，这时候就需要自己创建 nginx 来解决了。

2. 如果 Harbor 正在运行，请停止并删除现有实例。

   您的image数据保留在文件系统中，因此不会丢失任何数据。

   ```sh
   docker-compose down -v
   ```

3. 启动 Harbor

```sh
docker-compose up -d
```

### 验证 HTTPS 连接

为 Harbor 设置 HTTPS 后，您可以通过执行以下步骤来验证 HTTPS 连接

- 打开浏览器并输入 https://yourdomain.com。它应该显示Harbor 界面。

  某些浏览器可能会显示警告，指出证书颁发机构 (CA) 未知。当使用并非来自受信任的第三方 CA 的自签名 CA 时，会发生这种情况。您可以将 CA 导入浏览器以消除警告。

- 在运行 Docker 守护程序的计算机上，检查 `/etc/docker/daemon.json` 文件以确保未为 https://yourdomain.com 设置 `-insecure-registry` 选项。
- 从 Docker 客户端登录 Harbor。

  ```sh
  docker login yourdomain.com
  ```

  如果您已将 nginx 443 端口映射到其他端口，请在登录命令中添加该端口。

  ```sh
  docker login yourdomain.com:port
  ```
