好的，这是一份为您量身定制的、基于我们整个配置和调试过程的 **Markdown** 格式技术文档。

-----

# OpenWrt IPTV 网关配置指南 (双 NAT / WAN 口访问方案)

## 1\. 目标

本指南旨在将一台 OpenWrt/ImmortalWrt 路由器配置为专用的 IPTV 网关。它使用物理端口（`LAN 2`）接收光猫的 IPTV 专线信号，然后通过 `igmpproxy`（服务机顶盒）和 `udpxy`（服务电脑/手机）将视频流分发到局域网。

最终实现的效果是：**主路由（`192.168.0.x` 网段）下的所有设备，都可以通过访问 OpenWrt 路由器的 `WAN` 口 IP（例如 `192.168.0.13`）来观看 IPTV 直播**。

## 2\. 物理连接拓扑

  * **主路由 (192.168.0.x)**：
      * 负责家庭主网络、DHCP 和 Wi-Fi。
      * 您的电脑、手机等所有客户端都连接到此路由。
  * **OpenWrt IPTV 路由 (本机)**：
      * **`WAN` 口**：连接到**主路由**的 `LAN` 口。它将自动从主路由获取一个 IP (如 `192.168.0.13`)。
      * **`LAN 2` 口**：**物理隔离**，用于连接光猫的**IPTV专用口**。
      * **`LAN 1` 口**：(备用) 用于初次配置或连接专用的机顶盒 (STB)。

## 3\. 必备软件包

在开始前，请通过 `系统` -\> `软件包` 确保已安装（或点击 `刷新列表` 后安装）：

  * `igmpproxy`
  * `luci-app-udpxy` (此包会附带安装 `udpxy`)

## 4\. 详细配置步骤

### 步骤一：配置交换机 (VLAN 物理隔离)

这是实现本方案的基础。

1.  前往 `网络` -\> `交换机`。
2.  确保您的 VLAN 配置如下，以实现 `LAN 2` 口的物理隔离：
      * **VLAN 1 (LAN 区)**：`CPU (eth0): 已标记`，`LAN 1: 未标记`，`LAN 2: 关`，`WAN: 关`
      * **VLAN 2 (WAN 区)**：`CPU (eth0): 已标记`，`LAN 1: 关`，`LAN 2: 关`，`WAN: 未标记`
      * **VLAN 3 (IPTV 区)**：`CPU (eth0): 已标记`，`LAN 1: 关`，`LAN 2: 未标记`，`WAN: 关`
3.  点击 `保存并应用`。

### 步骤二：配置网络接口

此步骤将 VLAN 绑定到逻辑接口。

1.  前往 `网络` -\> `接口`。
2.  **配置 `lan` 接口**：
      * `设备`: 确保为 `eth0.1` (VLAN 1)。
      * `协议`: `静态地址`
      * `IPv4 地址`: `192.168.1.1` (或您自定义的，确保不与主路由冲突)。
3.  **配置 `wan` 接口**：
      * `设备`: 确保为 `eth0.2` (VLAN 2)。
      * `协议`: `DHCP 客户端` (它将从主路由获取 IP)。
4.  **创建 `IPTV` 接口**：
      * 点击 `添加新接口...`
      * `名称`: **`IPTV`** (注意大小写，后续配置将沿用此名称)
      * `协议`: `DHCP 客户端`
      * `设备`: **`eth0.3`** (VLAN 3)
      * 点击 `创建接口`。
5.  **编辑 `IPTV` 接口**：
      * **`高级设置`** 选项卡：
          * **取消勾选** `使用默认网关`。
          * `使用网关跃点`: 填 `20` (使其优先级低于 `wan` 口)。
      * **`防火墙设置`** 选项卡：
          * `创建 / 分配防火墙区域`: 输入 **`IPTV`** (与接口名一致)。
      * 点击 `保存`。
6.  点击 `保存并应用`。

### 步骤三：配置 `igmpproxy`

此服务用于将原始组播流转发到 `lan` 区（供机顶盒使用）。

1.  通过 SSH 登录 OpenWrt 路由器。
2.  编辑配置文件：`vi /etc/config/igmpproxy`
3.  确保文件内容如下 (注意 `network` 名称的大小写必须与接口名 `IPTV` 匹配)：
    ```ini
    config igmpproxy
        option quickleave 1

    config phyint
        option network 'IPTV'
        option direction 'upstream'
        list altnet '0.0.0.0/0'

    config phyint
        option network 'lan'
        option direction 'downstream'
    ```
