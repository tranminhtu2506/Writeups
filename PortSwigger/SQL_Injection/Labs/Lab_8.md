# Lab: SQL injection UNION attack, finding a column containing text
## Quá trình phân tích và khai thác 
### 1. Xác định điểm vào 
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:
![image](https://hackmd.io/_uploads/rkLndg3u-x.png)
&rarr; Ta thấy từ trang chủ có 3 chức năng chính là: xem chi tiết thông tin sản phẩm, lọc sản phẩm theo danh mục và đường dẫn tới trang login.
Theo lý thuyết khi xác định được chức năng hiện có của 1 trang web, ta cần phân loại và sắp xếp dựa trên khả năng có lỗi và tác động có thể có. Vì vậy với lỗ hổng SQL Injection, ta ưu tiên kiểm thử chức năng lọc sản phẩm theo danh mục trước, ở đây thường sử dụng mệnh đề `WHERE`. Đầu tiên, ta kích hoạt chức năng để quan sát request mà nó tạo ra:
![image](https://hackmd.io/_uploads/SklXLtghObl.png)
&rarr; Khi click vào `Gifts` thì trang web trả về 1 response chứa các sản phẩm thuộc về danh mục này. Request tạo ra gửi tới endpoint /filter với tham số category để chỉ định giá trị danh mục trong URL. 
Tiếp theo, từ request, ta cần xác định các điểm vào có thể sử dụng để kiểm thử lỗ hổng và sắp xếp chúng theo khả năng có lỗi. Vì vậy ta chọn ưu tiên tham số `category` vì đây thường là nơi phổ biến nhất. Thêm vào cuối giá trị tham số ký tự `'` và quan sát phản hồi:
![image](https://hackmd.io/_uploads/SygG9l2uWg.png)
&rarr; Ta thấy response trả về có status code 500, cho thấy ký tự ta chèn vào đã gây ra lỗi trong quá trình xử lý của hệ thống. Trong bối cảnh dữ liệu được sử dụng trong một câu truy vấn thì khả năng có lỗi SQLi ở đây là rất cao. 
Ta thử thay thế lần lượt bằng các ký tự đặc biệt khác xem phản hồi có khác biệt không:
![image](https://hackmd.io/_uploads/BJp_5gnd-e.png)
![image](https://hackmd.io/_uploads/H1YYce2Obx.png)
![image](https://hackmd.io/_uploads/Bk759eh_be.png)
&rarr; các ký tự không bị lọc đi, cũng không gây ra lỗi mà đều được xem như một chuỗi &rarr; chỉ ký tự `'` gây ra lỗi hệ thống.
Tiếp theo, ta thử thay thế `'` thành `'--` xem Server phản hồi thế nào:
![image](https://hackmd.io/_uploads/BygyjlnOZg.png)
&rarr; Không bị coi như một giá trị không tồn tại mà lại hiển thị sản phẩm bình thường &rarr; tới đây có thể khẳng định có lỗ hổng SQLi tại tham số `category`. 
### 2. Thu thập thông tin Database
Sau khi xác dịnh có lỗi, ta cần thu thập thêm thông tin về database của mục tiêu để gia tăng tác động hoặc tìm kiếm thêm vector tấn công. 
Trước hết ta cần xác định kiểu DBMS của mục tiêu, thử thay thế ký tự comment từ `--` thành `#` và quan sát phản hồi nhận được:
![image](https://hackmd.io/_uploads/Hy9dslnuZe.png)
&rarr; Hệ thống trả về lỗi 500 &rarr; ta loại MySQL. 
Tiếp theo, vì những loại DBMS khác nhau lại có ký tự nối chuỗi khác nhau, ta thử sử dụng chúng để nhận dạng:
![image](https://hackmd.io/_uploads/Skm2og3O-e.png)
![image](https://hackmd.io/_uploads/Sk1pje3Obg.png)
&rarr; Trong các loại DBMS phổ biến, loại sử dụng ký tự `||` để nối chuỗi bao gồm `Oracle` và `PostgreSQL`.
Vì 2 loại này lại có hàm trích xuất chuỗi khác nhau, ta thử sử dụng điều này để nhận dạng:
![image](https://hackmd.io/_uploads/SkTh3l3uWg.png)
![image](https://hackmd.io/_uploads/ry2png2dZx.png)
&rarr; Ta thấy cả 2 hàm đều gọi được. Qua tìm hiểu nếu là hành vi mặc định thì sẽ có `MySQL` và `PostgreSQL` làm được điều này, và `MySQL` thì đã bị loại nên khả năng cao là `PostgreSQL`. Tuy nhiên cũng chưa khẳng định vì trên thực tế nhà phát triển hoàn toàn có thể tự định nghĩa một hàm trong bất kỳ loại CSDL nào, vì vậy việc trích xuất phiên bản vẫn đáng tin hơn, vì metadata của mỗi loại DBMS là khác nhau và có cách truy vấn khác nhau. 
Dựa vào cách Server trả về kết quả truy vấn trong phản hồi, ta sử dụng kỹ thuật `UNION BASED` để trích xuất và thu thập thêm thông tin. Trước hết xác định số lượng cột trong truy vấn gốc:
![image](https://hackmd.io/_uploads/HJ326x3dWe.png)
&rarr; Khi tăng giá trị lên 4 thì hệ thống trả về lỗi &rarr; số lượng cột trong truy vấn gốc là 3. 
Tiếp theo, ta thử sử dụng `UNION SELECT` với giá trị `NULL` xem hệ thống phản hồi thế nào, nếu là `Oracle` thì hệ thống sẽ trả về lỗi nếu không có `FROM dual`, ngược lại thì sẽ là `PostgreSQL`:
![image](https://hackmd.io/_uploads/HJuXRg2u-l.png)
&rarr; Xác định được loại DBMS phía sau là `PostgreSQL`. Để xác minh, ta thực hiện trích xuất phiên bản. Nhưng trước đó, ta cần xác định trong truy vấn gốc cột nào là có định dạng là chuỗi:
![image](https://hackmd.io/_uploads/HkntRxhubl.png)
&rarr; Ta xác định được cột 2 là cột có định dạng chuỗi. 
Tiếp theo, ta thực hiện trích xuất phiên bản:
![image](https://hackmd.io/_uploads/Hkx2Cx3_Wx.png)
&rarr; Xác định được phiên bản là `PostgreSQL 12.22`.
### 3. Hoàn thành mục tiêu bài lab
Vì mục tiêu bài lab này không chỉ là đăng nhập admin hay xác định số lượng cột mà là hiển thị 1 chuỗi cho trước qua `UNION SELECT`, vì vậy ta cần phải thực hiện theo nếu muốn hoàn thành lab.
Chuỗi cho trước trong bài này là `qwXYFS`, vì vậy ta thực hiện truy vấn `UNION SELECT NULL,qwXYFS,NULL--` để hoàn thành bài lab:
![image](https://hackmd.io/_uploads/SJ1L1Zn_Ze.png)
&rarr; Hoàn thành bài lab.
### 4. Khám phá thêm về cấu hình 
Ta tiếp tục khám phá thêm xem current user và current database là gì:
![image](https://hackmd.io/_uploads/H1m51Zh_We.png)
&rarr; Vẫn là user `peter` và database `academy_labs`. 
Thử trích xuất các bảng có trong database hiện tại:
![image](https://hackmd.io/_uploads/SJM7gZ2_Wl.png)
&rarr; Chỉ có bảng `products`. Nó sẽ không có nhiều giá trị nếu ta không thể thao túng dữ liệu của bảng, vì vậy thử xem ta có thể thực hiện Stack Queries không:
![image](https://hackmd.io/_uploads/HkArlW3u-g.png)
&rarr; Việc sử dụng ký tự `;` khiến hệ thống trả về lỗi &rarr; khả năng sử dụng được Stack Queries là không cao. &rarr; Bảng `products` không có nhiều giá trị khai thác. 
Tiếp theo ta thử liệt kê những quyền mà user hiện có trên toàn database:
![image](https://hackmd.io/_uploads/Hy1Q--3dZl.png)
&rarr; Ta thấy chủ yếu chỉ có quyền trên bảng `products` nhưng lại không thể thao tác với bảng qua `UNION SELECT`, vì vậy nên không tìm được hướng khai thác khả thi với bối cảnh và quyền hạn này. 

