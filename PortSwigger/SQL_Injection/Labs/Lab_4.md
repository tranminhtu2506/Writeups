# Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft
## Quá trình phân tích và khai thác
### 1. Xác định điểm vào
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/H1ToBLFdZg.png)
&rarr; Ta thấy trang chỉ có liệt kê xem sản phẩm theo danh mục. Theo lý thuyết, đây là một trong những chức năng khả năng cao sẽ có lỗ hổng SQL Injection, vì nơi này thường sử dụng truy vấn với mệnh đề WHERE để lọc sản phẩm. 
Tiếp theo, ta cần thực hiện chức năng để quan sát request nhằm xác định các điểm vào có khả năng gây lỗi: 
![image](https://hackmd.io/_uploads/BySuI8tOWe.png)
&rarr; Request được gửi tới endpoint /filter với tham số category trong URL. Đây là một trong những điểm vào tiềm năng và phổ biến của loại lỗi này, vì vậy ta ưu tiên kiểm thử trước. 
Trước hết, thêm vào ký tự `'` trong giá trị của tham số và quan sát response trả về:
![image](https://hackmd.io/_uploads/ryxp6LIFubl.png)
&rarr; Đã thấy response với status code 500. Đây là một dấu hiệu cho thấy có lỗi trong quá trình xử lý ở phía Server, nhưng chưa thể khẳng định là có lỗ hổng SQL Injection. 
Tiếp theo, ta lần lượt thay thế `'` bằng những ký tự đặc biệt khác xem có cùng phản hồi không:
![image](https://hackmd.io/_uploads/HkcEDUtdWg.png)
![image](https://hackmd.io/_uploads/HyU8wIFdbg.png)
![image](https://hackmd.io/_uploads/HkzvP8Y_bx.png)
&rarr; Tất cả phản hồi đều trả về với status code 200 và không sản phẩm nào được liệt kê. Điều này cho thấy đối với một chuỗi không phù hợp, thông thường nó phải trả về kết quả trống, tuy nhiên ký tự `'` lại gây ra lỗi Server, mà giá trị tham số này khả năng cao lại được dùng trong truy vấn &rarr; Khả năng cao ký tự ta chèn vào đã gây lỗi trong câu truy vấn hoặc trong quá trình truy vấn. 
Để xác minh, ta thử thêm vào chuỗi ký tự `'--` với mục đích bổ sung thêm ký tự comment để khiến ký tự `'` bị thừa không gây ra lỗi. Nếu response trả về status code 200 thì coi như thành công:
![image](https://hackmd.io/_uploads/r12dd8YO-g.png)
&rarr; Ở đây ta thấy response lại trả về status code 500, chứng tỏ cú pháp comment của ta chưa giải quyết được vấn đề cú pháp. Trong các loại DBMS phổ biến, chỉ có MySQL là có thể sử dụng ký tự `#` thay thế để comment, ta thử thay thế trong request và quan sát:
![image](https://hackmd.io/_uploads/rJOMYLKubg.png)
&rarr; Đã khiến response trả về với status code 200, và các sản phẩm vẫn được liệt kê (chứng tỏ có giá trị tương đương với chuỗi gốc) &rarr; Tới đây, ta xác minh được trang này có lỗ hổng SQL Injection qua tham số category của chức năng filter.

### 2. Thu thập thông tin Database
Theo lý thuyết, sau khi xác định được có lỗ hổng, việc cần làm là dựa vào cách phản hồi của Server để lựa chọn kỹ thuật tấn công phù hợp. Ở đây kết quả của truy vấn được trả về trong response, vì vậy hướng tấn công phù hợp là sử dụng `UNION SELECT` để trích xuất và thu thập thêm thông tin. 
Trước hết, ta cần xác định số lượng cột trả về trong truy vấn gốc, ở đây ta sử dụng `ORDER BY 1#` và tăng dần giá trị cho tới khi gặp lỗi: 
![image](https://hackmd.io/_uploads/SJ2tcItOWg.png)
&rarr; Ta thấy khi tăng giá trị tới 3 thì Server trả về lỗi &rarr; xác định được số lượng cột trong truy vấn gốc là 2. 
Tiếp theo, cần xác định định dạng của dữ liệu trong các cột trả về là gì, ta sử dụng `UNION SELECT 'tmt1','tmt2'#` để xem response trả về thế nào (vì trong truy vấn gốc thì kết quả trả về bao gồm tên và mô tả sản phẩm nên dự đoán là kiểu chuỗi): 
![image](https://hackmd.io/_uploads/SJ74j8t_Ze.png)
&rarr; ta thấy dữ liệu có trả về trong respone mà không gây ra lỗi &rarr; xác minh được kiểu dữ liệu trả về ở dạng chuỗi. 
Tiếp theo, ta cần xác định phiên bản DBMS đang được sử dụng (vì nếu phiên bản sử dụng đã lỗi thời sẽ mở thêm cho ta nhiều hướng tấn công hơn) qua truy vấn `UNION SELECT 'tmt1',VERSION()#`:
![image](https://hackmd.io/_uploads/r1i2jUF_We.png)
&rarr; xác định được phiên bản hiện tại là `8.0.42-0ubuntu0.20.04.1`. Tới đây về cơ bản ta đã hoàn thành mục tiêu của bài lab:
![image](https://hackmd.io/_uploads/BJnk3LYdWg.png)
Tuy nhiên trong bối cảnh mô phỏng một cuộc tấn công, tôi vẫn sẽ tiếp tục khám phá và khai thác xem có thể gia tăng tác động hay không hay có thể mở ra hướng tấn công nào khác hay không.

### 3. Khám phá cấu hình và phân quyền
Tiếp theo, ta sử dụng truy vấn `UNION SELECT USER(),DATABASE()#` để xác định current user và current database:
![image](https://hackmd.io/_uploads/SktF3UYu-g.png)
&rarr; Ta thấy current user là `peter@localhost` và current database là `academy_labs`.
Tiếp theo, ta sẽ cần biết trong database hiện tại thì có những bảng nào qua truy vấn `UNION SELECT 'tmt1',table_name FROM information_schema.tables WHERE table_schema = DATABASE()#`:
![image](https://hackmd.io/_uploads/SJ6vpUKdWx.png)
&rarr; Chỉ có bảng products. 
Trước khi đi vào liệt kê các cột có trong bảng, ta thử kiểm tra xem có thể thực thi những hành động gì với bảng này qua truy vấn `UNION SELECT 'tmt1',GROUP_CONCAT(table_name,':',privilege_type) FROM information_schema.table_privileges WHERE table_schema=DATABASE()#`:
![image](https://hackmd.io/_uploads/Hk1SRLK_Zg.png)
&rarr; ta thấy kết quả trả về lại là trống (cho phần quyền). Có thể lý giải bằng một số lý do:
- Quyền được cấp ở cấp độ Database (Schema Privileges): 
    - Trong MySQL, quyền không phải lúc nào cũng được gán trực tiếp cho từng bảng (Table-level). Thông thường, quản trị viên sẽ cấp quyền cho User trên toàn bộ Database:
        - Ví dụ: GRANT SELECT ON shop_db.* TO 'user'@'localhost';
    - Khi đó, bảng table_privileges sẽ trống không, vì quyền nằm ở bảng schema_privileges.
    - Cách kiểm tra: 
        - `UNION SELECT 1, GROUP_CONCAT(privilege_type) FROM information_schema.schema_privileges 
WHERE table_schema = DATABASE()-- -`
- Đang dùng User có quyền "Global"
    - Nếu User của bạn có quyền cực cao (như SUPER hoặc quyền SELECT trên mọi database *.*), MySQL sẽ lưu thông tin này trong bảng user_privileges thay vì chi tiết từng bảng.
    - Cách kiểm tra: `UNION SELECT 1, GROUP_CONCAT(privilege_type) FROM information_schema.user_privileges-- -`
- Cơ chế implicit privileges (quyền ngầm định): 
    - Nếu bạn có thể nhìn thấy tên bảng thông qua UNION SELECT, điều đó có nghĩa là bạn chắc chắn có ít nhất quyền SELECT (hoặc quyền SHOW VIEW) trên bảng đó.
        - Trong MySQL, nếu bạn là chủ sở hữu (Owner) của database hoặc bảng, đôi khi các bảng hệ thống về privileges không liệt kê tường minh các quyền cơ bản vì chúng được mặc định là có toàn quyền.

Ta thử kiểm tra bằng những truy vấn vừa tìm được xem có phản hồi thế nào: 
![image](https://hackmd.io/_uploads/rJ1leDYdZx.png)
![image](https://hackmd.io/_uploads/SygPgwKdZl.png)
&rarr; Ta thấy user hiện tại chỉ có quyền `USAGE`. Theo tìm hiểu thì quyền USAGE thực chất là "không có quyền gì cả" (No privileges). Nó chỉ đơn giản xác nhận rằng User này có tồn tại trong hệ thống và có thể đăng nhập, nhưng không được cấp bất kỳ quyền thao tác đặc biệt nào như SELECT, INSERT, UPDATE hay DELETE trên các bảng quản trị hệ thống.
Ta thử xem với bài lab này Server có cho phép thực hiện stack queries không:
![image](https://hackmd.io/_uploads/SkNkzPFO-e.png)
&rarr; ta thấy ở đây khi thêm ký tự `;` vào thì vẫn nhận được phản hồi với status code 200. Tuy nhiên, khi thử nghiệm với `; SLEEP(3)#` thì response trả về:
![image](https://hackmd.io/_uploads/rkrfQwYObg.png)
&rarr; Có thể kết thúc một truy vấn nhưng không hỗ trợ Stack Queries. 

Tuy nhiên, có những lúc mật khẩu hoặc token của các dịch vụ khác (như AWS key, API key) được lưu trực tiếp vào cấu hình của database. Ta thử thực hiện truy vấn `UNION SELECT @@hostname, @@version_compile_os FROM products#` hoặc truy vấn `UNION SELECT @@datadir, @@basedir FROM products#` để kiểm tra các biến liên quan đến đường dẫn:
![image](https://hackmd.io/_uploads/HJE1Bwt_We.png)
![image](https://hackmd.io/_uploads/r1LWrDKuWe.png)
&rarr; Không có gì hữu ích trong việc mở rộng tấn công hoặc gia tăng tác động. 
Cuối cùng, vì dữ liệu được reflected vào trong response, ta thử xem có thể chèn script vào không:
![image](https://hackmd.io/_uploads/Hy-QIPK_We.png)
&rarr; bị encode HTML &rarr; không khả thi với tấn công này. 

&rArr; Vì tới đây không khai thác được gì thêm và cũng hết hướng đi nên tôi kết thúc bài viết ở đây.