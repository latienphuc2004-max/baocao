# BỘ CÂU HỎI & CÂU TRẢ LỜI VẤN ĐÁP BẢO VỆ BÀI TẬP LỚN
## MÔN: MẠNG MÁY TÍNH NÂNG CAO

> **Tài liệu bám sát 100% đề bài và công nghệ thực tế đã cấu hình:**
> Mô hình Doanh nghiệp / Trường Đại học Đa cơ sở (Multi-Campus) tích hợp: VLAN, L3 SVI, Router-on-a-Stick, HSRP, LACP EtherChannel, VTP, STP Rapid-PVST+, WLAN, Multi-Area OSPF, NAT Overload, Extended ACL, Port Security và SSH.

---

## MỤC LỤC BỘ CÂU HỎI

* [Phần 1: Kiến trúc mô hình, Quy mô các Site & Thiết bị](#phần-1-kiến-trúc-mô-hình-quy-mô-các-site--thiết-bị)
* [Phần 2: Chuyển mạch, Tạo/Thêm VLAN & Kết nối liên Switch (Trunk, VTP, LACP)](#phần-2-chuyển-mạch-tạothêm-vlan--kết-nối-liên-switch-trunk-vtp-lacp)
* [Phần 3: Chống vòng lặp & Bảo vệ chuyển mạch (STP Rapid-PVST+, PortFast, BPDU Guard)](#phần-3-chống-vòng-lặp--bảo-vệ-chuyển-mạch-stp-rapid-pvst-portfast-bpdu-guard)
* [Phần 4: Định tuyến Inter-VLAN, Dự phòng HSRP & OSPF đa vùng](#phần-4-định-tuyến-inter-vlan-dự-phòng-hsrp--ospf-đa-vùng)
* [Phần 5: Các dịch vụ mạng hạ tầng (DHCP Relay qua WAN, DNS, Web, WLAN)](#phần-5-các-dịch-vụ-mạng-hạ-tầng-dhcp-relay-qua-wan-dns-web-wlan)
* [Phần 6: An ninh mạng & Quản trị biên (NAT Overload, ACL, Port Security, SSH)](#phần-6-an-ninh-mạng--quản-trị-biên-nat-overload-acl-port-security-ssh)
* [Phần 7: Tình huống "Hỏi xoáy đáp xoay" & Yêu cầu thực hành trực tiếp trên CLI](#phần-7-tình-huống-hỏi-xoáy-đáp-xoay--yêu-cầu-thực-hành-trực-tiếp-trên-cli)

---

# PHẦN 1: KIẾN TRÚC MÔ HÌNH, QUY MÔ CÁC SITE & THIẾT BỊ

### ❓ Câu 1: Bài của em gồm có mấy Site (Khu vực / Cơ sở)? Vai trò và cách kết nối giữa các Site như thế nào?
* **Trả lời:**  
  Mô hình của em là **Mạng doanh nghiệp / trường đại học đa cơ sở (Multi-Campus Network)** gồm **2 Site chính và 1 Vùng biên Internet**:
  1. **Site 1: Trụ sở chính (Main Campus):** Nơi đặt Trung tâm dữ liệu (Data Center), cặp Switch lõi Layer 3 (`MLSW0`, `MLSW1`), Router biên (`Router0`) và 4 khu chức năng vệ tinh (Khu IT - VLAN 10, Đào tạo - VLAN 20, Giảng viên - VLAN 30, Sinh viên - VLAN 40).
  2. **Site 2: Phân hiệu Chi nhánh ở thành phố khác (Remote Campus):** Gồm 1 Router chi nhánh (`Router1`), Switch phân phối (`SW6`), Tòa nhà Giảng đường (VLAN 60) và Tòa nhà Ký túc xá KTX (VLAN 70).
  3. **Vùng biên Internet (`Cloud1`):** Đại diện cho đám mây nhà cung cấp dịch vụ Internet (ISP).
  * **Cách kết nối giữa các Site:**
    * Giữa các tòa nhà trong Site 1: Sử dụng **Cáp quang Single-Mode Fiber** cắm qua Switch gom `Switch0`.
    * Giữa Site 1 và Site 2: Kết nối qua đường truyền diện rộng **WAN Leased-line liên tỉnh** tốc độ cao giữa cổng `Gig1/0` của `Router0` và `Gig1/0` của `Router1` (đường mạng `10.0.2.0/30`).
    * Ra Internet: Chỉ có duy nhất `Router0` kết nối trực tiếp với `Cloud1` qua cổng `Gig0/0`.

---

### ❓ Câu 2: Tại sao không nối trực tiếp Router1 ở chi nhánh ra Cloud1 mà bắt buộc phải đi qua Router0 ở Trung tâm?
* **Trả lời:**  
  Đây là thiết kế chuẩn mực theo mô hình **Hub-and-Spoke Enterprise (Mô hình Trụ sở - Chi nhánh tập trung)**:
  1. **Tối ưu chi phí:** Không cần thuê thêm cổng Internet riêng và không cần mua thêm dải địa chỉ IP Public đắt đỏ cho chi nhánh.
  2. **Quản trị an ninh tập trung:** Toàn bộ lưu lượng chi nhánh muốn ra Internet đều đi qua `Router0`, giúp phòng Quản trị mạng tại Trung tâm kiểm soát, quét mã độc, lọc nội dung web và áp dụng tường lửa/NAT Overload đồng nhất tại 1 điểm duy nhất.
  3. **Đúng yêu cầu nghiệp vụ:** Chi nhánh cần nhận cấp phát IP từ DHCP Server trung tâm và truy cập Web/DNS nội bộ của trường, nên đường truyền nối thẳng về Trụ sở chính là đường truyền tối ưu và ổn định nhất.

---

### ❓ Câu 3: Mô hình của em áp dụng theo kiến trúc mạng phân cấp nào?
* **Trả lời:**  
  Áp dụng mô hình chuẩn **Cisco 3-Layer Hierarchical Model (Core - Distribution - Access)**:
  * **Tầng Core (Lõi):** Cặp Multilayer Switch `MLSW0` & `MLSW1`. Nhiệm vụ: Chuyển mạch và định tuyến gói tin với tốc độ cực cao (tốc độ dây - Wire speed), chạy HSRP dự phòng Gateway và OSPF.
  * **Tầng Distribution (Phân phối):** Gồm `Switch0` (tại Trung tâm), `SW1` → `SW4` (tại các khu) và `SW6` (tại chi nhánh). Nhiệm vụ: Gom lưu lượng, làm ranh giới truyền tải Trunking và định tuyến phân vùng.
  * **Tầng Access (Truy cập):** Các Switch tầng (`SW1.1`, `SW1.2`, ..., `Switch6.1`, `Switch6.2`). Nhiệm vụ: Cung cấp cổng cắm trực tiếp cho thiết bị người dùng (PC, Laptop, Wireless Router), thực thi bảo mật cổng (Port Security) và phân chia VLAN.

---

# PHẦN 2: CHUYỂN MẠCH, TẠO/THÊM VLAN & KẾT NỐI LIÊN SWITCH (TRUNK, VTP, LACP)

### ❓ Câu 4: Trong bài của em có bao nhiêu VLAN? Được tạo ở đâu và làm sao các Switch tầng dưới biết được sự tồn tại của các VLAN này?
* **Trả lời:**  
  * Hệ thống có tổng cộng **7 VLAN logic**:
    * VLAN 10: Khu Công nghệ thông tin (`192.168.10.0/24`)
    * VLAN 20: Khu Quản lý Đào tạo & Khảo thí (`192.168.20.0/24`)
    * VLAN 30: Khu Văn phòng Giảng viên (`192.168.30.0/24`)
    * VLAN 40: Khu Sinh viên & Phòng Lab (`192.168.40.0/24`)
    * VLAN 50: Khu Máy chủ Server Farm (`192.168.50.0/24`)
    * VLAN 60: Tòa Giảng đường Chi nhánh Remote (`192.168.60.0/24`)
    * VLAN 70: Tòa Ký túc xá KTX Chi nhánh Remote (`192.168.70.0/24`)
  * **Cách thức tạo và lan truyền VLAN:**
    * VLAN được tạo tập trung tại **VTP Server (`MLSW0`)**.
    * Nhờ giao thức **VTP v2 (VLAN Trunking Protocol)** với cùng Domain `UNIVERSITY` và Password `university123`, `MLSW0` sẽ phát các gói tin quảng bá VTP Advertisement qua các đường Trunk xuống tất cả các Switch còn lại (đang chạy ở chế độ **VTP Client**). Các switch tầng dưới sẽ tự động nhận diện và cập nhật toàn bộ cơ sở dữ liệu VLAN vào bộ nhớ mà không cần người quản trị phải gõ lệnh tạo VLAN thủ công trên từng Switch.

---

### ❓ Câu 5: Kết nối giữa các Switch với nhau dùng loại cổng gì? Chuẩn 802.1Q hoạt động như thế nào?
* **Trả lời:**  
  * Kết nối giữa các Switch với nhau (và giữa Switch lên Router) đều sử dụng **Cổng Trunk (Trunk Port)**. Cổng nối PC/AP dùng **Cổng Access (Access Port)**.
  * **Nguyên lý hoạt động của chuẩn IEEE 802.1Q:**
    * Khi một frame dữ liệu đi vào cổng Access thuộc một VLAN nào đó, Switch vẫn giữ nguyên cấu trúc frame chuẩn.
    * Khi frame đó cần truyền qua đường Trunk sang Switch khác, Switch gửi sẽ chèn thêm một thẻ nhận dạng **Tag 4-byte (gọi là 802.1Q Tag)** vào giữa header Ethernet, trong đó chứa trường **VLAN ID (12-bit)** xác định frame đó thuộc VLAN nào (từ 1 đến 4094).
    * Khi Switch nhận đọc được frame ở đầu bên kia, nó kiểm tra trường VLAN ID để chuyển gói tin đến đúng cổng của VLAN tương ứng, đồng thời bóc bỏ (untag) thẻ 4-byte này trước khi đẩy ra thiết bị đầu cuối.

---

### ❓ Câu 6: LACP EtherChannel là gì? Tại sao phải gộp cổng giữa MLSW0 và MLSW1? Nếu 1 sợi cáp bị đứt thì sao?
* **Trả lời:**  
  * **LACP (Link Aggregation Control Protocol - IEEE 802.3ad)** là giao thức chuẩn mở cho phép gom nhiều đường truyền vật lý song song thành một đường truyền logic duy nhất (`Port-channel`).
  * **Lý do áp dụng giữa MLSW0 và MLSW1:**
    * Bình thường nếu cắm 2 dây song song giữa 2 switch, giao thức STP sẽ khóa (Block) 1 dây để chống loop $\rightarrow$ Lãng phí 50% băng thông.
    * Khi gộp 2 cổng `Gig1/0/2` và `Gig1/0/3` thành `Port-channel 1`, STP chỉ xem đây là 1 liên kết duy nhất $\rightarrow$ Không bị khóa cổng, tăng băng thông liên Core lên **2 Gbps** và hỗ trợ cân bằng tải.
  * **Khi 1 sợi cáp bị đứt:** LACP tự động phát hiện và chuyển toàn bộ lưu lượng sang sợi cáp còn lại trong vòng vài mili-giây mà người dùng không hề bị rớt mạng và STP không phải tính toán lại.

---

# PHẦN 3: CHỐNG VÒNG LẶP & BẢO VỆ CHUYỂN MẠCH (STP RAPID-PVST+, PORTFAST, BPDU GUARD)

### ❓ Câu 7: Tại sao lại chọn Rapid-PVST+ thay vì STP cổ điển (802.1D)? Root Bridge được bầu chọn như thế nào trong bài?
* **Trả lời:**  
  * **Lý do chọn Rapid-PVST+ (IEEE 802.1w):**
    * STP cổ điển mất tới **30 đến 50 giây** để chuyển trạng thái từ Blocking sang Forwarding khi có biến động mạng.
    * Rapid-PVST+ sử dụng cơ chế bắt tay chủ động (Proposal/Agreement) giúp thời gian hội tụ mạng giảm xuống **dưới 2 giây**, đồng thời chạy tiến trình STP độc lập cho từng VLAN (Per-VLAN).
  * **Cách bầu chọn Root Bridge trong bài:**
    * Tiêu chí bầu Root Bridge: Thiết bị nào có **Bridge ID nhỏ nhất** (gồm: Priority + MAC Address).
    * Để kiểm soát hoàn toàn sơ đồ mạng, em đã can thiệp cấu hình:
      * `MLSW0`: Gán `priority 4096` cho tất cả các VLAN $\rightarrow$ Chắc chắn là **Primary Root Bridge**.
      * `MLSW1`: Gán `priority 8192` cho tất cả các VLAN $\rightarrow$ Chắc chắn là **Secondary Root Bridge** (sẵn sàng làm Root nếu MLSW0 gặp sự cố).

---

### ❓ Câu 8: PortFast và BPDU Guard hoạt động ra sao? Tại sao bật trên cổng PC mà TUYỆT ĐỐI KHÔNG bật trên cổng Trunk hay cổng cắm AP?
* **Trả lời:**  
  * **PortFast:** Bình thường cổng Switch phải trải qua các bước Listening, Learning mất 30 giây mới được truyền dữ liệu. Lệnh `spanning-tree portfast` cho phép cổng chuyển thẳng sang trạng thái **Forwarding ngay lập tức khi cắm dây**, giúp PC xin IP DHCP tức thì mà không bị timeout.
  * **BPDU Guard:** Đi kèm với PortFast. Nếu cổng Access nhận được bất kỳ gói tin BPDU nào (chứng tỏ có người cắm Switch cá nhân vào cổng này), BPDU Guard sẽ **lập tức khóa cổng (`err-disabled`)** để bảo vệ cấu trúc mạng của trường học.
  * **Tại sao cấm bật trên cổng Trunk hoặc cổng cắm AP:**
    * Cổng Trunk dùng để nối giữa các Switch, bắt buộc phải trao đổi BPDU để tính toán cây STP. Nếu bật BPDU Guard, cổng Trunk sẽ tự khóa ngay lập tức $\rightarrow$ Làm tê liệt liên kết mạng.
    * Các thiết bị Wireless Router (như WRT300N) đôi khi vẫn gửi gói tin quản trị Layer 2/BPDU. Nếu bật BPDU Guard trên cổng nối AP, cổng sẽ bị ngắt nhầm $\rightarrow$ Mất toàn bộ mạng Wi-Fi của tòa nhà.

---

# PHẦN 4: ĐỊNH TUYẾN INTER-VLAN, DỰ PHÒNG HSRP & OSPF ĐA VÙNG

### ❓ Câu 9: Trình bày sự khác nhau giữa Inter-VLAN qua SVI trên Switch L3 và Router-on-a-Stick trên Router1? Tại sao ở trung tâm dùng SVI mà ở chi nhánh dùng Router-on-a-Stick?
* **Trả lời:**  

| Tiêu chí | SVI trên Switch Layer 3 (`MLSW0`/`MLSW1`) | Router-on-a-Stick (`Router1`) |
|:---|:---|:---|
| **Cơ chế hoạt động** | Tạo interface ảo Layer 3 (`interface VlanX`) ngay bên trong Switch L3. | Dùng 1 cổng vật lý trên Router chia thành nhiều sub-interface ảo (`Gig3/0.60`, `.70`). |
| **Hiệu năng** | Định tuyến bằng phần cứng ASIC tốc độ dây (hàng chục Gbps), độ trễ cực thấp. | Định tuyến bằng CPU của Router, toàn bộ lưu lượng phải "chạy lên Router rồi chạy xuống Switch". |
| **Băng thông** | Không bị giới hạn bởi 1 cổng vật lý. | Dễ bị thắt cổ chai tại đường truyền vật lý duy nhất nối Switch - Router. |
| **Chi phí** | Đắt tiền (phải đầu tư Multilayer Switch). | Tiết kiệm chi phí phần cứng. |

* **Lý do lựa chọn trong bài:**  
  * **Tại Khu Trung tâm:** Lưu lượng trao đổi giữa 5 VLAN (sinh viên, giảng viên, đào tạo, server) là cực lớn $\rightarrow$ Bắt buộc dùng **SVI trên Multilayer Switch** để đạt hiệu năng tối đa.
  * **Tại Phân hiệu Chi nhánh:** Chỉ có 2 mạng nhỏ là Giảng đường và KTX với số lượng máy ít $\rightarrow$ Dùng **Router-on-a-Stick trên Router1** là giải pháp tối ưu nhất, vừa đáp ứng tốt nhu cầu định tuyến vừa **tiết kiệm chi phí đầu tư thiết bị** cho nhà trường.

---

### ❓ Câu 10: HSRP hoạt động như thế nào? Khi Core chính (MLSW0) bị sập thì cơ chế chuyển giao diễn ra ra sao? Vai trò của lệnh `preempt` là gì?
* **Trả lời:**  
  * **Nguyên lý HSRP (Hot Standby Router Protocol):**
    * Nhóm `MLSW0` (Active - Priority 110) và `MLSW1` (Standby - Priority 100) dùng chung một **Virtual IP** (ví dụ VLAN 10 là `192.168.10.1`) và một địa chỉ MAC ảo (`0000.0c07.acXX`).
    * Tất cả máy trạm chỉ trỏ Default Gateway về Virtual IP `192.168.x.1`.
    * Hai switch định kỳ trao đổi gói tin **Hello Multicast (3 giây/lần)**.
  * **Khi MLSW0 bị sự cố:**
    * Sau **10 giây (Holdtime)** không nhận được gói tin Hello từ MLSW0, `MLSW1` sẽ lập tức chuyển từ trạng thái `Standby` lên **`Active`**.
    * `MLSW1` gửi một gói tin Gratuitous ARP để cập nhật bảng MAC của các Switch tầng $\rightarrow$ Toàn bộ lưu lượng của người dùng tự động chuyển hướng qua MLSW1 mà không cần đổi IP Gateway trên máy trạm.
  * **Vai trò của lệnh `preempt`:**
    * Nếu không có `preempt`, khi MLSW0 phục hồi xong, nó vẫn phải đứng làm Standby dù có độ ưu tiên cao hơn (110 > 100).
    * Nhờ có lệnh **`standby X preempt`**, khi `MLSW0` bật lại, nó sẽ **tự động giành lại quyền Active** từ MLSW1, đưa hệ thống về đúng thiết kế ban đầu.

---

### ❓ Câu 11: Tại sao trong bài lại chia OSPF thành Multi-Area (Area 0 và Area 1) mà không gộp chung vào một Area duy nhất?
* **Trả lời:**  
  * Nếu đưa toàn bộ mạng vào chung **Area 0 (Single Area OSPF)**:
    1. Cơ sở dữ liệu trạng thái liên kết (LSDB) trên mỗi thiết bị sẽ rất cồng kềnh, tiêu tốn nhiều bộ nhớ RAM.
    2. Mỗi khi có 1 đường cáp ở chi nhánh bị chập chờn (link flapping), gói tin LSA loại 1/2 sẽ lan truyền khắp toàn mạng, buộc tất cả Core Switch và Router ở Trung tâm phải chạy lại thuật toán Dijkstra SPF $\rightarrow$ Gây quá tải CPU.
  * **Lợi ích khi chia Multi-Area:**
    * **Area 0 (Backbone Area):** Dành riêng cho vùng lõi Trung tâm (`Router0`, `MLSW0`, `MLSW1`) và đường link WAN.
    * **Area 1:** Dành riêng cho các mạng tại Phân hiệu Chi nhánh (`Router1`).
    * `Router1` đóng vai trò là **ABR (Area Border Router)**: Nó chỉ tóm tắt và gửi LSA dạng mạng tóm tắt (LSA Type 3) vào Area 0, giúp cô lập sự cố chập chờn tại chi nhánh, giảm tải xử lý CPU và giúp mạng hội tụ nhanh hơn gấp nhiều lần.

---

# PHẦN 5: CÁC DỊCH VỤ MẠNG HẠ TẦNG (DHCP RELAY QUA WAN, DNS, WEB, WLAN)

### ❓ Câu 12: Làm thế nào máy tính ở KTX (VLAN 70) và Giảng đường (VLAN 60) cách xa hàng chục km lại xin được IP từ DHCP Server trung tâm? Hãy giải thích cơ chế `ip helper-address`?
* **Trả lời:**  
  * **Vấn đề:** Gói tin xin cấp IP của máy tính (`DHCP Discover`) là gói tin **Broadcast Layer 2/3 (255.255.255.255)**. Theo nguyên lý, Router sẽ chặn và tiêu hủy (drop) toàn bộ các gói tin Broadcast này tại cổng vào, nên gói tin không thể tự bay qua mạng WAN về trung tâm.
  * **Giải pháp và cơ chế `ip helper-address 192.168.50.10`:**
    1. Khi máy tính ở KTX gửi gói DHCP Discover, gói tin đến cổng sub-interface `Gig3/0.70` của `Router1`.
    2. Nhờ lệnh `ip helper-address`, `Router1` đóng vai trò là **DHCP Relay Agent**: Nó nhận gói Broadcast, chèn địa chỉ IP của chính sub-interface đó (`192.168.70.1` - trường GIADDR) vào gói tin, rồi chuyển đổi thành gói tin **Unicast** có đích đến cụ thể là `192.168.50.10` (Server0) và gửi qua mạng WAN OSPF.
    3. Khi Server0 nhận được gói tin, nó nhìn vào trường GIADDR thấy địa chỉ thuộc mạng `192.168.70.0/24`, liền trích xuất dải IP từ `POOL-VLAN70` và gửi gói Unicast DHCP Offer trả ngược về cho Router1.
    4. Router1 nhận được sẽ chuyển tiếp trả về cho máy tính ở KTX $\rightarrow$ Máy nhận IP thành công.

---

### ❓ Câu 13: Trình bày quy trình 4 bước DORA của giao thức DHCP?
* **Trả lời:**  
  Quy trình cấp IP tự động gồm 4 bước (D-O-R-A):
  1. **D — Discover (Máy trạm gửi):** Máy tính mới khởi động gửi gói tin Broadcast tìm kiếm xem trong mạng có máy chủ DHCP nào không.
  2. **O — Offer (Máy chủ gửi):** DHCP Server nhận được yêu cầu, chọn 1 IP còn trống trong pool tương ứng và gửi gói tin Offer đề nghị máy trạm sử dụng IP đó.
  3. **R — Request (Máy trạm gửi):** Máy trạm đồng ý lấy IP đó và gửi thông báo chính thức xin thuê IP.
  4. **A — Acknowledge (Máy chủ gửi):** DHCP Server gửi xác nhận hoàn tất, cung cấp đầy đủ Subnet Mask, Default Gateway (`192.168.x.1`) và DNS Server (`192.168.50.20`), đồng thời khóa IP đó vào danh sách đã cấp (`DHCP Binding`).

---

### ❓ Câu 14: Tại sao trên các thiết bị Wireless Router (WRT300N), dây cáp lại cắm vào cổng LAN và phải tắt DHCP nội bộ? Nếu cắm vào cổng Internet (WAN) thì bị lỗi gì?
* **Trả lời:**  
  * **Mục đích:** Cắm dây vào cổng LAN và tắt DHCP nội bộ nhằm biến thiết bị Wireless Router thành một **Access Point cầu nối thuần túy (Layer 2 Bridge AP)**. Khi đó, sóng Wi-Fi chỉ đóng vai trò là "sợi dây mạng không dây", giúp thiết bị di động (Laptop, Smartphone) hòa mạng trực tiếp vào đúng VLAN của tòa nhà và nhận IP trực tiếp từ DHCP Server trung tâm.
  * **Nếu cắm vào cổng Internet (WAN) và bật DHCP:**
    1. **Lỗi Double NAT (Mạng 2 lớp):** Gói tin phải qua 2 lần dịch NAT (1 lần ở AP, 1 lần ở Router biên), gây tăng độ trễ và làm hỏng các dịch vụ chia sẻ file, máy in nội bộ.
    2. **Mất kiểm soát IP:** Người dùng Wi-Fi sẽ nhận dải IP riêng của router gia đình (như `192.168.0.x`) thay vì nhận IP của trường học $\rightarrow$ Khiến quản trị viên không thể áp dụng các chính sách an ninh mạng (ACL) theo từng đối tượng sinh viên hay cán bộ.

---

# PHẦN 6: AN NINH MẠNG & QUẢN TRỊ BIÊN (NAT OVERLOAD, ACL, PORT SECURITY, SSH)

### ❓ Câu 15: NAT Overload (PAT) trên Router0 hoạt động như thế nào? Tại sao máy nội bộ ra Internet được nhưng từ Internet không tự ý xâm nhập vào máy nội bộ được?
* **Trả lời:**  
  * **Cơ chế hoạt động:**
    * Lệnh: `ip nat inside source list 1 interface GigabitEthernet0/0 overload`.
    * Router0 sử dụng kỹ thuật **PAT (Port Address Translation)**. Khi máy nội bộ (ví dụ `192.168.10.15:52341`) gửi gói tin ra Internet, Router0 sẽ thay thế IP nguồn bằng IP Public của cổng `Gig0/0` (`203.0.113.1`) và gán cho nó một số hiệu cổng nguồn Public duy nhất (ví dụ `203.0.113.1:10055`), đồng thời lưu ánh xạ này vào bảng **NAT Translation Table**.
  * **Tính năng bảo mật một chiều:**
    * Khi gói tin phản hồi từ Internet gửi về `203.0.113.1:10055`, Router0 tra bảng NAT thấy khớp với phiên đang mở của `192.168.10.15` thì mới chuyển tiếp vào trong.
    * Ngược lại, nếu một tin tặc từ Internet tự ý gửi gói tin tấn công vào địa chỉ `203.0.113.1` mà gói tin đó **không nằm trong bảng phiên kết nối đang mở**, Router0 sẽ lập tức tiêu hủy (Drop) gói tin ngay tại cổng biên $\rightarrow$ Bảo vệ an toàn tuyệt đối cho toàn bộ mạng nội bộ trường học.

---

### ❓ Câu 16: ACL trong bài là loại nào? Đặt ở đâu và theo chiều nào (In hay Out)? Tại sao lại đặt như vậy?
* **Trả lời:**  
  * **Loại ACL:** Sử dụng **Extended ACL (ACL mở rộng)** có tên `BLOCK_STUDENT_ACCESS`.
  * **Vị trí và chiều đặt:** Đặt trên **SVI Vlan40 của Core Switch `MLSW0`** theo **chiều `in`** (`ip access-group BLOCK_STUDENT_ACCESS in`).
  * **Lý do lựa chọn vị trí và chiều đặt:**
    * Theo nguyên tắc vàng của Cisco: **"Standard ACL đặt gần đích, Extended ACL đặt càng gần nguồn càng tốt"**.
    * Việc áp dụng Extended ACL ngay tại cổng vào của VLAN 40 (`in`) giúp Core Switch **hủy bỏ các gói tin vi phạm ngay từ cửa ngõ** trước khi gói tin đó đi vào bảng định tuyến, giúp tiết kiệm tối đa băng thông đường truyền trục và tài nguyên xử lý CPU của Core Switch.

---

### ❓ Câu 17: Trình bày cơ chế Port Security? Khi bị vi phạm thì cổng Switch sẽ thế nào và làm cách nào để khôi phục?
* **Trả lời:**  
  * **Cơ chế:** Cấu hình trên cổng cắm PC của các Switch tầng:
    * `switchport port-security maximum 1`: Chỉ cho phép duy nhất 1 địa chỉ MAC kết nối.
    * `switchport port-security mac-address sticky`: Switch tự động "học" và dán chết địa chỉ MAC của chiếc máy tính hợp lệ đầu tiên cắm vào cổng vào file cấu hình.
    * `switchport port-security violation shutdown`: Chế độ xử lý vi phạm nghiêm ngặt nhất.
  * **Khi có vi phạm (rút dây cắm máy lạ):**
    * Khi phát hiện địa chỉ MAC lạ gửi frame vào cổng, Switch sẽ lập tức chuyển cổng sang trạng thái **`err-disabled`** (đèn cổng chuyển sang màu đỏ, cổng bị tắt hoàn toàn và gửi cảnh báo SNMP/Syslog).
  * **Cách khôi phục lại cổng:**
    * Quản trị viên phải cắm lại đúng máy tính hợp lệ, sau đó mở CLI của Switch gõ cặp lệnh:
      ```bash
      interface <tên_cổng>
       shutdown
       no shutdown
      ```
      (Bắt buộc phải gõ `shutdown` rồi mới gõ `no shutdown` thì cổng mới thoát khỏi trạng thái `err-disabled`).

---

# PHẦN 7: TÌNH HUỐNG "HỎI XOÁY ĐÁP XOAY" & YÊU CẦU THỰC HÀNH TRỰC TIẾP TRÊN CLI

### ❓ Tình huống 1: "Thầy bảo em tắt Core MLSW0 đi xem mạng có bị sập không?"
* **Cách thao tác và trả lời:**
  1. Trên một máy PC bất kỳ (ví dụ PC0), mở Command Prompt gõ lệnh ping liên tục về Gateway:
     ```cmd
     ping 192.168.10.1 -t
     ```
  2. Bấm vào thiết bị `MLSW0` $\rightarrow$ Tab **Physical** $\rightarrow$ Tắt công tắc nguồn (Power Switch).
  3. Chỉ cho thầy thấy: Trên màn hình máy PC, chỉ có khoảng 2-3 gói tin bị `Request timed out` (mất khoảng 6-9 giây để HSRP holdtime hết hạn), sau đó lập tức có **Reply trở lại bình thường**!
  4. Mở CLI của `MLSW1` gõ:
     ```bash
     show standby brief
     ```
     Chỉ cho thầy thấy cột State của MLSW1 đã chuyển từ **`Standby`** thành **`Active`**.
  5. Bật lại công tắc nguồn của `MLSW0` $\rightarrow$ Nhờ lệnh `preempt`, sau khi MLSW0 khởi động xong nó sẽ tự giành lại quyền `Active`.

---

### ❓ Tình huống 2: "Làm thế nào để chứng minh LACP EtherChannel đang chạy với băng thông 2Gbps?"
* **Cách thao tác và trả lời:**
  * Mở CLI của `MLSW0` hoặc `MLSW1` gõ lệnh:
    ```bash
    show etherchannel summary
    ```
  * Giải thích các ký hiệu cho thầy:
    * Cột nhóm hiện: `Po1(SU)` $\rightarrow$ **S** nghĩa là Layer 2, **U** nghĩa là In-use (đang hoạt động bình thường).
    * Cột Protocol hiện: **`LACP`**.
    * Cột Ports hiện: `Gig1/0/2(P)` và `Gig1/0/3(P)` $\rightarrow$ Chữ **(P)** nghĩa là "Port bundled in Port-channel" (hai cổng 1Gbps đã được gom thành công vào nhóm để tạo thành kênh 2Gbps).

---

### ❓ Tình huống 3: "Chứng minh OSPF đang định tuyến liên vùng giữa Trụ sở chính và Chi nhánh?"
* **Cách thao tác và trả lời:**
  * Mở CLI của `Router0` gõ:
    ```bash
    show ip route ospf
    ```
  * Chỉ cho thầy thấy các dòng có chữ **`O IA`** (OSPF Inter-Area):
    * `O IA 192.168.60.0/24 [110/...] via 10.0.2.2` (Mạng Giảng đường chi nhánh)
    * `O IA 192.168.70.0/24 [110/...] via 10.0.2.2` (Mạng KTX chi nhánh)
  * Giải thích: Ký hiệu `O IA` chứng minh Router0 (Area 0) đã học được các mạng từ Area 1 thông qua Router biên vùng ABR (`Router1`).

---

### ❓ Tình huống 4: "Chứng minh ACL đang chặn sinh viên nhưng không làm ảnh hưởng đến dịch vụ Web của trường?"
* **Cách thao tác và trả lời:**
  1. Đứng tại **PC của Sinh viên (VLAN 40)** hoặc **PC ở KTX (VLAN 70)**:
     * Gõ: `ping 192.168.10.11` (máy quản trị IT) $\rightarrow$ Kết quả: **`Destination host unreachable`** (Bị ACL chặn đứng).
  2. Vẫn tại chiếc máy đó:
     * Mở trình duyệt Web nhập: `http://www.university.local` $\rightarrow$ **Trang Web trường hiện lên bình thường**.
     * Hoặc gõ: `ping 192.168.50.30` (Web Server) $\rightarrow$ **Reply thành công 100%**.
  3. Mở CLI của `MLSW0` gõ:
     ```bash
     show access-lists BLOCK_STUDENT_ACCESS
     ```
     Chỉ cho thầy thấy số đếm **`(XX matches)`** ở dòng `deny` tăng lên sau mỗi lần ping chặn!

---

### ❓ Tình huống 5: "Tại sao khi vừa mở lại file Packet Tracer thì ping lần đầu tiên hay bị timeout?"
* **Cách trả lời cực kỳ ghi điểm chuyên môn:**
  > "Thưa thầy, hiện tượng timeout ở 1-2 gói tin đầu tiên khi vừa khởi động là **hoàn toàn bình thường theo đúng nguyên lý hoạt động của mạng máy tính**:
  > 1. **Tiến trình ARP:** Máy trạm chưa có địa chỉ MAC của Default Gateway trong bảng ARP Table, nó phải mất thời gian gửi gói tin ARP Request để hỏi MAC của Gateway trước khi đóng gói ICMP.
  > 2. **Sự hội tụ của các giao thức (STP, HSRP, OSPF):** Khi vừa bật file, các cổng switch phải mất vài giây để STP chuyển sang Forwarding, cặp Core Switch phải gửi Hello để bầu HSRP, và các Router phải thiết lập quan hệ Neighbor OSPF (chuyển từ INIT sang FULL).
  > Vì vậy, sau khi các bảng định tuyến và bảng MAC đã được học xong, từ gói ping thứ 2 trở đi tỉ lệ thành công luôn đạt **100%**."
