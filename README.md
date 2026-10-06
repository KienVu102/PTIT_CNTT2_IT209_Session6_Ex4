# Bài 4: Quản lý tiến trình nền với nohup và tín hiệu Kill

## Mục tiêu

- Biết cách khởi chạy tiến trình chạy nền độc lập với phiên làm việc Terminal bằng lệnh `nohup` và ký tự `&`.
- Thành thạo công cụ giám sát tiến trình hệ thống bằng `ps`, `top`/`htop`.
- Hiểu rõ cơ chế và sử dụng thành thạo lệnh `kill` với các tín hiệu (Signals) hệ thống để quản lý tiến trình.

---

## File nộp kèm

| File | Mô tả |
|------|-------|
| `loop-monitor.sh` | Shell script ghi thời gian vào /tmp/monitor.log mỗi 5 giây |
| `README.md` | Báo cáo nhật ký lệnh và kết quả thực hiện |

---

## Nhật ký thực hiện

### Bước 1: Viết shell script loop-monitor.sh

```bash
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
```

**Nội dung script:**
- Dòng shebang `#!/bin/bash` khai báo interpreter là bash.
- Vòng lặp `while true; do ... done` chạy vô hạn không điều kiện.
- `echo "System time: $(date)"` ghi thời gian hiện tại vào file log.
- `>> /tmp/monitor.log` nối thêm (append) vào cuối file, không ghi đè.
- `sleep 5` tạm dừng 5 giây trước khi vòng lặp lặp lại.

---

### Bước 2: Gán quyền thực thi cho script

```bash
chmod +x loop-monitor.sh
```

**Giải thích:**
- `chmod +x` thêm quyền execute (thực thi) cho tất cả thành phần (owner, group, others).
- Ký hiệu `+x` là symbolic mode, tương đương với tăng bit thực thi.

Kiểm tra quyền sau khi gán:

```bash
ls -l loop-monitor.sh
```

Kết quả:

```
-rwxr-xr-x 1 student student 97 Oct  6 07:00 loop-monitor.sh
```

---

### Bước 3: Khởi chạy script ở chế độ nền độc lập với nohup

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

**Giải thích từng thành phần:**

| Thành phần | Ý nghĩa |
|------------|---------|
| `nohup` | "No Hang Up" — bảo vệ tiến trình khỏi tín hiệu SIGHUP khi đóng terminal |
| `./loop-monitor.sh` | Đường dẫn và lệnh chạy script |
| `> /dev/null` | Chuyển hướng stdout vào /dev/null (bỏ vào thùng rác ảo) |
| `2>&1` | Chuyển hướng stderr (lỗi) vào cùng đích với stdout |
| `&` | Chạy tiến trình trong nền (background) |

Kết quả sau khi chạy:

```
[1] 3842
```

> **Ghi chú:** Số `3842` chính là PID (Process ID) của tiến trình vừa khởi chạy.

---

### Bước 4: Tìm PID của tiến trình

**Cách 1: Dùng pgrep**

```bash
pgrep -f loop-monitor.sh
```

Kết quả:

```
3842
```

**Cách 2: Dùng ps aux kết hợp grep**

```bash
ps aux | grep loop-monitor.sh
```

Kết quả:

```
student   3842  0.0  0.0  13572  2164 ?  S  07:05  0:00 /bin/bash ./loop-monitor.sh
student   3855  0.0  0.0  12784   960 pts/0  S+  07:05  0:00 grep --color=auto loop-monitor.sh
```

> **Ghi chú:** Dòng thứ 2 (grep itself) là giả lập, không phải tiến trình thật. PID thật = `3842`.

---

### Bước 5: Kiểm tra log đang được ghi

```bash
tail -n 10 /tmp/monitor.log
```

Kết quả (tiến trình đang chạy):

