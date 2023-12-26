## 配置网络交换机

需要解释一下为什么要自己创建交换机，hyper-v 其实有一个默认交换机他删除不了，并且每次重启系统后 ip 会发生变化 这会造成虚拟机不能稳定上网和使用外部ssh连接虚拟机。

### 1. 创建内部网络交换器

1. 两种方式

   - 在 hyper-v 中点击创建

   虚拟交换机管理器> 新建模拟网络交换器>选择内部 > 点击按钮创建虚拟交换机
   输入名称（这个名称会在控制面板->网络连接中显示）例如 hyper-v switch

   - 命令行创建

   ```powerShell
   New-VMSwitch -SwitchName "hyper-v switch" -SwitchType Internal
   ```

2. 查看结果  
   在控制面板->网络连接 中会多出一个适配器叫做 vEthernet (hyper-v switch) 这里假如他的 IPV4 是 172.29.240.1 子网掩码是 255.255.240.0 2. 为虚拟机配置网络交换机

- 解释一下 不同交换机的区别

      外部：交换器需要选择物理网卡 这么做 windows 的以太网（有线）会被顶替掉 造成主机不能用网了，可能还有其他原因，从控制面板中可以看到 以太网 不可用了， 一般不会考虑用这个
      内部：虚拟机间可以相互通信，虚拟机和主机间能通信。但是 虚拟机不能上网， 需要结合其他方式（网络共享 或者使用 Nat）
      专有：只能虚拟机间互相通信

### 2. 为虚拟指定交换机

    在 Hyper-v 管理器中 选中一个虚拟机打开他的设置 在菜单中选择网络适配器 右侧虚拟交换机下拉框中选择 刚刚创建的交换机 hyper-v switch

### 3. 解决内部交换器上网问题：

1. 网络共享方式使得内部网络交换器上网

   打开 以太网 属性，点击共享选项卡 勾选 允许其他用户通过此计算机的 Internet 连接来连接 选择家庭网络连接 选择 刚才创建的交换机 vEthernet (hyper-v switch)。这个时候会提示 internet 连接启用时，你的 LAN 适配器将被设置成使用 ip 地址 192.168.137。计算机可能会失去于其他计算机的连接，如果这些计算机有静态 IP 地址，你应该升职称自动获取 IP 地址。你确定要启用 Internet 连接吗？ 然后点击确认。这个时候，我们可以看到 vEthernet (hyper-v switch)中 ipv4 地址被修改为了 192.168.137.1。使用这个地址在虚拟机中配置网关为 192.168.137.1 就能上网了，当然我还要配置静态 ip 方便 ssh 连接。

2. NAT 方式使得内部网络交换器上网

   我是从这里看到 windows 下 NAT 网络 配置的
   https://learn.microsoft.com/zh-cn/virtualization/hyper-v-on-windows/user-guide/setup-nat-network

   1. 以管理员身份打开 PowerShell 控制台。
   2. 创建内部交换机。这一步可以跳过因为 我们在 Hyper-v 管理器中已经创建好了

   ```powerShell
   New-VMSwitch -SwitchName "hyper-v switch" -SwitchType Internal
   ```

   3. 查找刚创建的虚拟交换机的接口索引

   ```powerShell
   $ Get-NetAdapter
   Name                      InterfaceDescription                    ifIndex Status       MacAddress             LinkSpeed
   ----                      --------------------                    ------- ------       ----------             ---------
   WLAN                      Intel(R) Wi-Fi 6 AX201 160MHz                24 Disconnected 6C-94-66-A8-D4-6E          0 bps
   VMware Network Adapte...8 VMware Virtual Ethernet Adapter for ...      16 Up           00-50-56-C0-00-08       100 Mbps
   以太网                    Intel(R) Ethernet Controller (3) I225-V      15 Up           1C-69-7A-AC-28-8C         1 Gbps
   VMware Network Adapte...1 VMware Virtual Ethernet Adapter for ...      10 Up           00-50-56-C0-00-01       100 Mbps
   vEthernet (Default Swi... Hyper-V Virtual Ethernet Adapter             28 Up           00-15-5D-BB-49-8F        10 Gbps
   vEthernet (hyper-v swi... Hyper-V Virtual Ethernet Adapter #2           4 Up           00-15-5D-03-02-0D        10 Gbps
   ```

   记下其 ifIndex 以便在下一步中使用。我们可以看到最后一行就是之前创建交换机 他的 ifIndex 是 4。 4. 使用 New-NetIPAddress 配置 NAT 网关。

   ```powerShell
   New-NetIPAddress -IPAddress 172.29.240.1 -PrefixLength 20 -InterfaceIndex 4
   ```

   这里我就沿用创建时 IP 和掩码位 20 了，因该会覆盖之前的值的。 5. 使用 New-NetNat 配置 NAT 网络。

   ```powerShell
   New-NetNat -Name MyNATnetwork -InternalIPInterfaceAddressPrefix 172.29.240.0/20
   ```

   6. 查看一下 NAT

   ```powerShell
   $ Get-NetNat
   Name                             : MyNATnetwork
   ExternalIPInterfaceAddressPrefix :
   InternalIPInterfaceAddressPrefix : 172.29.240.0/20
   IcmpQueryTimeout                 : 30
   TcpEstablishedConnectionTimeout  : 1800
   TcpTransientConnectionTimeout    : 120
   TcpFilteringBehavior             : AddressDependentFiltering
   UdpFilteringBehavior             : AddressDependentFiltering
   UdpIdleSessionTimeout            : 120
   UdpInboundRefresh                : False
   Store                            : Local
   Active                           : True
   ```

### 4. 配置静态 IP

```bash
$ sudo vim /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses:
      - 172.29.240.2/20
      nameservers:
        addresses:
        - 1.1.1.1
        - 8.8.8.8
        search: []
      routes:
      - to: default
        via: 172.29.240.1
  version: 2
# 应用新ip
$ sudo netplan apply
```

这个时候 会发现 虚拟机间可以 ping 通了 ping baidu.com 也可以了
