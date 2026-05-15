# HachimiDataUniverse
## Ubuntu24.04的安装、基本操作、环境配置
略 懒得写
## OpenIPC图传
### 硬件
- 天空端：淘宝购买的铭威2w天空端 **（2w不是功耗，是发射功率）**

- 地面站：任意采用`RTL8812AU`方案的网卡
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
- 运行如下指令，将网卡的监听设置为通道为64
```bash
sudo iw dev wlx04d9f5115fdd set channel 64
```
铭威图传的默认通道为 **64**
- 替换`/etc/gs.key`
将铭威fpv4win文件夹中的gs.key文件替换到`/etc`目录中

4. **安装并运行**
- 安装[install_gs.sh](https://raw.githubusercontent.com/svpcom/wfb-ng/refs/heads/master/scripts/install_gs.sh):
```bash
curl -o install_gs.sh https://raw.githubusercontent.com/svpcom/wfb-ng/refs/heads/master/scripts/install_gs.sh
# 将 wlx04d9f5115fdd 替换成你自己的网卡名字
sudo bash ./install_gs.sh wlx04d9f5115fdd
```

-   完成! 在地面站终端输入如下指令以启动监测链路

```bash
wfb-cli gs
```
若`RX：gs video`数据跳动，表明接收到图传数据

- 打开一个新的终端，运行如下指令，将接收到的数据包解码，将弹出视频窗口
```bash
gst-launch-1.0 -v udpsrc port=5600 ! \
"application/x-rtp, payload=96" ! \
rtph265depay ! \
avdec_h265 ! \
videoconvert ! \
autovideosink sync=false
```
 **连不上？**
 1.  输入`sudo tcpdump -i wlx04d9f5115fdd -n`查看网卡是否接收到数据
 若终端没有数据跳动，则说明网卡没有接收到天空端信号。
   - 输入`ethtool -i wlx04d9f5115fdd`检查驱动是否正确安装
   - 输入`iwconfig`检查网卡是否为`Monitor`监听模式
 
 2. 查看`gs.key`是否被正确替换？
 替换完成后务必输入如下指令重启wfb-ng服务，并设置通道:
 ```bash
#停止服务
sudo systemctl stop wifibroadcast@gs
#确保没有任何残留进程
sudo killall wfb_rx wfb_tx 2>/dev/null
#重新启动
sudo systemctl start wifibroadcast@gs
#设置通道
sudo iw dev wlx04d9f5115fdd set channel 64
```
 3. 检查通道是否为`64`
 ```bash
#设置通道
sudo iw dev wlx04d9f5115fdd set channel 64
```

#### 在代码中拉流
数据已经在本地 udp://127.0.0.1:5600 了，直接在 Python 代码里用 OpenCV 调用 GStreamer 管道即可。
```python
#未经验证的AI代码
import cv2

# 这是专为 NVIDIA 显卡优化的管道（因为你有 ROG SCAR 18）
# 它直接在显卡里完成：解包 -> 硬件解码 -> 转换为 RGB -> 传给 YOLO
gst_str = (
    "udpsrc port=5600 ! "
    "application/x-rtp, payload=96 ! "
    "rtph264depay ! "
    "nvh264dec ! "  # 使用 NVIDIA 硬件解码
    "videoconvert ! "
    "appsink"
)

cap = cv2.VideoCapture(gst_str, cv2.CAP_GSTREAMER)

while True:
    ret, frame = cap.read()
    if not ret:
        break

    # 在这里运行你的 YOLO 模型
    # results = model(frame)
    
    cv2.imshow('WFB-ng YOLO Stream', frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```