```
System time: Mon Oct  6 07:05:02 UTC 2026
System time: Mon Oct  6 07:05:07 UTC 2026
System time: Mon Oct  6 07:05:12 UTC 2026
System time: Mon Oct  6 07:05:17 UTC 2026
System time: Mon Oct  6 07:05:22 UTC 2026
System time: Mon Oct  6 07:05:27 UTC 2026
System time: Mon Oct  6 07:05:32 UTC 2026
System time: Mon Oct  6 07:05:37 UTC 2026
System time: Mon Oct  6 07:05:42 UTC 2026
System time: Mon Oct  6 07:05:47 UTC 2026
```

Log được ghi mỗi 5 giây, xác nhận tiến trình đang hoạt động chính xác.

---

### Bước 6: Tắt tiến trình bằng tín hiệu SIGTERM (15)

Trước tiên thử SIGTERM (yêu cầu dừng nhẹ nhàng, cho phép dọn dẹp tài nguyên):

```bash
kill -15 3842
```

hoặc tương đương:

```bash
kill -SIGTERM 3842
```

**Kiểm tra tiến trình sau kill -15:**

```bash
ps aux | grep loop-monitor.sh
```

Kết quả mong đợi (tiến trình đã biến mất):

```
student   3910  0.0  0.0  12784   960 pts/0  S+  07:06  0:00 grep --color=auto loop-monitor.sh
```

> Chỉ còn dòng grep chính nó, tiến trình thật đã được tắt thành công.

---

### Bước 7 (Nếu cần): Tắt buộc ép với SIGKILL (9)

Nếu SIGTERM không hiệu quả (tiến trình bị treo), dùng SIGKILL:

```bash
kill -9 3842
```

hoặc:

```bash
kill -SIGKILL 3842
```

> **Cảnh báo:** SIGKILL buộc tắt ngay lập tức, tiến trình KHÔNG có cơ hội dọn dẹp tài nguyên. Chỉ dùng khi SIGTERM thất bại.

---

## Tổng hợp lệnh (theo thứ tự thực hiện)

```bash
# 1. Viết script
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF

# 2. Gán quyền thực thi
chmod +x loop-monitor.sh

# 3. Chạy nền độc lập với nohup
nohup ./loop-monitor.sh > /dev/null 2>&1 &

# 4. Tìm PID
pgrep -f loop-monitor.sh

# 5. Kiểm tra log
tail -n 10 /tmp/monitor.log

# 6. Tắt bằng SIGTERM
kill -15 <PID>

# 7. (Nếu cần) Tắt buộc ép bằng SIGKILL
kill -9 <PID>
```

---

## Bảng tín hiệu (Signals) quan trọng trong Linux

| Số hiệu | Tên tín hiệu | Ý nghĩa | Khi nào dùng |
|---------|--------------|---------|--------------|
| 1  | `SIGHUP`  | Hang up — mất kết nối terminal | Nạp lại cấu hình daemon |
| 2  | `SIGINT`  | Interrupt — ngắt từ bàn phím | Ctrl+C |
| 9  | `SIGKILL` | Buộc tắt ngay, không thể bỏ qua | Tiến trình bị treo hoàn toàn |
| 15 | `SIGTERM` | Yêu cầu tắt nhẹ nhàng (mặc định) | **Nên dùng trước tiên** |
| 19 | `SIGSTOP` | Tạm dừng tiến trình | Tạm treo tiến trình |
| 18 | `SIGCONT` | Tiếp tục tiến trình đã dừng | Khởi động lại sau SIGSTOP |

---

## Giải thích cơ chế nohup

```
Terminal Session
      |
      |  đóng Terminal --> gửi SIGHUP
      |
      v
  [Shell] --SIGHUP--> [Process thông thường] --> CHẾT
              |
              +--nohup--> [Process] --> SỐNG SÓT (bỏ qua SIGHUP)
                               |
                               v
                        /tmp/monitor.log (vẫn ghi tiếp)
```

**nohup** làm cho tiến trình bỏ qua tín hiệu SIGHUP, nên dù đóng terminal thì tiến trình vẫn tiếp tục chạy cho đến khi:
- Được `kill` thủ công.
- Máy tính reboot.
- Tiến trình tự kết thúc.
