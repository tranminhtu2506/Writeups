# Lab: SQL Injection Vulnerability in WHERE clause allowing retrieval of hidden data

## I. Quá trình phân tích và khai thác (Phần chính)
Tôi tiếp cận bài này dựa trên tài liệu SQL Injection Overview đã xây dựng trước đó, dưới góc nhìn của một kẻ tấn công. Mục tiêu của tôi không chỉ là hoàn thành thử thách, mà còn là ghi lại quá trình nhận diện, chứng minh và khai thác lỗ hổng một cách triệt để. Do đó, bài viết sẽ đi sâu vào chi tiết kỹ thuật thay vì chỉ đưa ra lời giải ngắn gọn. 

### 1. Xác định điểm vào 
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.

Đầu tiên, khi truy cập trang web, ta được trả về giao diện:

![image](https://hackmd.io/_uploads/BkZJDYSu-l.png)
![image](https://hackmd.io/_uploads/BJSJvtSu-x.png)

&rarr; Ta thấy có hai chức năng chính đó là xem sản phẩm theo danh mục và xem thông tin chi tiết của một sản phẩm.

Bước tiếp theo, ta cần xác định chức năng và tham số có thể làm vector cho cuộc tấn công. Trong lý thuyết có đề cập tới những chức năng như: Search, Filter, Sort là nơi dễ có lỗ hổng; vì tại đây SQL thường được dùng để xây dựng mệnh đề WHERE hoặc ORDER BY &rarr; Ta tiến hành kiểm thử chức năng này trước:

![image](https://hackmd.io/_uploads/BkeVFKSu-l.png)
![image](https://hackmd.io/_uploads/HJLNKFSdZl.png)

&rarr; Ta thấy request được tạo ra gửi tới endpoint /filter. 

Tiếp theo, việc cần làm là xác định các điểm vào có khả năng gây ra lỗi. Ta thấy tham số trong URL được liệt kê vào phần những điểm vào phổ biến nhất &rarr; ta tập trung kiểm tra trước. Thử chèn vào ký tự ‘ trong giá trị và nhận được phản hồi:

![image](https://hackmd.io/_uploads/HyMtKYHuWg.png)

&rarr; Ta thấy nhận được response có status code 500. Đây là 1 trong những tín hiệu tốt cho thấy ký tự ta thêm vào đã gây ra lỗi nào đó trên hệ thống. Trên thực tế, đây có thể là lỗi sinh ra từ quá trình xử lý dữ liệu phía Server, tuy nhiên giá trị ở đây vốn là 1 chuỗi, vì vậy khả năng bị xảy ra lỗi do ép kiểu dữ liệu là không cao &rarr; khả năng bị lỗi SQL Injection là rất cao.

Để kiểm chứng, ta thêm ký tự comment phổ biến trong các loại CSDL và quan sát response được trả về:

![image](https://hackmd.io/_uploads/HJbbcYruZg.png)

&rarr; Ta thấy phản hồi trở lại bình thường &rarr; có thể khẳng định được lỗi server trước đó là do ký tự ‘ ta chèn vào gây ra lỗi cú pháp trong câu truy vấn.

### 2. Thu thập thông tin Database 
Sau khi xác định được điểm vào có lỗ hổng, tiếp theo ta cần dựa trên các dấu hiệu cũng như tìm cách thu thập thêm thông tin về database để đưa ra hướng tấn công tiếp theo. Trong trường hợp này, ta thấy dữ liệu được trả về trong phản hồi, vì vậy hướng tới việc sử dụng UNION để trích xuất thêm thông tin từ các bảng khác.

Trước hết, ta sử dụng ORDER BY để xác định số lượng cột được trả về trong truy vấn gốc:

![image](https://hackmd.io/_uploads/HJ9DqKHdbe.png)
![image](https://hackmd.io/_uploads/ry0D9YBOWl.png)

&rarr; Ta thấy khi tăng giá trị cột lên 9 thì bị lỗi &rarr; số lượng cột được trả về trong truy vấn gốc là 8.

Tiếp theo, ta cần xác định kiểu dữ liệu của từng cột. Thử nghiệm bằng UNION SELECT NULL:

![image](https://hackmd.io/_uploads/BkzcqYBdZx.png)

&rarr; lần lượt thay giá trị NULL thành 1 chuỗi đơn giản như ‘a’ xem có cột nào có kiểu giá trị khác không:

![image](https://hackmd.io/_uploads/SJeoqtruZg.png)

&rarr; Sau 1 quá trình thử nghiệm xác định được những cột có kiểu dữ liệu là số và chuỗi. 

Tiếp theo, để có thể thu thập thêm thông tin, ta cần phải xác định được loại DBMS đang được sử dụng. Trong quá trình thử nghiệm, ta thấy khi sử dụng UNION SELECT không cần có from dual &rarr; loại Oracle khỏi danh sách.

Ta thử thay thế vào vị trí cột có kiểu chuỗi bất kỳ để xác định phiên bản của CSDL:
- `@@version`: Lỗi 500 &rarr; Loại bỏ MySQL và MSSQL

![image](https://hackmd.io/_uploads/Sy-8oFSdbg.png)

- `version()`: Không báo lỗi, nhưng cũng không hiển thị trên giao diện.

![image](https://hackmd.io/_uploads/rJeYjKr_bx.png)

&rarr; ta không thấy giá trị version() trên giao diện, nhưng thấy có chữ b, ta thử thay thế vị thành vị trí của b xem có phản hồi thế nào:

![image](https://hackmd.io/_uploads/B1v9iYHdZe.png)

&rarr; xác định được kiểu DBMS là PostgreSQL với phiên bản 12.22.

### 3. Khai thác thông tin cấu hình và phân quyền 
Sau khi biết hệ thống dùng PostgreSQL, tiếp tục trích xuất thông tin User và Database hiện tại:
`' UNION SELECT 111, 'a', current_user || '~' || current_database(), 222, 333, 'd' ...`

![image](https://hackmd.io/_uploads/rJOfhtHO-e.png)
![image](https://hackmd.io/_uploads/HJJXhKrObg.png)

&rarr; Xác định được có hai user là peter và postgres, nhưng không lấy được password. Tới đây, ta sẽ xác định xem hiện tại ta đang là user nào và database đang sử dụng là gì:

![image](https://hackmd.io/_uploads/SkkD3Yrd-l.png)

&rarr; Xác định được user hiện tại là peter và database hiện tại là academy_labs.

Tiếp theo, ta cần biết user peter có những quyền gì trong database. Vì việc liệt kê role có thể phức tạp và dài dòng, ta thực hiện kiểm tra xem user hiện tại có 1 số role nhạy cảm hoặc quan trọng không:
- Superuser: kết quả là "off" &rarr; ta không có quyền này.

![image](https://hackmd.io/_uploads/SJ-pnYBdbx.png)

- pg_read_server_files: kết quả là "false" &rarr; ta cũng không có quyền này.

![image](https://hackmd.io/_uploads/BJRxaKSdbg.png)

- pg_execute_server_program: kết quả vẫn là "false" &rarr; ta cũng không có quyền này. 

![image](https://hackmd.io/_uploads/BJr7TKBu-x.png)

&rArr; Tới đây, ta thay đổi từ việc tấn công hệ thống sang các quyền thông thường như tạo dữ liệu hay update hay xóa dữ liệu database. Tuy nhiên quyền này trên các table khác nhau lại có thể khác nhau, vì vậy trước hết ta cần liệt kê toàn bộ bảng đang có trong database hiện tại:

![image](https://hackmd.io/_uploads/S10KTtBu-g.png)

&rarr; Xác định chỉ có duy nhất bảng "products".

Tiếp theo liệt kê quyền của user hiện tại trên bảng này:

![image](https://hackmd.io/_uploads/BJP66tS_bx.png)

&rarr; Ta thấy ta có rất nhiều quyền như INSERT, SELECT, UPDATE, DELETE,…. 

Trong bối cảnh của điểm vào gây lỗi, có thể dễ dàng xác định nó nằm trong 1 câu mệnh đề WHERE, vì vậy dù có nhiều quyền thì tác động vẫn chỉ có hạn chế. Vì vậy, ta cần biết xem có thể stack queries ở đây không:

![image](https://hackmd.io/_uploads/Hy-1AYHube.png)

&rarr; Ta thấy khi thêm dấu ; vào cuối thì bị trả về lỗi 500 ngay &rarr; khả năng cao là không thể stack queries. 

### 4. Thử nghiệm khai thác nâng cao
Thông qua những quyền hiện có của user hiện tại với bảng products, tôi có tìm hiểu thì thấy có khả năng có thể thực hiện được những hành động độc hại sau: 
- Sửa đổi/Xóa dữ liệu thông qua "Data-Modifying CTE": Đây là vũ khí mạnh nhất của PostgreSQL. Khác với các DBMS khác, Postgres cho phép bạn chèn các câu lệnh INSERT, UPDATE, DELETE vào trong một câu lệnh SELECT duy nhất bằng từ khóa WITH.
    - Kịch bản: Xóa sạch bảng products (TRUNCATE trá hình):

![image](https://hackmd.io/_uploads/SkMV0KS_bl.png)

        - Hành động: Câu lệnh này xóa toàn bộ dữ liệu trong bảng products.
        - Tại sao nó nguy hiểm: Nó hoàn toàn là một câu lệnh SELECT hợp lệ về mặt cú pháp, nên nó vượt qua được các bộ lọc chặn dấu ;.

- Chiếm quyền điều khiển tài khoản (Account Takeover): Nếu bạn tìm thấy bảng users, bạn có thể đổi mật khẩu của Admin ngay lập tức:

![image](https://hackmd.io/_uploads/BJSO0Kr_bx.png)

- Khai thác quyền REFERENCES để tìm mối quan hệ ẩn:
    - Quyền REFERENCES cho phép bạn tạo ra các bảng mới tham chiếu đến bảng hiện có. Dù không thể CREATE TABLE trực tiếp (vì thiếu ;), nhưng nếu ứng dụng có một tính năng nào đó cho phép tạo bảng/view, bạn có thể lợi dụng nó để bypass các kiểm tra ràng buộc dữ liệu hoặc thực hiện các cuộc tấn công Inference Attack (suy luận dữ liệu dựa trên lỗi khóa ngoại).
- Khai thác quyền TRIGGER: 
    - Nếu bạn có quyền TRIGGER, bạn có thể tạo hoặc thay đổi hành vi của các trigger hiện có (nếu kết hợp được với CTE). Bạn có thể thiết lập để mỗi khi có ai đó chèn dữ liệu vào bảng products, một bản sao sẽ được gửi vào bảng mà bạn kiểm soát, hoặc thực hiện một hành động độc hại khác.
- Tấn công từ chối dịch vụ (DoS) thông qua hàm nặng: Với quyền SELECT, bạn có thể thực thi các hàm hệ thống làm "treo" CPU hoặc chiếm dụng hết bộ nhớ RAM của database server:

![image](https://hackmd.io/_uploads/SymBPoS_Zg.png)

- Sử dụng hàm set_config để làm thay đổi log hệ thống: Nếu bạn muốn xóa dấu vết, bạn có thể thử thay đổi các cấu hình session: UNION SELECT NULL, set_config('log_statement', 'none', false), NULL... Điều này có thể ngăn việc ghi lại các câu lệnh SQL độc hại của bạn vào file log của server.

&rarr; vì ở đây chỉ có bảng Product, ta sẽ thấy có 1 số hành động thú vị có thể thử nghiệm như update hoặc thêm 1 sản phẩm thông qua WITH. Hoặc thay đổi hành vi TRIGGER như được nói, hoặc sử dụng set_config,...

Đầu tiên, phải biết được tên của các cột trong bảng products:

![image](https://hackmd.io/_uploads/HJcYvsH_-g.png)

Kiểm tra 1 số biện pháp phòng thủ có thể ngăn chặn ta update hoặc ghi:

![image](https://hackmd.io/_uploads/HyJiwjB_Wx.png)

&rarr; transaction_read_only đã bị tắt. Cũng không có trigger:

![image](https://hackmd.io/_uploads/ryTiPordWx.png)

Tới đây, trên lý thuyết ta phải có thể thực hiện 1 số hành động như delete, update hay insert trên bảng products, tuy nhiên khi thực hiện câu lệnh lại bị trả về như sau:
- `UNION+SELECT+111,'a',(WITH+updated+AS+(UPDATE+products+SET+name+=+'PENTEST_SUCCESS'+WHERE+id=1+RETURNING+name)+SELECT+COALESCE((SELECT+name::text+FROM+updated+LIMIT+1),'Update_Failed'))`
![image](https://hackmd.io/_uploads/rJEldiBdZe.png)
- `UNION+SELECT+111,'a',(WITH+inserted+AS(INSERT+INTO+products+(id,name)+VALUES+(1337,'INJECTION_TEST')+RETURNING+name)+SELECT+name::text+FROM+inserted+LIMIT+1),'Update_Failed')),222,333,'"+onerror="alert(1)',444,'d'+--`
![image](https://hackmd.io/_uploads/BJvb_sBube.png)
- `UNION+SELECT+111,'a',(WITH+deleted+AS+(DELETE+FROM+products+WHERE+id+=+1337+RETURNING+id)+SELECT+COALESCE((SELECT+id::text+FROM+deleted+LIMIT+1),+'Delete_Failed')),222,333,'"+onerror="alert(1)',444,'d'+--`
![image](https://hackmd.io/_uploads/BkqM_sSOWx.png)

&rarr; Đều không thành công. Có thể giải thích qua 4 lý do kỹ thuật sau: 
- Giới hạn của PostgreSQL đối với DML trong Subqueries
    - Đây là nguyên nhân phổ biến nhất. PostgreSQL có một quy tắc ngầm: Không cho phép các câu lệnh thay đổi dữ liệu (DML) nằm trong một truy vấn con (Subquery) của một lệnh SELECT thuần túy, trừ khi nó được thực thi trong một bối cảnh cụ thể.
    - Mặc dù cú pháp WITH ... RETURNING là hợp lệ, nhưng khi nó bị ép vào làm một "cột" của lệnh UNION SELECT, trình tối ưu hóa truy vấn (Query Planner) của Postgres có thể từ chối thực thi vì nó không thể đảm bảo tính nhất quán của dữ liệu khi một câu lệnh đọc (SELECT) lại cố tình sinh ra tác động ghi (INSERT/UPDATE).
- Sự ngăn chặn của Database Driver (thư viện kết nối): Các thư viện như PDO (PHP), psycopg2 (Python), hoặc các Driver Java thường có cơ chế tách biệt luồng dữ liệu.
    - Khi ứng dụng gọi hàm executeQuery() (dành cho SELECT), nó mong đợi một tập kết quả trả về.
    - Nếu Driver phát hiện trong câu lệnh có chứa các từ khóa thay đổi dữ liệu như UPDATE/INSERT lồng bên trong, nó có thể tự ngắt kết nối và trả về lỗi trước khi gửi lệnh tới Database Server để bảo vệ an toàn hệ thống.
- Quy trình thực thi “Atomic” của UNION:
    - Trong một lệnh UNION, PostgreSQL cần xác định kiểu dữ liệu của tất cả các cột trước. Khi bạn đặt một CTE (WITH...) vào cột thứ 3, Postgres phải thực thi CTE đó để lấy giá trị trả về. Nếu quá trình này gặp bất kỳ sự không tương thích nào về kiểu dữ liệu giữa các "nhánh" của UNION, hoặc nếu kết quả trả về từ RETURNING không khớp hoàn hảo với định dạng cột, nó sẽ ném ra lỗi 500.
- Cơ chế WAF hoặc Database Firewall: Có thể có một Web Application Firewall (WAF) hoặc một lớp trung gian (như pg_firewall) đang quét nội dung câu lệnh SQL của bạn.
    - Nó cho phép các hàm "vô hại" như set_config hoặc string_agg.
    - Nhưng nó sẽ chặn đứng bất kỳ câu lệnh nào chứa tổ hợp: UNION + UPDATE/INSERT/DELETE. Đây là một dấu hiệu quá rõ ràng của một cuộc tấn công "Data-modifying SQL Injection".

Tới đây, ta thử thử nghiệm xem có thể sử dụng set_config không:
![image](https://hackmd.io/_uploads/SJCFOoSOZg.png)

&rarr; có thể gọi hàm này để tắt ghi log. 

### 5. Hoàn thành mục tiêu bài lab
Tới đây thì hướng tấn dễ và mang nhiều giá trị nhất hiện tại cho ta là trích xuất toàn bộ dữ liệu trong bảng products, tuy nhiên bảng này vốn dĩ đã được sử dụng trong câu truy vấn gốc nên không cần sử dụng UNION. Trong quá trình nỗ lực để gia tăng tác động, khi liệt kê tên các cột trong bảng này thấy có cột released (phát hành):

![image](https://hackmd.io/_uploads/ryusOoH_-l.png)

&rarr; khả năng cao có những sản phẩm không được phát hành đang ẩn dấu trong CSDL. Ta lợi dụng lỗ hổng SQL Injection, thêm vào điều kiện logic OR 1=1 nhằm liệt kê tất cả các sản phẩm trong database xem thế nào:

![image](https://hackmd.io/_uploads/H1Ah_iHuWx.png)

&rarr; Thành công liệt kê tất cả các sản phẩm. Tới đây cũng hoàn thành bài lab.



## II. Phần mở rộng: Chaining Vulnerabilities
Trong quá trình thay thế các giá trị giả (`'tmt1'`, `'tmt2'`,...) vào các cột của `UNION SELECT` để tìm vị trí phản chiếu (reflection) dữ liệu lên giao diện, tôi đã phát hiện một số điểm thú vị

* **Cột 1 & 4:** Phản chiếu vào thuộc tính `href` của thẻ `<a>`, nhưng chỉ nhận giá trị số => Khả năng tấn công thấp
![image](https://hackmd.io/_uploads/B1oUKiHdWl.png)
![image](https://hackmd.io/_uploads/B11wFsBObx.png)
* **Cột 3:** Phản chiếu vào nội dung thẻ `<h3>` => Bị HTML encode, khó chèn thẻ mới
![image](https://hackmd.io/_uploads/B1XFtjru-e.png)
* **Cột 6:** Phản chiếu vào thuộc tính `src` của thẻ `<img>` => **Mục tiêu rất tiềm năng!**
![image](https://hackmd.io/_uploads/H1oqKjHOZx.png)

### 1. Phân tích lỗ hổng Reflected XSS
Vì cột 6 nằm trong thuộc tính của thẻ HTML, ta có thể dùng kỹ thuật thoát khỏi thuộc tính và chèn thêm event handler độc hại. 
Payload thử nghiệm: 
`' UNION SELECT 111, 'tmt1', 'tmt2', 222, 333, 'tmt3" onerror="alert(1)', 444, 'tmt4'--`
![image](https://hackmd.io/_uploads/S1l8RYsHd-x.png)

Khi truy cập trang từ trình duyệt, mã JavaScript `alert(1)` đã được kích hoạt!

![image](https://hackmd.io/_uploads/S1ikqjH_bl.png)

=> **Kết luận:** Lỗ hổng SQL Injection đã trở thành vector để khai thác thêm lỗ hổng **Reflected XSS**

### 2. Phân tích lỗ hổng Local File Inclusion (LFI)
Vì có thể kiểm soát giá trị `src` trong thẻ `<img>`, tôi đã thử nhúng đường dẫn file hệ thống để ép đọc file nội bộ:
`../../../../../../../../etc/passwd`
![image](https://hackmd.io/_uploads/HyBN9oHObl.png)

Khi mở ảnh trong tab mới, server trả về `Not Found`

![image](https://hackmd.io/_uploads/rkKS5jSd-x.png)

=> Lý do có thể do server đã cố định đường dẫn trong thư mục `/image` hoặc có các ràng buộc bảo mật khác.