4.  保存退出 (`:wq`)。
5.  执行命令以重启并启用服务：
    ```sh
    /etc/init.d/igmpproxy restart
    /etc/init.d/igmpproxy enable
    ```

### 步骤四：配置 `udpxy`

此服务用于将组播流转换为 HTTP 流（供电脑、手机使用）。

1.  前往 `服务` -\> `udpxy`。
2.  勾选 `启用`。
3.  `绑定 IP / 接口`: **`0.0.0.0`** (关键！使其监听所有 IP，包括 `wan` 口的 IP)。
4.  `端口`: `4022`
5.  `源 IP / 接口`: **`eth0.3`** (即 IPTV 专线流入的设备)。
6.  `保存并应用`。

### 步骤五：配置防火墙

这是打通所有访问的核心。

1.  前往 `网络` -\> `防火墙`。
2.  **配置 `IPTV` 区域**：
      * 找到 `IPTV` 区域，点击 `修改`。
      * `入站数据`: `接受`
      * `出站数据`: `接受`
      * `区域内转发`: `拒绝`
      * **`IP 动态伪装`**: **勾选** (允许 IPTV 点播/EPG 访问互联网)。
      * **`MSS 钳制`**: **勾选** (提高网络兼容性)。
      * `涵盖的网络`: 确保 `IPTV` 已被勾选。
      * `允许转发到目标区域`: 选择 `wan`。
      * `保存`。
3.  **配置 `通信规则`** 选项卡：
      * **规则 1 (Allow-IGMP)**:
          * `名称`: `Allow-IGMP`
          * `协议`: `IGMP`
          * `源区域`: `IPTV`
          * `目标区域`: `设备 (输入)`
          * `操作`: `接受`
      * **规则 2 (Allow-Multicast)**:
          * `名称`: `Allow-Multicast`
          * `协议`: `UDP`
          * `源区域`: `IPTV`
          * `目标区域`: `lan`
          * `目标地址`: `224.0.0.0/4`
          * `操作`: `接受`
      * **规则 3 (Allow-udpxy-LAN)**:
          * `名称`: `Allow-udpxy-LAN`
          * `协议`: `TCP`
          * `源区域`: `lan`
          * `目标区域`: `设备 (输入)`
          * `目标端口`: `4022`
          * `操作`: `接受`
      * **规则 4 (Allow-udpxy-WAN)** (允许主路由访问的关键):
          * `名称`: `Allow-udpxy-WAN`
          * `协议`: `TCP`
          * `源区域`: **`wan`**
          * `目标区域`: `设备 (输入)`
          * `目标端口`: `4022`
          * `操作`: `接受`
4.  点击 `保存并应用`。

## 5\. 验证与使用

### 1\. 验证服务状态：

  * 在**主路由** (`192.168.0.x` 网段) 的电脑上，打开浏览器。
  * 访问 OpenWrt 路由器的 `WAN` 口 IP（例如 `192.168.0.13`）的 `4022` 端口的 `status` 页面：
    **`http://[OpenWrt-WAN-IP]:4022/status`**
  * 如果您能看到 `udpxy status:` 页面，说明 `udpxy` 服务已在运行且防火墙已成功打通。

### 2\. 播放视频流：

  * 在 Potplayer, VLC 等播放器中，使用 `udpxy` 转换后的 HTTP 地址。
  * **转换格式**：
      * **原始地址**: `rtp://@239.x.x.x:yyyy`
      * **播放地址**: `http://[OpenWrt-WAN-IP]:4022/rtp/239.x.x.x:yyyy`
      * (将 `[OpenWrt-WAN-IP]` 替换为您的 OpenWrt 路由器的 `WAN` 口 IP，例如 `192.168.0.13`)

-----

## 附录：一键配置脚本

**前提**：请**首先**在 LuCI 界面（`网络 -> 交换机`）手动配置好**步骤一**中的 VLAN 划分。

**操作**：通过 SSH 登录后，粘贴并运行以下脚本，可自动完成**步骤二至步骤五**的所有配置。

