# TỔNG HỢP DỊCH VỤ VÀ BỘ CÂU HỎI VẤN ĐÁP BẢO VỆ BÀI TẬP LỚN
## MÔN: MẠNG MÁY TÍNH NÂNG CAO

**Đề tài:** Thiết kế và triển khai hệ thống mạng doanh nghiệp / trường đại học đa cơ sở (Multi-Campus) tích hợp VLAN, L3 SVI, Router-on-a-Stick, HSRP, LACP, VTP, Rapid-PVST+, WLAN, Multi-Area OSPF, NAT Overload, Extended ACL, Port Security và SSH.

---

# PHẦN 1: TỔNG HỢP CÁC DỊCH VỤ & CHỨC NĂNG TRONG MÔ HÌNH

## 1.1. Bảng tra cứu nhanh các dịch vụ kỹ thuật

| STT | Tên dịch vụ / Công nghệ | Thiết bị đảm nhiệm | Giao thức / Port | Chức năng chính trong mô hình | Lệnh cấu hình cốt lõi |
|:---:|:---|:---|:---:|:---|:---|
| **1** | **DHCP & DHCP Relay** | `Server0`, `MLSW0`, `Router1` | UDP 67/68 | Cấp phát IP tự động toàn trường, tiếp sức gói tin DHCP qua mạng WAN liên tỉnh. | `ip helper-address 192.168.50.10`<br>`ip dhcp pool POOL-VLAN...` |
| **2** | **DNS Server** | `Server0` (`192.168.50.20`) | UDP/TCP 53 | Phân giải tên miền nội bộ `www.university.local` $\rightarrow$ `192.168.50.30`. | A Record trên Server GUI |
| **3** | **Web Server** | `Server0` (`192.168.50.30`) | TCP 80 (HTTP) | Cung cấp Cổng thông tin điện tử, tra cứu điểm và lịch học cho sinh viên. | Bật HTTP Service trên Server GUI |
| **4** | **SSH v2** | Toàn bộ Router và Switch | TCP 22 | Quản trị dòng lệnh từ xa bảo mật mã hóa RSA 2048-bit, thay thế Telnet. | `ip ssh version 2`<br>`transport input ssh` |
| **5** | **Inter-VLAN (SVI)** | `MLSW0`, `MLSW1` | Layer 3 IP Routing | Định tuyến giữa các VLAN tại Trụ sở chính bằng phần cứng ASIC tốc độ cao. | `ip routing`<br>`interface Vlan10` |
| **6** | **Inter-VLAN (RoaS)**| `Router1` | 802.1Q Sub-interface| Định tuyến giữa Giảng đường (VLAN 60) và KTX (VLAN 70) tại chi nhánh. | `interface Gig3/0.60`<br>`encapsulation dot1Q 60` |
| **7** | **HSRP** | `MLSW0` (Active)<br>`MLSW1` (Standby) | UDP 1985 (v1) | Dự phòng Gateway nóng, nhóm 2 Core thành Gateway ảo `192.168.x.1`. | `standby 10 ip 192.168.10.1`<br>`standby 10 priority 110 preempt` |
| **8** | **LACP EtherChannel** | `MLSW0` $\leftrightarrow$ `MLSW1` | IEEE 802.3ad | Gom 2 cổng vật lý thành kênh logic 2Gbps, tăng băng thông và chống đứt cáp. | `channel-group 10 mode active`<br>`interface Port-channel 10` |
| **9** | **VTP v2** | `MLSW0` (Server)<br>Các Switch (Client) | Layer 2 Multicast | Đồng bộ tự động cơ sở dữ liệu VLAN toàn trường từ 1 điểm quản trị duy nhất. | `vtp mode server/client`<br>`vtp domain UNIVERSITY` |
| **10**| **STP Rapid-PVST+** | Toàn bộ Switch L2 & L3 | IEEE 802.1w | Chống vòng lặp gói tin, hội tụ < 2s, MLSW0 là Primary Root (Pri 4096). | `spanning-tree mode rapid-pvst`<br>`spanning-tree vlan ... priority 4096` |
| **11**| **PortFast & BPDU Guard**| Cổng Access Switch tầng | STP Extensions | Vào mạng tức thì khi cắm dây, tự khóa cổng nếu phát hiện switch lạ cắm trộm. | `spanning-tree portfast`<br>`spanning-tree bpduguard enable` |
| **12**| **Multi-Area OSPF** | `Router0`, `Router1`, MLSW | IP Protocol 89 | Định tuyến động phân vùng: Area 0 (Core trung tâm) $\leftrightarrow$ Area 1 (Chi nhánh). | `router ospf 1`<br>`network 10.0.2.0 0.0.0.3 area 0` |
| **13**| **NAT Overload (PAT)**| `Router0` (Edge Router) | Layer 3/4 Translation | Dịch toàn bộ dải IP Private (`192.168.0.0/16`) ra 1 IP Public truy cập Cloud1. | `ip nat inside source list 1 interface Gig0/0 overload` |
| **14**| **Extended ACL** | `MLSW0` | Layer 3/4 Packet Filter | Ngăn chặn Sinh viên/KTX chọc phá dải Quản trị IT và Đào tạo, cho phép Web/DNS.| `ip access-list extended BLOCK_STUDENT`<br>`deny ip ... permit ip any any` |
| **15**| **Port Security** | Cổng PC Switch tầng | Layer 2 MAC Filter | Khóa địa chỉ MAC (`sticky`), tự ngắt cổng (`shutdown`) nếu rút dây cắm máy lạ. | `switchport port-security mac-address sticky`<br>`switchport port-security violation shutdown` |
| **16**| **WLAN (Bridge AP)** | 10 Wireless Router WRT300N| IEEE 802.11 b/g/n | Phát Wi-Fi cho Laptop/Phone, hòa mạng trực tiếp vào VLAN mà không bị Double NAT.| Cắm cổng LAN, tắt DHCP Server nội bộ, đặt SSID |
| **17**| **Static & Default Route**| `Router0`, `Router1` | Static Routing | Tuyến mặc định đẩy dữ liệu ra Internet và tuyến chỉ định từ chi nhánh về Core. | `ip route 0.0.0.0 0.0.0.0 203.0.113.2`<br>`ip route 0.0.0.0 0.0.0.0 10.0.2.1` |

