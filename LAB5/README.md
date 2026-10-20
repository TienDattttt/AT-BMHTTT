# BÁO CÁO BÀI TẬP THỰC HÀNH LAB 5: CẤU HÌNH TƯỜNG LỬA pfSense

---

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên sinh viên:** Nguyễn Tiến Đạt   
* **Mã số sinh viên (MSSV):** 1150080090
* **Lớp:** 11CNPM2
* **Môn học:** An toàn hệ thống thông tin
* **Bài thực hành:** LAB 5 — Thiết lập mô hình tường lửa pfSense (pfSense Firewall Configuration)

---

## 2. PHIÊN BẢN MÔI TRƯỜNG & KIẾN TRÚC MẠNG
* **Phần mềm ảo hóa:** VMware Workstation Pro.
* **Tường lửa pfSense:** pfSense Community Edition (CE) phiên bản 2.7.2-RELEASE (amd64), kiến trúc FreeBSD 14.0.
* **Hệ điều hành máy trạm & kiểm thử:**
  * **Máy thật quản trị (Host):** Windows 10/11 x64.
  * **Máy Domain Controller (DC):** Windows Server 2025 (IP: `10.0.0.2/8`, Gateway: `10.0.0.1`).
  * **Máy trạm kiểm thử LAN-Test / DMZ:** Ubuntu 20.04 LTS (IP: `10.0.0.3/8` trong LAN; `172.16.0.2/16` trong DMZ).
* **Quy hoạch kiến trúc mạng (Network Topology):**
  * **Adapter 1 (WAN - em0):** Chế độ Bridged / DHCP nhận IP từ mạng ngoài.
  * **Adapter 2 (LAN - em1):** Chế độ Host-only (VMnet1), IP tĩnh `10.0.0.1/8`, tắt DHCP Server.
  * **Adapter 3 (DMZ - em2):** Chế độ LAN Segment (`dmz-net`), IP tĩnh `172.16.0.1/16`.
  * **Card Host-only máy thật:** `10.0.0.100/8`, Gateway và DNS để trống. Truy cập WebGUI quản trị tại `https://10.0.0.1`.

---

## 3. CÁCH DỰNG & QUY TRÌNH TRIỂN KHAI HỆ THỐNG
1. **Khởi tạo và cài đặt pfSense:**
   * Tạo máy ảo pfSense trên VMware với 2 vCPU, 2GB RAM, 20GB HDD (Auto ZFS stripe).
   * Cấu hình độc lập 3 card mạng: Adapter 1 (WAN), Adapter 2 (LAN: Host-only), Adapter 3 (DMZ: LAN Segment `dmz-net`).
2. **Cấu hình IP LAN qua Console:**
   * Chọn option 2 trên console pfSense, đặt IP `10.0.0.1`, subnet bit count `8`, tắt DHCP, kích hoạt HTTPS.
3. **Cấu hình WebGUI & Phân vùng DMZ:**
   * Máy thật truy cập `https://10.0.0.1` bằng tài khoản `admin`.
   * Gán interface OPT1 cho card `em2`, kích hoạt và đặt tên `DMZ`, đặt IP `172.16.0.1/16`.
4. **Kiểm tra Outbound NAT:**
   * Xác nhận tính năng Automatic Outbound NAT đã tự động sinh các quy tắc NAT cho cả 2 dải mạng nội bộ `10.0.0.0/8` và `172.16.0.0/16` ra IP WAN.
5. **Chuẩn hóa LAN Ruleset (Bắt buộc):**
   * Vô hiệu hóa (Disable) 2 rule mặc định: `Default allow LAN to any` (IPv4 và IPv6).
   * Giữ nguyên `Anti-Lockout Rule` để tránh mất quyền truy cập WebGUI.
   * Tạo Rule nền tảng do sinh viên quản lý: `Pass | LAN subnets -> Any`.
   * Reset State Table (`Diagnostics -> States -> Reset States`).

---

## 4. CÁC TÌNH HUỐNG KIỂM THỬ THỰC TẾ (PASS / FAIL)

### 4.1. Kiểm thử Rule nền tảng (Domain Controller)
* **Kịch bản BẬT rule nền tảng:** Máy Windows Server (`10.0.0.2`) ping `8.8.8.8` thành công 100% (**PASS**).
* **Kịch bản TẮT rule nền tảng + Reset States:** Máy Windows Server ping `8.8.8.8` bị `Request timed out` 100% (**PASS** - Chứng minh tường lửa chặn đúng thiết kế).

