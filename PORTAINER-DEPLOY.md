# QRForge – Portainer nội bộ (fork hoangthang0106)

Bản triển khai riêng cho LAN, dùng **1 container QRForge + SQLite trong volume**.
Không cần cài PostgreSQL, không sử dụng ảnh Docker của upstream. Mọi thay đổi
giao diện/mã nguồn được build từ repository này.

## Thông tin đã cấu hình

| Mục | Giá trị |
| --- | --- |
| GitHub | https://github.com/hoangthang0106/QrForge |
| Compose | `docker-compose.portainer.yml` |
| Docker image build nội bộ | `qrforge-hoangthang:lan` |
| Container | `qrforge-lan` |
| IP LAN mặc định | `192.168.20.194` |
| Cổng LAN mặc định | **18889** |
| Cổng trong container | 8080 |
| URL mặc định | http://192.168.20.194:18889 |
| Lưu trữ | volume `qrforge-data`, file SQLite `/data/qrforge.db` |
| Đăng ký người dùng | **closed** |
| Upload tệp lên server | **disabled** |

**Chưa thể xác minh cổng 18889 đang trống trên máy Docker**. Trước khi Deploy,
xem trong Portainer → Containers → Published ports. Nếu đang bị chiếm,
đổi biến `QRFORGE_LAN_PORT` ở Portainer (ví dụ 18890); URL QRForge
sẽ tự cập nhật theo cổng mới.

## Cài đặt bằng giao diện Portainer (Docker Standalone / Local socket)

1. Mở Portainer → Stacks → Add stack → Git repository.
2. Name: `qrforge`.
3. Repository URL: `https://github.com/hoangthang0106/QrForge`.
4. Repository reference: `refs/heads/main`.
5. Compose path: `docker-compose.portainer.yml`.
6. Authentication: **off** vì fork public.
7. Trong mục **Environment variables**, thêm **bắt buộc**:
   - `QRFORGE_ADMIN_EMAIL`: email tài khoản quản trị riêng của bạn.
   - `QRFORGE_ADMIN_PASSWORD`: mật khẩu mạnh, **12–72 byte**, không dùng
     password ví dụ; đừng ghi vào GitHub hoặc chia sẻ trong chat.
8. Chọn endpoint **local Docker socket**; nhấn **Deploy the stack**.
9. Chờ build image xong, vào **Containers** → `qrforge-lan`:
   phải **Running / healthy**.
10. Mở `http://192.168.20.194:18889` trên máy ở LAN.

> Có báo cáo Portainer Agent không hỗ trợ ổn định tính năng build từ Git stack.
> Nếu dùng remote Agent và build lỗi, chọn local Docker socket trên đúng máy
> hoặc build image trực tiếp tại Docker host, rồi Deploy lại. Không tự chuyển
> sang image upstream không chứa những chỉnh sửa trong fork.

## Chỉnh thông số mà không sửa YAML

Portainer → Stacks → qrforge → Environment variables:

- `QRFORGE_LAN_IP`: địa chỉ IP LAN đang dùng của Docker host.
  Mặc định `192.168.20.194`.
- `QRFORGE_LAN_PORT`: cổng LAN khác khi 18889 bị chiếm.
  Mặc định `18889`.
- `QRFORGE_ADMIN_EMAIL`, `QRFORGE_ADMIN_PASSWORD`:
  dùng để tạo admin **chỉ khi database còn trống**. Khi tài khoản đã được tạo,
  đổi mật khẩu trong giao diện ứng dụng; thay biến môi trường không tự đổi
  mật khẩu đã lưu.

Đổi biến IP hoặc PORT rồi redeploy sẽ thay đổi URL **của những QR động tạo sau đó**.
Với QR động đã in, phải duy trì URL cũ để tránh hỏng link; không thay IP/port
khi đã dùng QR động thật mà chưa có URL ổn định.

## Sửa mã nguồn trên GitHub

- Giao diện: `apps/web/`.
- Mã QR: `packages/core/`.
- API, tài khoản, thống kê: `server/`.
- Docker image: `docker/Dockerfile`.
- Compose LAN: `docker-compose.portainer.yml`.

Sửa → Commit trên GitHub → Portainer → Stacks → qrforge →
**Pull and redeploy** (có thể cần chọn Rebuild image nếu Portainer cung cấp).
Mã GitHub thay đổi không có nghĩa image đang chạy tự cập nhật.
**KHÔNG bật GitOps auto-update** cho tới khi đã xác minh quy trình build/redeploy.

## Bảo mật

- Chỉ định IP LAN khi publish Docker port; không cấu hình Cloudflare Tunnel
  hoặc NAT/port-forward cổng này ra Internet.
- `ports:` không phải firewall: mọi thiết bị có thể kết nối đến
  `192.168.20.194:18889` đều xem được các phần công khai của ứng dụng.
  Đăng nhập admin cần mật khẩu, nhưng kết nối HTTP trên LAN chưa được mã hóa.
  Với LAN không đáng tin, đặt QRForge sau reverse proxy HTTPS và giới hạn ACL.
- QR tĩnh được tạo trong trình duyệt; QR động và analytics được lưu trong SQLite
  tại máy chủ của bạn. Không có quảng cáo/analytics bên thứ ba theo tài liệu
  của dự án, nhưng không thể bảo đảm an toàn tuyệt đối nếu chưa kiểm toán.
- Mặc định QRForge chặn đích chuyển hướng đến IP/host nội bộ nhằm chống lạm
  dụng. Nếu cần QR động trỏ về địa chỉ nội bộ, phải hiểu rủi ro trước khi
  thay đổi chính sách trong **Admin → Settings**.

## Sao lưu / khôi phục

Dữ liệu cần giữ là named volume của Docker Compose, tên thực tế có thể là
`qrforge_qrforge-data`. Không chọn Delete volumes khi xóa/cập nhật Stack.

SQLite dùng WAL. Để có bản sao nhất quán:
1. **Dừng container qrforge-lan** (không xóa).
2. Sao lưu toàn bộ thư mục dữ liệu trong volume sang ổ backup.
3. Khởi động lại container.
4. Kiểm tra bản backup định kỳ.

Nếu không có backup thì không thử xóa volume hoặc Reset database.