---

## 1.2. Chi tiết vai trò nghiệp vụ của từng dịch vụ

### 1. Dịch vụ cấp phát IP và chuyển tiếp DHCP (DHCP & DHCP Relay Agent)
* **Khái niệm:** Dịch vụ tự động cung cấp địa chỉ IP, Subnet Mask, Default Gateway và DNS Server cho các thiết bị đầu cuối khi hòa mạng.
* **Cơ chế hoạt động:** 
  * Máy tính gửi gói tin quảng bá `DHCP Discover` (Broadcast Layer 2/3).
  * Do Router và Switch Layer 3 chặn gói tin Broadcast, lệnh `ip helper-address 192.168.50.10` trên SVI Core Switch và Sub-interface của `Router1` sẽ đóng gói lại thành gói tin gửi đích danh (Unicast) gửi xuyên qua mạng WAN về máy chủ DHCP trung tâm (`Server0` hoặc `MLSW0`).
  * Máy chủ cấp phát IP tương ứng với dải mạng của interface gửi yêu cầu và gửi trả lại cho máy trạm.
* **Ưu điểm:** Quản lý địa chỉ IP tập trung một nơi, loại trừ hoàn toàn xung đột IP, tiết kiệm chi phí không phải trang bị máy chủ DHCP riêng tại từng chi nhánh.

### 2. Dịch vụ phân giải tên miền (DNS Server)
* **Khái niệm:** Hệ thống phân giải chuyển đổi tên miền dễ nhớ thành địa chỉ IP số học để định tuyến.
* **Cơ chế trong mô hình:** Máy chủ DNS đặt tại VLAN 50 (`192.168.50.20`), chứa bản ghi A Record phân giải tên miền `www.university.local` trỏ về IP `192.168.50.30` của Web Server.
* **Ưu điểm:** Người dùng toàn trường chỉ cần nhập tên miền vào trình duyệt web thay vì phải nhớ địa chỉ IP máy chủ.

