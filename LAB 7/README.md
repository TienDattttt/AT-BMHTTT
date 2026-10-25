# BÁO CÁO THỰC HÀNH LAB 7: XÂY DỰNG CHÍNH SÁCH WEB PROTECTION TRÊN MÔ HÌNH PFSENSE + WEB PROXY

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên sinh viên:** Nguyễn Tiến Đạt
* **Mã số sinh viên (MSSV):** 1150080090
* **Lớp:** 11CNPM2
* **Bài thực hành:** LAB 7 - Xây dựng chính sách Web Protection trên mô hình pfSense + Web Proxy
* **Video thuyết minh YouTube:** https://youtu.be/QdbywoFuKr4

---

## 2. TỔNG QUAN VỀ MÔ HÌNH HẠ TẦNG (TOPOLOGY)
Hệ thống LAB 7 được triển khai theo kiến trúc phòng thủ theo chiều sâu (Defense-in-Depth) chuẩn doanh nghiệp, tách bạch rõ ràng giữa hai tầng kiểm soát:
- **Tầng kiểm soát luồng mạng L3/L4 (pfSense CE 2.7.2):** Đóng vai trò Firewall biên, định tuyến giữa các phân vùng mạng (WAN, MGMT, LAN, DMZ) và thiết lập ruleset kiểm soát lưu lượng nghiêm ngặt, ép buộc toàn bộ máy trạm phải đi qua Proxy và chặn đứng mọi hành vi bypass.
- **Tầng kiểm soát nội dung ứng dụng L7 (Dedicated Web Proxy - Ubuntu Server 24.04):** Chạy chuỗi xử lý 3 lớp: E2Guardian (Port 8080 - Bộ lọc nội dung, URL, từ khóa & hook Antivirus) -> Squid (Port 3128 - Caching & Forwarding Proxy) -> ClamAV Daemon (Dịch vụ quét mã độc thời gian thực).

### Bảng phân bổ máy ảo và địa chỉ IP:
| STT | Tên máy ảo | Hệ điều hành | Phân vùng mạng | Địa chỉ IP | Vai trò & Dịch vụ chính |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | **pfSense** | pfSense CE 2.7.2 | WAN / MGMT / LAN / DMZ | 192.168.56.254, 10.6.0.1, 172.16.6.1 | Tường lửa biên L3/L4, định tuyến, ép luồng Proxy và chống bypass |
| 2 | **DC01** | Windows Server 2022 | LAN Segment lab6-lan | 10.6.0.10/24 | Domain Controller (lab6.test), máy chủ DNS phân giải 13 bản ghi .lab7.test |
| 3 | **WEB-PROXY** | Ubuntu Server 24.04 | LAN Segment lab6-lan | 10.6.0.30/24 | Web Proxy L7: E2Guardian (:8080) -> Squid (:3128) -> ClamAV Daemon |
| 4 | **DMZ-WEB** | Ubuntu Server 20.04 | LAN Segment lab6-dmz | 172.16.6.10/24 | Máy chủ Web nội bộ chạy Nginx (13 Virtual Host, tệp test tải về) |
| 5 | **Client-A** | Kali Linux | LAN Segment lab6-lan | 10.6.0.21/24 | Máy trạm kiểm thử người dùng, cấu hình Manual Proxy trình duyệt |

---

## 3. CÁC CÔNG VIỆC VÀ NỘI DUNG ĐÃ THỰC HIỆN ĐƯỢC

### 3.1. Thiết lập hạ tầng và phân vùng mạng
- Xây dựng 4 phân vùng mạng riêng biệt trên pfSense: WAN (DHCP), MGMT (192.168.56.0/24), LAN (10.6.0.0/24) và DMZ (172.16.6.0/24).
- Cấu hình máy chủ DNS Windows Server DC01 (10.6.0.10) phân giải nội bộ cho 13 tên miền kiểm thử thuộc zone lab7.test trỏ về IP của máy chủ Web nội bộ 172.16.6.10.
- Thiết lập máy chủ DMZ-WEB (172.16.6.10) chạy Nginx phục vụ đồng thời 13 Virtual Host tương ứng với các danh mục kiểm thử (bình thường, nhạy cảm, tin tức, việc làm, mạng xã hội, tải file...). Tạo sẵn các file kiểm thử tải về (10MB, 100MB, file chữ ký mẫu virus EICAR).

