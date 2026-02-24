# Lab: SQL injection UNION attack, determining the number of columns returned by the query
## Quá trình phân tích và khai thác 
### 1. Xác định điểm vào 
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/Hyo_pCq_-l.png)
&rarr; Từ trang chủ, có 3 chức năng chính là xem chi tiết thông tin sản phẩm, lọc sản phẩm theo danh mục và đường dẫn tới trang login. 
Theo lý thuyết, ta cần sắp xếp các chức năng dựa trên khả năng có lỗi của nó, vì vậy ta ưu tiên kiểm thử chức năng lọc sản phẩm theo danh mục trước. Trước hết, thực hiện chức năng để xem request nó tạo ra:
![image](https://hackmd.io/_uploads/By4pCCcO-l.png)
&rarr; Ruquest được gửi tới endpoint /filter với tham số category trong URL. Ta thử thêm ký tự `'` vào trong giá trị tham số xem phản hồi thế nào:
![image](https://hackmd.io/_uploads/rJ8lJyjube.png)
&rarr; Nhận được response với status code 500 &rarr; input ta chèn vào đã gây ra lỗi xử lý nào đó trên Server, tuy nhiên chưa thể khẳng định có lỗ hổng SQLi ở đây. 
Tiếp theo, thay thế lần lượt các ký tự đặc biệt để so sánh phản hồi được tạo ra:
![image](https://hackmd.io/_uploads/BJP41kj_Zg.png)
![image](https://hackmd.io/_uploads/rJmSk1s_-x.png)
![image](https://hackmd.io/_uploads/SkJUkyiuWe.png)
&rarr; Tất cả đều trả về phản hồi 200 với trang bình thường không có sản phẩm nào. Cho thấy chỉ có ký tự `'` mới gây ra lỗi hệ thống, và trong bối cảnh giá trị tham số rất có thể được sử dụng trong một truy vấn, khả năng rất cao ở đây có lỗ hổng SQLi.
Ta thử thay đổi giá trị thành chuỗi ký tự `'--` để so sánh phản hồi:
![image](https://hackmd.io/_uploads/SJ7ay1jdbx.png)
&rarr; lúc này lại nhận được phản hồi 200 như bình thường &rarr; có thể khẳng định ở đây có lỗ hổng SQLi. 
### 2. Thu thập thông tin Database:
Sau khi xác định có lỗ hổng và điểm vào cho cuộc tấn công, ta cần xác định chính xác kiểu DBMS phía sau. Trước hết, thử comment bằng ký tự `#`: 
![image](https://hackmd.io/_uploads/BJtXlys_bg.png)
&rarr; Server trả về lỗi 500 &rarr; Loại MySQL khỏi danh sách DBMS có thể. 
Tiếp theo, vì các DBMS có ký tự nối chuỗi khác nhau, ta thử một số trường hợp: 
- `||`:
![image](https://hackmd.io/_uploads/H18Oxko_-l.png)
- `+`:
![image](https://hackmd.io/_uploads/S18KgJsd-x.png)

&rarr; loại MSSQL khỏi danh sách, khoanh vùng được có thể là PostgreSQL hoặc Oracle. 
Tiếp theo, vì hàm trích xuất chuỗi của DBMS này là khác nhau, ta tiến hành thử nghiệm:
- `||(SUBSTRING('',1,1))--`:
![image](https://hackmd.io/_uploads/HkwkZJiOWe.png)
- `||(SUBSTR('',1,1))--`:
![image](https://hackmd.io/_uploads/B1brZ1o_Zl.png)

&rarr; cả 2 hàm này đều gọi được. 
Tới đây, có lẽ trích xuất phiên bản DBMS mới có thể khẳng định được. Vì vậy dựa vào cách phản hồi của bài lab, ta sử dụng kỹ thuật `UNION BASED` để trích xuất thông tin.
Trước hết, ta xác định số lượng cột được trả về trong truy vấn gốc qua `ORDER BY`:
![image](https://hackmd.io/_uploads/Sk4gfksdWg.png)
![image](https://hackmd.io/_uploads/Bkx-G1jdWx.png)
![image](https://hackmd.io/_uploads/ryjZzyo_Wx.png)
![image](https://hackmd.io/_uploads/H19zzJi_Wl.png)
&rarr; khi tăng giá trị tới 4 thì bị lỗi, vì vậy xác định được số lượng cột trả về trong truy vấn gốc là 3. 
Dựa trên thông tin được trả về, ta đoán ở đây có 2 cột là chuỗi và 1 cột là số. Vì vậy ta thử sử dụng `UNION SELECT 'a',1,'b' --` để kiểm chứng:
![image](https://hackmd.io/_uploads/rJyOzkiO-x.png)
&rarr; Bị lỗi 500, có thể là do sai kiểu dữ liệu, cũng có thể là do thiếu dual, thử thay lại bằng `NULL` xem sao:
![image](https://hackmd.io/_uploads/HJziM1jdWg.png)
&rarr; Phản hồi với mã 200 &rarr; loại DBMS là PostgreSQL do ta không cần `FROM dual` vẫn truy vấn thành công. Tới đây ta đã hoàn thành bài lab.
Tiếp theo, thử các trường hợp cho tới khi xác định được kiểu dữ liệu của các cột:
![image](https://hackmd.io/_uploads/S1pK7ko_-l.png)
&rarr; thành công xác định được cột 1 (giá trị productId được reflected vào trong thuộc tính href của thẻ a) và cột 3 (giá trị của sản phẩm được reflected vào trong giá trị của cặp thẻ th) có định dạng số, còn cột 2 (tên sản phẩm được reflected vào trong giá trị của cặp thẻ th) có định dạng chuỗi. 
Tiếp theo, ta sẽ trích xuất phiên bản của DBMS qua truy vấn `UNION SELECT 1,version(),2--`:
![image](https://hackmd.io/_uploads/H1fdVJjd-l.png)
&rarr; Lấy được phiên bản `PostgreSQL 12.22`.
### 3. Khám phá cấu trúc Database
Tiếp theo, ta sẽ xác định current user và current database qua truy vấn `UNION SELECT 1,current_user||'-'||current_database(),2--`:
![image](https://hackmd.io/_uploads/BkDHS1i_be.png)
&rarr; Vẫn là `peter` và `academy_labs`.
Tiếp theo, ta trích xuất các bảng có trong database hiện tại qua truy vấn `UNION SELECT 1,table_name,2 FROM information_schema.tables WHERE table_schema='public'--`:
![image](https://hackmd.io/_uploads/S19kLyi_bl.png)
&rarr; Chỉ có bảng `products` như một số bài trước. 
Ta thử xem có thể thực hiện Stack Queries ở bài này không:
![image](https://hackmd.io/_uploads/BJ2ZLkidWg.png)
&rarr; vẫn không thể thực hiện Stack Queries.
&rArr; Khả năng cao ta sẽ không thể thực hiện bất kỳ thay đổi gì trên bảng `products` này do bối cảnh hạn chế của điểm chèn. 
Ta thử xem với toàn bộ database thì ta có quyền hay role gì đặc biệt không: 
![image](https://hackmd.io/_uploads/Syo-PkiOZe.png)
![image](https://hackmd.io/_uploads/HJFvDyouZx.png)
&rarr; không có bất kỳ quyền hay role nào. 
Thử xem có thể tấn công XSS reflected không:
![image](https://hackmd.io/_uploads/BkkhDys_Zg.png)
&rarr; bị encode HTML. 2 cột còn lại chỉ nhận giá trị số nên ta không thể tấn công.
