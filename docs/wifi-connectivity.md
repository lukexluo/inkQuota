# WiFi 连接与数据获取方案

> 本文档详细说明 inkQuota 设备的 WiFi 连接方式、数据流架构和离线降级策略。

---

## 1. 核心设计：WiFi 是必备能力

ESP32-S3 原生支持 **WiFi 6 (802.11ax)** 和 **蓝牙 5.0**，本项目完全基于 WiFi 连接实现：

```
┌──────────────┐      WiFi      ┌──────────────────┐      HTTPS      ┌──────────────────┐
│  inkQuota    │  ───────────→  │  家庭路由器/网络  │  ───────────→  │  Coding Plan API │
│  桌面摆件     │               │                  │               │  阿里云/智谱/火山  │
│  (ESP32-S3)  │  ←───────────  │                  │  ←───────────  │                  │
└──────────────┘                └──────────────────┘                └──────────────────┘
```

**设备无 WiFi 时 = 高级电子时钟（显示最后一次数据 + 白色 LED 提示离线）**

---

## 2. WiFi 配置方式（三选一）

### 方案 A：硬编码（最简单，开发调试用）

在 `config.py` 中直接写死 WiFi 密码：

```python
WIFI_SSID = "YourWiFi"
WIFI_PASSWORD = "YourPassword"
```

- ✅ 开机自动连接，无需手动操作
- ❌ 换 WiFi 需要重新烧录代码
- 📌 适合：固定位置、不换路由器的场景

### 方案 B：Web 配网（推荐，最终部署用）

首次开机时，ESP32 创建一个热点 `inkQuota-Setup`，手机连接后自动弹出配置页面：

```
手机连接 inkQuota-Setup → 浏览器访问 192.168.4.1
→ 选择你的 WiFi → 输入密码 → 保存 → 自动重启连接
```

- ✅ 无需烧录代码，手机配置即可
- ✅ 支持多个 WiFi 配置（优先信号强的）
- ❌ 需要额外写一套 Web 服务器代码（约 200 行 MicroPython）
- 📌 适合：成品交付、可能需要换 WiFi 的场景

### 方案 C：SmartConfig（微信/手机一键配网）

使用乐鑫的 SmartConfig 协议，通过 App 或微信小程序广播 WiFi 信息：

```python
import smartconfig
smartconfig.start()
# 等待手机 App 发送 WiFi 配置
```

- ✅ 一键配网，体验最好
- ❌ 需要下载 App（ESP-TOUCH）或写小程序
- 📌 适合：有技术能力实现 App 配套的场景

**最终建议：开发期用方案 A，成品用方案 B（Web 配网）。**

---

## 3. 数据流架构

### 3.1 模式一：设备直连（Standalone）

```
inkQuota (ESP32-S3)
    │ WiFi
    ▼
各平台 Coding Plan API
    ├─ 阿里云百炼 API (coding.dashscope.aliyuncs.com)
    ├─ 智谱 GLM API (open.bigmodel.cn)
    ├─ 火山方舟 API (ark.volces.com)
    ├─ Kimi API (api.moonshot.cn)
    └─ 腾讯云 TI API (ti.tencentcloudapi.com)
```

**特点**：设备直接发起 HTTPS 请求，最简洁。
**风险**：API Key 存储在设备上，设备被读取会泄露 Key。

### 3.2 模式二：网关代理（Gateway，推荐）

```
inkQuota (ESP32-S3)
    │ WiFi
    ▼
家庭网关 (Raspberry Pi / NAS / 软路由 / 旧电脑)
    │ 本地局域网 HTTP
    ▼
网关服务 (Python Flask/FastAPI)
    │ HTTPS
    ▼
各平台 Coding Plan API
```

**特点**：
- API Key 藏在网关服务器，设备只存网关地址（如 `http://192.168.1.100:5000/quota`）
- 网关统一处理认证、滑动窗口计算、缓存
- 支持聚合多个平台，设备只拉一个接口
- 可扩展：手机 App 查看、历史趋势分析

**风险**：需要一台 24h 运行的网关设备（树莓派 Zero 2W 功耗仅 1W，约 ¥150）。

---

## 4. 无 WiFi / 离线降级策略

| 状态 | 墨水屏显示 | RGB 灯 | 行为 |
|------|-----------|--------|------|
| **WiFi 正常** | 实时数据 + 倒计时 | 绿/黄/红 | 正常刷新 |
| **WiFi 断开** | 保留最后数据 + 离线时间戳 | ⚪ 白色 | 停止刷新，每 5 分钟尝试重连 |
| **API 超时** | 保留数据 + "数据 stale" 提示 | 🟡 黄色快闪 | 使用缓存数据，标记过期 |
| **认证失败** | "API Key 错误" + 最后数据 | 🔴 红色快闪 | 停止请求，等待人工修复 |

**关键逻辑**：

```python
import network
import time

wlan = network.WLAN(network.STA_IF)

def ensure_wifi():
    if not wlan.isconnected():
        wlan.connect(WIFI_SSID, WIFI_PASSWORD)
        for _ in range(30):  # 等待 30 秒
            if wlan.isconnected():
                return True
            time.sleep(1)
        return False
    return True

def main_loop():
    if not ensure_wifi():
        show_offline()      # 显示离线状态
        set_led_white()     # 白色 LED
        deep_sleep(300)     # 睡 5 分钟再试
        return
    
    data = fetch_quota()    # 拉取数据
    render_display(data)   # 刷新屏幕
    set_led_by_status(data) # 更新 RGB 灯
    deep_sleep(300)        # 睡 5 分钟
```

---

## 5. 需要买的 WiFi 相关硬件

实际上 **不需要额外买任何东西** —— ESP32-S3 已经内置 WiFi。只要确保你的 2.4GHz WiFi 网络可用即可（ESP32 不支持 5GHz 频段）。

如果你要做**网关模式**，需要额外设备：

| 网关设备 | 淘宝搜索关键词 | 价格 | 功耗 | 推荐度 |
|---------|-------------|------|------|--------|
| 树莓派 Zero 2W | `树莓派 Zero 2W` | ¥150-180 | 1W | ⭐⭐⭐⭐⭐ |
| 旧手机 Termux | 无需购买 | 0 | 5W | ⭐⭐⭐⭐ |
| 软路由 / NAS | 已有设备 | 0 | 10W+ | ⭐⭐⭐ |
| 旧电脑 / 笔记本 | 已有设备 | 0 | 20W+ | ⭐⭐ |

---

## 6. 下一步：WiFi 配置代码

等我买到硬件后，我将提供：

1. `wifi_manager.py` — WiFi 连接 + 断线重连 + Web 配网服务器
2. `config_portal.html` — 手机端配网页面（内嵌到 MicroPython）
3. `quota_fetcher.py` — 直连模式下的 API 拉取
4. `gateway_client.py` — 网关模式下的数据获取

这些代码将在硬件到货后随 Phase 1 一起提供。

---

> 返回 [README.md](../README.md) 查看整体架构。
