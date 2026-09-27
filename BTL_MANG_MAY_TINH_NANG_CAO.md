# BÀI TẬP LỚN MẠNG MÁY TÍNH NÂNG CAO
## Thiết kế và triển khai hệ thống mạng LAN cho Trường Đại học

---

## 1. TỔNG QUAN ĐỀ TÀI

**Tên đề tài:** Thiết kế và triển khai hệ thống mạng LAN cho một trường đại học sử dụng VLAN, Switch Layer 2, Switch Layer 3, WLAN, DHCP, DNS và Web Server.

**Mô hình tổng quát:** Hệ thống được xây dựng theo mô hình mạng phân cấp (Hierarchical Network Model), trong đó:
- **MLSW1** đóng vai trò Core Layer 3 thực hiện định tuyến giữa các VLAN (Inter-VLAN Routing)
- Các **Switch Layer 2** (SW1–SW4) đảm nhiệm kết nối thiết bị đầu cuối tại từng khu vực
- Các khu vực ở xa được kết nối với Core bằng cáp **Single-Mode Fiber (SM Fiber)**
- Khu Sinh viên được triển khai **WLAN** qua Access Point
- Cụm Server VLAN 50 cung cấp các dịch vụ **DHCP, DNS và Web**

---

## 2. SƠ ĐỒ MẠNG TỔNG THỂ (LOGIC TOPOLOGY)

```
                              INTERNET
                                  |
                               Router R1
                             Gi0/0: 10.0.0.1/30  (kết nối về MLSW1)
                             Gi0/1: <IP ISP>      (kết nối Internet)
                                  |
                       [Link: 10.0.0.0/30 - SM Fiber]
                                  |
                    +-----------------------------+
                    |     KHU TRUNG TÂM / CORE    |
                    |   MLSW1 — Layer 3 Switch    |
                    |  Routed Port: 10.0.0.2/30   |
                    |  SVI VLAN10: 192.168.10.1   |
                    |  SVI VLAN20: 192.168.20.1   |
                    |  SVI VLAN30: 192.168.30.1   |
                    |  SVI VLAN40: 192.168.40.1   |
                    |  SVI VLAN50: 192.168.50.1   |
                    +----+-------+-------+----+---+
                         |       |       |        |
                      SM Fiber  SM    SM Fiber  (Local)
                         |    Fiber      |        |
              +----------+   |     +-----+--+   +---------+
              | KHU IT   |   |     |KHU GV  |   |  SERVER |
              | SW1 - L2 |   |     |SW3 - L2|   |  VLAN50 |
              +-----+----+   |     +----+---+   +---------+
                    |        |          |
                 PC-IT1   SM Fiber   PC-GV1
                 PC-IT2      |       PC-GV2
                 PC-IT3   +--+--------+
                          |KHU ĐÀO TẠO|
                          | SW2 - L2  |
                          +-----+-----+
                                |
                           PC-DAOTAO1
                           PC-DAOTAO2
                           PC-DAOTAO3

                    +-----------------------------+
                    |     KHU SINH VIÊN           |
                    |        SW4 - L2             |
                    +--------+-----------+--------+
                             |           |
                          PC-SV1      AP1 (Wi-Fi)
                          PC-SV2         |
                          PC-SV3    Laptop/Phone
                          PC-SV4    (SSID: UNIVERSITY-STUDENT)
```

---

## 3. PHÂN CHIA KHU VỰC VÀ THIẾT BỊ

### 3.1 Khu Trung tâm — Core / Phòng Server

**Đây là khu vực quan trọng nhất của toàn hệ thống.**

#### Router R1 — Edge Router

| Thuộc tính | Giá trị |
|:---|:---|
| Vai trò | Edge Router, kết nối LAN với Internet |
| Cổng Gi0/0 | 10.0.0.1/30 → kết nối về MLSW1 |
| Cổng Gi0/1 | IP do ISP cấp → kết nối Internet |
| Default Route | `ip route 0.0.0.0 0.0.0.0 <ISP_Gateway>` |
| Static Route (tùy chọn) | Trỏ các dải 192.168.x.0 về MLSW1 |

