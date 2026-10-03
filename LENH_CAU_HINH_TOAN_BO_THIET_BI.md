# TỔNG HỢP LỆNH CẤU HÌNH TOÀN BỘ THIẾT BỊ
## Hệ thống mạng Trường Đại học Đa Cơ sở — Có chú thích giải thích từng dòng lệnh

> **Lưu ý:** Sau khi cấu hình xong mỗi thiết bị, luôn chạy lệnh `write memory` để lưu cấu hình vào flash. Nếu không lưu, toàn bộ cấu hình sẽ bị mất khi tắt điện / tắt file Packet Tracer.

---

## MỤC LỤC THIẾT BỊ

### 🏢 KHU TRUNG TÂM (MAIN CAMPUS)
| STT | Thiết bị | Vai trò |
|:---:|:---|:---|
| 1 | **Router0** | Edge Router — NAT, OSPF, WAN Gateway |
| 2 | **MLSW0** | Core Switch Active — HSRP, VTP Server, SVI, ACL, DHCP |
| 3 | **MLSW1** | Core Switch Standby — HSRP Standby, SVI |
| 4 | **Switch0** | Aggregation Switch — Gom cáp quang |
| 5 | **SW1** | Distribution — Khu IT |
| 6 | **SW1.1** | Access — Tòa IT 1 |
| 7 | **SW1.2** | Access — Tòa IT 2 |
| 8 | **SW2** | Distribution — Khu Đào tạo |
| 9 | **SW2.1** | Access — Tòa Đào tạo 1 |
| 10 | **SW2.2** | Access — Tòa Đào tạo 2 |
| 11 | **SW3** | Distribution — Khu Giảng viên |
| 12 | **SW3.1** | Access — Tòa Giảng viên 1 |
| 13 | **SW3.2** | Access — Tòa Giảng viên 2 |
| 14 | **SW4** | Distribution — Khu Sinh viên |
| 15 | **SW4.1** | Access — Phòng Lab 1 |
| 16 | **SW4.2** | Access — Phòng Lab 2 |

### 🏙️ KHU CHI NHÁNH REMOTE (REMOTE CAMPUS)
| STT | Thiết bị | Vai trò |
|:---:|:---|:---|
| 17 | **Router1** | Branch Router — Router-on-a-Stick, OSPF Area 1 |
| 18 | **SW6** | Distribution — Chi nhánh |
| 19 | **Switch6.1** | Access — Tòa Giảng đường |
| 20 | **Switch6.2** | Access — Tòa KTX |

### 🖥️ MÁY CHỦ & THIẾT BỊ ĐẦU CUỐI
| STT | Thiết bị | Vai trò |
|:---:|:---|:---|
| 21 | **Server0** | DHCP + DNS + Web Server |
| 22 | **Wireless Routers** | Bridge Access Point (x10) |

---
---

# KHU TRUNG TÂM (MAIN CAMPUS)

---

## 1. Router0 — Edge Router

```bash
enable                          ! Chuyển từ User Mode sang Privileged Mode (toàn quyền)
configure terminal              ! Vào chế độ cấu hình toàn cục (Global Config)
hostname Router0                ! Đặt tên thiết bị hiển thị trên dòng lệnh

! ============================================================
! PHẦN 1: CỔng WAN GIG0/0 — Kết nối ra Internet (Cloud1)
! ============================================================
interface GigabitEthernet0/0
 description WAN-to-Cloud1-Internet  ! Ghi chú mô tả cổng (để dễ nhận biết)
 ip address 203.0.113.1 255.255.255.252  ! Gán IP Public cho cổng ra Internet
                                          ! /30 = 4 IP: .0 mạng, .1 Router0, .2 Cloud, .3 broadcast
 ip nat outside                  ! Đánh dấu cổng này là "phía ngoài" của NAT
                                  ! Gói tin từ trong mạng đi ra sẽ được dịch địa chỉ tại đây
 no shutdown                     ! Bật cổng lên (mặc định cổng Router bị tắt)
exit                             ! Thoát khỏi chế độ cấu hình cổng

! ============================================================
! PHẦN 2: CỔng GIG6/0 — Kết nối xuống MLSW0 (Core Active)
! ============================================================
interface GigabitEthernet6/0
 description Link-to-MLSW0-Core-Active
 ip address 10.0.0.1 255.255.255.252  ! IP của Router0 trên link nội bộ sang MLSW0
                                        ! MLSW0 sẽ dùng IP .2 đầu còn lại
 ip nat inside                   ! Đánh dấu cổng này là "phía trong" của NAT
                                  ! Gói tin từ mạng nội bộ đi qua cổng này sẽ được NAT
 no shutdown
exit

! ============================================================
! PHẦN 3: CỔng GIG8/0 — Kết nối xuống MLSW1 (Core Standby)
! ============================================================
interface GigabitEthernet8/0
 description Link-to-MLSW1-Core-Standby
 ip address 10.0.1.1 255.255.255.252  ! IP của Router0 trên link dự phòng sang MLSW1
 ip nat inside
 no shutdown
exit

! ============================================================
! PHẦN 4: CỔng GIG1/0 — Kết nối WAN sang Router1 (Chi nhánh)
! ============================================================
interface GigabitEthernet1/0
 description WAN-Leased-Line-to-Router1-Remote
 ip address 10.0.2.1 255.255.255.252  ! IP Router0 trên đường truyền liên tỉnh
                                        ! Router1 ở chi nhánh dùng IP .2
 ip nat inside                   ! Gói tin từ chi nhánh cũng cần được NAT khi ra Internet
 no shutdown
exit

! ============================================================
! PHẦN 5: NAT OVERLOAD (PAT) — Cho phép toàn trường ra Internet
! ============================================================
access-list 1 permit 192.168.0.0 0.0.255.255
! Tạo ACL chuẩn số 1, cho phép toàn bộ dải IP Private 192.168.x.x
! Wildcard 0.0.255.255 nghĩa là 2 octet cuối có thể là bất kỳ số nào (0-255)
! Đây là danh sách các IP nội bộ được phép dịch ra ngoài Internet

ip nat inside source list 1 interface GigabitEthernet0/0 overload
! Lệnh cấu hình NAT Overload (PAT):
! - "inside source list 1": Áp dụng NAT cho các IP khớp với access-list 1
! - "interface Gig0/0": Dùng IP của cổng Gig0/0 (203.0.113.1) làm IP Public dùng chung
! - "overload": Cho phép NHIỀU IP Private dùng chung 1 IP Public bằng cách phân biệt qua số Port

! ============================================================
! PHẦN 6: OSPF AREA 0 — Định tuyến động với Core và Chi nhánh
! ============================================================
router ospf 1                    ! Khởi động tiến trình OSPF số 1
 router-id 4.4.4.4               ! Đặt Router-ID duy nhất cho Router0 trong miền OSPF
                                  ! Dạng IP nhưng chỉ là số định danh, không cần là IP thật
 network 10.0.0.0 0.0.0.3 area 0 ! Quảng bá link 10.0.0.0/30 (nối MLSW0) vào OSPF Area 0
 network 10.0.1.0 0.0.0.3 area 0 ! Quảng bá link 10.0.1.0/30 (nối MLSW1) vào OSPF Area 0
 network 10.0.2.0 0.0.0.3 area 0 ! Quảng bá link 10.0.2.0/30 (nối Router1) vào OSPF Area 0
 default-information originate   ! Quảng bá tuyến Default Route (0.0.0.0/0) cho toàn hệ thống
                                  ! Nhờ lệnh này, MLSW0, MLSW1 và Router1 đều biết đường ra Internet
exit

! ============================================================
! PHẦN 7: DEFAULT ROUTE — Tuyến đường mặc định ra Internet
! ============================================================
ip route 0.0.0.0 0.0.0.0 203.0.113.2
! Tuyến đường tĩnh mặc định: Bất kỳ gói tin nào không biết đường đi đâu
! thì chuyển sang IP 203.0.113.2 (IP phía Cloud1/ISP)
! Đây là "đường thoát cuối cùng" ra Internet

! ============================================================
! PHẦN 8: SSH V2 — Quản trị từ xa an toàn
! ============================================================
ip domain-name university.local  ! Khai báo tên miền — bắt buộc để tạo khóa RSA cho SSH
crypto key generate rsa modulus 2048
! Tạo cặp khóa mã hóa RSA 2048-bit
! 2048 bit là độ bảo mật cao, đủ tiêu chuẩn doanh nghiệp
ip ssh version 2                 ! Chỉ cho phép dùng SSH phiên bản 2 (bảo mật hơn v1)
username admin privilege 15 secret Cisco@123
! Tạo tài khoản người dùng:
! - "admin": tên đăng nhập
! - "privilege 15": quyền cao nhất (tương đương enable mode)
! - "secret": mật khẩu được mã hóa MD5 (an toàn hơn "password" lưu dạng plaintext)

line vty 0 4                     ! Cấu hình 5 đường kết nối từ xa đồng thời (0 đến 4)
 transport input ssh             ! Chỉ chấp nhận kết nối qua SSH, từ chối Telnet
 login local                     ! Dùng cơ sở dữ liệu người dùng cục bộ (username/password đã khai báo)
exit

end                              ! Thoát khỏi chế độ cấu hình, quay về Privileged Mode
write memory                     ! Lưu cấu hình từ RAM (running-config) vào Flash (startup-config)
```

