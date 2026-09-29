# LAB 4 - KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

## 1. Thông tin sinh viên

- **Họ tên:** <Nguyễn Tiến Đạt>
- **MSSV:** <1150080090>
- **Môn học:** An toàn hệ thống thông tin
- **Lab:** LAB 4 - Khảo sát và đánh giá bề mặt mạng bằng Nmap

## 2. Môi trường thực hành

- **Host OS:** Windows 11
- **Hypervisor:** VirtualBox
- **Kali Linux:** máy quét
- **Metasploitable 2:** máy mục tiêu
- **Windows 11 VM:** máy đối chiếu/hardening
- **Network:** VirtualBox Host-Only `192.168.56.0/24`
- **Nmap:** 7.99

IP trong môi trường lab:

```text
Kali Linux       : 192.168.56.103
Metasploitable 2 : 192.168.56.101
Windows 11 VM    : 192.168.56.102
```

Metasploitable 2 chỉ sử dụng Host-Only, không dùng Bridged Adapter.

## 3. Cách dựng môi trường

1. Tạo 3 VM: Kali, Metasploitable 2 và Windows 11.
2. Tạo VirtualBox Host-Only Network `192.168.56.0/24`.
3. Gắn các VM vào cùng mạng Host-Only.
4. Kiểm tra IP bằng `ip -br addr`, `ifconfig`, `ipconfig`.
5. Ping kiểm tra kết nối giữa các VM.
6. Tạo snapshot `Before-LAB4`.
7. NAT chỉ được sử dụng tạm thời khi cần cài/cập nhật phần mềm.

## 4. Các tình huống đã thực hiện

| Nội dung | Kết quả |
|---|---|
| Host Discovery `-sn` | PASS |
| TCP Connect `-sT` | PASS |
| SYN Scan `-sS` | PASS |
| FIN Scan `-sF` | PASS |
| Xmas Scan `-sX` | PASS |
| NULL Scan `-sN` | PASS |
| ACK Scan `-sA` | PASS |
| UDP Scan `-sU` | PASS |
| Version Detection `-sV` | PASS |
| OS Detection `-O` | PASS |
| Aggressive Scan `-A` | PASS |
| SMB NSE | PASS |
| MS17-010 NSE | PASS |
| Export kết quả | PASS |
| Before/After Hardening | PASS |

> PASS nghĩa là tình huống được thực hiện thành công và có output hợp lệ, không đồng nghĩa mục tiêu có hoặc không có lỗ hổng.

## 5. Kết quả chính

Host Discovery phát hiện **5 host đang hoạt động** trong mạng Host-Only.

TCP Connect scan trên Metasploitable 2:

```text
Open     : 23
Closed   : 977
Filtered : 0
```

Một số dịch vụ được phát hiện:

```text
21/tcp   ftp
22/tcp   ssh
23/tcp   telnet
80/tcp   http
445/tcp  microsoft-ds
3306/tcp mysql
5432/tcp postgresql
5900/tcp vnc
```

FIN/Xmas/NULL scan có thể trả về `open|filtered`; trạng thái này không được xem là chắc chắn `open`.

ACK scan chủ yếu dùng để quan sát filtering; `unfiltered` không đồng nghĩa với cổng mở.

## 6. Before / After Hardening

Windows VM:

```text
192.168.56.102
```

Quét trước hardening:

```bash
sudo nmap -sV 192.168.56.102 -oN window_before.txt
```

Sau khi thay đổi Windows Defender Firewall:

```bash
sudo nmap -sV 192.168.56.102 -oN windows_after.txt
```

Hai kết quả được dùng để so sánh sự thay đổi trạng thái cổng/dịch vụ trước và sau hardening.

## 7. Cấu trúc repository

```text
LAB4/
├── README.md
├── evidence_sha256.csv
├── report/
│   └── BaoCao_LAB4.docx
├── evidence/
│   └── các ảnh chụp quá trình thực hành
└── outputs/
    ├── host_discovery.txt
    ├── tcp_connect.txt
    ├── syn_scan.txt
    ├── fin_scan.txt
    ├── xmas_scan.txt
    ├── null_scan.txt
    ├── ack_scan.txt
    ├── udp_scan.txt
    ├── service_version.txt
    ├── os_detection.txt
    ├── aggressive_scan.txt
    ├── smb_os_discovery.txt
    ├── ms17_010.txt
    ├── window_before.txt
    ├── windows_after.txt
    ├── metasploitable_full.nmap
    ├── metasploitable_full.xml
    └── metasploitable_full.gnmap
```

## 8. Evidence và SHA-256

Các output được làm sạch trước khi đưa vào repository.

`evidence_sha256.csv` lưu SHA-256 của các file trong `outputs/` để kiểm tra tính toàn vẹn.

Không upload:

- mật khẩu, token/API key, cookie/session;
- email hoặc dữ liệu cá nhân thật;
- public IP hoặc thông tin hệ thống thật;
- installer, `.exe`, `.msi`;
- file VM `.vdi`, `.vmdk`, `.iso`;
- file bị Defender quarantine.

## 9. Lỗi gặp phải và cách khắc phục

### Metasploitable dùng nhầm ổ `.vdi`
Khắc phục bằng cách bỏ ổ `.vdi` và gắn đúng file `Metasploitable.vmdk`.

### Sai đường dẫn output
Ban đầu dùng `/LAB4_outputs/...` nên Nmap báo không tìm thấy thư mục.  
Khắc phục bằng cách sử dụng `~/LAB4_outputs/`.

### VirtualBox Drag & Drop timeout
Drag & Drop báo `VERR_TIMEOUT`.  
Khắc phục bằng VirtualBox Shared Folder `/media/sf_outputs`.

## 10. Phạm vi an toàn

Toàn bộ thao tác quét chỉ thực hiện trên các VM do sinh viên tự dựng trong mạng Host-Only.

Không quét hệ thống bên ngoài lab.

Nếu repo có `local_load_test.py`, giữ nguyên mục tiêu:

```text
127.0.0.1:8080
```

Không thực hiện mail bomb, DDoS, spoofing hoặc MITM chủ động trên mạng bên ngoài VM lab.