### 3. Cổng thông tin trường học (Web Server HTTP Portal)
* **Khái niệm:** Dịch vụ web hoạt động trên giao thức HTTP cổng 80 cung cấp giao diện trực quan cho người dùng.
* **Cơ chế trong mô hình:** Cung cấp cổng thông tin đào tạo, cho phép sinh viên tra cứu thời khóa biểu, lịch thi, điểm số và thông báo từ nhà trường.
* **Ưu điểm:** Hiện đại hóa và số hóa các hoạt động truyền thông, đào tạo của trường đại học.

### 4. Quản trị từ xa an toàn (SSH Version 2)
* **Khái niệm:** Giao thức quản trị dòng lệnh từ xa được mã hóa bằng thuật toán bất đối xứng RSA 2048-bit.
* **Cơ chế trong mô hình:** Thay thế hoàn toàn giao thức Telnet truyền thống trên tất cả các Router và Switch. Người quản trị từ phòng IT (VLAN 10) có thể mở Command Prompt gõ `ssh -l admin <IP_Gateway>` để cấu hình thiết bị từ xa.
* **Ưu điểm:** Chống nghe lén thông tin đăng nhập và câu lệnh cấu hình trên đường truyền.

### 5. Định tuyến giữa các mạng ảo (Inter-VLAN Routing: SVI & Router-on-a-Stick)
* **Khái niệm:** Kỹ thuật chuyển tiếp gói tin giữa các VLAN độc lập ở tầng Network (Layer 3).
* **Mô hình kết hợp 2 giải pháp:**
  * **SVI (Switch Virtual Interface) trên Multilayer Switch:** Triển khai tại Khu Trung tâm (`MLSW0`, `MLSW1`). Các interface logic `Vlan10`, `Vlan20`... đóng vai trò Default Gateway với năng lực định tuyến bằng chip phần cứng ASIC tốc độ gigabit.
  * **Router-on-a-Stick (RoaS):** Triển khai tại Phân hiệu Chi nhánh (`Router1`). Cổng vật lý `Gig3/0` được chia thành các sub-interface `Gig3/0.60` (VLAN 60) và `Gig3/0.70` (VLAN 70) với đóng gói `encapsulation dot1Q`.
* **Ưu điểm:** Thể hiện sự linh hoạt và tối ưu chi phí — sử dụng Multilayer Switch đắt tiền tại Core trung tâm và tận dụng Router-on-a-Stick cho chi nhánh nhỏ.

### 6. Dự phòng Gateway nóng (HSRP - Hot Standby Router Protocol)
* **Khái niệm:** Giao thức dự phòng Default Gateway của Cisco, nhóm 2 Core Switch thành một Gateway ảo duy nhất (Virtual IP).
* **Cơ chế trong mô hình:**
  * `MLSW0` (Active): Đặt Priority 110, bật `preempt`, chịu trách nhiệm xử lý toàn bộ lưu lượng gửi đến Virtual IP `192.168.x.1`.
  * `MLSW1` (Standby): Đặt Priority 100, liên tục lắng nghe gói tin `Hello` (3 giây/lần). Nếu sau 10 giây (`holdtime`) không thấy MLSW0 phản hồi, MLSW1 tự động chiếm quyền Active.
* **Ưu điểm:** Đảm bảo hệ thống đạt độ sẵn sàng cao (High Availability), người dùng không bị rớt mạng khi Core chính gặp sự cố.

### 7. Gom kênh truyền tốc độ cao (LACP EtherChannel - IEEE 802.3ad)
* **Khái niệm:** Gộp nhiều liên kết vật lý song song thành một liên kết logic duy nhất.
* **Cơ chế trong mô hình:** Gộp 2 cổng `Gig1/0/2` và `Gig1/0/3` giữa MLSW0 và MLSW1 thành kênh `Port-channel 10`.
* **Ưu điểm:** Tăng băng thông kết nối liên Core lên **2 Gbps**, cân bằng tải lưu lượng và tự động chịu lỗi (nếu đứt 1 sợi cáp thì sợi còn lại vẫn hoạt động bình thường không gây gián đoạn).