### 3.2. Cấu hình Tường lửa pfSense chống Bypass Proxy triệt để
- Tạo các Network Aliases quản lý tập trung:
  - LAB7_CLIENTS: Dải mạng máy trạm người dùng (10.6.0.20 - 10.6.0.25).
  - LAB7_PROXY: Địa chỉ máy chủ Web Proxy chuyên dụng (10.6.0.30).
  - WEB_PORTS: Cổng web chuẩn TCP 80 (HTTP) và 443 (HTTPS).
  - MAIL_PORTS: Cổng dịch vụ thư điện tử (25, 465, 587, 110, 995, 143, 993).
- Thiết lập bộ 7 Firewall Rules nghiêm ngặt trên giao diện LAN của pfSense:
  1. Cho phép Client truy vấn DNS về DC01 (10.6.0.10:53).
  2. Cho phép Client kết nối đến cổng dịch vụ của Web Proxy (10.6.0.30:8080).
  3. Cho phép Client truy cập các cổng dịch vụ Mail Server được chỉ định.
  4. **Rule chống Bypass (Cốt lõi):** Chặn đứng (Block) toàn bộ kết nối trực tiếp từ LAB7_CLIENTS đến WEB_PORTS (80/443) đi ra ngoài Internet hoặc vào DMZ. Khi người dùng cố tình tắt cấu hình proxy trên trình duyệt, kết nối sẽ bị Firewall DROP ngay tại L3/L4.
  5. Cấp phép duy nhất cho máy chủ Web Proxy (LAB7_PROXY) được phép truy cập ra Internet qua cổng 80/443.
  6. Cấp phép cho Web Proxy truy cập vào vùng DMZ (172.16.6.10:80) để lấy dữ liệu về cho Client.
  7. Rule kiểm soát mặc định cho các dịch vụ còn lại.

### 3.3. Xây dựng chuỗi lọc nội dung Web Proxy L7 chuyên sâu
- **Cấu hình Forwarding Proxy Squid 6:**
  - Lắng nghe cục bộ tại cổng nội bộ 127.0.0.1:3128.
  - Cấu hình ACL chỉ cho phép kết nối từ localhost (chỉ nhận request được chuyển tiếp từ E2Guardian).
  - Trỏ DNS phân giải tên miền về máy chủ DC01 10.6.0.10.
- **Cấu hình Bộ lọc nội dung E2Guardian:**
  - Lắng nghe cổng công khai 10.6.0.30:8080 tiếp nhận yêu cầu từ các máy trạm LAN.
  - Chuyển tiếp luồng dữ liệu hợp lệ sang Squid tại 127.0.0.1:3128.
  - Thiết lập cơ chế lọc nội dung đa tầng: lọc URL, lọc danh sách tên miền (banned sites), lọc danh sách từ khóa nhạy cảm (banned phrases).
- **Tích hợp ClamAV Antivirus Daemon:**
  - Kích hoạt plugin contentscanner = clamdscan.conf trong E2Guardian để quét sâu vào nội dung và stream tệp tin tải về.
  - Tích hợp ClamAV Daemon với chữ ký mẫu chuẩn EICAR, phát hiện và cô lập mã độc theo thời gian thực.

---

## 4. KẾT QUẢ KIỂM THỬ CÁC CHÍNH SÁCH BẢO VỆ 

