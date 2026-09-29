# BÁO CÁO THIẾT KẾ MẠNG LAN TRƯỜNG ĐẠI HỌC TÍCH HỢP HSRP REDUNDANCY
## Đầy đủ: Dual Core HSRP, LACP EtherChannel, VTP, STP Rapid-PVST+, WLAN & Server Farm

---

## MỤC LỤC
1. [HSRP là gì và Những thay đổi cốt lõi khi tích hợp HSRP](#1-hsrp-là-gì-và-những-thay-đổi-cốt-lõi-khi-tích-hợp-hsrp)
2. [Sơ đồ tổng quát toàn mạng tích hợp HSRP (Hình ảnh)](#2-sơ-đồ-tổng-quát-toàn-mạng-tích-hợp-hsrp-hình-ảnh)
3. [Chi tiết từng khu vực: Sơ đồ hình ảnh & Bảng cổng nối](#3-chi-tiết-từng-khu-vực-sơ-đồ-hình-ảnh--bảng-cổng-nối)
   - [3.1. Khu Trung tâm (Dual Core MLSW1 Active + MLSW2 Standby & Server Room)](#31-khu-trung-tâm-dual-core-mlsw1-active--mlsw2-standby--server-room)
   - [3.2. Khu 1 — Khu IT (VLAN 10)](#32-khu-1--khu-it-vlan-10)
   - [3.3. Khu 2 — Khu Đào tạo (VLAN 20)](#33-khu-2--khu-đào-tạo-vlan-20)
   - [3.4. Khu 3 — Khu Giảng viên (VLAN 30)](#34-khu-3--khu-giảng-viên-vlan-30)
   - [3.5. Khu 4 — Khu Sinh viên & Lab (VLAN 40)](#35-khu-4--khu-sinh-viên--lab-vlan-40)
4. [Bảng quy hoạch IP chi tiết (Virtual IP vs Real IP)](#4-bảng-quy-hoạch-ip-chi-tiết-virtual-ip-vs-real-ip)
5. [Bảng tổng hợp toàn bộ cổng nối và đấu nối cáp hệ thống HSRP](#5-bảng-tổng-hợp-toàn-bộ-cổng-nối-và-đấu-nối-cáp-hệ-thống-hsrp)
6. [Cấu hình Cisco IOS chuẩn cho Dual Core HSRP, STP, VTP & LACP](#6-cấu-hình-cisco-ios-chuẩn-cho-dual-core-hsrp-stp-vtp--lacp)

---

## 1. HSRP LÀ GÌ VÀ NHỮNG THAY ĐỔI CỐT LÕI KHI TÍCH HỢP HSRP

### 1.1. Khái niệm HSRP (Hot Standby Router Protocol)
Trong mô hình cũ (1 Core Switch MLSW1), MLSW1 là điểm yếu chí tử (**Single Point of Failure**). Nếu MLSW1 hỏng bo mạch hoặc mất nguồn, toàn bộ mạng trường học sẽ mất kết nối hoàn toàn.  
**HSRP (Cisco Proprietary / FHRP)** giải quyết triệt để vấn đề này bằng cách kết hợp **2 Core Switch Layer 3 (MLSW1 và MLSW2)** thành một cặp dự phòng nóng (Active/Standby):
* **MLSW1 (Active):** Đảm nhiệm định tuyến chính và xử lý lưu lượng cho tất cả các VLAN.
* **MLSW2 (Standby):** Lắng nghe gói tin Hello (chu kỳ 3 giây). Nếu không nhận được Hello sau 10 giây (Holdtime), MLSW2 tự động nâng cấp thành **Active** để gánh toàn bộ hệ thống.
* **Client (PC, Laptop, WiFi):** Chỉ cấu hình Default Gateway trỏ về **Virtual IP (VIP)**. Do đó khi sự cố xảy ra, người dùng không hề bị gián đoạn hay phải đổi IP Gateway!

---

### 1.2. Bảng so sánh những thay đổi trước và sau khi tích hợp HSRP

| Hạng mục | Trước khi có HSRP (1 Core) | Sau khi tích hợp HSRP (Dual Core) | Ý nghĩa kỹ thuật |
|:---|:---|:---|:---|
| **Số lượng Core Switch** | 1 (MLSW1) | **2 (MLSW1 Active + MLSW2 Standby)** | Loại bỏ hoàn toàn điểm nghẽn đơn lẻ (No Single Point of Failure). |
| **Địa chỉ Gateway của VLAN** | IP vật lý của MLSW1 (`.1`) | **HSRP Virtual IP (VIP: `.1`)** | IP Gateway là ảo, dùng chung cho cả 2 switch Core. |
| **IP vật lý trên SVI Core** | MLSW1 mang IP `.1` | **MLSW1 mang `.2` (Pri 110, Preempt)<br>MLSW2 mang `.3` (Pri 100)** | Định danh phân biệt 2 switch vật lý trong bảng định tuyến. |
| **Đường liên kết Inter-Core** | Không có | **LACP EtherChannel (Po10: Gi1/0/23+24)** | Trunking 2Gbps truyền gói tin HSRP Hello, VTP và định tuyến. |
| **Uplink từ Access Switches** | 2 link gộp LACP vào duy nhất MLSW1 | **2 Uplink độc lập:<br>• Gi0/1 nối MLSW1 (Active)<br>• Gi0/2 nối MLSW2 (Standby)** | Dự phòng cả đứt cáp lẫn hỏng thiết bị Core. |
| **Cổng kết nối Router R1** | 1 cổng (`Gi0/0` về MLSW1) | **2 cổng:<br>• Gi0/0 về MLSW1 (`10.0.0.0/30`)<br>• Gi0/2 về MLSW2 (`10.0.1.0/30`)** | Dự phòng đường ra mạng ngoài / Internet. |
| **Vai trò STP (Rapid-PVST+)** | MLSW1 là Root Bridge duy nhất | **MLSW1 là Primary Root (Pri 4096)<br>MLSW2 là Secondary Root (Pri 8192)** | Đồng bộ đường đi L2 (STP) ăn khớp với đường định tuyến L3 (HSRP). |

---

## 2. SƠ ĐỒ TỔNG QUÁT TOÀN MẠNG TÍCH HỢP HSRP (HÌNH ẢNH)

![Sơ đồ tổng thể HSRP](./images/topo_tong_the_lacp_vtp_stp_hsrp.png)

---

## 3. CHI TIẾT TỪNG KHU VỰC: SƠ ĐỒ HÌNH ẢNH & BẢNG CỔNG NỐI

### 3.1. Khu Trung tâm (Dual Core MLSW1 Active + MLSW2 Standby & Server Room)

![Khu Trung tâm HSRP](./images/khu_trung_tam_core_hsrp.png)

#### Bảng cổng nối Edge Router R1
| Cổng | Cấu hình IP | Chế độ | Thiết bị đối ứng | Cổng đối ứng | Loại cáp |
|:---:|:---:|:---:|:---|:---:|:---:|
| **Gi0/0** | `10.0.0.1/30` | Routed Port | Core 1: MLSW1 | Gi1/0/1 | Cáp quang SM Fiber |
| **Gi0/2** | `10.0.1.1/30` | Routed Port | Core 2: MLSW2 | Gi1/0/1 | Cáp quang SM Fiber |
| **Gi0/1** | IP nhà mạng ISP | Routed Port | Internet / ISP Gateway | — | Cáp WAN |

#### Bảng cổng nối Core Switch 1: MLSW1 (HSRP ACTIVE — Priority 110)
| Cổng vật lý | Cổng logic | Chế độ | VLAN | Thiết bị đối ứng | Cổng đối ứng | Vai trò kỹ thuật |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi1/0/1** | — | Routed Port | `10.0.0.2/30` | Router R1 | Gi0/0 | Uplink chính ra Internet |
| **Gi1/0/2** | — | Trunk (802.1Q) | All | SW1 (Khu IT) | Gi0/1 | Đường Active xuống SW1 |
| **Gi1/0/4** | — | Trunk (802.1Q) | All | SW2 (Khu Đào tạo) | Gi0/1 | Đường Active xuống SW2 |
| **Gi1/0/6** | — | Trunk (802.1Q) | All | SW3 (Khu Giảng viên) | Gi0/1 | Đường Active xuống SW3 |
| **Gi1/0/8** | — | Trunk (802.1Q) | All | SW4 (Khu Sinh viên) | Gi0/1 | Đường Active xuống SW4 |
| **Gi1/0/23** | **Port-channel 10** | LACP Trunk | All | MLSW2 | Gi1/0/23 | Inter-Core Trunk (Sợi 1) |
| **Gi1/0/24** | **Port-channel 10** | LACP Trunk | All | MLSW2 | Gi1/0/24 | Inter-Core Trunk (Sợi 2) |
| **Gi1/0/10** | — | Access | VLAN 50 | DHCP Server | Fa0 | Cáp đồng Cat6 |
| **Gi1/0/11** | — | Access | VLAN 50 | DNS Server | Fa0 | Cáp đồng Cat6 |
| **Gi1/0/12** | — | Access | VLAN 50 | Web Server | Fa0 | Cáp đồng Cat6 |

#### Bảng cổng nối Core Switch 2: MLSW2 (HSRP STANDBY — Priority 100)
| Cổng vật lý | Cổng logic | Chế độ | VLAN | Thiết bị đối ứng | Cổng đối ứng | Vai trò kỹ thuật |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi1/0/1** | — | Routed Port | `10.0.1.2/30` | Router R1 | Gi0/2 | Uplink phụ ra Internet |
| **Gi1/0/2** | — | Trunk (802.1Q) | All | SW1 (Khu IT) | Gi0/2 | Đường Standby từ SW1 |
| **Gi1/0/4** | — | Trunk (802.1Q) | All | SW2 (Khu Đào tạo) | Gi0/2 | Đường Standby từ SW2 |
| **Gi1/0/6** | — | Trunk (802.1Q) | All | SW3 (Khu Giảng viên) | Gi0/2 | Đường Standby từ SW3 |
| **Gi1/0/8** | — | Trunk (802.1Q) | All | SW4 (Khu Sinh viên) | Gi0/2 | Đường Standby từ SW4 |
| **Gi1/0/23** | **Port-channel 10** | LACP Trunk | All | MLSW1 | Gi1/0/23 | Inter-Core Trunk (Sợi 1) |
| **Gi1/0/24** | **Port-channel 10** | LACP Trunk | All | MLSW1 | Gi1/0/24 | Inter-Core Trunk (Sợi 2) |

---

### 3.2. Khu 1 — Khu IT (VLAN 10)

![Khu IT HSRP](./images/khu_it_vlan10_hsrp.png)

#### Bảng cổng nối Switch SW1 (Layer 2)
| Cổng | Chế độ cổng | VLAN | Trạng thái STP | Thiết bị đối ứng | Cổng đối ứng | Ghi chú |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi0/1** | Trunk (802.1Q) | All | **Root Port (Forwarding)** | Core 1: MLSW1 | Gi1/0/2 | Đường chính (Active) |
| **Gi0/2** | Trunk (802.1Q) | All | **Alternate (Blocking)** | Core 2: MLSW2 | Gi1/0/2 | Đường dự phòng (Standby) |
| **Fa0/1** | Access | VLAN 10 | PortFast + BPDU Guard | PC-IT1 | NIC Fa0 | Nhận IP DHCP dải `.10.x` |
| **Fa0/2** | Access | VLAN 10 | PortFast + BPDU Guard | PC-IT2 | NIC Fa0 | Nhận IP DHCP dải `.10.x` |
| **Fa0/3** | Access | VLAN 10 | PortFast + BPDU Guard | PC-IT3 | NIC Fa0 | Nhận IP DHCP dải `.10.x` |
| **Fa0/4** | Access | VLAN 10 | PortFast + BPDU Guard | **Wireless Router WR-IT** | **LAN 1** | SSID: `IT-WIFI` |

---

### 3.3. Khu 2 — Khu Đào tạo (VLAN 20)

![Khu Đào tạo HSRP](./images/khu_daotao_vlan20_hsrp.png)

#### Bảng cổng nối Switch SW2 (Layer 2)
| Cổng | Chế độ cổng | VLAN | Trạng thái STP | Thiết bị đối ứng | Cổng đối ứng | Ghi chú |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi0/1** | Trunk (802.1Q) | All | **Root Port (Forwarding)** | Core 1: MLSW1 | Gi1/0/4 | Đường chính (Active) |
| **Gi0/2** | Trunk (802.1Q) | All | **Alternate (Blocking)** | Core 2: MLSW2 | Gi1/0/4 | Đường dự phòng (Standby) |
| **Fa0/1** | Access | VLAN 20 | PortFast + BPDU Guard | PC-DAOTAO1 | NIC Fa0 | Nhận IP DHCP dải `.20.x` |
| **Fa0/2** | Access | VLAN 20 | PortFast + BPDU Guard | PC-DAOTAO2 | NIC Fa0 | Nhận IP DHCP dải `.20.x` |
| **Fa0/3** | Access | VLAN 20 | PortFast + BPDU Guard | PC-DAOTAO3 | NIC Fa0 | Nhận IP DHCP dải `.20.x` |
| **Fa0/4** | Access | VLAN 20 | PortFast + BPDU Guard | **Wireless Router WR-DT** | **LAN 1** | SSID: `DAOTAO-WIFI` |

---

### 3.4. Khu 3 — Khu Giảng viên (VLAN 30)

![Khu Giảng viên HSRP](./images/khu_giangvien_vlan30_hsrp.png)

#### Bảng cổng nối Switch SW3 (Layer 2)
| Cổng | Chế độ cổng | VLAN | Trạng thái STP | Thiết bị đối ứng | Cổng đối ứng | Ghi chú |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi0/1** | Trunk (802.1Q) | All | **Root Port (Forwarding)** | Core 1: MLSW1 | Gi1/0/6 | Đường chính (Active) |
| **Gi0/2** | Trunk (802.1Q) | All | **Alternate (Blocking)** | Core 2: MLSW2 | Gi1/0/6 | Đường dự phòng (Standby) |
| **Fa0/1** | Access | VLAN 30 | PortFast + BPDU Guard | PC-GV1 | NIC Fa0 | Nhận IP DHCP dải `.30.x` |
| **Fa0/2** | Access | VLAN 30 | PortFast + BPDU Guard | PC-GV2 | NIC Fa0 | Nhận IP DHCP dải `.30.x` |
| **Fa0/3** | Access | VLAN 30 | PortFast + BPDU Guard | PC-GV3 | NIC Fa0 | Nhận IP DHCP dải `.30.x` |
| **Fa0/4** | Access | VLAN 30 | PortFast + BPDU Guard | **Wireless Router WR-GV** | **LAN 1** | SSID: `GIANGVIEN-WIFI` |

---

### 3.5. Khu 4 — Khu Sinh viên & Lab (VLAN 40)

![Khu Sinh viên HSRP](./images/khu_sinhvien_vlan40_hsrp.png)

#### Bảng cổng nối Switch SW4 (Layer 2)
| Cổng | Chế độ cổng | VLAN | Trạng thái STP | Thiết bị đối ứng | Cổng đối ứng | Ghi chú |
|:---:|:---:|:---:|:---:|:---|:---:|:---|
| **Gi0/1** | Trunk (802.1Q) | All | **Root Port (Forwarding)** | Core 1: MLSW1 | Gi1/0/8 | Đường chính (Active) |
| **Gi0/2** | Trunk (802.1Q) | All | **Alternate (Blocking)** | Core 2: MLSW2 | Gi1/0/8 | Đường dự phòng (Standby) |
| **Fa0/1** | Access | VLAN 40 | PortFast + BPDU Guard | PC-SV1 | NIC Fa0 | Nhận IP DHCP dải `.40.x` |
| **Fa0/2** | Access | VLAN 40 | PortFast + BPDU Guard | PC-SV2 | NIC Fa0 | Nhận IP DHCP dải `.40.x` |
| **Fa0/3** | Access | VLAN 40 | PortFast + BPDU Guard | PC-SV3 | NIC Fa0 | Nhận IP DHCP dải `.40.x` |
| **Fa0/4** | Access | VLAN 40 | PortFast + BPDU Guard | PC-SV4 | NIC Fa0 | Nhận IP DHCP dải `.40.x` |
| **Fa0/5** | Access | VLAN 40 | PortFast + BPDU Guard | **Wireless Router WR-SV** | **LAN 1** | SSID: `SINHVIEN-WIFI` |

---

## 4. BẢNG QUY HOẠCH IP CHI TIẾT (VIRTUAL IP vs REAL IP)

> **Nguyên tắc vàng của HSRP:** Tất cả thiết bị đầu cuối (PC, Laptop, Smartphone) chỉ biết đến **Virtual IP (VIP)**. Hai switch Core sử dụng IP thật trên SVI để trao đổi trạng thái và định tuyến.

| VLAN ID | Tên VLAN | HSRP Group | **Virtual IP (VIP - Gateway)** | **MLSW1 IP Thật (Active)** | **MLSW2 IP Thật (Standby)** | Dải DHCP cấp phát |
|:---:|:---|:---:|:---:|:---:|:---:|:---|
| **10** | IT | Group 10 | **`192.168.10.1`** | `192.168.10.2` (Pri 110) | `192.168.10.3` (Pri 100) | `192.168.10.11` - `.10.254` |
| **20** | DAOTAO | Group 20 | **`192.168.20.1`** | `192.168.20.2` (Pri 110) | `192.168.20.3` (Pri 100) | `192.168.20.11` - `.20.254` |
| **30** | GIANGVIEN | Group 30 | **`192.168.30.1`** | `192.168.30.2` (Pri 110) | `192.168.30.3` (Pri 100) | `192.168.30.11` - `.30.254` |
| **40** | SINHVIEN | Group 40 | **`192.168.40.1`** | `192.168.40.2` (Pri 110) | `192.168.40.3` (Pri 100) | `192.168.40.11` - `.40.254` |
| **50** | SERVER | Group 50 | **`192.168.50.1`** | `192.168.50.2` (Pri 110) | `192.168.50.3` (Pri 100) | IP gán tĩnh |
| **—** | WAN 1 (R1-M1) | — | — | `10.0.0.2/30` | `10.0.0.1/30` (R1 Gi0/0) | Đường mạng L3 |
| **—** | WAN 2 (R1-M2) | — | — | `10.0.1.2/30` (M2) | `10.0.1.1/30` (R1 Gi0/2) | Đường mạng L3 |

---

## 5. BẢNG TỔNG HỢP TOÀN BỘ CỔNG NỐI VÀ ĐẤU NỐI CÁP HỆ THỐNG HSRP

| STT | Thiết bị nguồn | Cổng nguồn | Thiết bị đích | Cổng đích | Kênh / Giao thức | Loại cáp | Vai trò trong hệ thống |
|:---:|:---|:---:|:---|:---:|:---:|:---:|:---|
| 1 | Router R1 | Gi0/0 | MLSW1 | Gi1/0/1 | Routed Port | 🟡 SM Fiber | Đường WAN chính ra Internet |
| 2 | Router R1 | Gi0/2 | MLSW2 | Gi1/0/1 | Routed Port | 🟡 SM Fiber | Đường WAN phụ ra Internet |
| 3 | MLSW1 | Gi1/0/23 | MLSW2 | Gi1/0/23 | **Po10 (LACP)** | 🟡 SM Fiber | Kênh Inter-Core Trunking 2Gbps |
| 4 | MLSW1 | Gi1/0/24 | MLSW2 | Gi1/0/24 | **Po10 (LACP)** | 🟡 SM Fiber | Truyền HSRP Hello & VTP update |
| 5 | SW1 (IT) | Gi0/1 | MLSW1 | Gi1/0/2 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Active (Forwarding) |
| 6 | SW1 (IT) | Gi0/2 | MLSW2 | Gi1/0/2 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Standby (STP Blocking) |
| 7 | SW2 (Đào tạo) | Gi0/1 | MLSW1 | Gi1/0/4 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Active (Forwarding) |
| 8 | SW2 (Đào tạo) | Gi0/2 | MLSW2 | Gi1/0/4 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Standby (STP Blocking) |
| 9 | SW3 (Giảng viên) | Gi0/1 | MLSW1 | Gi1/0/6 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Active (Forwarding) |
| 10 | SW3 (Giảng viên) | Gi0/2 | MLSW2 | Gi1/0/6 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Standby (STP Blocking) |
| 11 | SW4 (Sinh viên) | Gi0/1 | MLSW1 | Gi1/0/8 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Active (Forwarding) |
| 12 | SW4 (Sinh viên) | Gi0/2 | MLSW2 | Gi1/0/8 | Trunk 802.1Q | 🟡 SM Fiber | Uplink Standby (STP Blocking) |
| 13 | MLSW1 | Gi1/0/10 | DHCP Server | Fa0 | Access VLAN 50 | 🔵 UTP Cat6 | Máy chủ DHCP (192.168.50.10) |
| 14 | MLSW1 | Gi1/0/11 | DNS Server | Fa0 | Access VLAN 50 | 🔵 UTP Cat6 | Máy chủ DNS (192.168.50.20) |
| 15 | MLSW1 | Gi1/0/12 | Web Server | Fa0 | Access VLAN 50 | 🔵 UTP Cat6 | Máy chủ Web (192.168.50.30) |
| 16 | SW1 | Fa0/1 - Fa0/3 | PC-IT1, 2, 3 | NIC | Access VLAN 10 | 🔵 UTP Cat6 | Máy trạm có dây Khu IT |
| 17 | SW1 | Fa0/4 | WR-IT | LAN 1 | Access VLAN 10 | 🔵 UTP Cat6 | Phát Wi-Fi `IT-WIFI` |
| 18 | SW2 | Fa0/1 - Fa0/3 | PC-DT1, 2, 3 | NIC | Access VLAN 20 | 🔵 UTP Cat6 | Máy trạm có dây Khu Đào tạo |
| 19 | SW2 | Fa0/4 | WR-DT | LAN 1 | Access VLAN 20 | 🔵 Cáp Cat6 | Phát Wi-Fi `DAOTAO-WIFI` |
| 20 | SW3 | Fa0/1 - Fa0/3 | PC-GV1, 2, 3 | NIC | Access VLAN 30 | 🔵 Cáp Cat6 | Máy trạm có dây Khu Giảng viên |
| 21 | SW3 | Fa0/4 | WR-GV | LAN 1 | Access VLAN 30 | 🔵 Cáp Cat6 | Phát Wi-Fi `GIANGVIEN-WIFI` |
| 22 | SW4 | Fa0/1 - Fa0/4 | PC-SV1, 2, 3, 4 | NIC | Access VLAN 40 | 🔵 Cáp Cat6 | Máy trạm có dây Khu Sinh viên |
| 23 | SW4 | Fa0/5 | WR-SV | LAN 1 | Access VLAN 40 | 🔵 Cáp Cat6 | Phát Wi-Fi `SINHVIEN-WIFI` |

---

## 6. CẤU HÌNH CISCO IOS CHUẨN CHO DUAL CORE HSRP, STP, VTP & LACP

### 6.1. Cấu hình trên Core 1: MLSW1 (HSRP ACTIVE)
```cisco
! ==========================================
! 1. BẬT ĐỊNH TUYẾN & STP PRIMARY ROOT
! ==========================================
ip routing
spanning-tree mode rapid-pvst
spanning-tree vlan 1-50 root primary

! ==========================================
! 2. CẤU HÌNH VTP SERVER & TẠO VLAN
! ==========================================
vtp mode server
vtp domain UNIVERSITY
vtp password cisco
vtp version 2

vlan 10
 name IT
vlan 20
 name DAOTAO
vlan 30
 name GIANGVIEN
vlan 40
 name SINHVIEN
vlan 50
 name SERVER

! ==========================================
! 3. LACP ETHERCHANNEL INTER-CORE (MLSW1 <-> MLSW2)
! ==========================================
interface range GigabitEthernet1/0/23 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 10 mode active
 no shutdown

interface Port-channel 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all

! ==========================================
! 4. CẤU HÌNH TRUNK XUỐNG CÁC SWITCH NHÁNH (SW1-SW4)
! ==========================================
interface range GigabitEthernet1/0/2, GigabitEthernet1/0/4, GigabitEthernet1/0/6, GigabitEthernet1/0/8
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! ==========================================
! 5. CẤU HÌNH SVI & HSRP ACTIVE (PRIORITY 110, PREEMPT)
! ==========================================
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 no shutdown

interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt
 no shutdown

interface vlan 30
 ip address 192.168.30.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 30 ip 192.168.30.1
 standby 30 priority 110
 standby 30 preempt
 no shutdown

interface vlan 40
 ip address 192.168.40.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 40 ip 192.168.40.1
 standby 40 priority 110
 standby 40 preempt
 no shutdown

interface vlan 50
 ip address 192.168.50.2 255.255.255.0
 standby 50 ip 192.168.50.1
 standby 50 priority 110
 standby 50 preempt
 no shutdown

! ==========================================
! 6. ROUTED PORT NỐI R1 & DEFAULT ROUTE
! ==========================================
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.0.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.0.1

! Cổng nối Server
interface range GigabitEthernet1/0/10 - 12
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 no shutdown
```

---

### 6.2. Cấu hình trên Core 2: MLSW2 (HSRP STANDBY)
```cisco
! ==========================================
! 1. BẬT ĐỊNH TUYẾN & STP SECONDARY ROOT
! ==========================================
ip routing
spanning-tree mode rapid-pvst
spanning-tree vlan 1-50 root secondary

! ==========================================
! 2. CẤU HÌNH VTP CLIENT
! ==========================================
vtp mode client
vtp domain UNIVERSITY
vtp password cisco

! ==========================================
! 3. LACP ETHERCHANNEL INTER-CORE (MLSW2 <-> MLSW1)
! ==========================================
interface range GigabitEthernet1/0/23 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 10 mode active
 no shutdown

interface Port-channel 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all

! ==========================================
! 4. CẤU HÌNH TRUNK XUỐNG CÁC SWITCH NHÁNH
! ==========================================
interface range GigabitEthernet1/0/2, GigabitEthernet1/0/4, GigabitEthernet1/0/6, GigabitEthernet1/0/8
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! ==========================================
! 5. CẤU HÌNH SVI & HSRP STANDBY (PRIORITY 100)
! ==========================================
interface vlan 10
 ip address 192.168.10.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 10 ip 192.168.10.1
 standby 10 priority 100
 no shutdown

interface vlan 20
 ip address 192.168.20.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 20 ip 192.168.20.1
 standby 20 priority 100
 no shutdown

interface vlan 30
 ip address 192.168.30.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 30 ip 192.168.30.1
 standby 30 priority 100
 no shutdown

interface vlan 40
 ip address 192.168.40.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 40 ip 192.168.40.1
 standby 40 priority 100
 no shutdown

interface vlan 50
 ip address 192.168.50.3 255.255.255.0
 standby 50 ip 192.168.50.1
 standby 50 priority 100
 no shutdown

! ==========================================
! 6. ROUTED PORT NỐI R1 & DEFAULT ROUTE
! ==========================================
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.1.1
```

---

### 6.3. Cấu hình trên Edge Router R1
```cisco
! Cổng nối MLSW1
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown

! Cổng nối MLSW2
interface GigabitEthernet0/2
 ip address 10.0.1.1 255.255.255.252
 no shutdown

! Cổng nối Internet
interface GigabitEthernet0/1
 ip address <IP_ISP> <SUBNET_MASK>
 no shutdown

ip route 0.0.0.0 0.0.0.0 <GATEWAY_ISP>

! Định tuyến về các mạng nội bộ (Đường qua MLSW1 là chính, qua MLSW2 là dự phòng)
ip route 192.168.10.0 255.255.255.0 10.0.0.2
ip route 192.168.10.0 255.255.255.0 10.0.1.2 10
ip route 192.168.20.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.1.2 10
ip route 192.168.30.0 255.255.255.0 10.0.0.2
ip route 192.168.30.0 255.255.255.0 10.0.1.2 10
ip route 192.168.40.0 255.255.255.0 10.0.0.2
ip route 192.168.40.0 255.255.255.0 10.0.1.2 10
ip route 192.168.50.0 255.255.255.0 10.0.0.2
ip route 192.168.50.0 255.255.255.0 10.0.1.2 10
```

---

### 6.4. Cấu hình trên Access Switch (Ví dụ SW1 - Khu IT)
```cisco
spanning-tree mode rapid-pvst

vtp mode client
vtp domain UNIVERSITY
vtp password cisco

! 2 Uplink kết nối lên 2 Core Switch khác nhau
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

interface GigabitEthernet0/2
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! Cổng nối PC có PortFast + BPDU Guard
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Cổng nối Wireless Router WR-IT (LAN 1)
interface FastEthernet0/4
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Default Gateway quản trị trỏ về HSRP Virtual IP
ip default-gateway 192.168.10.1
```