### 8. Giao thức đồng bộ VLAN (VTP - VLAN Trunking Protocol)
* **Khái niệm:** Giao thức tầng 2 truyền bá và đồng bộ thông tin cấu hình VLAN trong toàn bộ hệ thống switch.
* **Cơ chế trong mô hình:** `MLSW0` là VTP Server; các switch tầng là VTP Client với chung Domain `UNIVERSITY` và mật khẩu `university123`.
* **Ưu điểm:** Tiết kiệm thời gian và loại bỏ sai sót khi cấu hình VLAN thủ công trên hàng chục switch.

### 9. Chống vòng lặp và tối ưu chuyển mạch (STP Rapid-PVST+, PortFast, BPDU Guard)
* **Khái niệm:** Giao thức ngăn chặn hiện tượng lặp vòng gói tin (Switching Loop) gây bão Broadcast.
* **Cơ chế trong mô hình:**
  * Chạy chuẩn `rapid-pvst` giảm thời gian hội tụ xuống dưới 2 giây.
  * `MLSW0` được chỉ định là Primary Root Bridge (Priority 4096), `MLSW1` là Secondary Root Bridge (Priority 8192).
  * Cổng Access nối PC/AP bật `portfast` để vào mạng ngay tức thì và bật `bpduguard enable` để tự khóa cổng nếu phát hiện có switch lạ cắm trộm vào mạng.

### 10. Định tuyến động đa vùng (Multi-Area OSPF)
* **Khái niệm:** Giao thức định tuyến trạng thái liên kết (Link-State) chuẩn mở hoạt động theo thuật toán Dijkstra.
* **Cơ chế trong mô hình:**
  * **Area 0 (Backbone Core):** Chạy giữa `Router0`, `MLSW0`, `MLSW1` và link WAN sang `Router1`.
  * **Area 1 (Branch):** Phân vùng riêng cho các mạng con tại chi nhánh (`192.168.60.0/24` và `192.168.70.0/24`) trên `Router1`.
* **Ưu điểm:** Thu hẹp phạm vi lan truyền gói tin LSA, giảm tải CPU cho các router biên và tự động tìm đường đi thay thế khi có đứt cáp.

### 11. Dịch địa chỉ mạng vùng biên (NAT Overload / PAT ra Internet)
* **Khái niệm:** Kỹ thuật chuyển đổi nhiều địa chỉ IP Private thành 1 địa chỉ IP Public duy nhất bằng cách ánh xạ qua các cổng nguồn khác nhau.
* **Cơ chế trong mô hình:** Triển khai tập trung trên `Router0` qua cổng `Gig0/0` nối ra `Cloud1` (`203.0.113.1`).
* **Ưu điểm:** Cho phép toàn bộ máy tính trong trường và chi nhánh truy cập Internet đồng thời chỉ với 1 IP Public, đồng thời che giấu dải IP nội bộ trước các nguy cơ tấn công từ mạng ngoài.

### 12. Danh sách kiểm soát an ninh đa tầng (Extended ACL)
* **Khái niệm:** Bộ quy tắc lọc gói tin dựa trên IP nguồn, IP đích, giao thức và cổng dịch vụ (Layer 3/4).
* **Cơ chế trong mô hình:** Áp dụng Extended ACL trên Core Switch:
  * Cho phép người dùng toàn trường truy cập Web Server, DNS Server và ra Internet.
  * **Chặn tuyệt đối** người dùng ở dải Sinh viên (VLAN 40) và KTX (VLAN 70) truy cập trái phép vào dải Quản trị IT (VLAN 10) và dải Dữ liệu điểm thi (VLAN 20).
* **Ưu điểm:** Ngăn chặn sinh viên chọc phá hệ thống mạng và phòng chống lây lan virus/mã độc nội bộ.

### 13. Bảo mật cổng vật lý (Port Security)
* **Khái niệm:** Tính năng bảo mật Layer 2 trên Switch giới hạn số lượng và cố định địa chỉ MAC được phép cắm vào cổng.
* **Cơ chế trong mô hình:** Cấu hình trên cổng cắm PC ở tất cả các Switch tòa nhà: `switchport port-security maximum 1`, `mac-address sticky`, `violation shutdown`.
* **Ưu điểm:** Nếu người dùng rút dây mạng cắm sang máy tính lạ, cổng Switch sẽ lập tức chuyển sang trạng thái `err-disabled` (tắt cổng hoàn toàn).