| Kịch bản | Chính sách kiểm thử | Tên miền / Mục tiêu | Kết quả thực tế đạt được |
| :---: | :--- | :--- | :--- |
| **01** | Baseline qua Proxy | 
ormal.lab7.test | Kết nối qua Proxy 10.6.0.30:8080 thành công, website tải nhanh và ổn định. |
| **02** | Cơ chế Anti-Bypass | 
ormal.lab7.test (Tắt Proxy) | Khi người dùng tắt cấu hình proxy trên trình duyệt, pfSense kích hoạt Rule L3/L4 chặn đứng kết nối trực tiếp đến cổng 80/443; trang web không thể tải (Connection timed out). |
| **03** | Chặn cứng danh mục (Hard Block) | china.lab7.test, jobs.lab7.test | E2Guardian chặn tuyệt đối, hiển thị trang thông báo Access Denied kèm biểu tượng bàn tay đỏ cảnh báo vi phạm chính sách công ty. |
| **04** | Cảnh báo & Mở tạm (Warning Bypass) | weapons.lab7.test | Hiển thị trang cảnh báo nội dung nhạy cảm kèm nút xác nhận "Acknowledge" để người dùng mở khóa tạm thời (Temporary Bypass) trong vòng 5 phút cho mục đích nghiệp vụ. |
| **05** | Danh sách trắng (Whitelist) | education.lab7.test | Được cấu hình trong danh sách Whitelist ưu tiên cao nhất; truy cập thẳng mượt mà, bỏ qua các bộ lọc từ khóa và quét sâu. |
| **06** | Giới hạn theo khung giờ (Schedule) | entertainment.lab7.test, 
ews.lab7.test | Áp dụng chính sách pfSense Schedules; tự động khóa truy cập vào giờ hành chính và mở quyền truy cập trong giờ nghỉ trưa / ngoài giờ. |
| **07** | Chặn phần mở rộng tệp tin | Tải tệp .exe, .msi, .bat | E2Guardian kiểm tra MIME Type và Extension trong luồng HTTP, chặn ngay lập tức hành vi tải tệp thực thi nguy hiểm. |
| **08** | Giới hạn dung lượng tệp tải | Tải file 110MB.bin (ngưỡng 100MB) | Chặn tải xuống các tệp vượt ngưỡng dung lượng cho phép nhằm bảo toàn băng thông hệ thống mạng doanh nghiệp. |
| **09** | Quét mã độc Antivirus (ClamAV) | Tệp mẫu eicar.com.txt | ClamAV Daemon quét luồng dữ liệu, phát hiện chính xác chữ ký EICAR-Test-Signature FOUND, E2Guardian chặn tải và thông báo virus bị phát hiện. |
| **10** | Giới hạn băng thông (Bandwidth Limit) | Tải tệp lớn 100MB.bin | Cấu hình Delay Pools trên Squid kiểm soát tốc độ tải về cho nhóm người dùng LAN, ngăn chặn nghẽn băng thông. |
| **11** | Nhật ký truy cập (Access Logging) | File log /var/log/e2guardian/access.log | Ghi nhận chi tiết từng phiên truy cập: thời gian, IP Client (10.6.0.21), tên miền, mã trạng thái HTTP, hành động (ALLOW / DENIED) và lý do vi phạm. |
| **12** | Kiểm soát Mạng xã hội | social.lab7.test | Chặn truy cập hoặc áp dụng giới hạn theo danh mục mạng xã hội trong giờ làm việc. |
| **13** | Phòng chống giao thức vượt rào | Các cổng Non-standard Web | pfSense và Proxy phối hợp đóng các cổng không xác thực, ngăn chặn người dùng sử dụng tunnel hoặc port khác để vượt rào. |

---

## 5. KẾT LUẬN & ĐÁNH GIÁ
- **Tính khả thi và hiệu quả:** Mô hình phân tách độc lập pfSense (L3/L4) và Dedicated Web Proxy (L7) giải quyết triệt để vấn đề Netgate ngừng phát triển gói Squid trên pfSense, đồng thời tối ưu hiệu năng phần cứng cho tường lửa biên.
- **Tính an toàn cao:** Cơ chế Anti-Bypass tại Firewall bảo đảm người dùng bắt buộc 100% phải tuân thủ chính sách kiểm soát nội dung; không thể tự ý thay đổi cấu hình máy trạm để qua mặt hệ thống bảo vệ.
- **Tính linh hoạt trong quản trị:** Hỗ trợ đa dạng các kịch bản: chặn cứng, cảnh báo mở tạm, danh sách trắng, giới hạn giờ giấc, giới hạn dung lượng và phòng chống mã độc chuyên sâu.