---

## 2. MLSW0 — Core Switch Active

```bash
enable
configure terminal
hostname MLSW0

! ============================================================
! PHẦN 1: BẬT ĐỊNH TUYẾN LAYER 3
! ============================================================
ip routing
! Lệnh quan trọng nhất trên Multilayer Switch!
! Mặc định Switch chỉ chuyển mạch Layer 2. Lệnh này bật khả năng
! định tuyến Layer 3 (IP Routing) bằng chip phần cứng ASIC tốc độ cao
no ip cef
! Tắt Cisco Express Forwarding (CEF) vì Packet Tracer không hỗ trợ đầy đủ
! Nếu để CEF, một số tính năng như DHCP Relay có thể bị lỗi trong Packet Tracer

! ============================================================
! PHẦN 2: VTP SERVER & CƠ SỞ DỮ LIỆU VLAN
! ============================================================
vtp mode server                  ! Đặt MLSW0 làm VTP Server — nơi tạo và quản lý VLAN cho toàn trường
vtp domain UNIVERSITY            ! Tên miền VTP — tất cả Switch trong mạng phải dùng cùng tên này
vtp password university123       ! Mật khẩu xác thực VTP — ngăn Switch lạ đồng bộ vào mạng

vlan 10                          ! Tạo VLAN 10
 name IT_DEPT                    ! Đặt tên mô tả cho VLAN 10 là "IT_DEPT"
vlan 20
 name DAO_TAO                    ! VLAN 20 - Phòng Quản lý Đào tạo
vlan 30
 name GIANG_VIEN                 ! VLAN 30 - Khu Văn phòng Giảng viên
vlan 40
 name SINH_VIEN                  ! VLAN 40 - Khu Sinh viên & Phòng Lab
vlan 50
 name SERVER_FARM                ! VLAN 50 - Khu Máy chủ Dịch vụ
vlan 60
 name GIANG_DUONG_REMOTE         ! VLAN 60 - Giảng đường ở Chi nhánh Remote
vlan 70
 name KTX_REMOTE                 ! VLAN 70 - Ký túc xá ở Chi nhánh Remote
exit

! ============================================================
! PHẦN 3: STP RAPID-PVST+ — Chống vòng lặp, Primary Root Bridge
! ============================================================
spanning-tree mode rapid-pvst
! Bật chế độ STP Rapid-PVST+ (IEEE 802.1w)
! "rapid": Hội tụ nhanh < 2 giây (thay vì 50 giây của STP cổ điển)
! "pvst+": Chạy STP riêng cho từng VLAN, tối ưu đường đi theo từng VLAN
spanning-tree vlan 10,20,30,40,50,60,70 priority 4096
! Đặt Bridge Priority = 4096 cho tất cả các VLAN
! Priority mặc định là 32768, giá trị càng thấp càng được bầu làm Root Bridge
! 4096 đảm bảo MLSW0 luôn được chọn làm Primary Root Bridge

! ============================================================
! PHẦN 4: LACP ETHERCHANNEL — Gom kênh 2Gbps sang MLSW1
! ============================================================
interface range GigabitEthernet1/0/2 - 3
! Cấu hình đồng thời trên 2 cổng Gig1/0/2 và Gig1/0/3
 switchport trunk encapsulation dot1q  ! Khai báo chuẩn đóng gói Trunk là IEEE 802.1Q
 switchport mode trunk                  ! Đặt cổng về chế độ Trunk (chuyển nhiều VLAN)
 channel-group 1 mode active           ! Gán 2 cổng vào EtherChannel nhóm 1
                                        ! "active": Dùng giao thức LACP và chủ động thương lượng
 description LACP-to-MLSW1
 no shutdown
exit

interface Port-channel 1               ! Vào cấu hình kênh logic Port-channel 1 (do 2 cổng trên tạo ra)
 switchport trunk encapsulation dot1q  ! Kênh logic cũng phải cấu hình Trunk như cổng vật lý
 switchport mode trunk
 description Inter-Core-LACP-2Gbps    ! Kênh này có băng thông 1+1 = 2 Gbps
 no shutdown
exit

! ============================================================
! PHẦN 5: CỔng ROUTED PORT — Nối lên Router0
! ============================================================
interface GigabitEthernet1/0/1
 no switchport                   ! Chuyển cổng từ chế độ Switch (Layer 2) sang chế độ Routed Port (Layer 3)
                                  ! Sau lệnh này cổng có thể gán IP trực tiếp như cổng Router
 ip address 10.0.0.2 255.255.255.252  ! IP của MLSW0 trên link nội bộ với Router0
 description Uplink-to-Router0
 no shutdown
exit

! ============================================================
! PHẦN 6: CỔng TRUNK — Nối xuống Switch0 (Aggregation)
! ============================================================
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk           ! Cho phép tất cả VLAN đi qua đường truyền này xuống Switch0
 description Trunk-to-Switch0-Aggregation
 no shutdown
exit

! ============================================================
! PHẦN 7: CỔng ACCESS — Nối thẳng đến Server0 (VLAN 50)
! ============================================================
interface GigabitEthernet1/0/5
 switchport mode access          ! Cổng Access chỉ thuộc 1 VLAN duy nhất
 switchport access vlan 50       ! Gán cổng vào VLAN 50 (Server Farm)
 spanning-tree portfast          ! Bật PortFast: cổng vào Forwarding ngay, không chờ 30 giây STP
                                  ! Dùng cho cổng cắm Server/PC (thiết bị đầu cuối, không phải Switch)
 description Connected-to-Server0
 no shutdown
exit

! ============================================================
! PHẦN 8: SVI + HSRP ACTIVE — Gateway ảo cho từng VLAN
! ============================================================
! --- VLAN 10 (Khu IT) ---
interface Vlan10
! Tạo giao diện ảo Layer 3 cho VLAN 10 (Switch Virtual Interface)
 ip address 192.168.10.2 255.255.255.0  ! IP thật của MLSW0 trong VLAN 10
 ip helper-address 192.168.50.10
 ! DHCP Relay Agent: Khi nhận gói Broadcast DHCP Discover từ máy VLAN 10,
 ! chuyển thành Unicast gửi thẳng đến DHCP Server tại 192.168.50.10 (Server0)
 standby 10 ip 192.168.10.1      ! Khai báo HSRP nhóm 10, Virtual IP (Gateway ảo) là 192.168.10.1
                                  ! Đây là IP mà tất cả máy tính VLAN 10 sẽ đặt làm Default Gateway
 standby 10 priority 110         ! MLSW0 có Priority 110 > MLSW1 (100) → được bầu làm Active
 standby 10 preempt              ! Nếu MLSW0 bị tắt rồi bật lại, nó sẽ tự giành lại quyền Active
                                  ! (thay vì để MLSW1 tiếp tục làm Active mãi mãi)
 no shutdown                     ! Bật SVI lên (mặc định SVI cần bật thủ công)
exit

! --- VLAN 20 (Khu Đào tạo) ---
interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt
 no shutdown
exit

! --- VLAN 30 (Khu Giảng viên) ---
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 30 ip 192.168.30.1
 standby 30 priority 110
 standby 30 preempt
 no shutdown
exit

! --- VLAN 40 (Khu Sinh viên) ---
interface Vlan40
 ip address 192.168.40.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby 40 ip 192.168.40.1
 standby 40 priority 110
 standby 40 preempt
 no shutdown
exit

! --- VLAN 50 (Server Farm) ---
interface Vlan50
 ip address 192.168.50.2 255.255.255.0
 ! Vlan 50 không cần ip helper-address vì DHCP Server chính đặt ngay trong VLAN này
 standby 50 ip 192.168.50.1
 standby 50 priority 110
 standby 50 preempt
 no shutdown
exit

! ============================================================
! PHẦN 9: DHCP POOLS — Cấp phát IP tập trung cho toàn trường
! ============================================================
ip dhcp excluded-address 192.168.10.1 192.168.10.10
! Loại trừ dải .1 đến .10 khỏi pool cấp phát tự động
! Dải này dành để gán tĩnh cho: Gateway ảo HSRP (.1), IP MLSW0 (.2), IP MLSW1 (.3)...
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
ip dhcp excluded-address 192.168.40.1 192.168.40.10
ip dhcp excluded-address 192.168.60.1 192.168.60.10
ip dhcp excluded-address 192.168.70.1 192.168.70.10

ip dhcp pool POOL-VLAN10         ! Tạo pool DHCP tên "POOL-VLAN10"
 network 192.168.10.0 255.255.255.0  ! Dải mạng sẽ cấp phát IP từ đây
 default-router 192.168.10.1     ! Gateway mặc định gửi cho máy trạm (= Virtual IP HSRP)
 dns-server 192.168.50.20        ! Địa chỉ DNS Server gửi cho máy trạm
exit

ip dhcp pool POOL-VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.50.20
exit

ip dhcp pool POOL-VLAN30
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 192.168.50.20
exit

ip dhcp pool POOL-VLAN40
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 192.168.50.20
exit

ip dhcp pool POOL-VLAN60         ! Pool cho VLAN 60 ở Chi nhánh Remote
 network 192.168.60.0 255.255.255.0
 default-router 192.168.60.1     ! Gateway là Router1 tại chi nhánh (192.168.60.1)
 dns-server 192.168.50.20        ! DNS vẫn là Server trung tâm
exit

ip dhcp pool POOL-VLAN70
 network 192.168.70.0 255.255.255.0
 default-router 192.168.70.1
 dns-server 192.168.50.20
exit

! ============================================================
! PHẦN 10: OSPF AREA 0
! ============================================================
router ospf 1
 router-id 1.1.1.1               ! Router-ID duy nhất của MLSW0 trong miền OSPF
 network 10.0.0.0 0.0.0.3 area 0 ! Quảng bá link nối Router0 vào OSPF Area 0
 network 192.168.10.0 0.0.0.255 area 0  ! Quảng bá mạng VLAN 10 vào OSPF
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 ! Lưu ý: VLAN 60 và 70 KHÔNG quảng bá ở đây vì chúng thuộc Area 1 trên Router1
exit

ip route 0.0.0.0 0.0.0.0 10.0.0.1
! Tuyến tĩnh mặc định: Gói tin không biết đi đâu thì chuyển lên Router0 (10.0.0.1)
! Router0 sẽ thực hiện NAT và chuyển tiếp ra Internet

! ============================================================
! PHẦN 11: EXTENDED ACL — Kiểm soát truy cập an ninh mạng
! ============================================================
ip access-list extended BLOCK_STUDENT_ACCESS
! Tạo Extended ACL có tên "BLOCK_STUDENT_ACCESS"
! Extended ACL có thể lọc theo IP nguồn, IP đích, giao thức và cổng dịch vụ
 permit ip 192.168.40.0 0.0.0.255 192.168.50.0 0.0.0.255
 ! Luật 1: CHO PHÉP Sinh viên (VLAN 40) truy cập Server Farm (VLAN 50)
 ! Nhờ luật này, sinh viên vẫn vào được Web, DNS và nhận được IP từ DHCP
 permit ip 192.168.70.0 0.0.0.255 192.168.50.0 0.0.0.255
 ! Luật 2: CHO PHÉP KTX (VLAN 70) truy cập Server Farm (VLAN 50)
 deny ip 192.168.40.0 0.0.0.255 192.168.10.0 0.0.0.255
 ! Luật 3: CHẶN Sinh viên truy cập dải Quản trị IT (VLAN 10)
 deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 ! Luật 4: CHẶN Sinh viên truy cập dải Đào tạo/Điểm thi (VLAN 20)
 deny ip 192.168.70.0 0.0.0.255 192.168.10.0 0.0.0.255
 ! Luật 5: CHẶN KTX truy cập dải Quản trị IT (VLAN 10)
 deny ip 192.168.70.0 0.0.0.255 192.168.20.0 0.0.0.255
 ! Luật 6: CHẶN KTX truy cập dải Đào tạo/Điểm thi (VLAN 20)
 permit ip any any
 ! Luật cuối: Cho phép tất cả lưu lượng còn lại đi qua bình thường
 ! (Giảng viên, IT, Đào tạo vẫn đi lại tự do với nhau và ra Internet)
exit

interface Vlan40
 ip access-group BLOCK_STUDENT_ACCESS in
 ! Áp dụng ACL vào SVI VLAN 40 theo chiều "in" (vào từ phía máy sinh viên đi vào Core)
 ! Chiều "in" nghĩa là lọc gói tin KHI NÓ ĐI VÀO cổng này (từ mạng sinh viên lên Core)
exit

! ============================================================
! PHẦN 12: SSH V2
! ============================================================
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 3. MLSW1 — Core Switch Standby

```bash
enable
configure terminal
hostname MLSW1