### 14. Mạng không dây nội bộ (WLAN Bridge Access Point)
* **Khái niệm:** Cung cấp kết nối không dây chuẩn IEEE 802.11 cho Laptop và Smartphone.
* **Cơ chế trong mô hình:** Cắm dây mạng vào cổng **LAN** của Wireless Router WRT300N và tắt tính năng DHCP nội bộ để biến thiết bị thành một điểm truy cập cầu nối thuần túy (Bridge AP).
* **Ưu điểm:** Người dùng bắt Wi-Fi sẽ nhận trực tiếp IP từ DHCP Server trung tâm và hòa vào đúng VLAN của tòa nhà đó, không bị lỗi mạng 2 lớp (Double NAT).

---

# PHẦN 2: BỘ CÂU HỎI VẤN ĐÁP TRỌNG TÂM & CÂU TRẢ LỜI MẪU

Dưới đây là 12 câu hỏi kinh điển mà các giảng viên chấm thi BTL Mạng máy tính thường hỏi nhất, kèm theo câu trả lời ngắn gọn, chuẩn xác và thuyết phục:

---

### ❓ CÂU HỎI 1: Tại sao trong mô hình này lại triển khai HSRP trên Switch Layer 3 mà không triển khai trên Router?
> **Trả lời:**  
> Trong mô hình này, việc định tuyến giữa các VLAN (Inter-VLAN Routing) được xử lý trực tiếp trên 2 Switch Layer 3 (`MLSW0` và `MLSW1`) thông qua các giao diện ảo SVI. Do đó, **Default Gateway của tất cả máy tính trong trường chính là các địa chỉ IP của Switch L3**.  
> Việc chạy HSRP trên cặp Switch Layer 3 giúp tạo ra một Default Gateway ảo (`192.168.x.1`) dự phòng nóng. Nếu Core chính (`MLSW0`) bị cháy nguồn hay đứt cáp, Core phụ (`MLSW1`) sẽ tự động tiếp quản Gateway ảo trong vòng vài giây, đảm bảo người dùng không bị rớt mạng. Router biên (`Router0`) chỉ làm nhiệm vụ kết nối WAN và NAT ra ngoài.

---

### ❓ CÂU HỎI 2: Sự khác nhau giữa định tuyến Inter-VLAN bằng SVI trên Switch L3 và Router-on-a-Stick trên Router1 là gì? Tại sao ở trung tâm dùng SVI mà ở chi nhánh lại dùng Router-on-a-Stick?
> **Trả lời:**  
> * **SVI trên Switch L3:** Định tuyến bằng phần cứng chuyên dụng (ASIC), băng thông chuyển mạch lên tới hàng chục Gbps, độ trễ cực thấp, không bị giới hạn bởi một đường cáp vật lý đơn lẻ. Rất thích hợp cho Khu Trung tâm nơi có lưu lượng dữ liệu khổng lồ.
> * **Router-on-a-Stick (RoaS):** Sử dụng 1 cổng vật lý trên Router chia thành nhiều Sub-interface (chuẩn 802.1Q). Toàn bộ lưu lượng giữa các VLAN phải đi lên Router rồi quay ngược lại Switch (gây hiện tượng thắt cổ chai nếu lưu lượng quá lớn).  
> * **Lý do lựa chọn:** Tại Khu Trung tâm cần hiệu năng cao nên dùng SVI. Tại Chi nhánh Phân hiệu (chỉ gồm Giảng đường và KTX với lượng máy ít), việc dùng RoaS trên `Router1` giúp **tiết kiệm chi phí đầu tư phần cứng**, không cần phải mua thêm Switch Layer 3 đắt tiền mà vẫn đáp ứng đầy đủ yêu cầu kỹ thuật.

---