#### MLSW1 — Core Layer 3 Switch (Thiết bị quan trọng nhất)

| Thuộc tính | Giá trị |
|:---|:---|
| Vai trò | Core L3, Inter-VLAN Routing, Default Gateway cho tất cả VLAN |
| Routed Port (kết nối R1) | 10.0.0.2/30 |
| SVI VLAN 10 | 192.168.10.1/24 |
| SVI VLAN 20 | 192.168.20.1/24 |
| SVI VLAN 30 | 192.168.30.1/24 |
| SVI VLAN 40 | 192.168.40.1/24 |
| SVI VLAN 50 | 192.168.50.1/24 |
| Default Route | `ip route 0.0.0.0 0.0.0.0 10.0.0.1` |
| DHCP Relay | `ip helper-address 192.168.50.10` (cấu hình trên từng SVI VLAN 10/20/30/40) |
| Kết nối uplink các SW | Trunk 802.1Q (cho phép tất cả VLAN) |

**Chức năng của MLSW1:**
- Tạo và quản lý VLAN
- Làm Default Gateway cho các VLAN thông qua SVI (Switched Virtual Interface)
- Inter-VLAN Routing (định tuyến giữa các VLAN)
- DHCP Relay Agent (`ip helper-address`) chuyển tiếp DHCP request về Server VLAN 50
- Kết nối các khu vực qua SM Fiber bằng Trunk Link
- Có thể triển khai ACL để kiểm soát truy cập giữa các VLAN

#### Cụm Server — VLAN 50 (192.168.50.0/24)

| Server | IP tĩnh | Chức năng |
|:---|:---|:---|
| DHCP Server | 192.168.50.10 | Cấp IP tự động cho VLAN 10/20/30/40 |
| DNS Server | 192.168.50.20 | Phân giải tên miền nội bộ |
| Web Server | 192.168.50.30 | Cung cấp website HTTP của trường |

---

### 3.2 Khu IT — VLAN 10

| Thuộc tính | Giá trị |
|:---|:---|
| Switch | SW1 — Layer 2 Switch |
| VLAN | 10 — IT |
| Subnet | 192.168.10.0/24 |
| Gateway | 192.168.10.1 (SVI MLSW1) |
| DHCP Range | 192.168.10.2 – 192.168.10.254 |
| DNS | 192.168.50.20 |
| Kết nối uplink | SM Fiber → MLSW1 (Trunk 802.1Q) |

**Thiết bị đầu cuối:** PC-IT1, PC-IT2, PC-IT3

**Cấu hình cổng SW1:**
```
Access Ports (kết nối PC):  switchport mode access / switchport access vlan 10
Trunk Port (uplink MLSW1):  switchport mode trunk / switchport trunk allowed vlan all
```

---

### 3.3 Khu Đào tạo — VLAN 20

| Thuộc tính | Giá trị |
|:---|:---|
| Switch | SW2 — Layer 2 Switch |
| VLAN | 20 — DAOTAO |
| Subnet | 192.168.20.0/24 |
| Gateway | 192.168.20.1 (SVI MLSW1) |
| DHCP Range | 192.168.20.2 – 192.168.20.254 |
| DNS | 192.168.50.20 |
| Kết nối uplink | SM Fiber → MLSW1 (Trunk 802.1Q) |

**Thiết bị đầu cuối:** PC-DAOTAO1, PC-DAOTAO2, PC-DAOTAO3

**Cấu hình cổng SW2:**
```
Access Ports (kết nối PC):  switchport mode access / switchport access vlan 20
Trunk Port (uplink MLSW1):  switchport mode trunk / switchport trunk allowed vlan all
```

---

### 3.4 Khu Giảng viên — VLAN 30

| Thuộc tính | Giá trị |
|:---|:---|
| Switch | SW3 — Layer 2 Switch |
| VLAN | 30 — GIANGVIEN |
| Subnet | 192.168.30.0/24 |
| Gateway | 192.168.30.1 (SVI MLSW1) |
| DHCP Range | 192.168.30.2 – 192.168.30.254 |
| DNS | 192.168.50.20 |
| Kết nối uplink | SM Fiber → MLSW1 (Trunk 802.1Q) |

