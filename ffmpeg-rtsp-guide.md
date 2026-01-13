# FFmpeg 7.1 RTSP 抓拍服务配置指南

## 概述

本指南适用于树莓派上安装的 **FFmpeg 7.1** 版本，提供了完全适配的 RTSP 流抓拍脚本解决方案。

---

## 1. 准备内存盘（只需运行一次）

如果还没挂载内存盘，请执行以下命令：

```bash
sudo mkdir -p /tmp/camera
sudo mount -t tmpfs -o size=20M tmpfs /tmp/camera
```

---

## 2. 完整脚本代码

创建脚本文件：

```bash
nano ~/start-rtsp.sh
```

删除旧内容，粘贴以下代码：

```bash
#!/bin/bash

# --- 配置区 ---
RTSP_URL="rtsp://admin:Ll568568@192.168.1.191:554/Streaming/Channels/101"
# 图片保存到内存盘，文件名 latest.jpg
OUTPUT_PATH="/tmp/camera/latest.jpg"

echo "FFmpeg 7.1 抓拍服务启动中..."

while true
do
    # 针对新版本 FFmpeg 的参数组合
    # -timeout 5000000: 5秒连接超时（单位为微秒）
    # -an: 禁用音频
    # -vf "fps=1/5": 每5秒一帧
    # -atomic_writing 1: 解决进程读取冲突
    ffmpeg -hide_banner -loglevel info \
        -rtsp_transport tcp \
        -timeout 5000000 \
        -i "$RTSP_URL" \
        -vf "fps=1/5" \
        -an \
        -update 1 \
        -atomic_writing 1 \
        -y "$OUTPUT_PATH"

    # 如果 ffmpeg 退出，显示日志并等待 5 秒重连
    echo "检测到连接断开或配置错误，5秒后自动重连..."
    sleep 5
done
```

---

## 3. 设置权限并启动

### 3.1 赋予执行权限

```bash
chmod +x ~/start-rtsp.sh
```

### 3.2 在后台运行

```bash
# 启动服务（即使关闭窗口也不停止）
nohup ~/start-rtsp.sh > rtsp_log.txt 2>&1 &

# 检查是否正在生成图片
ls -l /tmp/camera/latest.jpg
```

---

## 4. 进阶排错与说明

### 4.1 关于 `Unrecognized option` 错误

如果运行脚本依然提示 `timeout` 找不到，那是因为在某些 FFmpeg 编译版本中，这个参数必须放在 `-i` 之前。本脚本已将其放在正确位置。

### 4.2 查看实时日志

```bash
tail -f rtsp_log.txt
```

### 4.3 Home Assistant 调用

在 HA 配置文件中，可以直接引用：
```
/tmp/camera/latest.jpg
```

### 4.4 停止脚本

如果想彻底停止后台进程：

```bash
pkill -f start-rtsp.sh && pkill ffmpeg
```

---

## 5. 关键参数说明

| 参数 | 说明 |
|------|------|
| `-rtsp_transport tcp` | 使用 TCP 传输协议 |
| `-timeout 5000000` | 连接超时设置为 5 秒（微秒单位） |
| `-vf "fps=1/5"` | 每 5 秒提取一帧画面 |
| `-an` | 禁用音频流 |
| `-update 1` | 覆盖更新同一文件 |
| `-atomic_writing 1` | 原子化写入，避免读取冲突 |
| `-y` | 自动覆盖已存在的文件 |

---

## 6. 故障排除

### 问题：图片不更新

**解决方案**：
1. 检查 RTSP 连接是否正常
2. 查看日志：`tail -f rtsp_log.txt`
3. 确认 FFmpeg 进程是否运行：`ps aux | grep ffmpeg`

### 问题：权限错误

**解决方案**：
```bash
sudo chmod 755 /tmp/camera
sudo chown $USER:$USER /tmp/camera
```

### 问题：内存盘空间不足

**解决方案**：
```bash
# 卸载后重新挂载，调整大小
sudo umount /tmp/camera
sudo mount -t tmpfs -o size=50M tmpfs /tmp/camera
```

---

## 7. 开机自启动（可选）

如需开机自动运行，可创建 systemd 服务：

```bash
sudo nano /etc/systemd/system/rtsp-capture.service
```

添加以下内容：

```ini
[Unit]
Description=RTSP Camera Capture Service
After=network.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/start-rtsp.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable rtsp-capture.service
sudo systemctl start rtsp-capture.service
```

---

## 注意事项

1. FFmpeg 7.1 对参数要求更严格，参数顺序很重要
2. 内存盘重启后数据会丢失，需重新挂载或通过 systemd 自动挂载
3. 修改 RTSP_URL 变量以适配你的摄像头配置
4. 根据网络状况调整 `-timeout` 参数值

---

**文档创建时间：** 2026-01-13
