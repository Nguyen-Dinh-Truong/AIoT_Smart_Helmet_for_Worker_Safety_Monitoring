# Mũ bảo hộ thông minh giám sát an toàn lao động

Hệ thống mũ bảo hộ thông minh giám sát an toàn công trình giải quyết đồng thời 4 bài toán: độ trễ, độ chính xác định vị, tầm phủ và chi phí — bằng cách kết hợp TinyML phát hiện ngã trên thiết bị, UWB định vị cm và mạng LoRa relay đa tầng.

---

## Tổng quan kiến trúc

Hệ thống gồm 4 tầng:

```
[Smart Helmet Tag] → LoRa (433 MHz) → [Relay Station] → LoRa (433.5 MHz) → [Gateway RPi] → MQTT → [Cloud / Dashboard]
                          ↕ UWB TWR
                   [UWB Anchor Nodes]
```

| Tầng | Thành phần | Chức năng |
|---|---|---|
| Perception | ESP32-S3 (Helmet Tag) | Thu cảm biến, inference 1D-CNN, UWB ranging |
| Relay | ESP32 (Anchor Station) | Chuyển tiếp LoRa f1 → f2, UWB anchor |
| Network | Raspberry Pi (Gateway) | Nhận LoRa, đẩy MQTT lên cloud |
| Application | HiveMQ Cloud + Dashboard | Lưu trữ, hiển thị, cảnh báo |

---

## Cấu trúc thư mục

```
├── Full_System_DualCore.ino       # Firmware ESP32-S3 (Smart Helmet Tag)
├── LoRa_Anchor_dual_RX_TX.ino    # Firmware ESP32 (Relay Station)
├── lora_gateway.py                # Gateway Raspberry Pi (phiên bản cải tiến, qua relay)
├── lora_gateway_direct_from_tag.py # Gateway nhận trực tiếp từ mũ (phiên bản đầu)
└── Training_CNN_1D_Augment.ipynb  # Training notebook cho mô hình fall detection
```

---

## Smart Helmet Tag — `Full_System_DualCore.ino`

Chạy trên **ESP32-S3**, chia 2 core:

- **Core 0**: Fall detection — đọc MPU9250 (IMU 6 trục) @ 25 Hz, chạy inference TFLite Micro (1D-CNN 14 KB, INT8)
- **Core 1**: Thu thập cảm biến + truyền LoRa mỗi 2 giây

**Cảm biến tích hợp:**
- MPU9250 — IMU 6 trục (accel + gyro)
- MAX30105 — nhịp tim & SpO2
- MAX30205 — nhiệt độ cơ thể
- INA219 — điện áp & dòng pin
- GPS (UART) — tọa độ ngoài trời
- UWB (UART) — khoảng cách đến anchor

**Packet LoRa (74 bytes, packed struct):**

```
mac(8) | bodyTemp(4) | busVoltage(4) | current_mA(4) | batteryPercent(4)
latitude(8) | longitude(8) | heartRate(4) | spo2(4)
uwb_A0..A3(16) | uwb_Tag3(4) | uwb_Tag4(4)
fallDetected(1) | helpRequest(1)
```

**Logic cảnh báo cục bộ (không phụ thuộc cloud):**
- Fall → còi rung 250ms/200ms
- UWB Tag3/Tag4 < 3 m → còi proximity
- UWB < 2 m → còi liên tục
- Nút HELP giữ < 2 s → bật help request, giữ ≥ 2 s → tắt

---

## Relay Station — `LoRa_Anchor_dual_RX_TX.ino`

Chạy trên **ESP32**, 2 module LoRa RA-02:

| | RX Module | TX Module |
|---|---|---|
| Tần số | 433.0 MHz | 433.5 MHz |
| SyncWord | 0x12 | 0x56 |
| SF / BW | SF9 / 125 kHz | SF9 / 125 kHz |
| Vai trò | Nhận từ Tag | Gửi về Gateway |

FreeRTOS dual-core: Core 0 nhận, Core 1 gửi qua queue. Packet được forward nguyên vẹn không thay đổi.

---

## Gateway Raspberry Pi

### `lora_gateway_direct_from_tag.py` — Phiên bản đầu

Gateway nhận trực tiếp từ mũ bảo hộ (Tag): **433.0 MHz, SyncWord 0x12**, không qua relay. Kiến trúc đơn giản nhưng bộc lộ một số hạn chế khi triển khai thực tế — tầm phủ sóng bị giới hạn bởi công suất Tag, không xử lý packet trùng lặp khi nhiều tag phát đồng thời, và gateway phải "nghe thấy" trực tiếp từng thiết bị đầu cuối.

### `lora_gateway.py` — Phiên bản cải tiến

Chuyển sang nhận từ **Relay Station**: **433.5 MHz, SyncWord 0x56**. Relay đóng vai trò trung gian giải quyết các vấn đề trên — mở rộng tầm phủ lên ~1 km, tách biệt tần số uplink/backhaul tránh nhiễu, đồng thời gateway bổ sung lọc duplicate trong cửa sổ 1 giây (first-packet-wins) để loại bỏ packet trùng khi nhiều relay cùng forward một gói.

Các tính năng chính:
- Lọc duplicate trong cửa sổ 1 giây (first-packet-wins)
- Parse và validate struct 74 bytes trước khi publish
- Publish JSON lên HiveMQ Cloud qua MQTT TLS (`helmet/<MAC>`)
- Non-blocking MQTT loop, debug IRQ/OpMode mỗi 10 giây

---

## Fall Detection Model — `Training_CNN_1D_Augment.ipynb`

- **Input**: cửa sổ 65 mẫu × 6 kênh (2.6 s @ 25 Hz)
- **Kiến trúc**: Conv1D(24) → Conv1D(32) → GAP → Dense(32) → Dense(16) → Softmax(2)
- **Classes**: Fall / Non-Fall (Static, Walk, Run gộp thành Non-Fall)
- **Quantization**: INT8 post-training → 14 KB, inference < 30 ms trên ESP32-S3
- **Kết quả**: Accuracy 96.2%, sensitivity (Fall recall) 96.2%

---

## Cấu hình LoRa tóm tắt

| | Tag TX | Relay RX | Relay TX | Gateway RX |
|---|---|---|---|---|
| Freq | 433.0 MHz | 433.0 MHz | 433.5 MHz | 433.5 MHz |
| SF | 9 | 9 | 9 | 9 |
| BW | 125 kHz | 125 kHz | 125 kHz | 125 kHz |
| SyncWord | 0x12 | 0x12 | 0x56 | 0x56 |

---

## MQTT

- **Broker**: HiveMQ Cloud (TLS port 8883)
- **Topic**: `helmet/<MAC_HEX>`
- **Payload**: JSON gồm temp, voltage, battery, GPS, HR, SpO2, UWB distances, fallDetected, helpRequest, timestamp

---

## Yêu cầu

**Firmware (Arduino/PlatformIO):**
- Chirale_TensorFlowLite, LoRa, TinyGPSPlus, Adafruit_INA219, MAX30105, MPU9250

**Gateway (Python):**
```
pip install lgpio spidev paho-mqtt
```

**Hardware chính:**
- ESP32-S3 (Tag), ESP32 (Relay), Raspberry Pi (Gateway)
- SX1278 LoRa module RA-02 × 3
- DW3000 UWB module, MPU9250, MAX30105, MAX30205, INA219, GPS module
