# NOTE
学习笔记





根据你的磁盘信息（`nvme0n1` 为系统盘，`sda` 为 465.8G 外置硬盘），以下是使用 dd 命令直接备份系统镜像到外置硬盘的完整指南，包含关键步骤、优化技巧和风险规避策略：

---

一、准备工作
1. 确认设备路径
• 系统盘：`/dev/nvme0n1`（总容量 119.2G）

• 外置硬盘：`/dev/sda`（已挂载到 `/media/adam/新加卷`，容量 465.8G）


2. 卸载外置硬盘（避免数据冲突）
```bash
sudo umount /dev/sda1  # 卸载已挂载的分区
```

3. 检查磁盘健康状态
```bash
sudo smartctl -a /dev/nvme0n1  # 检查系统盘健康度
sudo smartctl -a /dev/sda      # 检查外置硬盘健康度
```

---

二、备份操作
1. **全盘镜像备份（推荐场景）**
```bash
sudo dd if=/dev/nvme0n1 of=/dev/sda bs=4M status=progress && sync
```
• 参数解析：

  • `if=/dev/nvme0n1`：源设备为系统盘

  • `of=/dev/sda`：目标设备为外置硬盘（直接覆盖整个磁盘）

  • `bs=4M`：块大小设为 4MB（平衡速度与稳定性，可提升至 8M 加速）

  • `status=progress`：实时显示进度与速度（如 `345 MB/s`）

  • `sync`：确保数据完全写入磁盘


2. **镜像文件备份（保留外置硬盘原有数据）**
```bash
sudo dd if=/dev/nvme0n1 of=/media/adam/新加卷/system_backup.img bs=4M status=progress
```
• 适用场景：外置硬盘需保留其他文件时，生成独立镜像文件


3. **压缩备份（节省空间）**
```bash
sudo dd if=/dev/nvme0n1 bs=4M | gzip > /media/adam/新加卷/system_backup.img.gz
```
• 压缩率通常为 30-50%，但会增加 CPU 负载


---

三、关键验证步骤
1. **校验完整性**
```bash
# 计算源盘哈希
sudo sha256sum /dev/nvme0n1

# 计算目标镜像哈希（若保存为文件）
sha256sum /media/adam/新加卷/system_backup.img
```
• 两次输出必须一致，否则备份可能损坏


2. **查看备份文件信息**
```bash
fdisk -l /media/adam/新加卷/system_backup.img  # 检查镜像分区表是否正常
```

---

四、恢复操作
1. **从全盘镜像恢复**
```bash
sudo dd if=/dev/sda of=/dev/nvme0n1 bs=4M status=progress
```
• 警告：此操作会覆盖系统盘所有数据


2. **从镜像文件恢复**
```bash
sudo dd if=/media/adam/新加卷/system_backup.img of=/dev/nvme0n1 bs=4M status=progress
```

---

五、风险规避与优化
1. **重要注意事项**
• 数据安全：操作前确认外置硬盘无重要数据（`/dev/sda` 将被完全覆盖）

• 设备路径：通过 `lsblk` 反复确认设备名，避免误操作

• 电源稳定：备份过程中断可能导致镜像损坏


2. **性能优化**
• 块大小调优：测试不同 `bs` 值（如 `1M/4M/8M`）选择最佳速度

• 绕过缓存：添加 `oflag=direct` 避免缓存干扰速度测试

• 多线程监控：另开终端运行 `watch -n 5 killall -USR1 dd` 查看实时进度


---

六、替代方案对比
| 方案            | 优势                          | 局限性                     |
|-----------------|-------------------------------|---------------------------|
| dd全盘镜像  | 精确物理级备份，兼容性强      | 耗时较长，无增量备份        |
| Clonezilla  | 支持压缩、网络存储            | 需制作启动盘               |
| Timeshift   | 增量备份，节省空间            | 仅文件级备份               |

推荐选择：  
• 需快速恢复裸机系统时选 dd  

• 日常备份优先 Timeshift + dd 组合（每月全盘镜像 + 每日增量）


---

七、引用说明
• 基础操作参考自酷盾教程和亿速云指南

• 压缩与验证方法结合博客园实践

• 风险提示综合多个技术文档
































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


















原因分析​​
​​分支名称不匹配​​
你尝试推送main和master分支均失败，因为：
​​本地分支是master​​：通过git commit日志显示本地分支为master。
​​远程仓库可能无对应分支​​：若远程仓库是新建的且未初始化，默认可能没有main或master分支（需手动创建）。
​​子仓库嵌套导致警告​​
添加的目录ros_ws/包含其他Git仓库，导致Git误判为子模块。直接添加会引发警告，但不会阻止提交，可能影响后续操作。
​​远程仓库已存在但分支未初始化​​
首次推送时，若远程仓库为空且未创建默认分支（如main或master），直接推送会因“源引用规格不匹配”失败。
​​解决方案​​
步骤1：检查并确认本地分支
git branch  # 查看本地分支名称（应为master）
步骤2：重命名本地分支（可选）
若需与远程分支main匹配（如GitHub默认分支），重命名本地分支：

git branch -m master main  # 将本地分支master改为main
步骤3：处理子仓库嵌套问题
若ros_ws/是独立仓库，需移除缓存中的嵌套仓库路径：

git rm --cached ros_ws/src/pb2025_sentry_nav/my_nav_controller  # 移除误添加的子仓库路径
git add .  # 重新添加其他文件（不含子仓库）
git commit -m "fix: remove nested repositories"
步骤4：强制推送到远程并创建分支
若远程仓库为空，强制推送以初始化远程分支：

git push -u origin master --force  # 若本地分支为master
# 或
git push -u origin main --force     # 若已重命名为main
​​补充说明​​
​​分支名称规范​​
GitHub等平台默认分支已从master改为main，建议本地同步此设置。
可通过全局配置统一默认分支名称：
git config --global init.defaultBranch main  # 设置默认分支为main
​​子模块的正确用法​​
若需保留嵌套仓库，应使用git submodule add命令添加为子模块：
git submodule add <仓库URL> ros_ws/src/pb2025_sentry_nav/my_nav_controller
​​操作验证​​
​​查看远程分支​​：
git ls-remote --heads origin  # 确认远程分支是否创建成功
​​重新拉取代码​​：
git pull origin main  # 测试分支同步
通过以上步骤，分支名称冲突和远程仓库初始化问题均可解决。若仍遇到错误，可检查网络权限或SSH密钥配置

（注：此模板根据常见配置方案整理，具体参数需根据实际硬件环境调整）