**Thiết bị đầu cuối:** PC-GV1, PC-GV2, PC-GV3

**Cấu hình cổng SW3:**
```
Access Ports (kết nối PC):  switchport mode access / switchport access vlan 30
Trunk Port (uplink MLSW1):  switchport mode trunk / switchport trunk allowed vlan all
```

---

### 3.5 Khu Sinh viên + WLAN — VLAN 40

| Thuộc tính | Giá trị |
|:---|:---|
| Switch | SW4 — Layer 2 Switch |
| VLAN | 40 — SINHVIEN |
| Subnet | 192.168.40.0/24 |
| Gateway | 192.168.40.1 (SVI MLSW1) |
| DHCP Range | 192.168.40.2 – 192.168.40.254 |
| DNS | 192.168.50.20 |
| Kết nối uplink | SM Fiber → MLSW1 (Trunk 802.1Q) |
| Access Point | AP1, SSID: `UNIVERSITY-STUDENT` |

**Thiết bị đầu cuối có dây:** PC-SV1, PC-SV2, PC-SV3, PC-SV4

**Thiết bị không dây:** Laptop, Điện thoại kết nối qua AP1

**Luồng kết nối WiFi:**
```
Laptop/Phone
    ↓ (Wi-Fi - SSID: UNIVERSITY-STUDENT)
AP1
    ↓ (Cáp đồng UTP)
SW4 (Access Port - VLAN 40)
    ↓ (SM Fiber - Trunk 802.1Q)
MLSW1
```

**Cấu hình cổng SW4:**
```
Access Ports (kết nối PC): switchport mode access / switchport access vlan 40
Access Port (kết nối AP1): switchport mode access / switchport access vlan 40
Trunk Port (uplink MLSW1): switchport mode trunk / switchport trunk allowed vlan all
```

---

## 4. QUY HOẠCH VLAN ĐẦY ĐỦ

| VLAN ID | Tên VLAN | Subnet | SVI / Gateway | Khu vực | Switch |
|:---:|:---|:---|:---|:---|:---|
| 10 | IT | 192.168.10.0/24 | 192.168.10.1 | Khu IT | SW1 |
| 20 | DAOTAO | 192.168.20.0/24 | 192.168.20.1 | Khu Đào tạo | SW2 |
| 30 | GIANGVIEN | 192.168.30.0/24 | 192.168.30.1 | Khu Giảng viên | SW3 |
| 40 | SINHVIEN | 192.168.40.0/24 | 192.168.40.1 | Khu Sinh viên + WiFi | SW4 |
| 50 | SERVER | 192.168.50.0/24 | 192.168.50.1 | Phòng Server (Local MLSW1) | MLSW1 |

---

## 5. QUY HOẠCH ĐỊA CHỈ IP ĐẦY ĐỦ

### 5.1 Link mạng giữa R1 và MLSW1

| Thiết bị | Cổng | IP | Subnet |
|:---|:---|:---|:---|
| R1 | Gi0/0 | 10.0.0.1 | /30 |
| MLSW1 | Routed Port Gi1/0/1 | 10.0.0.2 | /30 |

> **Lưu ý:** Cổng kết nối R1 trên MLSW1 phải được cấu hình `no switchport` để chuyển thành Routed Port (Layer 3), không phải Trunk/Access.

### 5.2 SVI (Default Gateway) trên MLSW1

| SVI | IP | Phục vụ VLAN |
|:---|:---|:---|
| interface vlan 10 | 192.168.10.1/24 | VLAN 10 - IT |
| interface vlan 20 | 192.168.20.1/24 | VLAN 20 - DAOTAO |
| interface vlan 30 | 192.168.30.1/24 | VLAN 30 - GIANGVIEN |
| interface vlan 40 | 192.168.40.1/24 | VLAN 40 - SINHVIEN |
| interface vlan 50 | 192.168.50.1/24 | VLAN 50 - SERVER |

### 5.3 Địa chỉ IP Server (tĩnh)

