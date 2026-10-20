# BÁO CÁO THỰC HÀNH LAB 6: XÁC THỰC ACTIVE DIRECTORY VÀ CHÍNH SÁCH TRUY CẬP TRÊN PFSENSE

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên sinh viên:** Nguyễn Tiến Đạt   
* **Mã số sinh viên (MSSV):** 1150080090
* **Lớp:** 11CNPM2
* **Môn học:** An toàn hệ thống thông tin
* **Bài thực hành:** LAB 6 - Xác thực Active Directory và Chính sách truy cập trên pfSense

---

## 2. PHIÊN BẢN MÔI TRƯỜNG & HẠ TẦNG TRIỂN KHAI
* **Nền tảng ảo hóa:** VMware Workstation Pro.
* **Firewall trung tâm:** pfSense CE 2.7.2-RELEASE (amd64) trang bị 4 Network Adapter độc lập:
  * `em0 (WAN)`: Chế độ NAT (VMnet8), nhận IPv4 động qua DHCP để định tuyến Internet.
  * `em1 (MGMT)`: Chế độ Host-Only (VMnet1), IPv4 tĩnh `192.168.56.254/24` (tắt DHCP, dành riêng quản trị WebGUI).
  * `em2 (LAN)`: LAN Segment `lab6-lan`, IPv4 tĩnh `10.6.0.1/24` (tắt DHCP, vùng nội bộ).
  * `em3 (DMZ)`: LAN Segment `lab6-dmz`, IPv4 tĩnh `172.16.6.1/24` (tắt DHCP, vùng máy chủ).
* **Máy tính quản trị (Host Windows 11):**
  * Card mạng `VMware Network Adapter VMnet1` đặt IP `192.168.56.1/24` (không gateway/DNS).
  * Quản trị WebGUI pfSense qua: `https://192.168.56.254`.
* **Domain Controller (DC01):**
  * Hệ điều hành: Windows Server 2025.
  * Địa chỉ IP: `10.6.0.10/24`, Default Gateway: `10.6.0.1`, DNS: `10.6.0.10`.
  * Dịch vụ: Active Directory Domain Services (AD DS) & DNS Server.
  * Cấu trúc: Domain Forest `lab6.test`, OU `LAB6-Users`, User `sv_a`, `sv_b`, Service Account `svc_pfsense_ldap`.
* **Máy chủ dịch vụ (DMZ-WEB):**
  * Hệ điều hành: Ubuntu Server chạy Nginx.
  * IP chính: `172.16.6.10/24` (GENERAL_SITE).
  * IP phụ: `172.16.6.20/24` (JOB_SITE), `172.16.6.30/24` (SOCIAL_SITE), `172.16.6.40/24` (RESTRICTED_SITE).

---

## 3. QUY TRÌNH DỰNG VÀ CẤU HÌNH (STEP-BY-STEP)
1. **Thiết lập mô hình mạng 4 phân vùng:**
   * Cấu hình VMnet1 Host-Only dải `192.168.56.0/24` và 2 LAN Segment `lab6-lan`, `lab6-dmz`.
   * Gán 4 Adapter cho pfSense và cấu hình địa chỉ IP qua Console & WebGUI.
2. **Cấu hình Active Directory:**
   * Dựng DC01 với domain `lab6.test`, tạo OU `LAB6-Users` cùng các tài khoản `sv_a`, `sv_b`, `svc_pfsense_ldap`.
3. **Cấu hình Captive Portal:**
   * Bật Captive Portal Zone `LAB6_USERS` trên interface LAN.
4. **Tạo Firewall Aliases & Schedule:**
   * Tạo Aliases cho máy trạm (`CLIENT_A`, `CLIENT_B`, `POLICY_CLIENTS`) và Web test (`GENERAL_SITE`, `JOB_SITE`, `SOCIAL_SITE`, `RESTRICTED_SITE`, `WEB_PORTS`).
   * Tạo Schedule khung giờ ăn trưa `LUNCH_WINDOW` (11:30 - 13:00) và giờ hành chính `WORK_HOURS` (08:00 - 17:00).
5. **Triển khai Ruleset Chính sách Truy cập:**
   * Áp dụng nguyên tắc First-Match, thiết lập thứ tự ruleset tối ưu trên tab LAN và cô lập vùng DMZ khỏi LAN trên tab DMZ.


---

## 4. CÁC LỖI GẶP PHẢI VÀ BIỆN PHÁP KHẮC PHỤC (TROUBLESHOOTING)
* **Lỗi 1 - Gõ nhầm địa chỉ IP Domain Controller:**
  * *Hiện tượng:* pfSense không thể kết nối hoặc ping tới máy Windows Server.
  * *Nguyên nhân:* Nhầm lẫn giữa dải mạng `10.6.0.0/24` thành `10.0.6.10` và đặt Default Gateway trỏ về chính nó.
  * *Khắc phục:* Đặt lại IP DC01 chuẩn `10.6.0.10/24`, Gateway trỏ về pfSense `10.6.0.1`.
* **Lỗi 2 - Không thể kéo thả file vào máy ảo VMware:**
  * *Nguyên nhân:* VMware Workstation chạy quyền Administrator trong khi Windows Explorer chạy quyền thường, cơ chế bảo vệ UIPI của Windows 11 chặn kéo thả giữa hai mức đặc quyền.
  * *Khắc phục:* Thao tác trực tiếp bằng giao diện hoặc cấu hình qua công cụ đồ họa tích hợp.
* **Lỗi 3 - Trùng lặp và thứ tự ưu tiên của Firewall Rules:**
  * *Nguyên nhân:* pfSense duyệt luật Top-Down theo nguyên tắc First-Match, nếu đặt rule Pass rộng lên trên sẽ làm vô hiệu hóa rule Block cụ thể.
  * *Khắc phục:* Đặt các rule Block đặc hiệu (như cấm Job site, cấm Restricted site) lên trên cùng trước các rule Pass dịch vụ chung.