### ❓ CÂU HỎI 3: Làm thế nào máy tính ở KTX (VLAN 70) và Giảng đường (VLAN 60) ở chi nhánh xa lại nhận được IP từ DHCP Server trung tâm? Cơ chế hoạt động là gì?
> **Trả lời:**  
> Bình thường, gói tin xin cấp IP của máy trạm là gói tin **Broadcast (255.255.255.255)** và sẽ bị chặn lại tại cổng của Router.  
> Để máy tính ở chi nhánh nhận được IP từ trung tâm, ta cấu hình lệnh **`ip helper-address 192.168.50.10`** trên các Sub-interface `Gig3/0.60` và `Gig3/0.70` của `Router1`. Khi nhận được gói tin DHCP Broadcast từ máy trạm, `Router1` sẽ đóng vai trò là **DHCP Relay Agent**, chuyển đổi gói tin Broadcast thành gói tin **Unicast** gửi xuyên qua mạng WAN về máy chủ DHCP trung tâm (`Server0`). Máy chủ trung tâm dựa vào địa chỉ mạng nguồn để cấp phát đúng dải IP của VLAN 60 hoặc VLAN 70 và gửi ngược lại cho máy trạm.

---

### ❓ CÂU HỎI 4: Vì sao cần chia OSPF thành Multi-Area (Area 0 và Area 1) thay vì để chung tất cả vào 1 Area?
> **Trả lời:**  
> Khi mạng mở rộng quy mô lớn, nếu đưa toàn bộ thiết bị vào một Single Area:
> 1. Cơ sở dữ liệu trạng thái liên kết (LSDB) sẽ rất lớn, khiến Router tốn nhiều bộ nhớ RAM và tiêu tốn CPU để chạy thuật toán Dijkstra tính toán bảng định tuyến.
> 2. Mỗi khi có một đường cáp ở chi nhánh bị chập chờn (flapping), toàn bộ các router trong mạng đều phải tính toán lại thuật toán SPF.
> * **Giải pháp Multi-Area:** Chia **Area 0 (Backbone)** cho Core trung tâm và **Area 1** cho Chi nhánh giúp giới hạn phạm vi lan truyền của gói tin LSA, giúp mạng hội tụ nhanh hơn, giảm tải xử lý cho Router biên và giúp hệ thống dễ dàng mở rộng thêm các chi nhánh mới trong tương lai.

---

### ❓ CÂU HỎI 5: Tại sao trên các thiết bị Wireless Router (WRT300N), dây cáp lại cắm vào cổng LAN và phải tắt DHCP nội bộ? Nếu cắm vào cổng Internet và bật DHCP thì sao?
> **Trả lời:**  
> * **Mục đích:** Cắm dây vào cổng LAN và tắt DHCP nội bộ nhằm biến Wireless Router thành một **Access Point cầu nối thuần túy (Layer 2 Bridge AP)**. Khi đó, người dùng bắt Wi-Fi sẽ hòa mạng trực tiếp vào đúng VLAN của tòa nhà và nhận IP trực tiếp từ DHCP Server trung tâm.
> * **Nếu cắm vào cổng Internet và bật DHCP:**
>   1. Sẽ xảy ra lỗi **Double NAT** (mạng 2 lớp), làm tăng độ trễ và khó khăn khi chia sẻ dữ liệu nội bộ (chia sẻ file, máy in).
>   2. Người dùng Wi-Fi sẽ nhận dải IP riêng của router gia đình (như `192.168.0.x`) thay vì dải IP của trường học, khiến quản trị viên không thể áp dụng các chính sách kiểm soát an ninh (ACL) theo từng đối tượng sinh viên hay giảng viên.

---

### ❓ CÂU HỎI 6: Hãy giải thích cách thức kiểm tra xem ACL có hoạt động chính xác hay không?
> **Trả lời:**  
> Để kiểm tra Extended ACL trên Core Switch:
> 1. **Kiểm tra chặn:** Đứng từ máy Sinh viên (VLAN 40) hoặc máy KTX (VLAN 70), thực hiện ping sang địa chỉ IP của máy Quản trị IT (`192.168.10.11`) hoặc máy Đào tạo (`192.168.20.11`). Kết quả phải báo **Destination host unreachable** $\rightarrow$ ACL đã chặn thành công.
> 2. **Kiểm tra cho phép:** Từ chính máy Sinh viên đó, thực hiện mở trình duyệt truy cập Web Server `192.168.50.30` hoặc ping `192.168.50.30` $\rightarrow$ Kết quả phải **Reply thành công** $\rightarrow$ Chứng minh ACL không chặn nhầm các dịch vụ chung.
> 3. Trên CLI của MLSW0 gõ lệnh: `show access-lists` để xem số lượng gói tin đã bị chặn (Matches).