| Server | IP | Subnet Mask | Gateway | DNS |
|:---|:---|:---|:---|:---|
| DHCP Server | 192.168.50.10 | 255.255.255.0 | 192.168.50.1 | 192.168.50.20 |
| DNS Server | 192.168.50.20 | 255.255.255.0 | 192.168.50.1 | 192.168.50.20 |
| Web Server | 192.168.50.30 | 255.255.255.0 | 192.168.50.1 | 192.168.50.20 |

### 5.4 DHCP Pool trên DHCP Server (4 Pool riêng biệt)

| Pool Name | Network | Subnet Mask | Gateway | DNS Server | Excluded (tĩnh) |
|:---|:---|:---|:---|:---|:---|
| POOL-VLAN10 | 192.168.10.0 | 255.255.255.0 | 192.168.10.1 | 192.168.50.20 | 192.168.10.1 |
| POOL-VLAN20 | 192.168.20.0 | 255.255.255.0 | 192.168.20.1 | 192.168.50.20 | 192.168.20.1 |
| POOL-VLAN30 | 192.168.30.0 | 255.255.255.0 | 192.168.30.1 | 192.168.50.20 | 192.168.30.1 |
| POOL-VLAN40 | 192.168.40.0 | 255.255.255.0 | 192.168.40.1 | 192.168.50.20 | 192.168.40.1 |

---

## 6. CẤU HÌNH KỸ THUẬT CHI TIẾT

### 6.1 Cấu hình MLSW1 (Core Layer 3 Switch)

```cisco
! ===== BƯỚC 1: Bật tính năng định tuyến L3 =====
ip routing

! ===== BƯỚC 2: Tạo các VLAN =====
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

! ===== BƯỚC 3: Tạo SVI (Default Gateway) cho từng VLAN =====
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.50.10
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.50.10
 no shutdown

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.50.10
 no shutdown

interface vlan 40
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 192.168.50.10
 no shutdown

interface vlan 50
 ip address 192.168.50.1 255.255.255.0
 no shutdown

! ===== BƯỚC 4: Cấu hình Routed Port kết nối về R1 =====
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.0.2 255.255.255.252
 no shutdown

! ===== BƯỚC 5: Default Route trỏ về R1 =====
ip route 0.0.0.0 0.0.0.0 10.0.0.1

! ===== BƯỚC 6: Cấu hình Trunk Port đến các SW nhánh =====
! --- Trunk đến SW1 (Khu IT) ---
interface GigabitEthernet1/0/2
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! --- Trunk đến SW2 (Khu Đào tạo) ---
interface GigabitEthernet1/0/3
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! --- Trunk đến SW3 (Khu Giảng viên) ---
interface GigabitEthernet1/0/4
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! --- Trunk đến SW4 (Khu Sinh viên) ---
interface GigabitEthernet1/0/5
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! ===== BƯỚC 7: Cổng kết nối Server VLAN 50 (Access) =====
interface GigabitEthernet1/0/10
 switchport mode access
 switchport access vlan 50
 no shutdown

interface GigabitEthernet1/0/11
 switchport mode access
 switchport access vlan 50
 no shutdown

interface GigabitEthernet1/0/12
 switchport mode access
 switchport access vlan 50
 no shutdown
```

---

### 6.2 Cấu hình Router R1

```cisco
! ===== Cổng kết nối về MLSW1 =====
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown

! ===== Cổng kết nối Internet (ISP) =====
interface GigabitEthernet0/1
 ip address <IP_DO_ISP_CAP> <SUBNET_MASK>
 no shutdown

! ===== Default Route ra Internet =====
ip route 0.0.0.0 0.0.0.0 <ISP_GATEWAY>

! ===== Static Route trỏ các dải nội bộ về MLSW1 =====
ip route 192.168.10.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.0.2
ip route 192.168.30.0 255.255.255.0 10.0.0.2
ip route 192.168.40.0 255.255.255.0 10.0.0.2
ip route 192.168.50.0 255.255.255.0 10.0.0.2
```

---

### 6.3 Cấu hình SW1 — Khu IT (VLAN 10)

```cisco
! ===== Tạo VLAN =====
vlan 10
 name IT

! ===== Access Port cho PC =====
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
 no shutdown

! ===== Trunk Port uplink về MLSW1 (qua SM Fiber) =====
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

! ===== Default Gateway cho SW1 (quản trị) =====
ip default-gateway 192.168.10.1
```