ip routing                       ! Bật định tuyến Layer 3 (bắt buộc như MLSW0)
no ip cef                        ! Tắt CEF để tránh lỗi DHCP Relay trong Packet Tracer

! ============================================================
! VTP CLIENT — Nhận thông tin VLAN từ MLSW0 (VTP Server)
! ============================================================
vtp mode client                  ! Đặt làm Client: Không tự tạo VLAN, chỉ nhận từ Server
vtp domain UNIVERSITY            ! Phải khớp với domain của VTP Server (MLSW0)
vtp password university123       ! Phải khớp với mật khẩu VTP của Server

! ============================================================
! STP — SECONDARY ROOT BRIDGE (Dự phòng cho MLSW0)
! ============================================================
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40,50,60,70 priority 8192
! Priority = 8192 > 4096 của MLSW0
! Nên MLSW1 chỉ được bầu làm Root Bridge khi MLSW0 gặp sự cố

! ============================================================
! LACP ETHERCHANNEL — Phải cấu hình giống hệt MLSW0
! ============================================================
interface range GigabitEthernet1/0/2 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active     ! Cũng dùng "active" để 2 đầu cùng chủ động thương lượng LACP
 description LACP-to-MLSW0
 no shutdown
exit

interface Port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit

! ============================================================
! ROUTED PORT — Nối lên Router0 (đường dự phòng)
! ============================================================
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.1.2 255.255.255.252  ! IP của MLSW1 trên link dự phòng sang Router0
 description Uplink-to-Router0-Standby-Path
 no shutdown