---

### ❓ CÂU HỎI 7: Port Security hoạt động như thế nào? Khi có thiết bị lạ cắm vào cổng thì điều gì xảy ra và làm thế nào để khôi phục lại cổng đó?
> **Trả lời:**  
> * **Cơ chế:** Cổng Switch được cấu hình `switchport port-security maximum 1` và `mac-address sticky`. Cổng sẽ tự động học và lưu địa chỉ MAC của máy tính hợp lệ đầu tiên cắm vào cấu hình đang chạy.
> * **Khi vi phạm (`violation shutdown`):** Nếu ai đó rút dây cắm sang một laptop lạ (khác địa chỉ MAC đã lưu), cổng Switch lập tức chuyển sang trạng thái **`err-disabled`** (đèn cổng chuyển sang màu đỏ và bị ngắt kết nối hoàn toàn).
> * **Cách khôi phục cổng:**
>   Quản trị viên phải cắm lại máy tính hợp lệ, sau đó vào giao diện dòng lệnh của Switch gõ:
>   ```bash
>   interface <tên_cổng>
>    shutdown
>    no shutdown
>   ```
>   Cổng mới có thể hoạt động trở lại bình thường.

---

### ❓ CÂU HỎI 8: Tại sao lại dùng LACP EtherChannel giữa 2 Core Switch? Nếu 1 sợi cáp bị đứt thì điều gì xảy ra?
> **Trả lời:**  
> * **Lý do dùng LACP:** Bình thường giao thức STP sẽ khóa (Block) 1 trong 2 đường cáp song song để chống loop, dẫn đến lãng phí 50% băng thông. Giao thức LACP (802.3ad) gộp 2 cổng vật lý 1Gbps thành một đường truyền logic duy nhất (**Port-channel 10**) có băng thông **2 Gbps**, đồng thời hỗ trợ cân bằng tải.
> * **Khi 1 sợi cáp bị đứt:** Nhờ cơ chế thương lượng động của LACP, hệ thống sẽ tự động chuyển toàn bộ lưu lượng sang sợi cáp còn lại trong vòng vài mili-giây mà không làm rớt kết nối mạng và không cần STP phải tính toán lại từ đầu.

---

### ❓ CÂU HỎI 9: STP Rapid-PVST+ có vai trò gì? Tại sao phải cấu hình PortFast và BPDU Guard trên các cổng cắm PC?
> **Trả lời:**  
> * **Rapid-PVST+ (802.1w):** Chống vòng lặp gói tin ở tầng Layer 2 cho từng VLAN riêng biệt, thời gian hội tụ lại cực nhanh (< 2 giây) so với STP cổ điển (50 giây).
> * **PortFast:** Bình thường khi cắm dây, cổng switch phải trải qua các trạng thái *Listening*, *Learning* mất 30 giây mới chuyển sang *Forwarding*. Tính năng PortFast cho phép cổng nối PC/AP chuyển ngay sang trạng thái Forwarding tức thì, giúp máy trạm nhận IP DHCP nhanh chóng không bị lỗi time-out.
> * **BPDU Guard:** Đề phòng người dùng vô tình hoặc cố ý cắm một chiếc Switch cá nhân vào cổng mạng trên tường gây lỗi cấu trúc STP của trường. Khi phát hiện nhận được gói tin BPDU, BPDU Guard sẽ tự động đóng cổng (`err-disabled`) để bảo vệ toàn mạng.

---