---

### 6.4 Cấu hình SW2 — Khu Đào tạo (VLAN 20)

```cisco
vlan 20
 name DAOTAO

interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 20
 no shutdown

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

ip default-gateway 192.168.20.1
```

---

### 6.5 Cấu hình SW3 — Khu Giảng viên (VLAN 30)

```cisco
vlan 30
 name GIANGVIEN

interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 30
 no shutdown

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

ip default-gateway 192.168.30.1
```

---

### 6.6 Cấu hình SW4 — Khu Sinh viên + AP (VLAN 40)

```cisco
vlan 40
 name SINHVIEN

! ===== Access Port cho PC =====
interface range FastEthernet0/1 - 4
 switchport mode access
 switchport access vlan 40
 no shutdown

! ===== Access Port cho AP1 =====
interface FastEthernet0/5
 switchport mode access
 switchport access vlan 40
 no shutdown

! ===== Trunk Port uplink về MLSW1 =====
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan all
 no shutdown

ip default-gateway 192.168.40.1
```

---

## 7. CÁP KẾT NỐI — SM FIBER vs CÁP ĐỒNG

### 7.1 Sơ đồ phân loại cáp

| Đoạn kết nối | Loại cáp | Lý do |
|:---|:---|:---|
| MLSW1 ↔ SW1 | **SM Fiber (Quang đơn mode)** | Kết nối giữa các tòa nhà/khu vực xa, >100m |
| MLSW1 ↔ SW2 | **SM Fiber** | Như trên |
| MLSW1 ↔ SW3 | **SM Fiber** | Như trên |
| MLSW1 ↔ SW4 | **SM Fiber** | Như trên |
| R1 ↔ MLSW1 | **SM Fiber / Cáp đồng** | Trong cùng tòa nhà trung tâm |
| SW1 ↔ PC-IT | **Cáp đồng UTP Cat6** | Kết nối thiết bị đầu cuối, ≤100m |
| SW2 ↔ PC-DAOTAO | **Cáp đồng UTP Cat6** | Như trên |
| SW3 ↔ PC-GV | **Cáp đồng UTP Cat6** | Như trên |
| SW4 ↔ PC-SV | **Cáp đồng UTP Cat6** | Như trên |
| SW4 ↔ AP1 | **Cáp đồng UTP Cat6** | Như trên |
| Server ↔ MLSW1 | **Cáp đồng UTP Cat6** | Cùng phòng server |

### 7.2 Lý do chọn Single-Mode Fiber cho đường backbone

> **Single-Mode Fiber (SM Fiber):**
> - Sử dụng tia laser đơn sắc, lõi cáp rất nhỏ (~9µm)
> - Khoảng cách truyền: **lên đến hàng chục km**
> - Suy hao tín hiệu rất thấp
> - Phù hợp cho kết nối **backbone giữa các tòa nhà trong khuôn viên đại học**
>
> **Multi-Mode Fiber:** Chỉ đến ~550m (tốc độ 10Gbps), không phù hợp khoảng cách xa.
>
> **Cáp đồng UTP:** Giới hạn tối đa **100 mét**, chỉ dùng trong phòng/tầng.

### 7.3 Lắp module SFP trong Packet Tracer

Để sử dụng cáp Fiber trong Cisco Packet Tracer:
1. Tắt nguồn thiết bị (nhấn nút nguồn)
2. Kéo module **GLC-LH-SMD** (SFP Single-Mode) vào slot trống
3. Bật nguồn lại
4. Dùng cáp **Fiber** (màu vàng) nối giữa các thiết bị

---

## 8. CÁC DỊCH VỤ CHI TIẾT

### 8.1 DHCP Service

**Server IP:** 192.168.50.10

**Nguyên lý DHCP Relay (ip helper-address):**