exit

! ============================================================
! TRUNK — Nối xuống Switch0
! ============================================================
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 description Trunk-to-Switch0-Aggregation
 no shutdown
exit

! ============================================================
! SVI + HSRP STANDBY — Dự phòng Gateway cho từng VLAN
! ============================================================
interface Vlan10
 ip address 192.168.10.3 255.255.255.0  ! IP thật của MLSW1 trong VLAN 10 (khác với MLSW0 là .2)
 ip helper-address 192.168.50.10
 standby 10 ip 192.168.10.1      ! Phải cùng Virtual IP với MLSW0 để tạo thành 1 cặp HSRP
 standby 10 priority 100         ! Priority 100 < 110 của MLSW0 → MLSW1 làm Standby
 ! Không có "preempt" → Khi MLSW0 quay lại, MLSW0 mới được tự lấy lại quyền Active
 no shutdown
exit

interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 20 ip 192.168.20.1
 standby 20 priority 100
 no shutdown
exit

interface Vlan30
 ip address 192.168.30.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 30 ip 192.168.30.1
 standby 30 priority 100
 no shutdown
exit

interface Vlan40
 ip address 192.168.40.3 255.255.255.0
 ip helper-address 192.168.50.10
 standby 40 ip 192.168.40.1
 standby 40 priority 100
 no shutdown
exit

interface Vlan50
 ip address 192.168.50.3 255.255.255.0
 standby 50 ip 192.168.50.1
 standby 50 priority 100
 no shutdown
exit

! ============================================================
! OSPF AREA 0
! ============================================================
router ospf 1
 router-id 2.2.2.2               ! Router-ID duy nhất của MLSW1 (khác MLSW0 là 1.1.1.1)
 network 10.0.1.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
