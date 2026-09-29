# BÁO CÁO THỰC HÀNH LAB 3
## NHẬN DIỆN VÀ ỨNG PHÓ CÁC MỐI ĐE DỌA AN TOÀN THÔNG TIN

---

### 1. Thông Tin Sinh Viên
- **Họ và tên:** Nguyễn Tiến Đạt
- **MSSV:** 1150080090
- **Lớp:** 11CNPM2
- **Môn học:** An toàn Hệ thống Thông tin
- **Giảng viên hướng dẫn:** ThS. Phạm Trọng Huynh

---

### 2. Môi Trường & Công Cụ Thực Hành
- **Hệ điều hành:** Windows 11 x64, OS Build 26200
- **Bảo vệ điểm cuối (Endpoint Protection):** Microsoft Defender Antivirus (RealTimeProtection: True, TamperProtection: True)
- **Tường lửa (Firewall):** Windows Defender Firewall bật trên cả 3 mạng (Domain, Private, Public)
- **Danh sách phiên bản công cụ chuẩn hóa:**
  - Python: `3.14.7`
  - Wireshark / TShark: `4.6.8`
  - Microsoft Sysmon: `15.22` (Schema 4.90)
  - Microsoft Autoruns: `14.3`
  - Microsoft Process Explorer: `17.14`

---

### 3. Các Bước Dựng Môi Trường
1. **Khởi tạo thư mục làm việc:** Tạo thư mục `C:\LAB3` và 4 thư mục con theo chuẩn: `Evidence`, `Tools`, `Downloads`, `Assets`.
2. **Ghi mốc thời gian:** Tự động lưu thời điểm bắt đầu vào file `C:\LAB3\Evidence\start_time.txt`.
3. **Đồng bộ công cụ & dữ liệu:** Chép bộ công cụ Sysinternals vào `Tools/` và gói tài nguyên bài lab vào `lab3_assets/`.
4. **Thu thập Baseline ban đầu:** Lưu thông tin nền tảng về OS, Defender, Firewall, Network và Process vào thư mục `Evidence/`.

---

### 4. Kết Quả Thực Hiện 7 Tình Huống

| STT | Tình huống | Mục tiêu & Thao tác chính | Kết quả | Bằng chứng hình ảnh / Log |
| :---: | :--- | :--- | :---: | :--- |
| **TH1** | **Risk Register** | Phân loại 5 nguồn đe dọa và lập bảng quản trị rủi ro gắn với tài sản hệ thống. | **PASS** | Bảng phân tích trong file Word |
| **TH2** | **Mã độc (Malware)** | Tạo chuỗi chuẩn EICAR; kiểm chứng Defender phát hiện, cách ly và ghi nhật ký. | **PASS** | `H4_ProtectionHistory_EICAR.png`<br>`defender_eicar.txt` |
| **TH3** | **Tấn công mật khẩu** | Bật Audit Logon, tạo user `lab3user`, sinh log đăng nhập đúng/sai và đổi mật khẩu. | **PASS** | `H5_Event4625.png`<br>`auth_events_before_rotation.txt` |
| **TH4** | **Cửa hậu (Backdoor)** | Nhận diện Persistence qua Registry Run, Scheduled Task và soi web server cổng 8080. | **PASS** | `H6_Sysmon_Event1.png`<br>`H7_Autoruns_LAB3_Run_Demo.png`<br>`H8_ProcessExplorer_Python.png` |
| **TH5** | **Bắt gói tin (Sniffing)** | Bắt gói HTTP thấy chuỗi văn bản thuần (Plaintext); so sánh với HTTPS/TLS được mã hóa an toàn. | **PASS** | `H9_HTTP_Plaintext.png`<br>`H10_TLS_443.png` |
| **TH6** | **DoS / DDoS / Mailbomb** | Chạy script DoS tải nội bộ (50 requests); phân tích dataset DDoS phân tán và log Mailbomb. | **PASS** | `H10_Load_and_Log_Analysis.png`<br>`local_load_test.txt` |
| **TH7** | **Social Engineering** | Chỉ ra 5 dấu hiệu lừa đảo trong file email mẫu và phân loại 6 kịch bản phi kỹ thuật. | **PASS** | `H10_Phishing_Offline.png`<br>File mẫu `phishing_email.txt` |
| **Mục 8** | **Cleanup & Recovery** | Dọn dẹp các mục thử nghiệm, khôi phục hệ thống sạch sẽ và băm mã toàn vẹn SHA-256. | **PASS** | `H11_Recovery_Verification.png`<br>`evidence_sha256.csv` |

---

### 5. Các Lỗi Gặp Phải & Cách Khắc Phục

1. **Lỗi lệnh `auditpol` báo sai tham số (Error 0x00000057):**
   - *Hiện tượng:* PowerShell hiểu nhầm cặp ngoặc nhọn `{GUID}` là khối mã ScriptBlock.
   - *Khắc phục:* Đặt GUID vào trong dấu ngoặc kép `"{...}"` hoặc dùng trực tiếp tên danh mục `auditpol /set /subcategory:"Logon" /success:enable /failure:enable`.

2. **Lỗi `runas` báo `Unable to acquire user password`:**
   - *Hiện tượng:* Cửa sổ dòng lệnh PowerShell trên Windows 11 chặn thao tác dán phím qua clipboard (Ctrl + V) ở khung nhập mật khẩu ẩn.
   - *Khắc phục:* Dùng bàn phím gõ trực tiếp từng ký tự hoặc sử dụng lệnh PowerShell gọi đối tượng `PSCredential` để đăng nhập tự động.

3. **Lỗi không tìm thấy `tshark.exe` của Wireshark:**
   - *Hiện tượng:* Wireshark được cài đặt tại ổ `D:\Software\Wireshark` thay vì đường dẫn mặc định `C:\Program Files\Wireshark`.
   - *Khắc phục:* Bổ sung câu lệnh điều kiện tự động nhận diện và trỏ đúng đường dẫn thực tế của Wireshark trên hệ thống.

4. **Lỗi Autoruns không hiển thị mục `LAB3_Run_Demo`:**
   - *Hiện tượng:* Autoruns mặc định bật tính năng ẩn các tệp của Microsoft (`Hide Windows Entries`), mà `notepad.exe` là tệp hệ thống nên bị ẩn.
   - *Khắc phục:* Vào menu `Options` > bỏ tích chọn `Hide Windows Entries` và `Hide Microsoft Entries`, sau đó nhấn `F5` để làm mới.

---

### 6. Tính Toàn Vẹn Của Bằng Chứng Số
- Toàn bộ các tệp bằng chứng thực hành trong thư mục `Evidence` đã được tính toán mã băm mật mã học **SHA-256** và xuất ra tệp `evidence_sha256.csv`.
- Đảm bảo tính bất biến, không bị chỉnh sửa hay làm sai lệch kể từ thời điểm kết thúc bài thực hành.