```
Thiết bị đầu cuối (VLAN 40)
    ↓ DHCP Discover (Broadcast)
SW4
    ↓ Broadcast trong VLAN 40
MLSW1 — SVI VLAN 40 (192.168.40.1)
    ↓ ip helper-address 192.168.50.10
    ↓ Chuyển thành Unicast gửi về DHCP Server
DHCP Server (192.168.50.10)
    ↓ DHCP Offer → DHCP Ack
    ↓ Cấp IP + Gateway + DNS
Thiết bị đầu cuối nhận:
    IP:      192.168.40.x
    Gateway: 192.168.40.1
    DNS:     192.168.50.20
```

**4 DHCP Scope cần tạo trên DHCP Server:**

| Scope | Default Gateway | DNS | Start IP | End IP |
|:---|:---|:---|:---|:---|
| 192.168.10.0/24 | 192.168.10.1 | 192.168.50.20 | 192.168.10.2 | 192.168.10.254 |
| 192.168.20.0/24 | 192.168.20.1 | 192.168.50.20 | 192.168.20.2 | 192.168.20.254 |
| 192.168.30.0/24 | 192.168.30.1 | 192.168.50.20 | 192.168.30.2 | 192.168.30.254 |
| 192.168.40.0/24 | 192.168.40.1 | 192.168.50.20 | 192.168.40.2 | 192.168.40.254 |

---

### 8.2 DNS Service

**Server IP:** 192.168.50.20

**Bản ghi DNS cần tạo:**

| Loại Record | Tên miền | Giá trị |
|:---|:---|:---|
| A Record | www.university.local | 192.168.50.30 |
| A Record | university.local | 192.168.50.30 |

**Luồng phân giải DNS:**
```
Laptop gõ: http://www.university.local
    ↓ DNS Query (UDP port 53)
DNS Server (192.168.50.20)
    ↓ Tra bảng A Record
    ↓ Trả về: www.university.local → 192.168.50.30
Laptop kết nối đến 192.168.50.30
    ↓ HTTP GET
Web Server
    ↓ HTTP Response
Website hiển thị
```

---

### 8.3 Web Service

**Server IP:** 192.168.50.30

**Giao thức:** HTTP (Port 80)

**Nội dung website mẫu:**
```
================================
     ABC UNIVERSITY PORTAL
================================

Chào mừng đến với hệ thống cổng thông tin
Trường Đại học ABC

[ Phòng IT ]         [ Phòng Đào tạo ]
[ Giảng viên ]       [ Sinh viên ]

================================
```

**URL truy cập:** `http://www.university.local`

---

## 9. INTER-VLAN ROUTING — NGUYÊN LÝ HOẠT ĐỘNG

```
Ví dụ: PC-IT1 (VLAN 10) muốn truy cập Web Server (VLAN 50)

PC-IT1
IP: 192.168.10.100
    ↓ Gói tin gửi đến Gateway: 192.168.10.1
SW1 (Access VLAN 10)
    ↓ Trunk 802.1Q → MLSW1
MLSW1 — SVI VLAN 10 (nhận gói)
    ↓ Tra bảng định tuyến (Routing Table):
    ↓ 192.168.50.0/24 via SVI VLAN 50
MLSW1 — SVI VLAN 50 (chuyển tiếp)
    ↓ Gói tin đến Web Server
Web Server (192.168.50.30)
    ↓ HTTP Response ngược lại
PC-IT1 nhận được phản hồi
```

**Bảng định tuyến trên MLSW1 (tự động tạo khi bật `ip routing`):**

| Mạng đích | Next-Hop / Interface |
|:---|:---|
| 192.168.10.0/24 | directly connected (Vlan10) |
| 192.168.20.0/24 | directly connected (Vlan20) |
| 192.168.30.0/24 | directly connected (Vlan30) |
| 192.168.40.0/24 | directly connected (Vlan40) |
| 192.168.50.0/24 | directly connected (Vlan50) |
| 0.0.0.0/0 | 10.0.0.1 (R1) |

---

## 10. KỊCH BẢN DEMO ĐẦY ĐỦ

### Kịch bản 1: Laptop WiFi nhận IP tự động và truy cập Web

