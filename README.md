# NOTE
学习笔记


```markdown
# Ubuntu 虚拟显示器配置指南（无物理显示器）

## 目录
1. [场景需求](#场景需求)
2. [配置步骤](#配置步骤)
   - [安装虚拟显示器驱动](#1-安装虚拟显示器驱动)
   - [创建配置文件](#2-创建配置文件)
   - [重启Xorg服务](#3-重启xorg服务)
   - [分辨率设置](#4-分辨率设置)
3. [远程控制方案](#远程控制方案)
4. [注意事项](#注意事项)
5. [问题排查](#问题排查)

---

## 场景需求
通过虚拟显示器实现以下功能：
- 无物理显示器接入时正常使用图形界面
- 支持远程桌面控制（VNC/XRDP）
- 解决服务器无显卡导致的显示异常

---

## 配置步骤

### 1. 安装虚拟显示器驱动
```bash
sudo apt install xserver-xorg-video-dummy
```

### 2. 创建配置文件
新建 `/usr/share/X11/xorg.conf.d/xorg.conf` 文件：
```conf
Section "Device"
    Identifier  "DummyDevice"
    Driver      "dummy"
    VideoRam    256000
EndSection

Section "Monitor"
    Identifier  "DummyMonitor"
    HorizSync   30.0-70.0
    VertRefresh 50.0-75.0
    Modeline "1920x1080" 148.50 1920 2008 2052 2200 1080 1084 1089 1125
EndSection

Section "Screen"
    Identifier  "DummyScreen"
    Device      "DummyDevice"
    Monitor     "DummyMonitor"
    DefaultDepth 24
    SubSection "Display"
        Depth   24
        Modes   "1920x1080"
    EndSubSection
EndSection
```

### 3. 重启Xorg服务
```bash
sudo systemctl restart display-manager
```

### 4. 分辨率设置
生成自定义分辨率：
```bash
gtf 1920 1080 60
# 将输出内容填入Modeline字段
xrandr --newmode "1920x1080_60" 172.80 1920 2040 2248 2576 1080 1081 1084 1118 -HSync +Vsync
xrandr --addmode Virtual1 1920x1080_60
```

---

## 远程控制方案
推荐工具：
- **VNC Server** 
- **XRDP** 
- **NoMachine**

配置示例（VNC）：
```bash
sudo apt install tightvncserver
vncserver :1 -geometry 1920x1080
```

---

## 注意事项
1. 必须使用 `nouveau` 开源显卡驱动
2. 配置文件需命名为 `xx-nvidia.conf` 格式（xx为数字）
3. 推荐分辨率组合：
   ```conf
   3840x2160 | 1920x1080 | 1280x720
   ```
4. 若出现黑屏，通过 `Ctrl+Alt+F2` 进入TTY终止Xorg进程：
   ```bash
   sudo pkill Xorg
   ```

---

## 问题排查
1. **分辨率不生效**  
   - 检查Modeline参数计算准确性
   - 验证xrandr命令是否成功执行

2. **服务启动失败**  
   - 查看日志：`journalctl -u display-manager`
   - 检查配置文件权限：`chmod 644 xorg.conf`

3. **远程连接模糊**  
   - 调整VNC客户端色彩设置为True Color
   - 增加视频内存分配值（VideoRam）
``` 

（注：此模板根据常见配置方案整理，具体参数需根据实际硬件环境调整）
