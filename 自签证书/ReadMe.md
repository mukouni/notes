一、 安装 OpenSSL

```bash
# ubuntu
sudo apt update
sudo apt install openssl
vim /etc/ssl/openssl.cnf

# centos 7
sudo yum -y install openssl openssl-devel
vim /etc/pki/tls/openssl.cnf
```

二、 生成自签名 CA 根证书

1. 生成 CA 根证书 RSA 密钥

```bash
openssl genrsa -out ca.key 2048
# 也可以使用genpkey
# -outform PEM 默认是PEM文件输出
# -pkeyopt rsa_keygen_bits:2048 长度默认 2048
# 使用-aes-256-cbc对密钥 二次加密
openssl genpkey -algorithm RSA -out cakey.pem -aes-256-cbc
# 腾讯云要key文件 需要转换一下
openssl rsa -in cakey.pem -out ca.key
```

[genrsa 文档](https://www.openssl.org/docs/man3.1/man1/openssl-genrsa.html)
[genpkey 文档](https://www.openssl.org/docs/man3.1/man1/openssl-genpkey.html)

2. 生成 CA 根证书申请请求（CSR）

```bash
openssl req -new -key ca.key -out ca.csr
```

![生成 CA 根证书申请请求（CSR）](generate_ca.csr.png)
Tip:查看 csr 文件

```
openssl req -text -in ca.csr -noout
```

3. 生成 CA 根证书并签名

```bash
openssl x509 -req -days 3650 -in ca.csr -signkey ca.key -out ca.crt
```

三、 生成客户端 SSL 证书

1. 生成客户端 SSL 证书私钥

```bash
openssl genrsa -out server.key 2048
// 也可以使用genpkey
openssl genpkey -algorithm RSA -out clientkey.pem -outform PEM -pkeyopt rsa_keygen_bits:2048
```

2. 生成 CA 根证书申请请求（CSR）

```bash
openssl req -new -key server.key -out server.csr
```

![生成客户端证书申请请求（CSR）](generate_server.csr.png)

3. 准备客户端证书签名环境

```bash
mkdir -p demoCA/newcerts
touch demoCA/index.txt
touch demoCA/serial
echo "01" > demoCA/serial
cat << EOF >server.ext
subjectAltName = DNS:*.kougen.asia, DNS:kougen.asia
EOF
```

demoCA 目录是/etc/ssl/openssl.cnf 中指定的 修改目录名 需要先修改这个文件
![生成客户端证书申请请求（CSR）](openssl.cnf.png)

4. 生成客户端 SSL 证书并完成签名

```bash
#  这个命令会把 签署的证书 编号记录到 demoCA目录中
#  但是生成的server.crt 包含了一切其他内容  导致不能安装到 nginx和导入到腾讯云中 需要把
# -----BEGIN CERTIFICATE-----
# ...
# -----END CERTIFICATE-----
# 这部分内容 复制到新的 crt文件中 上传给服务器使用
openssl ca -days 365 -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -extfile server.ext
# 单独生成证书 不做任何记录 生成的文件 以-----BEGIN CERTIFICATE-----开头
# openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt -extfile server.ext
```

四、 nginx 部署

1. 编辑 nginx/conf.d/default.conf

```conf
server {
        listen       80;
        listen  [::]:80;
        server_name *.kougen.asia; #填写域名
        #将所有HTTP请求通过rewrite指令重定向到HTTPS
        rewrite ^(.*) https://$server_name$1 permanent;
}

#配置443端口
server {
        listen 443 ssl;  # 1.1版本后这样写
        server_name kougen.asia; #填写域名 二级域名也被包含在内
        ssl_certificate certs/server.crt;  # der或者crt后缀的估计都行
        ssl_certificate_key certs/server.key; # pem和key后缀的文件都可以

        ssl_session_timeout 5m;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;

        ssl_session_cache shared:SSL:1m;

        fastcgi_param  HTTPS        on;
        fastcgi_param  HTTP_SCHEME     https;

  location / {
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   Host              $http_host;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        root html;
        index index.html index.htm;

```

server.crt 内容大概长这样

```
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----
```

server.key 大概长这样

```
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

五、本地系统安装 CA 证书

在浏览器中也能操作 手机中也是同理
(1). 双击 ca.crt 安装证书
![双击ca.crt安装证书](install_ca.png)
(2). 选择本地计算机也可以选个人
![选择本地计算机](install_ca2.png)
(3). 选择安装到 受信任的根证书颁发机构
![选择安装到根证书目录](install_ca3.png)
(4). 最后点击完成
开始菜单 运行 certmgr.msc 看到安装好的证书
![选择安装到根证书目录](install_ca_check1.png)
浏览器中也看到 https 是安全连接的
![选择安装到根证书目录](install_ca_check2.png)

[腾讯云上传 ssl 证书](https://console.cloud.tencent.com/ssl/dsc/upload)