```
Bước 1: Laptop kết nối Wi-Fi SSID "UNIVERSITY-STUDENT"
         AP1 → SW4 → MLSW1 (VLAN 40)

Bước 2: Laptop gửi DHCP Discover
         MLSW1 relay về DHCP Server (192.168.50.10)
         Laptop nhận: IP 192.168.40.x / GW 192.168.40.1 / DNS 192.168.50.20

Bước 3: Laptop ping 192.168.40.1 (Gateway) → thành công

Bước 4: Laptop gõ http://www.university.local
         DNS Query → DNS Server → 192.168.50.30
         HTTP Request → Web Server
         Website hiển thị ✅

Kết quả chứng minh: WLAN + DHCP Relay + DNS + Web + Inter-VLAN Routing
```

### Kịch bản 2: Inter-VLAN Routing giữa các khu vực

```
Bước 1: PC-IT1 (192.168.10.x) ping PC-DAOTAO1 (192.168.20.x)
         Gói tin: PC-IT1 → GW 192.168.10.1 → MLSW1 → 192.168.20.1 → PC-DAOTAO1
         Kết quả: ping thành công ✅

Bước 2: PC-GV1 (192.168.30.x) truy cập http://www.university.local
         DNS phân giải → Web Server (192.168.50.30)
         Inter-VLAN: VLAN 30 → VLAN 50
         Kết quả: Website hiển thị ✅
```

### Kịch bản 3: Kiểm tra phân tách VLAN (tùy chọn ACL)

```
Mô tả: Bằng cách cấu hình ACL trên MLSW1, có thể ngăn
       VLAN 40 (Sinh viên) truy cập trực tiếp VLAN 10 (IT)
       nhưng vẫn cho phép truy cập VLAN 50 (Server)

→ Thể hiện tính năng bảo mật mạng nâng cao
```

---

## 11. TỔNG HỢP VAI TRÒ THIẾT BỊ

| Thiết bị | Loại | Vai trò chính | VLAN liên quan |
|:---|:---|:---|:---|
| R1 | Router | Edge Router, kết nối Internet | - |
| MLSW1 | L3 Switch | Core, Inter-VLAN Routing, Gateway, DHCP Relay | 10/20/30/40/50 |
| SW1 | L2 Switch | Access Switch Khu IT | VLAN 10 |
| SW2 | L2 Switch | Access Switch Khu Đào tạo | VLAN 20 |
| SW3 | L2 Switch | Access Switch Khu Giảng viên | VLAN 30 |
| SW4 | L2 Switch | Access Switch Khu Sinh viên + AP | VLAN 40 |
| AP1 | Access Point | Wi-Fi SSID: UNIVERSITY-STUDENT | VLAN 40 |
| DHCP Server | Server | Cấp IP tự động cho VLAN 10/20/30/40 | VLAN 50 |
| DNS Server | Server | Phân giải www.university.local → 192.168.50.30 | VLAN 50 |
| Web Server | Server | Cung cấp website HTTP của trường | VLAN 50 |
| PC-IT1~3 | End Device | Máy trạm khu IT | VLAN 10 |
| PC-DAOTAO1~3 | End Device | Máy trạm khu Đào tạo | VLAN 20 |
| PC-GV1~3 | End Device | Máy trạm khu Giảng viên | VLAN 30 |
| PC-SV1~4 | End Device | Máy trạm khu Sinh viên | VLAN 40 |
| Laptop/Phone | End Device | Thiết bị không dây (WiFi) | VLAN 40 |

---

## 12. CÂU HỎI VẤN ĐÁP THƯỜNG GẶP & ĐÁP ÁN