exit

ip route 0.0.0.0 0.0.0.0 10.0.1.1
! Tuyến tĩnh mặc định của MLSW1 đẩy về Router0 qua đường dự phòng (10.0.1.1)

ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 4. Switch0 — Aggregation Switch

```bash
enable
configure terminal
hostname Switch0

vtp mode client                  ! Nhận VLAN từ MLSW0 qua VTP
vtp domain UNIVERSITY
vtp password university123

spanning-tree mode rapid-pvst    ! Bật STP nhanh để tránh vòng lặp

! ============================================================
! 2 UPLINK LÊN CORE (MLSW0 và MLSW1)
! STP sẽ tự động chọn 1 đường là Root Port, 1 đường là Alternate
! ============================================================
interface GigabitEthernet8/1
 description Uplink-to-MLSW0-Primary     ! Đường chính lên MLSW0 (Active Core)
 switchport mode trunk           ! Cho phép tất cả VLAN đi qua uplink lên Core
 no shutdown
exit

interface GigabitEthernet9/1
 description Uplink-to-MLSW1-Backup      ! Đường dự phòng lên MLSW1 (Standby Core)
 switchport mode trunk
 no shutdown
exit

! ============================================================
! 4 DOWNLINK QUANG ĐI ĐẾN CÁC KHU CHỨC NĂNG
! Mỗi sợi cáp quang Single-Mode Fiber truyền nhiều VLAN qua Trunk
! ============================================================
interface GigabitEthernet0/1
 description SM-Fiber-to-SW1-KhuIT       ! Cáp quang nối sang Khu Công nghệ thông tin
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet1/1
 description SM-Fiber-to-SW2-KhuDaoTao
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet2/1
 description SM-Fiber-to-SW3-KhuGiangVien
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet3/1
 description SM-Fiber-to-SW4-KhuSinhVien
 switchport mode trunk
 no shutdown
exit

ip default-gateway 192.168.50.1
! Đặt Default Gateway cho Switch0 (dùng IP của Switch L2, không có ip routing)
! Dùng để Switch0 có thể nhận lệnh SSH từ quản trị viên ở mạng khác

ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 5. SW1 — Distribution Khu IT

```bash
enable
configure terminal
hostname SW1

vtp mode client                  ! Nhận VLAN từ MLSW0 qua VTP
vtp domain UNIVERSITY
vtp password university123
spanning-tree mode rapid-pvst

interface GigabitEthernet0/1
 description Uplink-to-Switch0   ! Đường quang đón từ Switch0 (Aggregation) về Core
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet1/1
 description Trunk-to-SW1.1-ToaIT1   ! Phân phối Trunk xuống Switch tầng Tòa IT 1
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet2/1
 description Trunk-to-SW1.2-ToaIT2   ! Phân phối Trunk xuống Switch tầng Tòa IT 2
 switchport mode trunk
 no shutdown
exit

ip default-gateway 192.168.10.1  ! Gateway mặc định dùng Virtual IP HSRP của VLAN 10
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 6. SW1.1 — Access Tòa IT 1

```bash
enable
configure terminal
hostname SW1.1

vtp mode client
vtp domain UNIVERSITY
vtp password university123
spanning-tree mode rapid-pvst

interface GigabitEthernet1/1
 description Uplink-to-SW1        ! Cổng đi lên SW1 (Switch phân phối khu IT)
 switchport mode trunk             ! Trunk để truyền thông tin VLAN từ Core xuống
 no shutdown
exit

! ============================================================
! CỔNG NỐI WIRELESS ROUTER — Không bật BPDU Guard
! ============================================================
interface GigabitEthernet4/1
 description AP-IT-WIFI-1         ! Access Point phát Wi-Fi cho khu IT Tòa 1
 switchport mode access            ! Access Port: chỉ truyền 1 VLAN
 switchport access vlan 10         ! Gán vào VLAN 10
 spanning-tree portfast            ! Vào mạng ngay, không chờ 30s STP
 ! Không bật BPDU Guard vì Wireless Router gửi BPDU → sẽ tắt cổng nhầm
 no shutdown
exit

! ============================================================
! CỔNG NỐI PC — Bật đầy đủ bảo mật Port Security
! ============================================================
interface GigabitEthernet5/1
 description PC0-VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast            ! Máy tính kết nối ngay lập tức, không chờ STP
 spanning-tree bpduguard enable    ! Nếu có Switch lạ cắm vào → tự khóa cổng
 switchport port-security          ! Bật tính năng Port Security trên cổng này
 switchport port-security maximum 1          ! Chỉ cho phép 1 địa chỉ MAC duy nhất
 switchport port-security mac-address sticky ! Tự học và ghi nhớ MAC đầu tiên cắm vào
 switchport port-security violation shutdown ! Nếu phát hiện MAC lạ: đóng cổng (err-disabled)
 no shutdown
exit

interface GigabitEthernet6/1
 description PC1-VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

interface GigabitEthernet7/1
 description PC2-VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

ip default-gateway 192.168.10.1
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

> **Lưu ý:** SW1.2, SW2, SW2.1, SW2.2, SW3, SW3.1, SW3.2, SW4, SW4.1, SW4.2 cấu hình **hoàn toàn tương tự** SW1 và SW1.1 — chỉ thay đổi:
> * **Hostname** (tên thiết bị)
> * **VLAN số** (20 cho Đào tạo, 30 cho Giảng viên, 40 cho Sinh viên)
> * **Số hiệu cổng** (theo đúng sơ đồ topology thực tế)
> * **Default Gateway** (192.168.20.1 / 30.1 / 40.1 tương ứng)

---

## 7—16. SW1.2 đến SW4.2 — (Cấu hình mẫu đại diện: SW4.1)

```bash
enable
configure terminal
hostname SW4.1                   ! Thay tên tương ứng: SW2, SW2.1, SW3.2, SW4.2...

vtp mode client
vtp domain UNIVERSITY
vtp password university123
spanning-tree mode rapid-pvst

interface GigabitEthernet0/1
 description Uplink-to-SW4       ! Đổi thành SW2/SW3 cho đúng khu vực
 switchport mode trunk
 no shutdown
exit

interface GigabitEthernet4/1
 description AP-SINHVIEN-WIFI-1  ! SSID: SINHVIEN-WIFI-1 (đổi theo khu vực)
 switchport mode access
 switchport access vlan 40        ! Đổi VLAN: 20 (ĐT), 30 (GV), 40 (SV)
 spanning-tree portfast
 no shutdown
