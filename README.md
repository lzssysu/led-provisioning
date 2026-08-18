[README.md](https://github.com/user-attachments/files/31164587/README.md)
# BLE WiFi 配网方案 (rpi-bt-provisioning)

通过蓝牙 BLE 传输 WiFi 凭据，无需网络通道，无需下载 App。

## 方案特点

- **免 App**: 手机 Chrome 浏览器直接配网（Web Bluetooth API）
- **免网络**: 蓝牙传输，不需要 WiFi 或移动数据
- **不占用 wlan0**: BLE 独立于 WiFi，无 AP/STA 驱动冲突
- **全平台兼容**: Android Chrome / Edge 均可使用
- **开机自启**: 有保存的 WiFi 自动连接，无 WiFi 自动进入配网模式

## 工作原理

```
手机 Chrome 打开配网页页
        │
        ├── Web Bluetooth API 连接树莓派 BLE
        │
        ├── 写入 WiFi 名称 (ff01)
        ├── 写入 WiFi 密码 (ff02)
        ├── 写入触发指令 (ff04)
        │
        └── 轮询读取状态 (ff03)
              ├── 02 = 连接成功
              ├── 03 = 连接失败
              └── 04 = 密码错误
```

## 文件结构

```
rpi-bt-provisioning/
├── install.sh                          # 一键安装脚本
├── README.md                           # 本文档
├── scripts/
│   ├── ble_provisioning_server.py      # BLE GATT 服务器 (核心)
│   ├── ble_switch.sh                   # BLE/STA 模式切换
│   ├── boot_logic.sh                   # 开机自启逻辑
│   ├── ble-provisioning.service        # BLE 配网 systemd 服务
│   └── ble-provisioning-boot.service   # 开机逻辑 systemd 服务
└── web/
    ├── index.html                      # 配网页面 (部署到 GitHub Pages)
    └── qr-generator.html               # 二维码生成工具
```

---

## 部署步骤（全新树莓派）

### 前提条件

- 树莓派（带蓝牙的型号：Pi 3B/3B+/4/Zero W/Zero 2W）
- 已烧录 Raspberry Pi OS 并能 SSH 连接
- SD 卡已创建 `ssh` 文件启用 SSH

### 第一步：上传项目到树莓派

**Windows (PowerShell):**

```powershell
scp -r rpi-bt-provisioning pi@<树莓派IP>:~/
```

**Mac/Linux:**

```bash
scp -r rpi-bt-provisioning pi@<树莓派IP>:~/
```

### 第二步：运行安装脚本

```bash
cd ~/rpi-bt-provisioning
sudo bash install.sh
```

安装脚本会自动完成：
1. 检测网络连通性
2. 安装依赖包（bluez, python3-dbus, python3-gi 等）
3. 停止旧服务（WiFi AP / 蓝牙 PAN）
4. 复制文件到 `/opt/bt-provisioning/`
5. 配置蓝牙服务
6. 安装 systemd 开机自启服务
7. 配置 DNS 和 IPv4

### 第三步：重启

```bash
sudo reboot
```

重启后树莓派自动运行开机逻辑：
- 有已保存的 WiFi → 自动连接
- 无保存的 WiFi → 进入 BLE 配网模式

### 第四步：部署配网页面（一次性）

配网页面需要 HTTPS 托管（Web Bluetooth API 要求）。使用 GitHub Pages 免费永久托管：

1. 注册 GitHub 账号（https://github.com）
2. 新建仓库，名称为 `led-provisioning`，选 Public
3. 上传 `web/index.html` 到仓库
4. Settings → Pages → Source 选 `main` branch → Save
5. 等待 1 分钟，得到网址：`https://<用户名>.github.io/led-provisioning/`
6. 用电脑浏览器打开 `web/qr-generator.html`，生成二维码
7. 打印二维码贴在设备上

### 第五步：验证安装

SSH 连接树莓派，检查：

```bash
# 检查 BLE 服务状态
systemctl status ble-provisioning-boot

# 查看开机日志
cat /tmp/boot_logic.log

# 检查 BLE 服务器是否运行
ps aux | grep ble_provisioning

# 查看配网日志
tail -20 /tmp/ble_provisioning.log
```

---

## 配网操作流程

### 方式一：Web Bluetooth（推荐，免 App）

1. 手机用 Chrome 扫描设备上的二维码
2. 打开配网页面，点击「连接蓝牙设备」
3. 选择 `LED_SCREEN_0682`
4. 输入 WiFi 名称和密码
5. 点击「开始连接」，等待结果
6. Chrome 菜单 →「添加到主屏幕」，下次直接点图标

**要求：**
- Android 手机
- Chrome 或 Edge 浏览器
- 手机蓝牙开启

### 方式二：nRF Connect App（备选）

1. 下载 **nRF Connect** (Android) 或 **LightBlue** (iOS)
2. 扫描并连接 `LED_SCREEN_0682`
3. 找到 Service `0000ff00-0000-1000-8000-00805f9b34fb`
4. 写 SSID → `0000ff01` (UTF-8 文本)
5. 写密码 → `0000ff02` (UTF-8 文本)
6. 写 `01` → `0000ff04` (触发连接)
7. 读 `0000ff03` 查状态（02=成功, 03=失败, 04=密码错误）

---

## GATT 服务定义

**Service UUID:** `0000ff00-0000-1000-8000-00805f9b34fb`

| 特征 | UUID 后缀 | 属性 | 说明 |
|------|-----------|------|------|
| SSID | `ff01` | Write | WiFi 名称 (UTF-8) |
| Password | `ff02` | Write | WiFi 密码 (UTF-8) |
| Status | `ff03` | Read + Notify | 连接状态 (uint8) |
| Trigger | `ff04` | Write | 写入任意值触发连接 |
| DeviceInfo | `ff05` | Read | 设备信息 |

### Status 值

| 值 | 状态 |
|----|------|
| 0 | IDLE 空闲 |
| 1 | CONNECTING 连接中 |
| 2 | CONNECTED 连接成功 |
| 3 | FAILED 连接失败 |
| 4 | WRONG_PASSWORD 密码错误 |

---

## 开机逻辑

```
开机
  │
  ├── 检查 wpa_supplicant.conf 有无保存的 WiFi
  │     │
  │     ├── 无 → 进入 BLE 配网模式
  │     │
  │     └── 有 → 等待 NetworkManager 就绪
  │               │
  │               ├── 已自动连上 → 正常运行
  │               │
  │               └── 尝试连接 (最多3次)
  │                     ├── 成功 → 正常运行
  │                     └── 失败 → 进入 BLE 配网模式
  │
  └── BLE 配网模式 → 等待手机连蓝牙配网
```

---

## 手动操作命令

```bash
# 进入 BLE 配网模式
sudo /opt/bt-provisioning/scripts/ble_switch.sh to_ble

# 切换到 WiFi 连接模式
sudo /opt/bt-provisioning/scripts/ble_switch.sh to_sta

# 查看当前模式
sudo /opt/bt-provisioning/scripts/ble_switch.sh status

# 查看配网日志
tail -f /tmp/ble_provisioning.log

# 查看切换日志
tail -f /tmp/ble_switch.log

# 查看开机日志
cat /tmp/boot_logic.log

# 清除已保存的 WiFi（重新配网）
sudo nmcli device disconnect wlan0
sudo nmcli connection delete provisioning
sudo rm /etc/wpa_supplicant/wpa_supplicant.conf
sudo /opt/bt-provisioning/scripts/ble_switch.sh to_ble

# 清除蓝牙配对记录
for dev in $(bluetoothctl devices | awk '{print $2}'); do bluetoothctl remove $dev; done
sudo systemctl restart bluetooth
```

---

## 故障排查

### BLE 服务无法启动

```bash
# 检查蓝牙服务
systemctl status bluetooth

# 检查蓝牙适配器
bluetoothctl show

# 如果蓝牙被阻止
sudo rfkill unblock bluetooth
sudo systemctl restart bluetooth

# 重启 BLE 服务
sudo killall -9 ble_provisioning_server.py
sudo /opt/bt-provisioning/scripts/ble_switch.sh to_ble
```

### 手机搜不到设备

```bash
# 检查 BLE 服务器是否运行
ps aux | grep ble_provisioning

# 检查蓝牙适配器状态
bluetoothctl show

# 确认可发现
bluetoothctl discoverable on

# 清除旧配对记录
for dev in $(bluetoothctl devices | awk '{print $2}'); do bluetoothctl remove $dev; done
sudo systemctl restart bluetooth
sudo /opt/bt-provisioning/scripts/ble_switch.sh to_ble
```

### WiFi 连接失败

```bash
# 查看配网日志
tail -50 /tmp/ble_provisioning.log

# 手动测试 WiFi 连接
nmcli connection add type wifi ifname wlan0 con-name test ssid "WiFi名称" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "WiFi密码"
nmcli connection up test ifname wlan0

# 检查 WiFi 接口
ip addr show wlan0
iwgetid -r
```

### 开机不自动连接 WiFi

```bash
# 查看开机日志
cat /tmp/boot_logic.log

# 检查 wpa_supplicant.conf
cat /etc/wpa_supplicant/wpa_supplicant.conf

# 检查 NetworkManager 连接
nmcli connection show

# 手动测试开机逻辑
sudo /opt/bt-provisioning/scripts/boot_logic.sh
```

### Web Bluetooth 页面无法打开

- 确认使用 **Chrome** 或 **Edge** 浏览器（Firefox 不支持）
- 确认是 **Android** 手机（iOS Safari 不支持 Web Bluetooth）
- 确认手机蓝牙已开启
- 确认网页是 **HTTPS** 地址（GitHub Pages 自动提供）
- 清除浏览器缓存后重试

---

## 技术规格

| 项目 | 规格 |
|------|------|
| 蓝牙名称 | LED_SCREEN_0682 |
| 配网服务 UUID | 0000ff00-0000-1000-8000-00805f9b34fb |
| 部署路径 | /opt/bt-provisioning/ |
| 配网页面 | https://lzssysu.github.io/led-provisioning/ |
| BLE 日志 | /tmp/ble_provisioning.log |
| 切换日志 | /tmp/ble_switch.log |
| 开机日志 | /tmp/boot_logic.log |
| WiFi 连接超时 | 25 秒 |
| 开机重试次数 | 3 次 |
| 配对方式 | Just Works (NoInputNoOutput) |

---

## 与 WiFi AP 方案对比

| 特性 | WiFi AP 方案 | BLE GATT 方案 |
|------|-------------|--------------|
| 配网方式 | 连 WiFi 热点，浏览器访问 | 连蓝牙，网页/App 写入 |
| 占用 wlan0 | 是 | 否 |
| 驱动状态风险 | 有 (AP/STA 切换) | 无 |
| 需要 hostapd/dnsmasq | 是 | 否 |
| 需要 C 编译 | 是 | 否 (纯 Python) |
| iOS 支持 | 是 (Captive Portal) | 需 App (LightBlue) |
| Android 支持 | 是 | 是 (Chrome/Edge) |

---

## 卸载

```bash
# 停止并禁用服务
sudo systemctl stop ble-provisioning ble-provisioning-boot
sudo systemctl disable ble-provisioning ble-provisioning-boot

# 删除服务文件
sudo rm /etc/systemd/system/ble-provisioning.service
sudo rm /etc/systemd/system/ble-provisioning-boot.service
sudo systemctl daemon-reload

# 删除项目文件
sudo rm -rf /opt/bt-provisioning/

# 恢复蓝牙默认配置
sudo apt-get install --reinstall bluez
```