| Câu hỏi | Đáp án ngắn gọn |
|:---|:---|
| Tại sao dùng Switch L3 thay vì Router-on-a-Stick? | L3 Switch xử lý Inter-VLAN Routing trong phần cứng (ASIC), tốc độ cao hơn, không tạo bottleneck tại một cổng vật lý duy nhất như Router-on-a-Stick |
| DHCP Relay là gì? Tại sao cần nó? | DHCP Discover là broadcast, không qua Router/L3 Switch. `ip helper-address` chuyển broadcast thành Unicast và chuyển tiếp đến DHCP Server ở VLAN khác |
| Trunk Link là gì? Tại sao cần? | Trunk Link (802.1Q) cho phép nhiều VLAN đi trên một đường vật lý bằng cách gắn thêm VLAN tag vào frame. Cần thiết cho đường uplink từ Access Switch lên Core Switch |
| SVI là gì? | Switched Virtual Interface — cổng ảo Layer 3 gắn với một VLAN, đóng vai trò Default Gateway cho các thiết bị trong VLAN đó |
| Tại sao dùng SM Fiber không dùng MM Fiber? | SM Fiber dùng laser đơn sắc, truyền hàng km, suy hao thấp — phù hợp kết nối backbone giữa các tòa nhà đại học. MM Fiber chỉ ~550m |
| Tại sao Server cần IP tĩnh? | Server cần địa chỉ IP cố định để DHCP Relay, DNS record và ip helper-address luôn trỏ đúng đích |
| ip routing trên L3 Switch có ý nghĩa gì? | Lệnh này bật tính năng định tuyến Layer 3 trên Multilayer Switch, cho phép nó định tuyến giữa các VLAN thông qua SVI |

---

## 13. CHECKLIST KIỂM TRA TRƯỚC KHI NỘP

### Cấu hình kỹ thuật
- [ ] MLSW1: Đã bật `ip routing`
- [ ] MLSW1: Đã tạo đủ 5 VLAN (10/20/30/40/50)
- [ ] MLSW1: Đã tạo đủ 5 SVI với đúng IP
- [ ] MLSW1: Đã cấu hình `ip helper-address 192.168.50.10` trên SVI VLAN 10/20/30/40
- [ ] MLSW1: Routed Port Gi1/0/1 → 10.0.0.2/30 (kết nối R1)
- [ ] MLSW1: Default Route `ip route 0.0.0.0 0.0.0.0 10.0.0.1`
- [ ] MLSW1: Trunk Port đến SW1/SW2/SW3/SW4
- [ ] SW1/SW2/SW3/SW4: Access Port đúng VLAN, Trunk Port uplink
- [ ] SW4: Cổng AP1 là Access VLAN 40
- [ ] DHCP Server: Đủ 4 Pool (VLAN 10/20/30/40) với đúng Gateway và DNS
- [ ] DNS Server: Bản ghi A `www.university.local → 192.168.50.30`
- [ ] Web Server: HTTP service đang chạy (Active)
- [ ] R1: Static Route trỏ 5 dải 192.168.x.0 về 10.0.0.2

### Kiểm tra kết nối (Packet Tracer)
- [ ] PC trong mỗi VLAN nhận IP đúng range từ DHCP
- [ ] PC ping được Default Gateway của VLAN mình
- [ ] PC ping được Server (192.168.50.10/20/30)
- [ ] Laptop WiFi nhận IP VLAN 40 từ DHCP
- [ ] Truy cập `http://www.university.local` thành công từ mọi VLAN
- [ ] Inter-VLAN ping thành công (PC VLAN 10 ping PC VLAN 20)

---

## 14. MÔ TẢ HỆ THỐNG (DÙNG TRONG BÁO CÁO)

> Hệ thống được xây dựng theo mô hình mạng phân cấp (Hierarchical Network Model), trong đó **MLSW1** đóng vai trò **Core Layer 3 Switch** thực hiện Inter-VLAN Routing thông qua các SVI (Switched Virtual Interface), đồng thời hoạt động như DHCP Relay Agent chuyển tiếp yêu cầu cấp địa chỉ IP từ các VLAN về DHCP Server tập trung tại VLAN 50. Các **Switch Layer 2** (SW1–SW4) đảm nhiệm kết nối thiết bị đầu cuối tại từng khu vực chức năng, với các khu vực ở xa được kết nối với Core bằng cáp **Single-Mode Fiber** để đảm bảo truyền dẫn khoảng cách lớn với suy hao thấp. Khu Sinh viên được triển khai **WLAN** thông qua Access Point, cho phép thiết bị di động kết nối và nhận IP tự động. Cụm Server tại **VLAN 50** cung cấp ba dịch vụ cốt lõi: **DHCP** (cấp IP tự động), **DNS** (phân giải tên miền nội bộ) và **Web Server** (cổng thông tin trường), tạo nên một hệ thống mạng hoàn chỉnh, phục vụ đa dạng nhu cầu của trường đại học.
```