exit

interface GigabitEthernet5/1
 description PC-LAB1-1-VLAN40    ! Đổi tên mô tả theo khu vực
 switchport mode access
 switchport access vlan 40        ! Đổi VLAN tương ứng
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

! (Thêm các cổng PC khác tương tự Gig6/1, Gig7/1...)

ip default-gateway 192.168.40.1  ! Đổi thành .20.1 / .30.1 / .40.1 cho đúng VLAN
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---
---

# KHU CHI NHÁNH REMOTE (REMOTE CAMPUS)

---

## 17. Router1 — Branch Router Remote

```bash
enable
configure terminal
hostname Router1

! ============================================================
! CỔNG WAN GIG1/0 — Kết nối về Trụ sở chính qua Router0
! ============================================================
interface GigabitEthernet1/0
 description WAN-Leased-Line-to-Router0
 ip address 10.0.2.2 255.255.255.252  ! IP của Router1 trên đường truyền liên tỉnh
                                        ! Router0 đầu kia dùng IP 10.0.2.1
 no shutdown
exit

! ============================================================
! CỔNG VẬT LÝ GIG3/0 — Phải bật lên TRƯỚC khi tạo sub-interface
! ============================================================
interface GigabitEthernet3/0
 description Trunk-RoaS-to-SW6   ! Cổng vật lý nối xuống SW6 bằng cáp Trunk
 no shutdown                      ! BẮT BUỘC bật cổng cha trước, sau đó sub-interface mới UP
exit

! ============================================================
! SUB-INTERFACE GIG3/0.60 — Gateway cho VLAN 60 (Giảng đường)
! ============================================================
interface GigabitEthernet3/0.60
! Sub-interface được tạo từ cổng vật lý Gig3/0
! Tên ".60" chỉ là quy ước đặt tên, không bắt buộc trùng với VLAN ID
 encapsulation dot1Q 60           ! Khai báo sub-interface này xử lý gói tin mang thẻ VLAN 60
                                  ! Khi Switch6.1 gửi gói tin VLAN 60, thẻ "60" này được đóng vào header
 ip address 192.168.60.1 255.255.255.0  ! Đây là Default Gateway của toàn bộ máy trong VLAN 60
 ip helper-address 192.168.50.10  ! DHCP Relay: Gói tin xin IP từ VLAN 60 được chuyển về Server0
 description Gateway-VLAN60-GiangDuong
exit

! ============================================================
! SUB-INTERFACE GIG3/0.70 — Gateway cho VLAN 70 (KTX)
! ============================================================
interface GigabitEthernet3/0.70
 encapsulation dot1Q 70           ! Xử lý gói tin mang thẻ VLAN 70
 ip address 192.168.70.1 255.255.255.0
 ip helper-address 192.168.50.10
 description Gateway-VLAN70-KTX
exit

! ============================================================
! OSPF MULTI-AREA — Area 0 cho WAN, Area 1 cho mạng nội bộ chi nhánh
! ============================================================
router ospf 1
 router-id 5.5.5.5               ! Router-ID duy nhất của Router1
 network 10.0.2.0 0.0.0.3 area 0
 ! Link WAN sang Router0 đặt vào Area 0 (Backbone)
 ! Nhờ đây Router1 là ABR (Area Border Router) — kết nối 2 Area khác nhau
 network 192.168.60.0 0.0.0.255 area 1
 ! Mạng VLAN 60 đặt vào Area 1 (vùng mạng chi nhánh)
 network 192.168.70.0 0.0.0.255 area 1
 ! Mạng VLAN 70 đặt vào Area 1
exit

! ============================================================
! STATIC ROUTE — Tuyến tĩnh dự phòng về Trụ sở chính
! ============================================================
ip route 0.0.0.0 0.0.0.0 10.0.2.1
! Default Route: Tất cả gói tin không biết đi đâu → chuyển về Router0 (10.0.2.1)
! Khi OSPF chưa hội tụ kịp, tuyến này đảm bảo Router1 vẫn thông với trung tâm
ip route 192.168.50.0 255.255.255.0 10.0.2.1
! Tuyến tĩnh chỉ định: Gói tin DHCP Relay đến Server0 (192.168.50.x) → đi về Router0

ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 18. SW6 — Distribution Chi nhánh

```bash
enable
configure terminal
hostname SW6

vtp mode client                  ! Nhận VLAN từ MLSW0 VTP Server
vtp domain UNIVERSITY
vtp password university123

! Khai báo VLAN thủ công phòng khi VTP chưa đồng bộ về kịp sau khi bật lại
vlan 60
 name GIANG_DUONG
vlan 70
 name KTX
exit

spanning-tree mode rapid-pvst
spanning-tree vlan 60,70 priority 4096
! Đặt SW6 làm Root Bridge cho VLAN 60 và 70 trong vùng chi nhánh
! Giúp đường đi từ Switch6.1 / Switch6.2 lên SW6 là đường tối ưu nhất

! ============================================================
! CỔNG TRUNK LÊN ROUTER1 — Phía Router làm RoaS
! ============================================================
interface GigabitEthernet3/1
 description Uplink-to-Router1-RoaS      ! Lên Router1 để thực hiện Inter-VLAN
 switchport mode trunk            ! Cổng Trunk truyền cả VLAN 60 và 70 lên Router1
 switchport trunk allowed vlan 60,70      ! Chỉ cho phép 2 VLAN này đi qua, lọc bỏ các VLAN khác
 no shutdown
exit

! ============================================================
! CỔNG TRUNK XUỐNG SWITCH6.1 (Giảng đường)
! ============================================================
interface GigabitEthernet1/1
 description Trunk-to-Switch6.1-GiangDuong
 switchport mode trunk
 switchport trunk allowed vlan 60         ! Chỉ cần truyền VLAN 60 xuống tòa Giảng đường
 no shutdown
exit

! ============================================================
! CỔNG TRUNK XUỐNG SWITCH6.2 (KTX)
! ============================================================
interface GigabitEthernet0/1
 description Trunk-to-Switch6.2-KTX
 switchport mode trunk
 switchport trunk allowed vlan 70         ! Chỉ cần truyền VLAN 70 xuống tòa KTX
 no shutdown
exit

