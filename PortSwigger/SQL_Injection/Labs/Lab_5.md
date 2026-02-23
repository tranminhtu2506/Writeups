# Lab: SQL injection attack, listing the database contents on non-Oracle databases
## Quá trình phân tích và khai thác
### 1. Xác định điểm vào
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/B16tMjKOWe.png)
&rarr; Từ trang chủ, có 2 chức năng chính đó là lọc sản phẩm theo danh mục và đường dẫn tới trang login. 
Theo lý thuyết, ta cần ưu tiên kiểm thử những chức năng có khả năng cao có lỗi, cụ thể là những chức năng: Search, Filter, Sort vì tại đó những mệnh đề như `WHERE` hay `ORDER BY` có thể được sử dụng. Vậy nên trước hết, ta thử thực hiện chức năng lọc sản phẩm theo danh mục và quan sát request nó tạo ra:
![image](https://hackmd.io/_uploads/rJDYmjK_-g.png)
&rarr; Ta thấy request gửi tới endpoint /filter với tham số category trong URL. 
Trong các điểm vào của 1 request, tham số trong body hoặc trong URL là những điểm vào tiềm năng nhất và phổ biến có thể có lỗi, vì vậy, ta ưu tiên kiểm tra nó trước. Ta thử thêm ký tự `'` vào sau giá trị của tham số và quan sát response trả về:
![image](https://hackmd.io/_uploads/HyIeNiKOZx.png)
&rarr; Ta thấy response có status code 500 &rarr; ký tự ta chèn vào đã gây ra lỗi xử lý phía Server, tuy nhiên chưa thể khẳng định được có lỗi SQLi ở đây.
Tiếp theo, ta thử thêm lần lượt các ký tự đặc biệt vào khác xem có nhận được phản hồi tương tự không:
![image](https://hackmd.io/_uploads/Hk6EEitubg.png)
![image](https://hackmd.io/_uploads/Sk_SEiKuZe.png)
![image](https://hackmd.io/_uploads/HyG84oYuWg.png)
&rarr; Tất cả đều trả về response với status code 200, nhưng không có bất kỳ sản phẩm nào. 
&rArr; Từ đây có thể thấy trong bối cảnh giá trị tham số có khả năng được sử dụng trong 1 câu truy vấn, việc ta chèn vào ký tự `'` gây ra lỗi phía Server mà không ký tự đặc biệt nào khác làm được có thể xác minh rằng hệ thống đang bị lỗi SQL Injection. 
Để tiếp tục cuộc tấn công, ta cần biết chính xác kiểu database là gì. Trước hết, ta thử thêm vào chuỗi `'--` xem Server phản hồi thế nào:
![image](https://hackmd.io/_uploads/B1zVroYOWe.png)
&rarr; hệ thống trở về phản hồi như bình thường. Ta thử thay đổi `--` thành `#` xem sao:
![image](https://hackmd.io/_uploads/SJWDriFObl.png)
&rarr; vì ký tự `#` không thể comment được nên ta loại MySQL ra khỏi danh sách các DBMS.
### 2. Thu thập thông tin Database
Vì mỗi loại DBMS lại có cách truy vấn phiên bản khác nhau, vì vậy ta có thể lợi dụng việc này để tìm kiếm chính xác loại DBMS của mục tiêu.
Trước hết, có thể thấy kết quả của truy vấn được trả về trong response, vì vậy kỹ thuật nên được chọn sử dụng là `UNION BASED` nhằm trích xuất và thu thập thêm thông tin. 
Đầu tiên, ta cần xác định chính xác số lượng cột trong truy vấn gốc thông qua `ORDER BY`:
![image](https://hackmd.io/_uploads/BkdF8stOWx.png)
&rarr; Ta thấy khi tăng giá trị tới 3 thì Server trả về lỗi (do giá trị này lớn hơn số lượng cột trả về nên không thể sắp xếp theo cột đó được). Vì vậy số lượng cột trả về trong truy vấn gốc là 2.
Tiếp theo, dựa trên dữ liệu trả về, ta có thể dự đoán định dạng dữ liệu của cả 2 cột trả về đều là chuỗi, vì vậy thử sử dụng `UNION SELECT 'tmt1','tmt2'--` để kiểm chứng:
![image](https://hackmd.io/_uploads/SyxmwstuZg.png)
&rarr; Ta thấy input được trả về thành công trong response &rarr; ta tiếp tục loại được Oracle vì truy vấn phụ của ta không cần có `FROM dual`
Tiếp theo, còn 2 loại DBMS phổ biến là PostgreSQL và MSSQL, ta thử thực hiện truy vấn đề lấy phiên bản của 2 loại này để nhận diện chính xác:
- PostgreSQL:
![image](https://hackmd.io/_uploads/rkN6PoFuZx.png)

&rarr; May mắn thay chỉ cần thử một lần ta đã xác định được DBMS là PostgreSQL với phiên bản `PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit`.
### 3. Khám phá cấu hình và phân quyền
Sau khi đã xác định được phiên bản, ta cần xác định current user và current database qua truy vấn `UNION SELECT current_user,current_database()--`:
![image](https://hackmd.io/_uploads/ByBFdsYOZe.png)
&rarr; Ta lấy được user `peter` và database `academy_labs`.
Tiếp theo, ta cần xác định trong database này có những bảng nào thông qua truy vấn `UNION SELECT table_schema, table_name FROM information_schema.tables WHERE table_schema = 'public'--`:
![image](https://hackmd.io/_uploads/Sk34KoYdZx.png)
&rarr; Ta thấy có 2 bảng đó là `users_bkjmop` và `products`.
Ta thấy bảng `users_bkjmop` có khả năng chứa dữ liệu nhạy cảm, nhưng không chắc các cột của nó đã đặt tên theo quy tắc thông thường (bằng chứng là tên của bảng được thêm vào một chuỗi ngẫu nhiên), vì vậy, ta sẽ đi liệt kê cột của bảng này qua truy vấn `UNION SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users_bkjmop'--`:
![image](https://hackmd.io/_uploads/BymXqjK_-g.png)
&rarr; xác định được 3 cột: `username_ojkcqu, password_ycgill, email` và cả 3 đều ở định dạng chuỗi. 
### 4. Hoàn thành mục tiêu bài lab
Tiếp theo, ta sử dụng truy vấn `UNION SELECT username_ojkcqu,password_ycgill FROM users_bkjmop LIMIT 1--` để trích xuất username và password của user đầu tiên (thường là user admin):
![image](https://hackmd.io/_uploads/ryHC5iYOZe.png)
&rarr; Lấy được thông tin đăng nhập thành công. Ta có thể mang thông tin này để thực hiện login với tài khoản admin:
![image](https://hackmd.io/_uploads/S1BXijK_Zl.png)
### 5. Tiếp tục khám phá 
Tuy đã hoàn thành mục tiêu của bài lab, tuy nhiên trong bối cảnh của cuộc tấn công, tôi sẽ tiếp tục khá phá thêm các thông tin để xem có thể mở ra hướng tấn công mới hay gia tăng tác động không. 
Sau khi đã lấy được tài khoản của admin, ta thử xem có thể sử dụng tài khoản này để thực hiện những chức năng gì thay đổi trạng thái của hệ thống không.
Ta thử xem source code của trang này:
![image](https://hackmd.io/_uploads/rke-G19u-g.png)
&rarr; Không có đường dẫn ẩn nào trên trang. Vì ta thấy ở đây trang tải profile dựa trên tham số id, thử thay đổi giá trị này: 
![image](https://hackmd.io/_uploads/S1cVfy9u-g.png)
&rarr; Ta bị điều hướng tới trang /login &rarr; Không có lỗ hổng IDOR ở điểm này. 
Thử truy cập một số đường dẫn tới trang dashboard admin phổ biến:
![image](https://hackmd.io/_uploads/SyRFzycObl.png)
&rarr; Cũng không có gì. Quay trở lại trang chủ cũng không có gì:
![image](https://hackmd.io/_uploads/Hk-2f1qObx.png)
Ta nhận thấy khi liệt kê các cột hiện có trong bảng user, chỉ có `username, password, email`, không hề có cột `role`, vì vậy khả năng tài khoản admin cũng chỉ ngang quyền với các tài khoản khác. Vì vậy, ta tạm bỏ qua việc khai thác chức năng dành cho admin vì không khả thi, thử quay trở lại với lỗ hổng SQLi. Giờ ta cần xem ta có những quyền gì trên bảng user này thông qua truy vấn `UNION SELECT 'tmt1',string_agg(privilege_type,',') FROM information_schema.table_privileges WHERE grantee=current_user AND table_name = 'users_...'--`:
![image](https://hackmd.io/_uploads/rJZAmy5dbg.png)
&rarr; Ta thấy có khá nhiều quyền trên bảng này như: `INSERT, SELECT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER`. Tuy nhiên, hầu hết sẽ khá vô dụng nếu ta không thể thực hiện được Stack Queries, vì vậy ta thử thêm ký tự `;` vào cuối xem phản hồi từ Server thế nào:
![image](https://hackmd.io/_uploads/rkuhV15_Zg.png)
&rarr; Response trả về có status code 500 cho thấy hệ thống có cơ chế ngăn chặn hành động này. 
### 6. Tổng kết 
Trong bối cảnh này, tôi nghĩ đến việc liệt kê hết tất cả các tài khoản hiện có trong hệ thống. Mục đích là để có nhiều vector tấn công sau này và để lỡ tài khoản admin có bị thay đổi thông tin đăng nhập, thì vẫn có thể truy cập lại hệ thống với tài khoản khác. Bên cạnh đó cũng có khả năng tài khoản admin ta tìm được chỉ là một mồi nhử, là một tài khoản rác không có bất kỳ quyền gì còn tài khoản admin thật thì lại được đặt với một username rất bình thường. Tuy nhiên vì đây là một bài lab nên khả năng này không cao, và mỗi lần làm lại thì thông tin đăng nhập cũng thay đổi nên không cần phải lưu trữ thông tin tài khoản khác làm gì. 
Dựa vào tình hình hiện tại: chiếm được tài khoản admin nhưng không thể thực hiện hành động nào thay đổi trạng thái hệ thống ngoại trừ update email (có một khả năng nhỏ có chứa lỗ hổng IDOR trong quá trình update email nhưng chiếm một tài khoản admin chỉ để update email của tài khoản thường qua IDOR thì không hợp lý cho lắm, vì đáng lẽ admin đã tự có quyền làm được điều đó nếu muốn rồi); có lỗ hổng SQLi và có rất nhiều quyền nhưng lại không thể tận dụng vì không có Stack Queries; Bảng còn lại là bảng Product như từ những bài lab trước cũng đã thấy không có quá nhiều giá trị khi liệt kê bảng này và kể cả có thì cũng không làm gì được trong bối cảnh `UNION SELECT`.
&rArr; Vì vậy tôi xin dừng bài viết tại đây.
