# Lab: SQL injection attack, querying the database type and version on Oracle
## Quá trình phân tích và khai thác
### 1. Xác định điểm vào 
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/BkRoLF_dbx.png)
![image](https://hackmd.io/_uploads/r1MkDFuubx.png)


&rarr; Từ trang chủ, ta thấy trang chỉ có chức năng hiển thị mô tả sản phẩm các sản phẩm dựa trên danh mục. 
Tới đây, ta cần thực hiện chức năng và quan sát mà nó tạo ra để xác định các điểm vào có khả năng kích hoạt lỗi:
![image](https://hackmd.io/_uploads/S1XYwtOO-e.png)

&rarr; Theo lý thuyết, các điểm vào được coi là phổ biến của loại lỗi này là những tham số trong URL hoặc Body của request, vì vậy ta ưu tiên kiểm thử trước. 
Trước hết, ta thử thêm vào ký tự `'` để quan sát phản hồi trả về:
![image](https://hackmd.io/_uploads/HJ9fut_uZx.png)

&rarr; Ta thấy response và về có status code 500 &rarr; input ta truyền vào đã gây ra lỗi trong quá trình xử lý ở phía BE. Đây là 1 dấu hiệu tốt cho thấy có khả năng có lỗi SQLi ở đây; tuy nhiên chưa thể khẳng định được, vì vậy ta thử thay thế bằng một số ký tự đặc biệt khác xem sao:
![image](https://hackmd.io/_uploads/HkiddYudbg.png)
![image](https://hackmd.io/_uploads/SJ5tdt_uWl.png)
![image](https://hackmd.io/_uploads/H1Yc_YO_We.png)

&rarr; ta thấy không có bất kỳ ký tự nào khác gây ra lỗi 500. Ta thử thêm chuỗi ký tự `'--` xem hệ thống phản hồi thế nào:
![image](https://hackmd.io/_uploads/rkOAdFd_-l.png)

&rarr; Không những không bị lỗi 500, mà còn trả về thông tin mô tả sản phẩm &rarr; khẳng định được ở đây có lỗi SQL Injection.

### 2. Thu thập thông tin database
Theo lý thuyết, sau khi xác định được có lỗ hổng, cần dựa vào cách Server phản hồi mà xác định phương pháp tấn công. Ở đây, ta thấy kết quả của truy vấn trả về trong response, vì vậy hướng tấn công phù hợp là sử dụng truy vấn UNION để trích xuất và thu thập thêm thông tin về database của mục tiêu. 
Trước hết, ta sẽ sử dụng `ORDER BY` để xác định số lượt cột được trả về trong truy vấn gốc:
![image](https://hackmd.io/_uploads/SycTYFd_Wg.png)
![image](https://hackmd.io/_uploads/S17AYtddbg.png)

&rarr; Ta thấy khi tăng giá trị lên 3 thì Server bắt đầu trả về lỗi, vì vậy số lượng cột trong truy vấn gốc là 2. 
Tiếp theo, ta cần xác định kiểu dữ liệu của từng cột. Từ nội dung trả về trong response bao gồm tên và mô tả của bài viết, ta có thể suy luận được khả năng cao cả 2 cột trả về đều có kiểu chuỗi. Để xác minh, ta sử dụng `UNION SELECT 'a','a'--` xem có phản hồi thế nào:
![image](https://hackmd.io/_uploads/Hkp_qtuObx.png)

&rarr; ta thấy Server trả về response mã 500, vì vậy ta thử thử nghiệm các trường hợp khác có thể:
![image](https://hackmd.io/_uploads/SJFo5K_uWl.png)
![image](https://hackmd.io/_uploads/r1jn9KduWx.png)
![image](https://hackmd.io/_uploads/Sk4JiY__be.png)

&rarr; ta thấy cả 4 trường hợp có thể đều bị lỗi. Thử với `UNION SELECT NULL,NULL--` xem sao:
![image](https://hackmd.io/_uploads/Sy-GjYd_be.png)

&rarr; vẫn bị lỗi, vì vậy khả năng cao suy luận của ta về kiểu dữ liệu là không sai. Ta bác bỏ khả năng bị chặn bởi các thành phần trung gian như IDS hay WAF vì status code trong response là 500, không phải 403, vì vậy khả năng cao là lỗi trong cú pháp của câu truy vấn ta chèn vào. 
Vậy thì điều gì có thể khiến một câu truy vấn `UNION` bị lỗi ? số lượng cột đã chính xác, kiểu dữ liệu cũng đã chính xác, ký tự comment cũng đã chính xác &rarr; khả năng cao bị lỗi vì câu truy vấn không phù hợp với kiểu DBMS. Mà đa phần các DBMS phổ biến đều có thể sử dụng cú pháp này, chỉ có Oracle là cần phải có `FROM dual`, vì vậy ta thử thêm vào phần này xem sao:
![image](https://hackmd.io/_uploads/H17UnFd_bl.png)

&rarr; Thành công khiến response trả về status code 200. Tiếp theo, ta thay thế giá trị `NULL` thành `'a'` xem sao:
![image](https://hackmd.io/_uploads/SJ3thYddbx.png)

&rarr; Thấy giá trị trong truy vấn của ta đã được trả về trong response. 
Sau khi xác định được loại DBMS, ta dễ dàng tra cứu cú pháp của những câu truy vấn nhằm xác định thêm thông tin về database. Trước hết, ta hướng tới xác định phiên bản của database thông qua truy vấn `UNION SELECT BANNER,'tmt2' FROM v$version--`:
![image](https://hackmd.io/_uploads/r1zT6FuuZg.png)
![image](https://hackmd.io/_uploads/SkM0pFO_Zg.png)

&rarr; Xác định được phiên bản được sử dụng là 11.2.0.2.0.
### 3. Khai thác cấu hình và phân quyền
Tới đây, mặc dù đã hoàn thành mục tiêu của bài lab, tuy nhiên trong bối cảnh mô tả một cuộc tấn công thực tế, tôi vẫn sẽ tiếp tục khai thác thêm thông tin. Tiếp theo, cần xác định current user và current database thông qua truy vấn `UNION SELECT user,'tmt2' FROM dual--` và `UNION SELECT sys_context('USERENV','DB_NAME'),'tmt2' FROM dual--`:
![image](https://hackmd.io/_uploads/BJgq0YOu-e.png)
![image](https://hackmd.io/_uploads/B1BiAY_Obl.png)

&rarr; Xác định được current user là PETER và database hiện tại có tên là XE. 
Tiếp theo, cần xác định user PETER có những quyền gì trong database:
- `UNION SELECT privilege,'tmt2' FROM user_sys_privs--`:

![image](https://hackmd.io/_uploads/rJMoJ5_Obx.png)
![image](https://hackmd.io/_uploads/SJpjy5_u-x.png)

&rarr; Xác định được có các quyền `CREATE SESSION, CREATE TABLE, UNLIMITED TABLESPACE`.
- `UNION SELECT granted_role,'tmt2' FROM user_role_privs`:

![image](https://hackmd.io/_uploads/HJ0p1cddZl.png)
![image](https://hackmd.io/_uploads/B1tCy5Od-g.png)

&rarr; Xác định được có các role `CONNECT, XDBADMIN`.

Thực hiện phân tích giá trị của các quyền và các role này để xác định hướng tấn công tiếp theo:
- Các quyền: 
    - CREATE SESSION: Đây là quyền cơ bản nhất. Nó xác nhận bạn có thể duy trì kết nối với database. Nếu không có quyền này, mọi nỗ lực SQL Injection sau đó sẽ bị ngắt kết nối ngay lập tức.
    - CREATE TABLE: Đây là "chìa khóa" quan trọng. Bạn có thể tạo các bảng mới trong schema của mình.
        - Ứng dụng: Bạn có thể tạo các bảng trung gian để lưu trữ kết quả truy vấn phức tạp hoặc dùng để lừa người dùng khác tương tác (nếu có các trigger hệ thống).
    - UNLIMITED TABLESPACE: Bạn không bị giới hạn về dung lượng lưu trữ.
        - Ứng dụng: Bạn có thể thực hiện một cuộc tấn công Storage DOS bằng cách tạo các bảng cực lớn hoặc chèn dữ liệu rác liên tục cho đến khi làm tràn ổ cứng của server database, gây treo hệ thống.
- Các role:
    - CONNECT: Role này thực chất bao gồm CREATE SESSION và một số quyền kết nối cơ bản khác. Nó xác nhận bạn là một user hợp lệ của hệ thống.
    - XDBADMIN: Đây là điểm yếu nhất trong cấu hình này. Role này dành cho Oracle XML DB.
        - Nguy cơ: XDBADMIN có quyền quản trị các tài nguyên XML, cấu hình các repository của XML DB. Trong một số phiên bản Oracle cũ hoặc cấu hình sai, role này cho phép bạn truy cập vào các gói như DBMS_XDB hoặc DBMS_XDBZ, từ đó có thể đọc/ghi các file trên hệ điều hành hoặc thay đổi cấu hình mạng của database.

&rArr; Có thể thấy ta có hướng tấn công rõ ràng &rarr; hướng tới việc thu thập thêm các thông tin khác. Tiếp theo, ta thực hiện việc kê các bảng có trong database hiện tại xem có gì đặc biệt không:
![image](https://hackmd.io/_uploads/Bknn-9_ubl.png)

&rarr; Chỉ có bảng PRODUCTS.
Giờ ta xác định các cột có trong table này:
![image](https://hackmd.io/_uploads/ryw0Xqud-l.png)

&rarr; Xác định được những cột: ID, CATEGORY, NAME, RATING, PRICE, IMAGE, RELEASED, DESCRIPTION. Ta thấy ở đây có cột RELEASED, thử xem cột này có định dạng gì:
![image](https://hackmd.io/_uploads/S1oVBcuuWg.png)

&rarr; Có vẻ nó không phải định dạng chuỗi nên đã gây ra lỗi Server. Thử ép kiểu kết quả trong truy vấn:
![image](https://hackmd.io/_uploads/Hk4uB9OObe.png)

&rarr; Thành công, ta có thể sử dụng cách này để tìm ra sản phẩm có giá trị ở cột này khác 1 qua intruder:
![image](https://hackmd.io/_uploads/rJofLcduWx.png)
&rarr; Không có bất kỳ sản phẩm nào. Ta thấy trong các cột, còn có cột IMAGE có thể được sử dụng để tiết lộ thêm thông tin cho ta về cấu trúc thư mục của hệ thống hoặc may mắn hơn có thể là IP của 1 máy nội bộ, thử liệt kê nội dung cột này xem sao:

![image](https://hackmd.io/_uploads/ryWxvc_dbe.png)

&rarr; Ta thấy đưỡng dẫn thư mục tới các ảnh, thử truy cập từ trình duyệt xem sao (vì các chức năng của trang web không có chức năng nào hiển thị ảnh sản phẩm):
![image](https://hackmd.io/_uploads/SyTNwq_Obg.png)
&rarr; Phản hồi trả về lại là "Not Found", vì vậy khả năng cao đây không phải là đường dẫn đầy đủ hoặc có thể chúng được lưu trữ tại 1 máy chủ nội bộ nào đó. 
Trong trường hợp này, nếu như kích hoạt được một chức năng nào có có thể truy vấn tới máy chủ hiện tại hoặc một máy chủ nội bộ khác thì có khả năng sẽ mở thêm được hướng khai thác cho ta, nhưng trong bối cảnh bài lab hiện tại thì không có.

Ta thử liệt kê xem với bảng hiện tại ta có những quyền gì qua truy vấn `UNION SELECT LISTAGG(privelege,',') WITHIN GROUP (ORDER BY privilege),'tmt2' FROM all_tab_privs WHERE table_name = 'PRODUCTS'--`:
![image](https://hackmd.io/_uploads/BJD8t5OdZx.png)

&rarr; Ta thấy kết quả trả về cho quyền là trống. Để giải thích tại sao, có thể phỏng đoán do một số lý do:
- Bạn chính là chủ sở hữu bảng (Owner): Trong Oracle, nếu bạn là người tạo ra bảng PRODUCTS (nó nằm trong schema của bạn), quyền của bạn sẽ không hiển thị trong all_tab_privs. View này thường chỉ lưu các quyền được cấp (GRANT) từ người này sang người khác.
    - Cách kiểm tra: Thử truy vấn từ view dành cho chủ sở hữu:
`+UNION+SELECT+LISTAGG(privilege,+',+')+WITHIN+GROUP+(ORDER+BY+privilege),+'tmt2'+FROM+user_tab_privs+WHERE+table_name='PRODUCTS'--`
- Quyền được cấp thông qua một ROLE: Trong Oracle, nếu quyền SELECT được gán cho một Role, và Role đó được gán cho bạn, thì nó sẽ không xuất hiện trong all_tab_privs (vốn chủ yếu liệt kê quyền cấp trực tiếp cho User).
    - Cách khắc phục: Truy vấn view dành cho Role:
`+UNION+SELECT+LISTAGG(privilege,+',+')+WITHIN+GROUP+(ORDER+BY+privilege),+'tmt2'+FROM+role_tab_privs+WHERE+table_name='PRODUCTS'--`

&rarr; Thử nghiệm lại: 
![image](https://hackmd.io/_uploads/B1-9q9OOWg.png)
![image](https://hackmd.io/_uploads/B1dj99u_-e.png)

&rarr; Vẫn là kết quả trống. Ta thử kiểm tra xem bảng hiện tại có phải có một cái tên nào khác không thông qua truy vấn `+UNION+SELECT+table_owner+||+'.'+||+table_name,+NULL+FROM+all_synonyms+WHERE+synonym_name='PRODUCTS'--`:
![image](https://hackmd.io/_uploads/Sk94sq_dWg.png)

&rarr; Không có kết quả nào trả về. 
Tuy nhiên, trong bối cảnh của một truy vấn `UNION`, những hành động ta có thể thực hiện cũng bị hạn chế. Ta thử kiểm tra xem có thể stack queries không:
![image](https://hackmd.io/_uploads/rk9235uOWl.png)
&rarr; Không thể stack queries. 

Tới đây, vì dữ liệu được phản ánh trong response, mà lại còn ở định dạng chuỗi, vì vậy tôi nghĩ tới việc sử dụng lỗ hổng này làm điểm vào cho một cuộc tấn công XSS Reflected. Vì vậy tôi thử nghiệm:
![image](https://hackmd.io/_uploads/SkOhp9_dWx.png)

&rarr; Dựa vào bối cảnh của dữ liệu là nội dung của một thẻ. Hướng tấn công phổ biến nhất là chèn thêm một thẻ mới có khả năng kính hoạt mã. Tuy nhiên ở đây có thể thấy các ký tự đặc biệt đã được encode HTML, vì vậy khả năng tấn công là không cao. 
Vì bài viết cũng đã khá dài và ta cũng không thể tìm được hướng tấn công nào khác, vì vậy tôi xin dừng bài viết lại ở đây. 