ip default-gateway 192.168.60.1  ! Default Gateway quản trị SSH qua Router1
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 19. Switch6.1 — Access Tòa Giảng đường

```bash
enable
configure terminal
hostname Switch6.1

vtp mode client
vtp domain UNIVERSITY
vtp password university123

vlan 60
 name GIANG_DUONG
exit                             ! Khai báo thủ công phòng khi VTP chưa đồng bộ kịp

spanning-tree mode rapid-pvst

! ============================================================
! UPLINK TRUNK LÊN SW6
! ============================================================
interface GigabitEthernet1/1
 description Uplink-to-SW6
 switchport mode trunk
 switchport trunk allowed vlan 60
 no shutdown
exit

! ============================================================
! CỔNG ACCESS CHO PC 20 — Bật đầy đủ Port Security
! ============================================================
interface GigabitEthernet7/1
 description PC20-GiangDuong-VLAN60
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast           ! Máy tính vào mạng ngay không chờ STP
 spanning-tree bpduguard enable   ! Nếu phát hiện Switch cắm vào → khóa cổng ngay
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

! ============================================================
! CỔNG ACCESS CHO WIRELESS ROUTER 8 — Không bật BPDU Guard
! ============================================================
interface GigabitEthernet6/1
 description AP-GIANG-DUONG-WIFI  ! Access Point phát sóng SSID: GIANG-DUONG-WIFI
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
 ! Không bật bpduguard: Wireless Router WRT300N đôi khi gửi BPDU
 ! Nếu bật BPDU Guard sẽ tự khóa cổng AP và mất Wi-Fi!
 no shutdown
exit

ip default-gateway 192.168.60.1
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---

## 20. Switch6.2 — Access Tòa KTX

```bash
enable
configure terminal
hostname Switch6.2

vtp mode client
vtp domain UNIVERSITY
vtp password university123

vlan 70
 name KTX
exit

spanning-tree mode rapid-pvst

interface GigabitEthernet0/1
 description Uplink-to-SW6       ! Cổng Trunk lên SW6 (phân phối chi nhánh)
 switchport mode trunk
 switchport trunk allowed vlan 70
 no shutdown
exit

interface GigabitEthernet7/1
 description PC21-KTX-VLAN70
 switchport mode access
 switchport access vlan 70
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 no shutdown
exit

interface GigabitEthernet6/1
 description AP-KTX-WIFI          ! Access Point phát sóng SSID: KTX-WIFI
 switchport mode access
 switchport access vlan 70
 spanning-tree portfast
 ! Không bật bpduguard cho cổng AP
 no shutdown
exit

ip default-gateway 192.168.70.1
ip domain-name university.local
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret Cisco@123
line vty 0 4
 transport input ssh
 login local
exit

end
write memory
```

---
---

# MÁY CHỦ & THIẾT BỊ ĐẦU CUỐI

---

## 21. Server0 — DHCP + DNS + Web Server

> Cấu hình hoàn toàn qua giao diện **GUI** (Click vào Server → Tab Services). Không dùng CLI.

### 🔹 Bước 1: Gán IP tĩnh cho Server0
```
Click Server0 → Tab "Desktop" → "IP Configuration"
  Chọn "Static" (IP cố định, không dùng DHCP)
  IP Address   : 192.168.50.10   ← IP của DHCP Server
  Subnet Mask  : 255.255.255.0
  Default GW   : 192.168.50.1    ← Virtual IP HSRP của VLAN 50
  DNS Server   : 192.168.50.20   ← Trỏ sang DNS Server
```

### 🔹 Bước 2: Cấu hình DHCP Service
```
Click Server0 → Tab "Services" → "DHCP"
  Bật Service: ON (chuyển nút sang ON)
  
  Xóa pool mặc định "serverPool" nếu có (click chọn → Delete)
  
  Thêm pool VLAN10:
    Pool Name   : POOL-VLAN10
    Network     : 192.168.10.0    ← Dải mạng cấp IP
    Subnet Mask : 255.255.255.0
    Default GW  : 192.168.10.1    ← Virtual IP HSRP (máy sẽ dùng làm Gateway)
    DNS Server  : 192.168.50.20
    Start IP    : 192.168.10.11   ← Bắt đầu cấp từ .11 (tránh .1-.10 đã dùng)
    → Bấm nút "Add"

  Tương tự thêm pool cho VLAN20 (192.168.20.x / GW .20.1 / Start .20.11)
  Tương tự thêm pool cho VLAN30 (192.168.30.x / GW .30.1 / Start .30.11)
  Tương tự thêm pool cho VLAN40 (192.168.40.x / GW .40.1 / Start .40.11)
  Tương tự thêm pool cho VLAN60 (192.168.60.x / GW .60.1 / Start .60.11)
  Tương tự thêm pool cho VLAN70 (192.168.70.x / GW .70.1 / Start .70.11)
```

### 🔹 Bước 3: Cấu hình DNS Service (trên máy DNS Server - IP 192.168.50.20)
```
Click DNS-Server → Tab "Services" → "DNS"
  Bật Service: ON
  
  Thêm bản ghi A Record:
    Name    : www.university.local  ← Tên miền người dùng nhập vào trình duyệt
    Type    : A Record              ← Loại bản ghi ánh xạ tên → IP
    Address : 192.168.50.30         ← IP của Web Server
    → Bấm nút "Add"
```

### 🔹 Bước 4: Cấu hình Web Server (trên máy Web Server - IP 192.168.50.30)
```
Click Web-Server → Tab "Services" → "HTTP"
  Bật HTTP  Service: ON    ← Cổng 80, không mã hóa
  Bật HTTPS Service: OFF   ← Không bắt buộc trong Packet Tracer
  
  Có thể sửa nội dung file "index.html" bằng cách click vào tên file
  và chỉnh sửa nội dung trang chủ Web Portal của trường
```

---

## 22. Wireless Routers — Bridge AP (Tất cả 10 thiết bị)

> Cấu hình qua giao diện **GUI**. Không dùng CLI.

### 🔹 Cấu hình chung cho MỌI Wireless Router:

```
Bước 1: Click Wireless Router → Tab "GUI"

Bước 2: Vào "Setup" → "Basic Setup"
  → Mục "Internet Connection Type": Chọn "Disabled"
    (Tắt chức năng Router/NAT nội bộ — biến thiết bị thành AP thuần)
  → Mục "DHCP Server": Chọn "Disabled"
    (Tắt DHCP nội bộ — máy kết nối Wi-Fi sẽ xin IP từ Server0 trung tâm)
  → Bấm "Save Settings"

