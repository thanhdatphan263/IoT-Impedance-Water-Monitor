# IoT-Impedance-Water-Monitor
Thiết bị phân tích và giám sát chất lượng nước

## 1. Giới thiệu đề tài
Đề tài xây dựng một thiết bị đo lường và phân tích chất lượng nước sử dụng vi điều khiển ESP32, kết hợp chip đo tổng trở chuyên dụng AD5933, mạch khuếch đại thuật toán AD8606, giao thức truyền thông MQTT và hệ thống giám sát Node-RED Dashboard. Thiết bị này đóng vai trò như một trạm phân tích chất lượng nước sinh hoạt tự động dựa trên phương pháp đo phổ tổng trở.

## 2. Vấn đề thực tế và giải pháp
* **Thực trạng nguồn nước:** Chất lượng nước sinh hoạt tại các khu nhà trọ hoặc nước giếng khoan thường bị lẫn tạp chất, ion kim loại nặng và muối hòa tan làm thay đổi đặc tính điện hóa, gây ảnh hưởng nghiêm trọng đến sức khỏe nhưng rất khó nhận biết bằng mắt thường.
* **Hạn chế của cảm biến phổ thông:** Các cảm biến đo độ đục hay cảm biến TDS thông thường chỉ đo giá trị điện trở thuần hoặc độ truyền quang ở một tần số cố định, độ chính xác thấp, dễ bị trôi tuyến tính và hoàn toàn không phản ánh được bản chất phức tạp của dung dịch điện ly.
* **Giải pháp chuyên sâu:** Đề tài ứng dụng phương pháp tính phổ tổng trở điện hóa bằng phần cứng chuyên dụng. Hệ thống thực hiện quét dải tần số rộng qua mẫu nước, đo đạc đồng thời biên độ và góc pha của tổng trở phức ($\dot{Z} = R + jX$). Từ đó, hệ thống bù trừ sai số do nhiệt độ môi trường và truyền dữ liệu qua MQTT lên Node-RED để dựng đồ thị Bode Plot trực quan từ xa, giúp phân tích chính xác bản chất điện hóa của nguồn nước.

## 3. Danh mục phần cứng
* **Vi điều khiển ESP32:** 
  * Đóng vai trò là "bộ não" điều phối toàn hệ thống.
  * Cấu hình giao tiếp phần cứng I2C để điều khiển chip AD5933 thực hiện các chu trình quét tần số.
  * Đọc dữ liệu nhiệt độ từ cảm biến DS18B20 để thực hiện thuật toán bù nhiệt phần mềm.
  * Đóng gói dữ liệu thành bản tin JSON và kết nối Wi-Fi truyền tải qua giao thức MQTT.
* **IC đo tổng trở chuyên dụng AD5933:**
  * Là một hệ thống đo tổng trở tích hợp sẵn bộ phát sóng sin kỹ thuật số và bộ chuyển đổi tương tự - số 12-bit tốc độ 1 MSPS.
  * Có khả năng thực hiện phép biến đổi Fourier rời rạc trực tiếp trên chip ở từng tần số quét, trả về kết quả là các thành phần số phức thực và ảo.
* **Mạch khuếch đại thuật toán AD8606:**
  * Op-amp kép độ chính xác cao, dòng rò cực thấp, hoạt động ở dải điện áp thấp.
  * Đóng vai trò làm mạch đệm, mạch chuyển đổi dòng điện sang điện áp hoặc khuếch đại biên độ tín hiệu phản hồi từ dung dịch nước trước khi đưa vào chân ADC của AD5933, giúp loại bỏ hiện tượng sụt áp và sai số do trở kháng nguồn.
* **Cảm biến nhiệt độ nước DS18B20:**
  * Giao tiếp qua chuẩn 1-Wire, đo nhiệt độ nước thời gian thực. 
  * Cực kỳ quan trọng vì tổng trở của dung dịch điện ly phụ thuộc rất lớn vào nhiệt độ ($R$ giảm khi nhiệt độ tăng). Dữ liệu này dùng để hiệu chuẩn và bù sai số nhiệt độ.
* **Giao diện quan sát:** Màn hình OLED 0.96 inch giao tiếp I2C hiển thị thông số tổng trở thực tế và trạng thái hoạt động trực tiếp tại thiết bị.

## 4. Nguyên lý hoạt động chi tiết
* **Bước 1: Khởi tạo và cấu hình dải quét tần số**
  * Người dùng hoặc hệ thống kích hoạt lệnh đo. ESP32 cấu hình chip AD5933 thông qua giao diện I2C với các tham số: Tần số bắt đầu, bước nhảy tần số, và tổng số điểm quét  trong dải tần cho phép, thường từ vài kHz đến 100 kHz.
* **Bước 2: Phát sóng và kích thích điện hóa**
  * Bộ DDS bên trong AD5933 phát ra tín hiệu điện áp xoay chiều hình sin có biên độ cố định để đặt lên hai điện cực ngâm trong mẫu nước. 
  * Dòng điện chạy qua dung dịch nước phụ thuộc vào nồng độ ion và tổng trở phức của mẫu. Op-amp AD8606 đóng vai trò khuếch đại và ổn định dòng điện phản hồi này quay trở lại chân nhận tín hiệu của AD5933.
* **Bước 3: Xử lý tín hiệu DFT và thu thập dữ liệu**
  * Tại mỗi bước tần số trong dải quét, mạch tích hợp thực hiện lấy mẫu tín hiệu và tính toán thuật toán DFT để bóc tách ra hai thành phần: Phần thực và Phần ảo của tổng trở phức.
  * Cùng lúc đó, cảm biến DS18B20 đo chính xác nhiệt độ của mẫu nước tại thời điểm đó.
* **Bước 4: Bù nhiệt và đóng gói dữ liệu**
  * ESP32 đọc các giá trị thực, ảo và nhiệt độ. Vi điều khiển thực hiện thuật toán toán học để tính toán biên độ tổng trở ($|\dot{Z}| = \sqrt{\text{Real}^2 + \text{Imaginary}^2}$), góc pha ($\theta = \arctan(\text{Imaginary} / \text{Real})$), đồng thời áp dụng hệ số hồi quy tuyến tính để bù trừ sai số do nhiệt độ nước gây ra.
  * Dữ liệu sau khi xử lý được đóng gói gọn gàng thành chuỗi bản tin JSON.
* **Bước 5: Truyền thông MQTT và Giám sát trực quan trên Node-RED**
  * ESP32 đẩy bản tin dữ liệu qua Wi-Fi lên MQTT Broker thông qua các Topic được định nghĩa sẵn.
  * **Node-RED Dashboard** nhận chuỗi dữ liệu, sử dụng các hàm xử lý JavaScript để tách mảng dữ liệu, sau đó vẽ ồ thị biên độ tổng trở và góc pha theo tần số ngay trên giao diện web. Dựa vào hình dạng và độ dịch chuyển của đặc tuyến phổ tổng trở, hệ thống đưa ra cảnh báo nếu nguồn nước xuất hiện bất thường hoặc nhiễm ion lạ.
