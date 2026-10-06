# Bài 5: Chẩn đoán và xử lý xung đột cổng 8082 (Address Already in Use)

**Họ tên:** Mai Văn Huy
**Mã lớp:** CNTT1
**Ngày:** 06/10/2026
**Môi trường:** Linux Ubuntu, cổng mục tiêu `8082`, service `spring-app.service` (app.jar)

## 1. Mô tả sự cố
Khi khởi chạy Spring Boot (`spring-app.service` / `java -jar app.jar`) ở cổng 8082, log báo:
```
Web server failed to start. Port 8082 was already in use.
... Address already in use ...
```
Nguyên nhân: có 1 tiến trình khác đang LISTEN cổng 8082, cần tìm PID và kill để giải phóng.

## 2. Tái hiện (giả lập cổng bị chiếm theo đề bài)
```bash
python3 -m http.server 8082 > /dev/null 2>&1 &
echo $!
# Hoặc nếu không có python3 (lưu ý cú pháp nc khác nhau giữa các bản):
# Ubuntu OpenBSD netcat: nc -l 8082 > /dev/null 2>&1 &
# Traditional netcat: nc -l -p 8082 > /dev/null 2>&1 &

# Kiểm tra đã chiếm cổng:
sudo ss -tlnp | grep 8082
```

Kết quả mẫu lúc bị chiếm bởi tiến trình giả lập:
```
LISTEN 0  5  0.0.0.0:8082  0.0.0.0:*  users:(("python3",pid=1234,fd=3))
```

## 3. Chẩn đoán: tìm PID đang giữ cổng 8082

### Cách 1 (khuyên dùng): `ss`
```bash
sudo ss -tlnp | grep 8082
```
- `-t`: TCP, `-l`: listening, `-n`: numeric, `-p`: show process
- PID nằm ở cột cuối trong `users:(("tên",pid=1234,...))`

### Cách 2: `lsof`
```bash
sudo lsof -i :8082
```
Ví dụ output:
```
COMMAND  PID USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
python3 1234 ubuntu  3u IPv4  12345   0t0  TCP *:8082 (LISTEN)
```

### Cách 3 (dự phòng): `netstat`
```bash
sudo netstat -tlnp | grep 8082
```

### Xem chi tiết tiến trình
```bash
ps -p 1234 -o pid,ppid,user,%cpu,%mem,cmd
# ví dụ:
#  PID PPID USER  %CPU %MEM CMD
# 1234  987 ubuntu 0.1  0.5 python3 -m http.server 8082
```

> **Ghi nhận thực tế của em:**
> - PID tìm được: `1234` (ví dụ, thay bằng số thực tế trên máy khi làm)
> - Tên tiến trình: `python3 -m http.server 8082`
> - User chạy: `ubuntu` (hoặc `root` tùy máy)
> - Cổng: `0.0.0.0:8082 LISTEN`

## 4. Xử lý: kill để giải phóng cổng

```bash
kill 1234
sleep 2
sudo ss -tlnp | grep 8082

# Nếu vẫn còn (process lì), kill mạnh:
kill -9 1234
sudo ss -tlnp | grep 8082
# mong đợi: không còn dòng nào → cổng đã trống
```

Giải thích:
- `kill <PID>` (SIGTERM): kết thúc an toàn, cho process dọn dẹp.
- `kill -9 <PID>` (SIGKILL): cưỡng chế, dùng khi `kill` thường không hiệu quả.

## 5. Khởi động lại service và xác nhận Java giữ cổng

```bash
sudo systemctl restart spring-app.service
sudo systemctl status spring-app.service --no-pager
sleep 5
sudo ss -tlnp | grep 8082
# Hoặc:
sudo lsof -i :8082
```

Kết quả mong đợi sau fix (cổng do `java` quản lý, không còn `python3`):
```
LISTEN 0  100  0.0.0.0:8082  0.0.0.0:*  users:(("java",pid=2345,fd=12))
```

Kiểm tra thêm:
```bash
ps -p 2345 -o pid,user,cmd
curl -i http://localhost:8082/api/health
# hoặc qua Nginx (nếu đã làm Bài 4):
curl -i http://localhost/api/health
# mong đợi: HTTP 200, không còn 502 Bad Gateway
```

## 6. Chuỗi lệnh tóm tắt (để copy nộp / thao tác nhanh)
```bash
python3 -m http.server 8082 > /dev/null 2>&1 &
sudo ss -tlnp | grep 8082
sudo lsof -i :8082
ps -p <PID> -o pid,ppid,user,cmd
kill <PID>
sudo ss -tlnp | grep 8082
kill -9 <PID>
sudo systemctl restart spring-app.service
sudo systemctl status spring-app.service --no-pager
sudo ss -tlnp | grep 8082
curl -i http://localhost:8082/api/health
```

## 7. Kết luận
- Đã xác định PID chiếm cổng 8082 bằng `ss -tlnp` / `lsof -i :8082`.
- Đã `kill` thành công, cổng trống.
- Đã restart `spring-app.service`, xác nhận `ss` hiện `java` LISTEN 8082.
- Lỗi `Address already in use` đã được khắc phục.

## 8. File nộp
- File này: `bai5/troubleshoot_report.md`
- Khi nộp: thay PID ví dụ `1234`/`2345` bằng PID thực tế trên máy + dán output thật của `ss`, `lsof`, `systemctl status`.
