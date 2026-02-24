# Lab: SQL injection attack, listing the database contents on Oracle
## Quá trình đánh giá và khai thác 
### 1. Xác định điểm vào
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/SyEr-25uWe.png)
&rarr; Từ trang chủ có 2 chức năng chính là lọc sản phẩm theo danh mục và đường dẫn truy cập tới trang Login. 
Theo lý thuyết, sau khi xác định được các chức năng của trang web, ta cần đánh giá chúng theo khả năng có lỗi để ưu tiên. Chức năng lọc sản phẩm dễ có lỗi hơn vì nó thường được sử dụng trong mệnh đề WHERE của truy vấn, chưa kể ta chưa có thông tin đăng nhập nào, nếu chức năng này có lỗ hổng có khả năng sẽ liệt kê được dữ liệu hỗ trợ cho tấn công login. 
Đầu tiên, ta thực hiện chức năng để quan sát request tạo ra:
![image](https://hackmd.io/_uploads/Syf-z35ubg.png)
&rarr; Nó gửi request tới /filter với tham số category trong URL. Đây là một trong những điểm vào phổ biến nhất của lỗi này, vì vậy ta ưu tiên kiểm thử trước. Thêm vào ký tự `'` trong giá trị tham số và quan sát response:
![image](https://hackmd.io/_uploads/ByBIMh9ubx.png)
&rarr; Response trả về có status code 500, cho thấy input của ta đã gây ra lỗi xử lý trên Server. Tuy nhiên chưa khẳng định được là có lỗi SQLi, ta thử thay thế lần lượt bằng các ký tự đặc biệt phổ biến khác:
![image](https://hackmd.io/_uploads/Hkexmn5ubg.png)
![image](https://hackmd.io/_uploads/r1pgX35dZx.png)
![image](https://hackmd.io/_uploads/rJhZ7ncOZg.png)
&rarr; Không có ký tự nào gây ra response status code 500. 
Tiếp theo, để xác minh, ta thêm ký tự comment phổ biến `--` vào sau ký tự `'` để xem có phản hồi thế nào:
![image](https://hackmd.io/_uploads/S1qI7ncObg.png)
&rarr; Phản hồi trả về status 200 thành công (mặc dù có chứa ký tự `'`) &rarr; Khẳng định được có lỗi SQLi ở đây. 
### 2. Thu thập thông tin Database
Sau khi xác định có lỗi, cần biết chính xác loại DBMS được sử dụng bởi mục tiêu là gì. Trước hết ta thử thay thế ký tự comment thành `#` để xem phản hồi thế nào:
![image](https://hackmd.io/_uploads/ByRJNh5uWe.png)
&rarr; Phản hồi lỗi 500 &rarr; Ký tự # không thể comment, vì vậy ta loại MySQL khỏi danh sách có thể.
Vì mỗi loại DBMS lại có ký tự nối chuỗi khác nhau, ta thử sử dụng nối chuỗi `||` xem phản hồi thế nào:
![image](https://hackmd.io/_uploads/r1Z8H2quZx.png)
&rarr; Phản hồi bình thường &rarr; ký tự `||` có thể được sử dụng để nối chuỗi &rarr; Có thể là Oracle hoặc PostgreSQL. 
Vì hàm trích xuất chuỗi của 2 loại DBMS này là khác nhau, vì vậy ta có thể sử dụng cú pháp ví dụ: `||(SUBSTRING('',1,1))--` để xác định:
- PostgreSQL:
![image](https://hackmd.io/_uploads/rkpwUhcO-l.png)
- Oracle:
![image](https://hackmd.io/_uploads/BJMFInqd-e.png)

&rarr; Xác định được loại DBMS là Oracle. 
Tiếp theo, cần dựa vào cách Server phản hồi kết quả truy vấn mà xác định kỹ thuật tấn công. Ở đây kết quả truy vấn là thông tin của sản phẩm được trả về trong response, vì vậy hướng tới sử dụng kỹ thuật `UNION BASED` để trích xuất và tiết lộ dữ liệu nhạy cảm. 
Trước hết sử dụng `ORDER BY` để xác định số lượng cột trả về trong truy vấn gốc:
![image](https://hackmd.io/_uploads/Hyz3ya9dZl.png)
&rarr; Khi tăng giá trị lên 3 thì trả về lỗi 500 &rarr; số lượng cột trong truy vấn gốc là 2. 
Tiếp theo, sử dụng `UNION SELECT 'tmt1','tmt2' FROM dual--` để xác định xem định dạng của 2 cột trả về có phải là chuỗi không:
![image](https://hackmd.io/_uploads/rkZMl69ubl.png)
&rarr; thành công &rarr; cả 2 cột đều có định dạng chuỗi. 
Tiếp theo, thay câu truy vấn thành `UNION SELECT 'tmt1',banner FROM v$version--` để trích xuất phiên bản của database:
![image](https://hackmd.io/_uploads/rk2jZa5_-x.png)
&rarr; Xác định được phiên bản là `11.2.0.2.0`.
### 3. Khám phá cấu trúc database
Tiếp theo, ta sẽ thực hiện xác định current user và current database thông qua truy vấn `UNION SELECT SYS_CONTEXT('USERENV','DB_NAME'), SYS_CONTEXT('USERENV','CURRENT_USER') FROM dual --`:
![image](https://hackmd.io/_uploads/rJ2_M6cuZx.png)
&rarr; Database là `XE` và user là `PETER`. 
Tiếp theo, ta trích xuất các bảng dữ hiệu hiện có trong database này qua truy vấn `UNION SELECT table_name, owner FROM all_tables--`:
![image](https://hackmd.io/_uploads/r1mZmT9uZe.png)
![image](https://hackmd.io/_uploads/HkTW7acO-l.png)
&rarr; Ta thấy có 2 bảng thuộc sở hữu của user hiện tại là `PRODUCTS` và `USERS_UHGEMQ`. 
Sau khi đã xác định được bảng chứa dữ liệu nhạy cảm, ta tiến hành trích xuất tên các cột có trang bảng đó thông qua truy vấn `UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name = 'USERS_UHGEMQ'--`:
![image](https://hackmd.io/_uploads/HkepX6qubl.png)
&rarr; xác định có 3 cột, trong đó có 2 cột cần chú ý là `USERNAME_HHSDBW` và `PASSWORD_ZKJNDZ`. 
Tiếp theo, vì truy vấn gốc trả về 2 cột nên ta có thể thực hiện truy vấn `UNION SELECT USERNAME_HHSDBW,PASSWORD_ZKJNDZ FROM USERS_UHGEMQ--` để trích xuất toàn bộ toàn khoản web:
![image](https://hackmd.io/_uploads/HkQwETcdbl.png)
&rarr; Lấy được thành công tài khoản của admin.
Sử dụng thông tin này để đăng nhập sẽ hoàn thành mục tiêu của bài lab: 
![image](https://hackmd.io/_uploads/H13cEa5ube.png)
### 4. Tiếp tục khám phá và phân tích thêm
Vì trên thực tế quá trình khai thác có thể không dễ dàng được như vậy, có thể mật khẩu được hash với salt bằng một thuật toán đủ mạnh nên ta không thể dịch ngược lại password để đăng nhập được. 
Vậy nên trước hết, ta thử xem với user hiện tại thì có thể thực hiện những hành động gì trong database:
![image](https://hackmd.io/_uploads/HkhdrTqOWg.png)
&rarr; thấy có 3 quyền đó là `CREATE SESSION`, `CREATE TABLE`, `UNLIMITED TABLESPACE`. 
Thử xem role được gán là gì:
![image](https://hackmd.io/_uploads/r1P-I69_Zl.png)
&rarr; 2 role được gán là `CONNECT` và `XDBADMIN`. 
Phân tích quyền và role có được:
- Quyền: 
    - `CREATE SESSION`: Đây là quyền tối thiểu nhất. Nếu không có nó, user thậm chí không thể đăng nhập vào database.
        - Khả năng: Kết nối vào database thông qua các công cụ như SQL Plus, SQL Developer hoặc ứng dụng web.
        - Giới hạn: Chỉ được phép kết nối, chưa được phép làm gì khác nếu không có các quyền tiếp theo.
    - `CREATE TABLE`: Đây là quyền cho phép user tạo ra các cấu trúc lưu trữ dữ liệu của riêng mình.
        - Khả năng: Tự tạo bảng mới: 
            - Tự tạo các index để tăng tốc truy vấn trên bảng của mình.
            - Toàn quyền INSERT, UPDATE, DELETE, DROP trên chính những bảng mà user đó tạo ra.
        - Giới hạn: Tự tạo các index để tăng tốc truy vấn trên bảng của mình.
    - `UNLIMITED TABLESPACE`: 
        - Khả năng: Cho phép user sử dụng không giới hạn dung lượng lưu trữ trong bất kỳ Tablespace nào của hệ thống.
        - Tầm quan trọng: Thông thường, khi tạo bảng, bạn cần có "Quota" (hạn mức dung lượng). Nếu có UNLIMITED TABLESPACE, bạn sẽ không bao giờ bị lỗi "insufficient quota" khi dữ liệu phình to.
- Role: 
    - `CONNECT`: Trong các phiên bản Oracle hiện đại (từ 10g trở đi), Role này khá khiêm tốn. Nó chủ yếu chỉ chứa quyền CREATE SESSION.
        - Ý nghĩa: Cho phép bạn giữ kết nối với Database. Nếu không có nó, các hành động khác đều vô nghĩa.
    - `XDBADMIN`: Oracle XML DB (XDB) là một thành phần dùng để quản lý dữ liệu XML, Web Services, và các giao thức như FTP/HTTP trực tiếp trên Database. Khi có quyền này bạn có thể:
        - Quản lý Repository: Bạn có quyền thay đổi cấu trúc cây thư mục của XDB (thường thấy qua đường dẫn /sys/acls/, /public/, v.v.).
        - Cấu hình giao thức: Bạn có thể thay đổi các tham số cấu hình cho HTTP/HTTPS, FTP của Database.
        - Thay đổi ACL (Access Control Lists): Bạn có quyền kiểm soát ai có thể xem hoặc sửa các tài nguyên XML trong hệ thống.
        - Thực thi các Package hệ thống: Bạn thường có quyền thực thi trên các package như DBMS_XDB hoặc DBMS_XDB_CONFIG.

Tiếp theo, với quyền này, ta nên kiểm tra xem User này có quyền `DBA` ẩn dưới một Role nào khác không, hoặc thử xem mình có quyền tác động vào các bảng của Schema `XDB` không qua truy vấn `UNION SELECT owner, table_name FROM all_tables WHERE owner = 'XDB'--`
![image](https://hackmd.io/_uploads/r1vd_pc_-x.png)
&rarr; có 3 bảng đó là `APP_ROLE_MEMBERSHIP`, `APP_USERS_AND_ROLES`, `XDB$XIDX_IMP_T`. Phân tích tác động:
- `APP_USERS_AND_ROLES` & `APP_ROLE_MEMBERSHIP`:
    - Ý nghĩa: Hai bảng này thường chứa danh sách toàn bộ người dùng của ứng dụng (không chỉ là user database) và các quyền hạn (Role) tương ứng của họ trong phần mềm.
    - Tại sao nó quan trọng: Bạn có thể trích xuất được thông tin đăng nhập, email, hoặc phân quyền của các Admin cấp cao hơn trong ứng dụng web.
- `XDB$XIDX_IMP_T`: Đây là một bảng hệ thống liên quan đến XML Index của Oracle (hệ quả của việc bạn có quyền XDBADMIN).
    - Ý nghĩa: Nó thường chứa thông tin tạm thời hoặc siêu dữ liệu (metadata) khi hệ thống import/export các chỉ mục XML.
    - Giá trị: Bảng này ít khi chứa dữ liệu người dùng trực tiếp, nhưng nó xác nhận rằng các tính năng XML nâng cao đang được sử dụng, cho phép bạn thực hiện các kỹ thuật như XXE (XML External Entity) nếu ứng dụng có cổng nhận dữ liệu XML.

Tới đây, ta cần liệt kê xem với những bài này thì user hiện tại có quyền gì qua truy vấn `UNION SELECT table_name||'->'||privilege,grantee FROM all_tab_privs WHERE table_name IN ('APP_ROLE_MEMBERSHIP','APP_USERS_AND_ROLES','XDB$XIDX_IMP_T')--`:
![image](https://hackmd.io/_uploads/SJroCp5Obl.png)
&rarr; hầu hết đều có quyền `DELETE`, `INSERT`, `SELECT`. 
Tiếp theo, liệt kê các cột trong từng bảng:
- `APP_USERS_AND_ROLES`:
![image](https://hackmd.io/_uploads/H1aukRcO-l.png)
- `APP_ROLE_MEMBERSHIP`:
![image](https://hackmd.io/_uploads/H1C51R9u-x.png)
- `XDB$XIDX_IMP_T`:
![image](https://hackmd.io/_uploads/H1Y6kAqubx.png)
![image](https://hackmd.io/_uploads/BkGRJRqdWg.png)

Tiến hành trích xuất nội dung của từng bảng:
- `APP_USERS_AND_ROLES`:
![image](https://hackmd.io/_uploads/By-Ez0qOWl.png)
&rarr; rõ ràng ta thấy bản thân có quyền SELECT, INSERT và DELETE nhưng lại không thể trích xuất nổi dữ liệu giống như một số bài trước đó, khá khó hiểu.
- `APP_ROLE_MEMBERSHIP`:
![image](https://hackmd.io/_uploads/Sk7sG0cdZl.png)
&rarr; tương tự bảng này cũng không được
- `XDB$XIDX_IMP_T`:
![image](https://hackmd.io/_uploads/HkfkQAquWl.png)

&rArr; Ta thấy cả 3 đều không thể truy xuất được gì. Khả năng cao do còn có một lớp phòng thủ nào đó phía hệ thống mà tôi không thể bypass được. 
Tới đây thì tôi cũng đã hết ý tưởng về việc mở rộng và tìm thêm hướng khai thác nên xin dừng bài viết ở đây.