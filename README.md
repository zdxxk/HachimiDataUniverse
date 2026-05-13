# HachimiDataUniverse
## Ubuntu24.04的安装、基本操作、环境配置
略 懒得写
## OpenIPC图传
### 硬件
-天空端：淘宝购买的铭威2w天空端 **（2w不是功耗，是发射功率）**

-地面站：任意采用`RTL8812AU`方案的网卡
### 在Ubuntu地面站上出图
#### 采用[wfb-ng](https://github.com/svpcom/wfb-ng)方案
1. **安装驱动**
	
```bash
#在终端依次输入如下指令
sudo apt-get install dkms
git clone -b v5.2.20 https://github.com/svpcom/rtl8812au.git
cd rtl8812au/
```
安装完成后记得重新插拔网卡

2.  **检测**
`ifconfig`
- 如果出现类似输出
 **不一定非得是 `wlan0`，也可以是`wlx04d9f5115fdd`等**
```bash
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 2312
        ether 0c:91:60:0a:5a:8b  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

-  运行  `ethtool -i wlan0`  来确认驱动是否为:  `rtl88xxau_wfb`  or  `rtl8812eu`
3.  **准备**
- 运行如下指令，切换网卡的监听通道
`sudo iw dev wlx04d9f5115fdd set channel 64`
铭威图传的默认通道为 **64**
- 替换`/etc/gs.key`
将铭威fpv4win文件夹中的gs.key文件替换到`/etc`目录中

4. **安装并运行  [install_gs.sh](https://raw.githubusercontent.com/svpcom/wfb-ng/refs/heads/master/scripts/install_gs.sh):**
```bash
curl -o install_gs.sh https://raw.githubusercontent.com/svpcom/wfb-ng/refs/heads/master/scripts/install_gs.sh
# 将 wlx04d9f5115fdd 替换成你自己的网卡名字
sudo bash ./install_gs.sh wlx04d9f5115fdd
```

-   完成! 在地面站终端输入如下指令以监测链路

```bash
wfb-cli gs
```
若`RX：gs video`数据跳动，表明接收到图传数据

- 打开一个新的终端，运行如下指令，将接收到的数据包解码
```bash
gst-launch-1.0 -v udpsrc port=5600 ! \
"application/x-rtp, payload=96" ! \
rtph265depay ! \
avdec_h265 ! \
videoconvert ! \
autovideosink sync=false
```
