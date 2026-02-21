# Lab: SQL injection vulnerability allowing login bypass
## Quá trình phân tích và khai thác

### 1. Xác định điểm vào
Giả định đã hoàn tất giai đoạn Recon và sàng lọc, chúng ta sẽ bắt đầu khoanh vùng các vector tấn công trên đối tượng này, từ đó đánh giá liệu đây có phải là một điểm khai thác tiềm năng hay không.
Đầu tiên, khi truy cập trang web, ta được trả về giao diện:

![image](https://hackmd.io/_uploads/By4NM08ubl.png)
![image](https://hackmd.io/_uploads/rk4HGC8ubl.png)

&rarr; Ta thấy từ trang chủ, có 2 chức năng chính là xem chi tiết thông tin sản phẩm và đường dẫn tới trang Login. 

Theo lý thuyết, những chức năng liên quan đến định danh (ở đây là xem chi tiết thông tin của 1 sản phẩm) và những chức năng liên quan tới truy vấn dữ liệu (ở đây là chức năng đăng nhập) đều là những chức năng khả năng cao có lỗi; vì vậy ta tiến hành kiểm thử lần lượt. 

#### Với chức năng xem chi tiết sản phẩm: 
Đầu tiên, khi thực hiện chức năng, nó tạo ra request như sau:

![image](https://hackmd.io/_uploads/r1swX0UdZg.png)

&rarr; Không có chức năng mới nào xuất hiện. Request được gửi tới endpoint /product với tham số productId. 

Tới đây, ta cần xác định các điểm vào có thể gây ra lỗi, phổ biến nhất là các tham số trong URL hoặc body của request. Vì vậy ta ưu tiên kiểm thử trước bằng cách thêm ký tự `'` vào sau giá trị gốc:

![image](https://hackmd.io/_uploads/r1n-NALOZx.png)

&rarr; Ta thấy nhận được response có status code 400 với nội dung "Invalid Product ID", điều này có thể lý giải bằng một số lý do:
- Cơ chế Validation (kiểm chứng dữ liệu): Đa số các framework hiện đại (như Spring Boot, ASP.NET, NestJS) đều có lớp trung gian để kiểm tra định dạng dữ liệu trước khi gửi nó xuống Database.
        - Dự đoán: Tham số productId trong code được định nghĩa là kiểu số.
        - Hành động: Khi ta thêm dấu `'`, giá trị trở thành một chuỗi không hợp lệ. Lớp Validation phát hiện ra đây không phải một ID hợp lệ và chặn đứng yêu cầu ngay lập tức. 
- Sử dụng Prepared Statements: Nếu ứng dụng được viết tốt, nó sẽ sử dụng Parameterized Queries
    - Khi thêm dấu `'`, thay vì dấu này được thực thi như một phần của câu lệnh SQL, thư viện kết nối Database sẽ coi cả cụm 123' chỉ là một giá trị chuỗi thuần túy. Database tìm kiếm ID đó, không thấy, và logic phía Backend trả về thông báo lỗi "thân thiện" là "Invalid product ID".
- Error Handling (xử lý lỗi tùy chỉnh): Các lập trình viên thường viết các khối try-catch hoặc Global Exception Handler để "bọc" các lỗi tiềm ẩn.
    - Thay vì để mặc định Server "văng" ra lỗi 500 kèm theo một đống log chi tiết (vốn rất nguy hiểm vì lộ thông tin hệ thống), họ bắt lỗi và trả về một mã lỗi 400 cùng thông báo ngắn gọn.
    - X-Frame-Options: SAMEORIGIN: Việc có header này cho thấy server đã được cấu hình bảo mật khá kỹ để chống các cuộc tấn công như Clickjacking.

=> Có thể xác định đây không phải là một dấu hiệu rõ ràng cho một cuộc tấn công SQLi. Vì tham số ở đây là ProductID, vì vậy giá trị thường được sử dụng cho phần này thường là số &rarr; việc chèn ký tự `'` có thể gây ra lỗi không mong muốn trong quá trình xử lý hoặc ép kiểu. Vì vậy ta thử thay thế nó bằng ký tự comment phổ biến xem sao:

![image](https://hackmd.io/_uploads/SkIrU0UOZl.png)
![image](https://hackmd.io/_uploads/HJH8LAUd-x.png)

Ta thấy không có gì thay đổi, thử thêm vào các điều kiện logic để khiến câu truy vấn luôn đúng hoặc luôn sai xem có phản hồi khác biệt nào không:

![image](https://hackmd.io/_uploads/SyboL0Udbg.png)
![image](https://hackmd.io/_uploads/B1I2URIuWe.png)

&rarr; Ta thấy vẫn không có bất kỳ sự khác biệt nào trong phản hồi. 

Để khảo sát, ta thử thay thế giá trị tham số thành một giá trị khả năng cao không tồn tại để so sánh phản hồi:

![image](https://hackmd.io/_uploads/SypZwCL_bg.png)

&rarr; Ta thấy nếu như là một giá trị không tồn tại, response trả về phải có mã 404 với nội dung "Not Found". 

=> Tới đây có thể khẳng định response với status code 400 ta nhận được là kết quả của một quá trình xử lý dữ liệu phía Server và vì nội dung nhận được cũng không phản ánh được cách dữ liệu được xử lý hay thay đổi, vì vậy đây có thể là điểm vào không quá tiềm năng để tấn công. 

Thông thường tới giai đoạn này, ta thường có một vài hướng xử lý:
- 1. Lần lượt thêm ký tự đặc biệt vào từng điểm vào khác trong request như header, cookie,... để gây ra lỗi nhằm so sánh các response để tìm điểm tấn công tiềm năng.
- 2. Sử dụng các kiểu biểu diễn dữ liệu khác nhau để tìm ra cách có thể bypass cơ chế phòng thủ phía Server.
- 3. Thực hiện kiểm thử chức năng khác để tìm điểm tấn công tiềm năng hơn. 

Ở đây, tôi chọn hướng xử lý thứ 3, vì không có gì đảm bảo chức năng đầu tiên có lỗ hổng. Việc tiếp tục thử nghiệm có thể gây tốn nhiều thời gian hoặc bị chặn nếu ở trong một cuộc tấn công thực tế (vì ta phải gửi nhiều request để đoán cơ chế xử lý và cách bypass); trong khi đó ta có thể ta sẽ tìm được một vector tấn công dễ dàng hoặc hiệu quả hơn ở chức năng khác. 

#### Với chức năng login
Trước hết, khi ta click vào đường dẫn "My account", ta nhận về giao diện:

![image](https://hackmd.io/_uploads/H1Pg5RIube.png)

&rarr; Ta thấy chỉ có duy nhất chức năng "Log in". 

Trong một cuộc kiểm thử, việc có thể sở hữu hay sử dụng một tài khoản mang lại nhiều giá trị. Trong nhiều trường hợp, các chức năng có thể nới lỏng cơ chế bảo vệ vì tin tưởng vào trạng thái user sau login, hoặc có thể xuất hiện thêm nhiều chức năng hơn &rarr; từ đó mở rộng được bề mặt kiểm thử và tấn công. 
Vì vậy khi phát hiện một trang login mà không được cung cấp thông tin đăng nhập, thứ ta muốn xác định đầu tiên là chức năng register có được bật hay không. Trong trường hợp này ở phía giao diện không có chức năng này, tuy nhiên từ đường dẫn tới /login, ta có thể suy luận được nếu chức năng register có tồn tại, thì đường dẫn tới chức năng thường nằm ở /register:

![image](https://hackmd.io/_uploads/S17_oRUuWl.png)

&rarr; Response trả về có status code 404. Ở đây nếu trong một cuộc tấn công hoặc kiểm thử thực tế, ta cần dựa vào một số thông tin thu thập được trong quá trình recon để đưa ra nhận định:
- 1. Dựa vào giá trị của các endpoint đã tìm được để suy luận những giá trị có thể có của chức năng register này. 
- 2. Tra cứu trong những đường dẫn tìm được, có endpoint nào có thể được sử dụng hoặc bổ trợ cho quá trình khám phá nội dung của trang web hay không. 

=> Kết quả của quá trình này có thể được chia thành hai trường hợp:
- Nếu tìm được đường dẫn bị ẩn của chức năng register, ta sẽ tiến hành kiểm thử những lỗ hổng có thể xảy ra trong quá trình đăng ký trước, nếu không có thì tạo một tài khoản thường và bắt đầu kiểm thử tới chức năng login. 
- Nếu trong trường hợp không tìm thấy đường dẫn tới register hoặc xác định được trang web không có chức năng này &rarr; khẳng định được chức năng đăng ký bị hạn chế trên trang web này &rarr; những tài khoản được tạo ra và đang được sử dụng là cần thiết hoặc có đặc quyền cao với hệ thống &rarr; tập trung vào các vector hoặc lỗ hổng có thể giúp ta chiếm được hoặc đăng nhập một cách trái phép vào tài khoản đang có trên hệ thống. 

Trong bối cảnh của bài lab, ta không có bất kỳ thông tin nào về việc recon, và việc thử nghiệm truy cập tới /register cũng không thành công. Ta giả định rằng trên trang web này không có chức năng đăng ký &rarr; do đó ta tiến hành tập trung kiểm thử với chức năng login. 
Thông thường ở bước này, ta sẽ sử dụng những công cụ hỗ trợ cho quá trình brute-force. Tuy nhiên, việc brute-force một cách thiếu suy nghĩ sẽ có thể dấn tới bị chặn địa chỉ IP hoặc gây ảnh hưởng tới hoạt động của trang web. Vì phỏng đoán được tài khoản hiện có trên hệ thống thường phải có đặc quyền cao, vì vậy ta nghĩ tới username phổ biến có thể là: admin, administrator, admin123, ...
Vì vậy bước tiếp theo là thử đăng nhập với username này và quan sát nội dung phản hồi: 

![image](https://hackmd.io/_uploads/Hk9-ekDOWe.png)
![image](https://hackmd.io/_uploads/rknMe1PdWx.png)
![image](https://hackmd.io/_uploads/SJGVg1vd-l.png)

&rarr; Ta thấy không có sự khác biệt gì trong phản hồi giữa ba lần thử. &rarr; khả năng khai thác sự khác biệt thông qua thông báo đăng nhập thất bại là không quá cao. Hoặc phải cần thử nghiệm nhiều hơn. 
Tuy nhiên, kiểm thử chức năng login không chỉ có mỗi đoán mò hoặc brute-force, mà còn có những lỗ hổng giúp ta đăng nhập mà không cần mật khẩu như: Access Control, SQLi hay NoSQL,... và dựa vào cách giá trị input được gửi đi trong request, thiên về hướng SQLi hơn (vì dữ liệu không ở định dạng JSON, và cũng không biết endpoint tới trang profile của account, và session cũng không ở định dạng đặc biệt). 
Trước hết, ta thử đăng nhập như sau:

![image](https://hackmd.io/_uploads/Hkei-yvOZg.png)

&rarr; Ta thấy khi thêm ký tự `'` vào trong trường username, response trả về có status code 500 &rarr; Đây là một dấu hiệu tốt cho thấy dữ liệu của ta đã gây hỏng hoặc sai xót trong quá trình xử lý phía Server, cho thấy dữ liệu đã thực sự tới được BE. Tuy nhiên, vẫn chưa thể khẳng định ở đây có lỗ hổng SQL Injection, ta thử thay thế ký tự này thành các ký tự đặc biệt khác:

![image](https://hackmd.io/_uploads/rkOEzJP_be.png)
![image](https://hackmd.io/_uploads/ByMrfJP_-x.png)
![image](https://hackmd.io/_uploads/B1JUf1DOZg.png)

&rarr; Ta thấy chỉ có ký tự `'` là gây ra phản hồi 500 &rarr; gần như chắc chắn có lỗ hổng SQLi. Tiếp theo, ta thử thêm ký tự comment vào giá trị tham số:

![image](https://hackmd.io/_uploads/SyU5z1PO-g.png)

&rarr; vẫn còn ký tự `'` nhưng không còn bị lỗi 500 &rarr; lỗi trước đó là do ký tự ta chèn vào gây ra lỗi cú pháp trong câu truy vấn. 
Tiếp theo, trong một hệ thống, tài khoản đầu tiên được tạo thường là tài khoản của admin. Vì không biết cụ thể username và password, ta có thể thêm điều kiện logic OR 1=1 và LIMIT 1 để khiến truy vấn luôn đúng và trả về kết quả đầu tiên, lúc này ta sẽ đăng nhập được thành công vào tài khoản của admin:

![image](https://hackmd.io/_uploads/HyeS71DdZg.png)

&rarr; Đăng nhập thành công với tư cách là admin:

![image](https://hackmd.io/_uploads/r1lC71DdZe.png)

&rArr; Hoàn thành bài lab. 

