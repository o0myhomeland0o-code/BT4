# BT4
# Môn: Phát triển ứng dụng với mã nguồn mở-TEE0421
### Họ tên: Dương Quang Minh
### MSSV: K225480106047
### Lớp: K58KTP

### Bài 4: KHAI THÁC N8N ĐỂ TỰ ĐỘNG ĐĂNG BÀI LÊN WORDPRESS
### 1.
##### file docker-compose.yml (cập nhật từ bài 3)
```
services:
  db:
    image: mariadb:latest
    container_name: mariadb_db
    restart: always
    environment:
      TZ: "Asia/Ho_Chi_Minh"
      MARIADB_ROOT_PASSWORD: "123456_db_root_pass"
      MARIADB_DATABASE: "wordpress_db"
      MARIADB_USER: "wp_user"
      MARIADB_PASSWORD: "123456_wp_password"
    volumes:
      - db_data:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin_ui
    restart: always
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
      PMA_ARBITRARY: 1
    depends_on:
      - db

  wordpress:
    image: wordpress:latest
    container_name: wordpress_app
    restart: always
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: "wordpress_db"
      WORDPRESS_DB_USER: "wp_user"
      WORDPRESS_DB_PASSWORD: "123456_wp_password"
    depends_on:
      - db
    volumes:
      - wp_data:/var/www/html

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n_automation
    restart: always
    ports:
      - "5678:5678"
    environment:
      - TZ=Asia/Ho_Chi_Minh
      - WEBHOOK_URL=https://n8n.tnut58ktp047.id.vn/
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - db

  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare_tunnel
    restart: always
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=eyJhIjoiMzljZDJiMmQyNDNhZDU3MzA2ZjFjMGY0OTA5ZTZlYWIiLCJ0IjoiNmY0M2Q4NjktNzBhYy00NjY1LWI2OWYtOTlkYjUwNjI4ODkwIiwicyI6IlpXSTFNbVV4TkRjdFptTTNOeTAwTURJeExUZzFObUl0WWpaak9EUTVOVFU0T0RnMiJ9
    depends_on:
      - wordpress
      - n8n
      - phpmyadmin

volumes:
  db_data:
  wp_data:
  n8n_data:
```
###### docker compose down
###### docker compose up -d
<img width="368" height="136" alt="image" src="https://github.com/user-attachments/assets/b6026835-656d-4ee4-8280-b2a32a6e8898" />
<img width="652" height="237" alt="image" src="https://github.com/user-attachments/assets/d4189a3d-9e7b-4439-8060-90d88d9202d2" />

### 2.
##### thêm sub-domain vào tunnel đã tạo cho wordpress ở bài 3
<img width="847" height="391" alt="image" src="https://github.com/user-attachments/assets/33df4193-90f0-4f74-be41-d0be79f436d3" />


##### tạo các bài viết ở wordpress
##### cấu hình n8n:
######
+ tạo tài khoản admin
+ bấm Send me a Licence key, lấy key ở gmail
+ Activate License key: vào trang chủ => SETTING (góc dưới trái) => Usage and plan => Enter activation key: paste key từ email vào đây => Activate => sẽ nhận đc thông báo (góc dưới phải) Your Registered Community Edition has been successfully activated.
+ create workflow
<img width="311" height="176" alt="image" src="https://github.com/user-attachments/assets/59956b08-c130-42e2-af25-f366c2d767f3" />

+ Add trigger node:
  - node: Telegram => OnMessage
  - cấu hình Credential: Set up Credential => Nhập Access Token (chat với @BotFather, tạo bot, lấy token)
<img width="269" height="76" alt="image" src="https://github.com/user-attachments/assets/c50fddd9-10df-491b-80fd-e626bef38df4" />
<img width="251" height="129" alt="image" src="https://github.com/user-attachments/assets/09fe841a-83bb-400c-b6f2-f1f2d19359c0" />
  - chat với bot (vd: Hello bot) để lấy giá trị

+ Add node: AI Google Gemini
  - Set up Credential, nhập API KEY
  - tạo nội dung promt
  - Output Content as JSON: bật
<img width="259" height="196" alt="image" src="https://github.com/user-attachments/assets/135d89ba-57a4-49ae-877f-e1109cf805a2" />

+ Add node: Code in JavaScript
  - mode: Run Once for All Items
  - nội dung code:
```
// 1. lấy dữ liệu gốc
const rawText = $input.first().json.content.parts[0].text;

// 2. Chuyển đổi chuỗi (đã được bọc JSON) thành Object trong JavaScript
const cleanData = JSON.parse(rawText);

// 3. Trả về kết quả định dạng lại gọn gàng cho n8n sử dụng
return {
  title: cleanData.post_title,
  content: cleanData.post_content
};
```
<img width="178" height="147" alt="image" src="https://github.com/user-attachments/assets/ce9f112a-d5cf-4708-aa6e-841ef26736fc" />

+ Add node: WordPress => Create a Post
  - Set up Credential: sub-domain wordpress. => Tài Khoản => chọn user đã tạo lúc setup wordpress => Mật khẩu ứng dụng => Nhập n8n và bấm "Thêm mật khẩu ứng dụng" => copy chuỗi 24 kí tự : Đây là mật khẩu ứng dụng => paste vào mục Password của n8n Credential
  - nhập username, password (chuỗi 24 kí tự), wordpress url (sub-domain1)
  - Ignore SSL Issues (Insecure): bật
  - Cấu hình node Create a Post: bấm nút Execute previous nodes để thấy trường giá trị của node trước trả về, kéo nội dung phần title (bên trái) vào trường title
  - Add field (Thêm thuộc tính): Status == Publish (bài đăng sẽ ở trạng thái xuất bản ngay lập tức, mặc định nó ở giá trị Draft bản nháp)
<img width="158" height="174" alt="image" src="https://github.com/user-attachments/assets/d03bcb8e-ea0c-4c92-8a87-036db9dd7d6c" />

+ PUBLISH flow
<img width="931" height="215" alt="image" src="https://github.com/user-attachments/assets/a855a11b-c502-4645-b4d7-f6e68ebf92ed" />

### 3. Test:
###### chat với bot:
<img width="347" height="42" alt="image" src="https://github.com/user-attachments/assets/c34176f9-86e8-436f-a79f-de46a866ced5" />

###### kiểm tra trên wordpress:
<img width="922" height="678" alt="image" src="https://github.com/user-attachments/assets/83e35f2f-0753-48bd-9f96-7159e288cc7a" />
