# Hướng dẫn cài lab AD từ file .ova (VirtualBox)

Bộ lab gồm hai máy ảo đã cấu hình sẵn domain Active Directory:

- **LAB-DC01**: Windows Server 2025, làm Domain Controller.
- **LAB-CLIENT01**: Windows 11, đã join domain.

Bạn sẽ nhận được hai file:

- `lab-dc01-v1.0.ova`
- `lab-client01-v1.0.ova`

## Thông tin lab

| Máy | Hệ điều hành | Vai trò | IP |
|---|---|---|---|
| `LAB-DC01` | Windows Server 2025 | Domain Controller + DNS, domain `lab.local` | `192.168.56.10` |
| `LAB-CLIENT01` | Windows 11 | Máy trạm đã join domain | `192.168.56.20` |

| Tài khoản | Mật khẩu | Ghi chú |
|---|---|---|
| `LAB\<tài-khoản-quản-trị>` | `<điền>` | Quản trị domain |
| `LAB\<tài-khoản-người-dùng>` | `<điền>` | Người dùng thường |

## Yêu cầu với máy nhận

- **CPU:** Intel hoặc AMD 64-bit, đã bật ảo hoá **VT-x / AMD-V** trong BIOS.
- **RAM:** tối thiểu 16 GB để chạy cả hai máy cùng lúc.
- **Ổ trống:** khoảng gấp 3 lần tổng dung lượng hai file `.ova`.
- **VirtualBox:** cùng bản hoặc mới hơn bản người gửi đã dùng (`<điền phiên bản>`). Tải tại https://www.virtualbox.org/wiki/Downloads.
- **Trên Windows:** nên tắt Docker Desktop, WSL2 và Hyper-V khi dùng VirtualBox (xem mục 5).

---

## 1. Chuyển file sang máy mới

1. Copy hai file `.ova` sang máy mới bằng ổ cứng rời, mạng LAN hoặc dịch vụ lưu trữ.
2. Nếu dùng USB, ổ phải định dạng **exFAT** hoặc **NTFS**. Ổ FAT32 không chứa được file lớn hơn 4 GB.
3. Đặt hai file vào một thư mục dễ tìm, ví dụ `D:\lab-ova\`.

---

## 2. Import vào VirtualBox

1. Mở **VirtualBox Manager** → **File** → **Import Appliance**.
2. **Source:** chọn **Local File System**. **File:** chọn `lab-dc01-v1.0.ova` → **Next**.
3. Ở trang **Appliance settings**:
   - **Name:** giữ `LAB-DC01`.
   - **Machine Base Folder:** chọn ổ còn nhiều chỗ trống để chứa máy ảo.
   - **MAC Address Policy:** chọn **Include all network adapter MAC addresses**, để Windows giữ nguyên cấu hình IP tĩnh.
   - Bỏ tick **Import hard drives as VDI**.
4. Bấm **Finish** và chờ import xong (khoảng 10–30 phút).
5. Lặp lại từ bước 1 với file `lab-client01-v1.0.ova` (Name: `LAB-CLIENT01`).

> VirtualBox tự kiểm tra tính toàn vẹn của file khi import. Nếu file bị hỏng trong lúc copy, VirtualBox sẽ báo lỗi ngay ở bước này (xem mục 5).

Sau khi import xong cả hai máy, có thể xoá hai file `.ova` để lấy lại dung lượng.

---

## 3. Gắn lại mạng host-only (bắt buộc)

Adapter 2 của hai máy đang trỏ tới mạng host-only trên máy của người gửi, nên trên máy mới phải chọn lại.

**Kiểm tra / tạo mạng host-only:**

1. **File** → **Tools** → **Network Manager** → tab **Host-only Networks**.
2. Cần có một mạng với:
   - **IPv4 Address:** `192.168.56.1`
   - **IPv4 Network Mask:** `255.255.255.0`
3. Nếu chưa có: bấm **Create**, sửa hai giá trị trên → **Apply**.

**Gắn mạng cho từng máy** (làm cho cả LAB-DC01 và LAB-CLIENT01):

1. Chọn máy → **Settings** → **Network** → tab **Adapter 2**.
2. **Attached to:** chọn **Host-only Adapter**.
3. **Name:** chọn mạng vừa kiểm tra ở trên → **OK**.

---

## 4. Bật máy và kiểm tra

**Thứ tự bật:**

1. Bật **LAB-DC01** trước.
2. Đợi tới màn hình đăng nhập, rồi chờ thêm 1–2 phút cho dịch vụ AD và DNS lên hết.
3. Bật **LAB-CLIENT01** và đăng nhập bằng tài khoản domain.

**Kiểm tra domain hoạt động:** trên LAB-CLIENT01, mở CMD và chạy:

```
nltest /dsgetdc:lab.local
```

Kết quả phải trả về `\\DC01.lab.local` với địa chỉ `192.168.56.10`.

**Thứ tự tắt:** tắt **LAB-CLIENT01** trước, rồi tới **LAB-DC01**. Luôn tắt từ trong Windows (**Start** → **Shut down**).

---

## 5. Xử lý sự cố

### Bật máy báo "Nonexistent host networking interface"

Adapter 2 chưa được gắn vào mạng host-only trên máy này. Làm lại mục 3.

### Import báo lỗi file hỏng (manifest / digest mismatch)

File `.ova` bị hỏng trong lúc copy. Copy lại file đó rồi import lại.

### Máy chạy rất chậm, treo ở logo, hoặc thanh trạng thái có biểu tượng con rùa xanh

VirtualBox đang phải chạy chồng lên Hyper-V (do Docker Desktop, WSL2 hoặc Core Isolation). Cách sửa:

1. Mở CMD bằng **Run as administrator**, chạy:
   ```
   bcdedit /set hypervisorlaunchtype off
   ```
2. Khởi động lại máy thật.

Khi cần dùng lại Docker Desktop hoặc WSL2: chạy `bcdedit /set hypervisorlaunchtype auto` rồi khởi động lại.

### LAB-CLIENT01 không đăng nhập được domain ("There are currently no logon servers available")

1. Kiểm tra LAB-DC01 đã bật xong chưa (mục 4).
2. Trên LAB-CLIENT01, mở CMD và chạy `ping 192.168.56.10`.
3. Nếu không ping được:
   - Kiểm tra Adapter 2 của cả hai máy có cùng một mạng host-only không (mục 3).
   - Mở `ncpa.cpl`, kiểm tra card mạng thứ hai (thường tên *Ethernet 2*) vẫn giữ IP tĩnh như bảng **Thông tin lab**. Nếu IP bị mất (thường do lúc import không chọn giữ MAC), đặt lại IP theo bảng. Trên LAB-CLIENT01, DNS phải là `192.168.56.10`.

### Cảnh báo "Shared folder ... could not be set up"

Máy ảo còn khai báo một thư mục chia sẻ của máy người gửi. Cảnh báo này vô hại. Muốn hết cảnh báo: tắt máy → **Settings** → **Shared Folders** → xoá dòng đó.
