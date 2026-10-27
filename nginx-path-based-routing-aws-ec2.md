# Lab: Nginx Path-Based Routing trên AWS EC2

## 1. Mục tiêu

-   Triển khai Nginx làm reverse proxy trên VPS Ubuntu.
-   Định tuyến request theo URL path:
    -   `/api/` → Backend API tại `127.0.0.1:8080`
    -   Các path còn lại → Frontend tại `127.0.0.1:3000`
-   Truyền các header `Host`, `X-Real-IP`, `X-Forwarded-For` và
    `X-Forwarded-Proto`.
-   Kiểm tra luồng truy cập từ chính VPS và từ máy cá nhân.

## 2. Môi trường lab

  Thành phần             Giá trị
  ---------------------- ---------------------------------
  VPS                    AWS EC2
  Hệ điều hành VPS       Ubuntu Server 24.04.5 LTS
  Máy client             Ubuntu 26.04
  Reverse proxy          Nginx 1.24.0 (Ubuntu package)
  Frontend giả lập       Python HTTP server trong Docker
  Backend giả lập        Python HTTP server trong Docker
  HTTP port công khai    `80`
  Frontend port nội bộ   `127.0.0.1:3000`
  Backend port nội bộ    `127.0.0.1:8080`

> Thay địa chỉ IP public mẫu trong các lệnh kiểm thử bằng IP hiện tại
> của VPS nếu địa chỉ đã thay đổi.

## 3. Kiến trúc

``` text
Personal PC
    |
    | HTTP :80
    v
AWS Security Group
    |
    v
UFW
    |
    v
Nginx Reverse Proxy (:80)
    |                         |
    | location /api/          | location /
    v                         v
Backend API               Frontend
127.0.0.1:8080            127.0.0.1:3000
```

Client chỉ cần truy cập địa chỉ public của VPS trên port 80. Nginx lựa
chọn service đích dựa trên URL path. Frontend và backend được bind vào
loopback để không trực tiếp mở các port ứng dụng ra Internet.

## 4. Cấu hình Nginx

File cấu hình được sử dụng trong lab:

`/etc/nginx/sites-available/default`

``` nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Kiểm tra cấu hình và reload Nginx:

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` kiểm tra cú pháp và tính hợp lệ của cấu hình. Chỉ reload sau
khi bước kiểm tra thành công.

### Vì sao `/api/products` được giữ nguyên?

Cấu hình sử dụng:

``` nginx
proxy_pass http://127.0.0.1:8080;
```

`proxy_pass` không có phần URI ở cuối, nên Nginx chuyển tiếp URI request
ban đầu. Vì thế backend nhận `/api/products`, không phải `/products`.

## 5. Ý nghĩa các header

  -----------------------------------------------------------------------
  Header                              Mục đích
  ----------------------------------- -----------------------------------
  `Host`                              Chuyển hostname mà client dùng để
                                      truy cập tới backend.

  `X-Real-IP`                         Chuyển IP client mà Nginx nhìn
                                      thấy.

  `X-Forwarded-For`                   Chuyển tiếp chuỗi IP qua các proxy,
                                      hữu ích cho logging và truy vết
                                      request.

  `X-Forwarded-Proto`                 Cho backend biết client kết nối với
                                      Nginx bằng giao thức nào, ví dụ
                                      `http` hoặc `https`.
  -----------------------------------------------------------------------

### Tại sao cần `X-Forwarded-For`?

Backend nhận kết nối TCP trực tiếp từ Nginx, nên địa chỉ peer mà backend
nhìn thấy thường là địa chỉ của Nginx, không phải IP client ban đầu.
Nginx có thể chuyển tiếp thông tin IP qua header:

``` nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Trong bài lab, request từ máy cá nhân trả về `X-Real-IP` và
`X-Forwarded-For` là `1.55.175.46`. Khi request được chạy ngay trên VPS
bằng `curl http://127.0.0.1/api/products`, các giá trị này là
`127.0.0.1`.

