# 樹莓派4 RTMP 推流伺服器

使用樹莓派 4 取代原有硬體推流模組（Encoder），從 SangLuWoo V3-365C-AR IP 攝影機接收 RTSP 串流，透過 FFmpeg 推送至私有化 RTMP 伺服器，供 T100 機庫系統使用。

## 網路架構

```
SangLuWoo V3-365C-AR
  IP：192.168.8.5
  RTSP：rtsp://admin:123456@192.168.8.5:554/h264/ch1/main/av_stream
  規格：H.264 1920x1080@25fps + AAC 16000Hz mono
        │
        │ RTSP over TCP
        ▼
樹莓派 4 (eth0: 192.168.8.12)
  FFmpeg：RTSP → RTMP copy 模式（不轉碼）
  nginx-rtmp：本機 HLS 輸出（備用播放）
        │
        │ RTMP
        ▼
私有化伺服器 (192.168.8.8)
  Docker：7yhong/nginx-rtmp
  推流地址：rtmp://192.168.8.8:1935/app/oBYS2OomLmSXZ6QUbqRq__dock
  觀看地址：rtmp://192.168.8.8:1935/app/oBYS2OomLmSXZ6QUbqRq__dock
```

## 關鍵設定

| 項目 | 值 |
|------|-----|
| 攝影機 IP | 192.168.8.5 |
| 攝影機 RTSP 路徑 | `/h264/ch1/main/av_stream`（HiSilicon 格式） |
| 攝影機帳密 | admin / 123456 |
| 樹莓派 IP (eth0) | 192.168.8.12 |
| RTMP 伺服器 IP | 192.168.8.8:1935 |
| RTMP Application | `app`（非 `live`，為 7yhong/nginx-rtmp 預設） |
| 串流金鑰 | `oBYS2OomLmSXZ6QUbqRq__dock` |
| T100 機庫 SN | `oBYS2OomLmSXZ6QUbqRq` |
| T100 MQTT Broker | 192.168.8.8:1883 |

> **注意**：攝影機（192.168.8.5）有防火牆限制，只允許同子網路（192.168.8.x）的設備存取，Mac 無法直接連線。

## 目錄結構

```
Rasp-RTMP-Server/
├── config/
│   ├── config.env          # 主設定檔（INPUT_SOURCE、RTMP_OUTPUT 等）
│   └── nginx.conf          # 本機 nginx-rtmp 設定（HLS 備用輸出）
├── scripts/
│   ├── setup.sh            # 一鍵安裝（nginx、FFmpeg、systemd 服務）
│   ├── start_server.sh     # 快速啟動 / 停止 / 狀態
│   ├── stream_relay.sh     # FFmpeg 串流中繼主腳本
│   ├── diagnose.sh         # 診斷工具（網路、RTSP、防火牆、HLS）
│   └── stream_monitor.sh   # 串流監控介面
├── dock/
│   ├── dock_probe.py       # T100 機庫 MQTT 控制（命令列）
│   ├── dock_control.py     # T100 機庫控制核心模組
│   ├── dock_ui.py          # T100 機庫控制 GUI（PySide6）
│   └── requirements.txt    # Python 相依套件
├── systemd/
│   ├── rtmp-server.service # nginx RTMP systemd 服務
│   └── stream-relay.service# 串流中繼 systemd 服務
└── docs/
    └── setup-guide.md      # 詳細安裝與排解指南
```

## 快速開始

### 1. 安裝

```bash
git clone https://github.com/shian-chi/rasp-rtmp-server.git
cd rasp-rtmp-server
sudo ./scripts/setup.sh
```

### 2. 啟動串流

```bash
# 方法一：tmux 背景執行（斷線不中斷）
sudo systemctl start nginx
tmux new-session -d -s stream 'cd ~/Rasp-RTMP-Server && ./scripts/stream_relay.sh'
tmux attach -t stream   # 查看輸出
# Ctrl+B 再按 D 離開（串流繼續跑）

# 方法二：systemd 服務（需先執行 setup.sh）
sudo systemctl start nginx
sudo systemctl start stream-relay

# 方法三：一鍵啟動腳本
sudo ./scripts/start_server.sh
```

### 3. 確認串流狀態

```bash
pgrep -a ffmpeg                    # 確認 FFmpeg 在跑
./scripts/diagnose.sh              # 完整診斷
./scripts/diagnose.sh --rtsp       # 測試 RTSP 連線
tail -f /var/log/rtmp-stream/stream.log  # 查看日誌
```

## Mac 播放方式

RTMP 伺服器在 192.168.8.8 時用 VLC：
```
rtmp://192.168.8.8:1935/app/oBYS2OomLmSXZ6QUbqRq__dock
```

本機 nginx HLS 備用（Pi 推到 localhost 時）：
```
http://192.168.8.12:8080/hls/oBYS2OomLmSXZ6QUbqRq__dock.m3u8
```

## T100 機庫 Dock 控制

```bash
cd dock
pip3 install -r requirements.txt

# 開蓋
python3 dock_probe.py open --wait 30 --after-send-wait 12

# 關蓋
python3 dock_probe.py close --wait 30 --after-send-wait 12
```

MQTT Broker：192.168.8.8:1883

## 私有化伺服器（192.168.8.8）Docker 服務

```bash
# RTMP 伺服器（7yhong/nginx-rtmp）
sudo docker run -d --name rtmp -p 1935:1935 --restart=always 7yhong/nginx-rtmp

# MQTT Broker（eclipse-mosquitto）
sudo docker run -d -it -p 1883:1883 --name mqtt --restart=always eclipse-mosquitto

# 查看容器狀態
sudo docker container ls
```

## 已知問題與排解

| 問題 | 原因 | 解決方式 |
|------|------|---------|
| RTSP Connection refused | 攝影機 RTSP 服務未啟用 | 登入 `http://攝影機IP:80` 啟用 RTSP；或執行 `./scripts/diagnose.sh --rtsp` 掃描 |
| Mac 無法 ping / 連攝影機 | 攝影機防火牆僅允許同子網路 | 從 Pi（192.168.8.12）操作，不從 Mac |
| RTMP Input/output error | Application 名稱錯誤 | 確認 RTMP URL 的 app name：7yhong/nginx-rtmp 用 `app`，Pi 本機 nginx 用 `live` |
| HLS 無法播放（Mac VLC） | UFW 封鎖 8080 埠 | `sudo ufw allow 8080/tcp` |
| Non-monotonous DTS 警告 | RTSP 時間戳問題 | 已在 stream_relay.sh 加入 `-use_wallclock_as_timestamps 1` |
| stream-relay.service not found | systemd 服務未安裝 | `sudo ./scripts/setup.sh` 或手動安裝服務 |