### 4.2. Tình huống 1: Chặn ICMP nhưng cho phép Web và DNS
* **Cấu hình Ruleset tab LAN:**
  1. `Block | ICMP (any) | Source: LAN subnets | Destination: Any` (Ưu tiên cao nhất)
  2. `Pass | TCP/UDP | Source: LAN subnets | Destination: Any | Port: 53 (DNS)`
  3. `Pass | TCP | Source: LAN subnets | Destination: Any | Port: 80, 443 (HTTP/HTTPS)`
* **Kết quả kiểm thử từ Windows Server:**
  * `ping 8.8.8.8`: Bị `Request timed out` (**PASS** - ICMP bị Block).
  * `nslookup google.com 8.8.8.8`: Trả về IP chính xác (**PASS** - DNS cổng 53 được duyệt).
  * `curl.exe -4 https://example.com`: Nhận mã HTML trang web (**PASS** - HTTPS cổng 443 được duyệt).

### 4.3. Tình huống 2: Chỉ cho một Host cụ thể ra Internet
* **Cấu hình Ruleset tab LAN:**
  1. `Pass | Any | Source: 10.0.0.2 | Destination: Any` (Nằm trên)
  2. `Block | Any | Source: LAN subnets | Destination: Any` (Nằm dưới)
* **Kết quả kiểm thử đối chứng:**
  * **Host 1 (Windows Server - 10.0.0.2):** `ping 8.8.8.8` thành công, nhận phản hồi Reply (**PASS**).
  * **Host 2 (Ubuntu - 10.0.0.3):** `ping 8.8.8.8` bị `100% packet loss` (**PASS** - Bị rule Block phía dưới chặn triệt để).

### 4.4. Tình huống 3: Cô lập phân vùng DMZ khỏi mạng LAN
* **Kiểm thử Baseline (Trước khi chặn):**
  * Tab DMZ chỉ có rule `Pass DMZ subnets to Any`.
  * Từ máy DMZ (`172.16.0.2`), chạy `ping 10.0.0.2` nhận phản hồi Reply thành công (**PASS** - Hai vùng thông suốt).
* **Kiểm thử Cô lập (Sau khi thêm Block):**
  * Thêm rule `Block DMZ subnets to LAN subnets` nằm TRÊN rule `Pass Any`.
  * Từ máy DMZ (`172.16.0.2`), chạy `ping 10.0.0.2` bị `100% packet loss` (**PASS** - Bị cô lập hoàn toàn khỏi LAN).
  * Từ máy DMZ (`172.16.0.2`), chạy `ping 8.8.8.8` nhận phản hồi Reply bình thường (**PASS** - Vẫn truy cập Internet bình thường).

---

## 5. LỖI GẶP PHẢI VÀ BIỆN PHÁP KHẮC PHỤC (TROUBLESHOOTING)
1. **Lỗi `Not enough information: "dev" argument is required` khi gán IP trên Ubuntu:**
   * *Nguyên nhân:* Lệnh `sudo ip addr add 10.0.0.3/8` thiếu khai báo tên interface mục tiêu.
   * *Khắc phục:* Bổ sung tên thiết bị card mạng: `sudo ip addr add 10.0.0.3/8 dev ens33`.
2. **Lỗi `connect: Network is unreachable` khi ping từ Ubuntu:**
   * *Nguyên nhân:* Bảng định tuyến chưa có default gateway trỏ về pfSense.
   * *Khắc phục:* Thêm route mặc định bằng lệnh: `sudo ip route add default via 172.16.0.1 dev ens33`.
3. **Lỗi máy DMZ (Ubuntu) không ping được gateway `172.16.0.1` và DC `10.0.0.2` dù mạng đã cắm:**
   * *Nguyên nhân:* Khi tạo rule Baseline trên tab DMZ, chọn nhầm giao thức là `IPv4 TCP`. Do `ping` chạy giao thức ICMP chứ không phải TCP nên gói tin bị pfSense chặn.
   * *Khắc phục:* Chỉnh sửa rule trên tab DMZ, chuyển Protocol từ `TCP` thành `Any` và Reset State Table.
4. **Lỗi kết nối cũ vẫn thông dù đã tắt rule firewall:**
   * *Nguyên nhân:* pfSense là stateful firewall; các phiên kết nối đã thành lập trước đó vẫn được lưu trong bảng State Table.
   * *Khắc phục:* Sau mỗi lần sửa hoặc tắt rule, luôn thực hiện thao tác xóa bảng trạng thái tại `Diagnostics -> States -> Reset States`.