Bước 3: Vào "Wireless" → "Basic Wireless Settings"
  → Network Name (SSID): [Đặt tên theo bảng bên dưới]
  → Standard Channel: 6 (hoặc chọn kênh khác tránh nhiễu)
  → Bấm "Save Settings"

Bước 4 — RẤT QUAN TRỌNG — Kiểm tra kết nối dây:
  → Dây mạng PHẢI cắm vào cổng "Ethernet 1" (cổng LAN có nhãn số 1,2,3,4)
  → TUYỆT ĐỐI KHÔNG cắm vào cổng "Internet" (cổng màu khác / WAN port)
  → Lý do: Cắm vào cổng LAN để AP hoạt động ở chế độ cầu nối (Layer 2 Bridge)
            Cắm vào cổng Internet sẽ kích hoạt NAT → gây lỗi Double NAT
```

### 🔹 Bảng SSID cho từng Wireless Router:

| Thiết bị | SSID | VLAN | Switch cắm vào | Cổng Switch |
|:---|:---|:---:|:---|:---:|
| WR khu IT Tòa 1 | `IT-WIFI-1` | 10 | SW1.1 | Gig4/1 |
| WR khu IT Tòa 2 | `IT-WIFI-2` | 10 | SW1.2 | Gig6/1 |
| WR khu Đào tạo 1 | `DAOTAO-WIFI-1` | 20 | SW2.1 | Gig4/1 |
| WR khu Đào tạo 2 | `DAOTAO-WIFI-2` | 20 | SW2.2 | Gig6/1 |
| WR khu Giảng viên 1 | `GIANGVIEN-WIFI-1` | 30 | SW3.1 | Gig4/1 |
| WR khu Giảng viên 2 | `GIANGVIEN-WIFI-2` | 30 | SW3.2 | Gig6/1 |
| WR khu Sinh viên 1 | `SINHVIEN-WIFI-1` | 40 | SW4.1 | Gig4/1 |
| WR khu Sinh viên 2 | `SINHVIEN-WIFI-2` | 40 | SW4.2 | Gig6/1 |
| Wireless Router 8 | `GIANG-DUONG-WIFI` | 60 | Switch6.1 | Gig6/1 |
| Wireless Router 9 | `KTX-WIFI` | 70 | Switch6.2 | Gig6/1 |

---
---

# PHỤ LỤC: LỆNH KIỂM TRA NHANH (SHOW COMMANDS)

```bash
! ===== KIỂM TRA HSRP =====
MLSW0# show standby brief
! Xem: MLSW0 phải có State = "Active" | MLSW1 phải có State = "Standby"
! Cột "Active addr" phải là IP thật của MLSW0 (192.168.x.2)
! Cột "Virtual addr" phải là Virtual IP (192.168.x.1)

! ===== KIỂM TRA ETHERCHANNEL =====
MLSW0# show etherchannel summary
! Xem: Po1(SU) → S=Layer2, U=in-use (đang hoạt động)
! Gig1/0/2(P) và Gig1/0/3(P) → P=bundled (đã gom vào kênh thành công)

! ===== KIỂM TRA OSPF =====
Router0# show ip ospf neighbor
! Xem: State phải là "FULL" → 2 Router đã trao đổi đủ thông tin định tuyến
! Nếu State là "INIT" hay "2WAY" → OSPF chưa hội tụ xong, chờ thêm

Router0# show ip route ospf
! Xem: Dòng "O" = học từ OSPF cùng Area | "O IA" = học từ Area khác (Inter-Area)
! Phải thấy O IA 192.168.60.0/24 và O IA 192.168.70.0/24 (mạng chi nhánh)

! ===== KIỂM TRA VLAN & VTP =====
MLSW0# show vlan brief
! Xem: Tất cả 7 VLAN (10,20,30,40,50,60,70) phải ở trạng thái "active"

MLSW0# show vtp status
! Xem: VTP Operating Mode = Server | VTP Domain Name = UNIVERSITY
! Configuration Revision phải > 0 (có ít nhất 1 lần thay đổi VLAN)

! ===== KIỂM TRA STP =====
MLSW0# show spanning-tree vlan 10
! Xem: "This bridge is the root" → MLSW0 đang là Root Bridge VLAN 10

Switch# show spanning-tree brief
! Xem: Tất cả cổng phải ở trạng thái FWD (Forwarding)
! Nếu thấy BLK (Blocking) → cổng đó bị STP chặn để tránh loop (bình thường)

! ===== KIỂM TRA NAT =====
Router0# show ip nat translations
! Xem: Cột "Inside global" phải là 203.0.113.1 (IP Public)
! Cột "Inside local" là IP Private của máy đang truy cập Internet

Router0# show ip nat statistics
! Xem: Hits (số lượt dịch thành công) tăng dần khi có máy ra Internet

! ===== KIỂM TRA ACL =====
MLSW0# show access-lists
! Xem: Mỗi dòng lệnh permit/deny có thêm "(X matches)" = số gói tin đã khớp
! Nếu deny matches > 0 → ACL đang chặn gói tin thành công

! ===== KIỂM TRA PORT SECURITY =====
Switch# show port-security interface GigabitEthernet7/1
! Xem: Port Status = Secure-up (bình thường) hoặc Secure-shutdown (bị kích hoạt)
! Last Source Address: Địa chỉ MAC đã học (sticky)
! Security Violation Count: Số lần vi phạm

! ===== KIỂM TRA GIAO DIỆN IP =====
MLSW0# show ip interface brief
! Xem: TẤT CẢ cổng Vlan và cổng vật lý phải có:
! Status = "up" | Protocol = "up"
! Nếu Status = "administratively down" → cần chạy "no shutdown"

! ===== KIỂM TRA BẢNG ĐỊNH TUYẾN ĐẦY ĐỦ =====
Router0# show ip route
! Xem toàn bộ bảng định tuyến: C=Connected, S=Static, O=OSPF, O IA=OSPF Inter-Area

! ===== KIỂM TRA DHCP HOẠT ĐỘNG =====
MLSW0# show ip dhcp binding
! Xem: Danh sách các máy đã được cấp IP (IP Address, MAC Address, VLAN)

MLSW0# show ip dhcp pool
! Xem: Thống kê từng pool (Total/Allocated/Available)
```