```sh
#!/bin/sh

# 步骤 1: 检查 VLAN 设备是否存在
echo "--- 正在检查 VLAN 设备 ---"
if [ ! -d "/sys/class/net/eth0.1" ] || [ ! -d "/sys/class/net/eth0.2" ] || [ ! -d "/sys/class/net/eth0.3" ]; then
    echo "错误：未找到 eth0.1, eth0.2 或 eth0.3 设备。"
    echo "请先在 '网络 -> 交换机' 页面手动配置好 VLAN 1, 2, 3。"
    echo "脚本已终止。"
    exit 1
fi
echo "VLAN 检查通过！"

# 步骤 2: 安装软件包
echo "--- 正在安装软件包 ---"
opkg update
opkg install igmpproxy luci-app-udpxy

# 步骤 3: 配置网络接口
echo "--- 正在配置网络接口 ---"
uci set network.lan.device='eth0.1'
uci set network.wan.device='eth0.2'
uci set network.wan.proto='dhcp'

uci -q delete network.IPTV
uci set network.IPTV='interface'
uci set network.IPTV.proto='dhcp'
uci set network.IPTV.device='eth0.3'
uci set network.IPTV.defaultroute='0'
uci set network.IPTV.metric='20'

# 步骤 4: 配置 igmpproxy
echo "--- 正在配置 igmpproxy ---"
uci -q delete igmpproxy.igmpproxy
uci set igmpproxy.igmpproxy='igmpproxy'
uci set igmpproxy.igmpproxy.quickleave='1'
while uci -q delete igmpproxy.@phyint[0]; do :; done
uci add igmpproxy phyint
uci set igmpproxy.@phyint[-1].network='IPTV'
uci set igmpproxy.@phyint[-1].direction='upstream'
uci add_list igmpproxy.@phyint[-1].altnet='0.0.0.0/0'
uci add igmpproxy phyint
uci set igmpproxy.@phyint[-1].network='lan'
uci set igmpproxy.@phyint[-1].direction='downstream'

# 步骤 5: 配置 udpxy
echo "--- 正在配置 udpxy ---"
uci set udpxy.udpxy.disabled='0'
uci set udpxy.udpxy.status='1'
uci set udpxy.udpxy.bind='0.0.0.0'
uci set udpxy.udpxy.port='4022'
uci set udpxy.udpxy.source='eth0.3'

# 步骤 6: 配置防火墙
echo "--- 正在配置防火墙 ---"
uci -q delete firewall.IPTV
uci set firewall.IPTV='zone'
uci set firewall.IPTV.name='IPTV'
uci set firewall.IPTV.input='ACCEPT'
uci set firewall.IPTV.output='ACCEPT'
uci set firewall.IPTV.forward='REJECT'
uci set firewall.IPTV.masq='1'
uci set firewall.IPTV.mtu_fix='1'
uci add_list firewall.IPTV.network='IPTV'
uci add_list firewall.IPTV.forwarding_to='wan'

uci -q delete firewall.Allow_IGMP
uci -q delete firewall.Allow_Multicast
uci -q delete firewall.Allow_udpxy_LAN
uci -q delete firewall.Allow_udpxy_WAN

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-IGMP'
uci set firewall.@rule[-1].src='IPTV'
uci set firewall.@rule[-1].proto='igmp'
uci set firewall.@rule[-1].target='INPUT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Multicast'
uci set firewall.@rule[-1].src='IPTV'
uci set firewall.@rule[-1].dest='lan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_ip='224.0.0.0/4'
uci set firewall.@rule[-1].target='ACCEPT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-udpxy-LAN'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest_port='4022'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='INPUT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-udpxy-WAN'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='4022'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='INPUT'

# 步骤 7: 提交配置并重启服务
echo "--- 正在提交配置并重启服务 ---"
uci commit network
uci commit igmpproxy
uci commit udpxy
uci commit firewall

/etc/init.d/network restart
/etc/init.d/firewall restart
/etc/init.d/igmpproxy enable
/etc/init.d/igmpproxy restart
/etc/init.d/udpxy enable
/etc/init.d/udpxy restart

echo ""
echo "==================================================================="
echo "--- 所有配置已完成！---"
echo "您的 OpenWrt IPTV 路由器已配置完毕。"
echo "请从主路由的 192.168.0.x 网段访问 http://[OpenWrt的WAN口IP]:4022/status"
echo "==================================================================="
```