### ❓ CÂU HỎI 10: NAT Overload (PAT) trên Router0 hoạt động như thế nào? Tại sao máy nội bộ ra Internet được nhưng từ Internet không tự ý xâm nhập vào máy nội bộ được?
> **Trả lời:**  
> * **Cơ chế:** NAT Overload sử dụng bảng ánh xạ gồm địa chỉ IP Private nguồn kết hợp với số hiệu cổng nguồn (Source Port) để chuyển thành IP Public duy nhất trên cổng WAN của `Router0`.
> * **Tính năng bảo mật 1 chiều:** 
>   * Khi máy nội bộ gửi yêu cầu ra ngoài, Router0 ghi nhớ phiên kết nối vào bảng NAT Table để khi gói tin phản hồi về, nó biết gửi trả lại cho máy nào.
>   * Ngược lại, nếu từ Internet có một hacker gửi gói tin tấn công vào địa chỉ IP Public của Router0 mà gói tin đó **không nằm trong bảng phiên kết nối đang mở**, Router0 sẽ tự động hủy bỏ (Drop) gói tin đó ngay tại cửa ngõ.

---

### ❓ CÂU HỎI 11: Tại sao trong mô hình lại bỏ đường nối trực tiếp từ Router 1 sang Cloud 1 mà chỉ để Router 0 nối ra Cloud 1?
> **Trả lời:**  
> Đây là thiết kế chuẩn mực theo mô hình **Hub-and-Spoke Enterprise Network**:
> 1. **Tối ưu chi phí:** Tránh việc phải thuê đường truyền Internet riêng và mua thêm dải địa chỉ IP Public đắt đỏ tại chi nhánh.
> 2. **Bảo mật tập trung:** Toàn bộ lưu lượng từ chi nhánh muốn ra ngoài Internet đều phải đi qua Trụ sở chính (`Router0`), giúp bộ phận an ninh mạng trung tâm dễ dàng áp dụng chính sách lọc web, phát hiện xâm nhập (IDS/IPS) và kiểm soát an toàn thông tin tại một điểm duy nhất.
> 3. **Tận dụng kênh thuê riêng:** Đảm bảo toàn bộ lưu lượng dữ liệu và xin cấp IP DHCP của chi nhánh đi thẳng tắp về Data Center trung tâm một cách ổn định nhất.

---

### ❓ CÂU HỎI 12: Tổng hợp các câu lệnh kiểm tra (Verification Commands) quan trọng nhất cần nhớ khi demo trước Hội đồng chấm thi?
> **Trả lời:**  
> Khi thầy cô yêu cầu kiểm tra trạng thái hoạt động của từng dịch vụ trên CLI, chỉ cần gõ các câu lệnh sau:
> 
> * **Kiểm tra HSRP Core:**  
>   `MLSW0# show standby brief` *(Xem trạng thái Active/Standby, Virtual IP và Priority)*
> * **Kiểm tra gộp kênh EtherChannel:**  
>   `MLSW0# show etherchannel summary` *(Phải thấy cờ `Po10(SU)` và các port `(P)`)*
> * **Kiểm tra định tuyến động OSPF:**  
>   `Router0# show ip route ospf` *(Xem các mạng nội bộ và chi nhánh được học qua OSPF)*  
>   `Router0# show ip ospf neighbor` *(Xem trạng thái kết nối láng giềng `FULL`)*
> * **Kiểm tra bảng dịch địa chỉ mạng NAT:**  
>   `Router0# show ip nat translations` *(Xem các luồng Private IP dịch sang Public IP)*
> * **Kiểm tra cơ sở dữ liệu VLAN và VTP:**  
>   `Switch# show vlan brief` *(Xem danh sách 7 VLAN)*  
>   `Switch# show vtp status` *(Xem chế độ Server/Client và Revision Number)*
> * **Kiểm tra cây chống vòng lặp STP:**  
>   `Switch# show spanning-tree brief`  
>   `MLSW0# show spanning-tree vlan 10` *(Phải có dòng "This bridge is the root")*
> * **Kiểm tra an ninh cổng Port Security:**  
>   `Switch# show port-security interface Gig7/1` *(Xem địa chỉ MAC sticky và trạng thái port)*
> * **Kiểm tra thông số IP trên máy tính:**  
>   Vào Command Prompt của PC gõ: `ipconfig /all` và gõ `ping <IP_Gateway>`