**Lưu ý bảo mật:** ứng dụng backend không nên tin tưởng mù quáng
`X-Forwarded-For` do client gửi. Trong production, chỉ nên tin header
được chuyển tiếp bởi proxy đáng tin cậy và cấu hình trusted proxies phù
hợp.

## 6. Kiểm thử

### 6.1. Kiểm thử từ VPS

Chạy các lệnh này trên VPS:

``` bash
curl -i http://127.0.0.1/
curl -i http://127.0.0.1/about
curl -i http://127.0.0.1/api/products
```

Kết quả mong đợi:

-   `/` trả về HTML của Dummy E-commerce Frontend.
-   `/about` trả về HTML của Dummy E-commerce Frontend và hiển thị path
    `/about`.
-   `/api/products` trả về JSON của Dummy Backend API, có `path` là
    `/api/products`.

### 6.2. Kiểm thử từ máy cá nhân

Chạy từ máy client, thay `16.178.73.16` bằng public IP hiện tại của VPS
nếu cần:

``` bash
curl -i http://16.178.73.16/
curl -i http://16.178.73.16/about
curl -i http://16.178.73.16/api/products
```

Kết quả thực tế trong lab: cả ba request đều trả về `HTTP/1.1 200 OK`.

### 6.3. Kết quả backend quan sát được

Request từ máy cá nhân tới `/api/products` trả về JSON tương tự:

``` json
{
  "service": "Dummy Backend API",
  "path": "/api/products",
  "host": "16.178.73.16",
  "x_real_ip": "1.55.175.46",
  "x_forwarded_for": "1.55.175.46",
  "x_forwarded_proto": "http"
}
```

Địa chỉ IP và trường `Host` phụ thuộc vào client, mạng và cách request
được gửi; các giá trị trên là kết quả của lần kiểm thử trong lab, không
phải giá trị cố định.

## 7. Checklist nghiệm thu

-   [x] `sudo nginx -t` thành công.
-   [x] Nginx reload cấu hình thành công.
-   [x] `/` được định tuyến tới frontend.
-   [x] `/about` được định tuyến tới frontend.
-   [x] `/api/products` được định tuyến tới backend.
-   [x] Backend nhận nguyên path `/api/products`.
-   [x] Các header proxy được chuyển tiếp.
-   [x] Kiểm thử từ VPS thành công.
-   [x] Kiểm thử từ máy cá nhân qua public IP thành công.
-   [x] Cả ba endpoint trả về HTTP 200.

## 8. Bài học rút ra

1.  **Reverse proxy:** client truy cập Nginx; Nginx chuyển tiếp request
    tới service phù hợp.
2.  **Path-based routing:** các `location` block phân loại request dựa
    trên URL path.
3.  **Header forwarding:** backend có thể nhận thông tin về host, IP
    client và giao thức ban đầu.
4.  **Giảm bề mặt tấn công:** chỉ mở Nginx trên port công khai; bind các
    service giả lập vào `127.0.0.1`.
5.  **Kiểm tra trước khi áp dụng:** dùng `nginx -t` trước khi reload cấu
    hình.

## 9. Lưu ý khi chuyển sang production

-   Dùng HTTPS/TLS thay vì chỉ phục vụ HTTP.
-   Chỉ mở các port cần thiết trong AWS Security Group và UFW.
-   Giữ các port ứng dụng nội bộ không public nếu không cần truy cập
    trực tiếp.
-   Cấu hình trusted proxies khi ứng dụng xử lý `X-Forwarded-For`.
-   Bổ sung health checks, logging, giám sát và xử lý lỗi phù hợp.
-   Dummy frontend/backend trong bài này chỉ dùng để học định tuyến,
    không phải ứng dụng production.

## 10. Bài lab tiếp theo đề xuất

**HTTPS/TLS với Nginx:** tìm hiểu certificate, HTTPS listener, chuyển
hướng HTTP sang HTTPS và cách kiểm thử chứng chỉ. Sau đó có thể triển
khai ứng dụng thật bằng Docker Compose.
