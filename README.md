# Tổng quan về sản phẩm
## Mô tả sản phẩm
NekoTech Garage là giải pháp quản trị dịch vụ ô tô toàn diện "All-in-One", được thiết kế chuyên biệt để số hóa quy trình vận hành cho các Gara và chuỗi trung tâm chăm sóc xe chuyên nghiệp. Hệ thống chuẩn hóa quy trình dịch vụ 12 bước khép kín từ khâu tiếp nhận, báo giá, sửa chữa đến hậu mãi, kết nối liền mạch giữa bộ phận điều hành (Web Admin), kỹ thuật viên (Staff App) và chủ xe (Customer App). Điểm đột phá của iCar nằm ở công nghệ quản lý hồ sơ xe định danh bảo mật theo đời chủ, cùng khả năng tích hợp sâu rộng với Zalo OA, hóa đơn điện tử và thanh toán không tiền mặt, giúp doanh nghiệp không chỉ tối ưu năng suất, chống thất thoát mà còn nâng tầm trải nghiệm khách hàng lên chuẩn mực mới.
## Phân hệ dự án
- **Phân hệ Core (Lõi):** Quản lý hồ sơ xe, lịch sử sửa chữa, và quản lý khách hàng.
- **Phân hệ Operation (Vận hành):** Quy trình 12 bước từ tiếp nhận đến hậu mãi (kiểm tra file Workflow).
- **Phân hệ Inventory & Finance (Kho & Tài chính):** Quản lý phụ tùng, định mức tồn kho, công nợ, thu chi và tích hợp kế toán.
- **Phân hệ HR (Nhân sự):** Chấm công, tính lương khoán/sản phẩm cho thợ.   
- **Phân hệ Digital Touchpoints:** Web Portal, Landing Pages, Mobile App cho khách và thợ, Status Page công khai.
- **Hạ tầng & Bảo mật:** Thiết lập Proxy, Caddyfile, Backup, và tuân thủ quy định bảo mật dữ liệu.
## Những điều cần lưu ý
### Thuật toán lưu dữ liệu xe
Một xe có thể nhiều chủ, cho nên nếu xe bị bán lại cho chủ khác thì có nguy cơ gây lộ thông tin cá nhân của chủ cũ. Giải pháp là thiết kế một mô hình dữ liệu **Time-Variant Relationship (Quan hệ biến thiên theo thời gian)**.

Thiết kế bảng dữ liệu xe bao gồm các dữ liệu **bắt buộc** như sau:

> Bảng VehicleOwnership (Quyền sở hữu xe) gồm:
> - `VehicleID`: Tham chiếu đến xe.
> - `CustomerID`: Tham chiếu đến chủ sở hữu.
> - `StartDate`: Ngày bắt đầu sở hữu (Ngày mua xe hoặc ngày đầu tiên đến gara).
> - `EndDate`: Ngày kết thúc sở hữu (Ngày bán xe/chuyển nhượng). Nếu `NULL`, nghĩa là đang sở hữu.

Logic truy vấn dữ liệu được thiết kế như sau:
1. Hệ thống kiểm tra bảng `VehicleOwnership`.
2. Tìm các bản ghi khớp `CustomerID = UserA` và `VehicleID = CarX`.
3. Lấy ra khoảng thời gian `StartDate` và `EndDate`.
4. Hệ thống truy vấn bảng ServiceOrders (Lệnh sửa chữa) với điều kiện ngày tạo lệnh nằm trong khoảng thời gian trên.

Thiết kế logic này đảm bảo việc đúng xe, đúng chủ vào từng thời điểm chuyển nhượng xe.
### Các vòng lặp quan trọng
Trong workflow hoạt động theo bước sẽ có một số đoạn check, nếu check failed thì sẽ phải thực hiện lại bước trước đó, cụ thể như sau:
- **Bước kĩ thuật**: Bước 5 (Thực hiện DV) -> Bước 6 (Test) -> Nếu check fail -> Trở về bước 5
> Hệ thống phải ghi nhận đây là "Rework" (Làm lại). Việc này ảnh hưởng đến KPI của thợ. Nếu một thợ sửa chữa bị rework quá nhiều lần, hệ thống cần cắm cờ cảnh báo (Flagging) cho người quản lý kỹ thuật. Chúng ta không thể chỉ đơn giản là đổi trạng thái, mà phải tạo ra một bản ghi `ServiceActionLog` để lưu lại lần sửa lỗi này, vật tư tiêu hao cho việc sửa lỗi có thể không được tính tiền cho khách mà tính vào chi phí rủi ro của gara.
- **Bước vệ sinh**: Bước 8 (Kiểm tra trước bàn giao) -> Nếu vệ sinh chưa tốt -> Quay lại Bước 7 (Vệ sinh)
> Đây là bước thường bị coi nhẹ. Tuy nhiên, trong các mô hình dịch vụ cao cấp, "Sạch sẽ" là yếu tố cảm xúc quan trọng nhất. Hệ thống cần có Checklist vệ sinh cụ thể trên App của nhân viên vệ sinh (ví dụ: Hút bụi sàn, lau kính, khử mùi).
### Các bước trao đổi giữa khách và Garage
Bước 3 (Tư vấn và báo giá) và Bước 4 (Lên lịch sửa chữa) là bước mà 2 bên cần làm việc trực tiếp. Từ đó đưa ra 2 hướng xử lý như sau:

1. **Làm việc trực tiếp tại quầy:** Tạo một `waiting_list` để xử lý từng xe, từng khách hàng, khi khách tới lượt sẽ gửi notify về app của khách và thông báo tại quầy để mời khách tới quầy làm việc với bên tư vấn.
2. **Làm việc online:** Phải đảm bảo tốc độ truyền/gửi dữ liệu nhanh chóng để cập nhật dữ liệu giữa khách hàng & bộ phận tư vấn, đảm bảo khách hàng có sử dụng app của Garage. Phương pháp này có thể áp dụng Internet Banking (thanh toán trực tuyến) để đơn giản hóa quá trình.
### Độ ưu tiên
Sắp xếp hàng chờ theo độ ưu tiên cần làm cho khách dựa trên các tiêu chí sau.

1. Mood (Tâm trạng: Vui vẻ, Vội vã, Bực dọc (do xe hỏng), Lo lắng,...): Nếu khách "Vội vã", quy trình trên App của Cố vấn dịch vụ (CVDV) sẽ ưu tiên hiện các mục "Fast Track" (Làm nhanh), bỏ qua các bước tư vấn phụ kiện rườm rà. Nếu khách "Bực dọc", hệ thống cảnh báo CVDV cần cử người có kinh nghiệm nhất ra tiếp. Đồng thời ưu tiên hàng chờ cho khách.
2. Requirement (yêu cầu của khách hàng): Nếu khách hàng cần nhận xe sớm thì sẽ ưu tiên làm trước, query cao hơn các xe gửi ở Garage lâu dài hoặc khách hàng có thể chờ được.
### Hệ Thống Hỗ Trợ (Support System)
- **Knowledge Base:** Xây dựng trang Help Center chứa các bài viết: "Hướng dẫn đặt lịch", "Quy trình bảo hiểm", "Ý nghĩa các đèn báo lỗi".
- **Live Chat:** Tích hợp Chat widget trên Web và App. Có thể dùng Chatbot AI để trả lời các câu hỏi thường gặp (Giờ mở cửa, Bảng giá cơ bản).
- **Ticket System:** Đối với các khiếu nại phức tạp, khách hàng gửi Ticket. Hệ thống theo dõi SLA (Service Level Agreement) để đảm bảo khiếu nại được giải quyết trong 24h.
# Tech Stack & Framework
## Mô hình kiến trúc 
Sử dụng **mô hình Modular Monolith (Khối đơn nhất dạng mô-đun)** trong giai đoạn đầu, sẵn sàng chuyển đổi sang Microservices khi quy mô mở rộng.
> Refer to https://www.milanjovanovic.tech/blog/what-is-a-modular-monolith for more information.

**Phân hệ logic:**
1. **Identity Provider (IDP):** Quản lý User, Auth, Roles.   
2. **Core Service:** Quản lý xe, khách hàng, lịch sử.
3. **Workflow Engine:** Quản lý State Machine của quy trình 12 bước.   
4. **Inventory Service:** Quản lý kho, nhập xuất, cảnh báo.   
5. **Accounting Service:** Thu chi, công nợ, HĐĐT.
6. **Notification Service:** Zalo, SMS, Email, Push Notification.

## Tech Stack
### Backend
- Lựa chọn Golang + Gin Framework (khuyến khích)/Javascript (altenative)

> **Hiệu năng:** Go là ngôn ngữ biên dịch (compiled), xử lý concurrency (đồng thời) cực tốt. Với yêu cầu "không giới hạn người dùng" và xử lý dữ liệu thời gian thực từ nhiều chi nhánh, Go vượt trội so với Node.js hay.NET cũ.

> **Triển khai:** Go biên dịch ra một file binary duy nhất, rất dễ đóng gói vào Docker container nhẹ (chỉ vài chục MB), giúp tiết kiệm chi phí Server.

> **Tính ổn định:** Go ép buộc xử lý lỗi (error handling) rất chặt chẽ, giảm thiểu các lỗi Crash server bất ngờ (Runtime panic) thường thấy ở các ngôn ngữ khác.
### Database
- **Primary DB:** PostgreSQL 15+
> PostgreSQL là mã nguồn mở tiên tiến nhất hiện nay, hỗ trợ JSONB để lưu các thuộc tính xe linh hoạt (ví dụ: xe độ thêm loa, đèn - những trường không cố định). Nó hỗ trợ Partitioning tốt cho dữ liệu lớn theo thời gian.
- **Caching:** Redis
> Dùng để lưu session đăng nhập, cache danh sách phụ tùng, và quan trọng nhất là làm Message Broker đơn giản cho các tác vụ Real-time (ví dụ: Báo tiến độ sửa chữa lên màn hình TV ở phòng chờ).
### Frontend
- **Website (bao gồm Landing, ClientArea,...):** ReactJS (hoặc Next.js) + Ant Design Pro. Ant Design cung cấp sẵn các bộ component bảng biểu phức tạp (Data Tables), rất phù hợp cho các báo cáo kế toán và kho vận dày đặc dữ liệu trong. 
- **Mobile App:** Flutter/ngôn ngữ khác tùy lựa chọn - Google's Flutter cho phép build ra iOS và Android từ một codebase. Hiệu năng render 60fps mượt mà. Quan trọng là khả năng tùy biến giao diện cao để tạo ra các Dashboard đẹp cho Giám đốc.
### Network & Reverse Proxy
- Sử dụng Caddyfile để làm Proxy
> Automatic HTTPS: Caddy tự động xin chứng chỉ SSL từ Let's Encrypt và gia hạn nó. Chúng ta không cần lo về việc chứng chỉ hết hạn làm gián đoạn dịch vụ.
> Reverse Proxy: Giấu kín các Backend Service bên trong mạng nội bộ (Docker Network), chỉ mở cổng 80/443 ra ngoài.
> Load Balancing: Nếu sau này hệ thống quá tải, có thể bật thêm nhiều Container Backend, Caddy sẽ tự động chia tải.
### Docker
- Chạy trên nền tảng Containerization, theo mô hình dưới đây:

<div>
    <img src="image.png" align="center"></img>
</div>

### Backup
**Yêu cầu**: "Tự động backup và phục hồi số liệu theo ngày hoặc theo ý muốn". Với PostgreSQL và Docker, chúng ta thiết lập một container phụ pg_dump chạy theo Cronjob.   

**Backup Policy:**
> Full Backup: 02:00 AM hàng ngày.

> Transaction Log (WAL) Archiving: Liên tục (Continuous) để có thể khôi phục về bất kỳ thời điểm nào (Point-in-Time Recovery).

**Lưu trữ Backup:** File backup sau khi tạo ra sẽ được mã hóa (GPG encryption) và upload ngay lập tức lên Cloud Storage (như AWS S3 hoặc Google Cloud Storage) để đề phòng rủi ro server vật lý bị cháy nổ hoặc hỏng ổ cứng hoặc sync sang một server khác để đảm bảo lưu trữ dữ liệu của hệ thống.
### Các dịch vụ ngoài
- **Chatwoot:** sử dụng cho hệ thống Support System.
- **Misa:** xuất hóa đơn điện tử.
- **Sepay:** sử dụng cho module thanh toán online.
- **UptimeKuma:** trang status dịch vụ

_(liệt kê thêm sau)_
# User Flow
## Phân Hệ Tiếp Nhận & Cố Vấn Dịch Vụ (Front Desk Module)
- **Nhận diện khách hàng:** Khi xe vào xưởng, Cố vấn dịch vụ (CVDV) nhập biển số hoặc quét QR Code trên sổ bảo hành điện tử.
- **Tra cứu nhanh:** Hệ thống hiển thị ngay lập tức: Tên khách, Loại xe, Lịch sử sửa chữa gần nhất, và Số dư tài khoản/Công nợ.
- **Kiểm tra xe (Vehicle Inspection):**
+ **Mobile App:** CVDV dùng máy tính bảng đi quanh xe.
+ **Chụp ảnh:** Chụp 4 góc xe, chụp vết xước cũ, chụp mức xăng/km. Ảnh được upload lên Server (qua MinIO) và gắn vào phiếu tiếp nhận. Điều này tránh tranh cãi sau này nếu khách khiếu nại "Gara làm xước xe".
+ **Checklist:** Tích vào các mục kiểm tra sơ bộ (Đèn, Còi, Gạt mưa).
- **Tạo phiếu yêu cầu:** Ghi nhận mô tả của khách ("Xe có tiếng kêu lạ ở gầm", "Điều hòa không mát").
- **Ghi nhận tâm lý:** CVDV chọn trạng thái khách (Vui/Buồn) để hệ thống lưu ý các bộ phận sau.
## Phân Hệ Kỹ Thuật & Sửa Chữa (Workshop Module)
- **Bảng kế hoạch (Digital Kanban Board):**
+ Hiển thị danh sách xe đang chờ, xe đang sửa, xe đã xong.
+ Quản lý Garage (Technical Manager) hoặc quản lý trực ca kéo thả xe vào tên từng thợ để giao việc (Assign).
- **Thao tác của Thợ (Mechanic App):**
+ Nhận thông báo "Có xe mới được giao".
+ Bấm "Bắt đầu" (Start Job) -> Đồng hồ bấm giờ chạy. Dữ liệu này dùng để tính năng suất thợ thực tế so với định mức (Flat Rate).
+ **Yêu cầu vật tư:** Thợ chọn phụ tùng cần thay trên App -> Lệnh chuyển sang Kho.
+ **Cập nhật tiến độ:** Thợ chụp ảnh phụ tùng cũ/mới để làm bằng chứng minh bạch cho khách hàng.
## Phân Hệ Kho & Mua Hàng (Inventory & Procurement)
Quản lý kho gara ô tô phức tạp hơn bán lẻ thông thường vì có hàng thay thế lẫn nhau (Part Substitution).
- **Quản lý mã phụ tùng:**
+ Mã OE (Chính hãng), Mã OEM (Nhà sản xuất linh kiện), Mã hàng thay thế.
+ Tính năng Cross-Reference: Khi hết hàng mã A, hệ thống gợi ý mã B có thông số tương đương.
- **Quy trình xuất kho:**
+ Thủ kho nhận yêu cầu từ Thợ -> Soạn hàng -> Quét mã vạch (Barcode/QR) để xuất.
+ **Nguyên tắc FIFO (First-In, First-Out):** Hệ thống chỉ định lấy lô hàng nhập trước (đặc biệt quan trọng với dầu nhớt, lốp xe có hạn sử dụng).
- **Cảnh báo tồn kho:** Thiết lập Min/Max cho từng mã. Khi tụt xuống Min, hệ thống tự động tạo "Phiếu đề nghị mua hàng" gửi bộ phận Mua hàng.
## Phân Hệ Tài Chính & Kế Toán (Finance & Accounting)
- **Báo giá (Quotation):** Tự động lấy giá phụ tùng + % Lợi nhuận + Tiền công thợ. Hỗ trợ nhiều bảng giá (Giá khách lẻ, Giá khách buôn, Giá bảo hiểm).
- **Quyết toán:**
+ Khi xe sửa xong, Thu ngân nhận lệnh "Sẵn sàng thanh toán".
+ **Hệ thống tổng hợp:** Phụ tùng thực xuất + Công thợ thực tế (hoặc khoán) + Các dịch vụ ngoài (Gia công cơ khí, cứu hộ).
+ **Khuyến mãi:** Áp dụng mã giảm giá, thẻ VIP tích điểm.
- Tích hợp Hóa đơn điện tử: Kết nối API với các nhà cung cấp (Misa + Sepay) để xuất hóa đơn ngay lập tức khi thu tiền.
# UI/UX Design
Cần chú trọng các nguyên tắc thiết kế sau:   
## Giao diện Web Admin (Dashboard)
**Nguyên tắc:** "Information Density" (Mật độ thông tin). Người quản lý cần nhìn thấy nhiều số liệu trên một màn hình mà không cần cuộn trang quá nhiều.
- Dashboard: Sử dụng các biểu đồ (Charts) trực quan:
- Biểu đồ cột: Doanh thu 7 ngày gần nhất.
- Biểu đồ tròn: Tỷ trọng các loại dịch vụ (Sơn, Máy, Gầm).
- Widget cảnh báo: Số xe quá hạn giao, Số phụ tùng sắp hết.
## Giao diện Mobile App (End-User)
**Nguyên tắc:** "Simplicity" (Đơn giản). Khách hàng không phải chuyên gia kỹ thuật.
- **Tính năng chính:**
+ **Trang chủ:** Hiển thị xe của tôi, Lịch bảo dưỡng sắp tới.
+ **Sổ theo dõi:** Timeline lịch sử sửa chữa (Ngày nào, làm gì, hết bao nhiêu tiền, ảnh hóa đơn).
+ **Đặt lịch:** Chọn ngày, chọn gói dịch vụ, chọn Cố vấn quen thuộc.
+ **Tracking:** Giống như Grab/Uber, khách hàng thấy trạng thái xe: "Đang kiểm tra" -> "Đang đợi phụ tùng" -> "Đang rửa xe".
## Landing Pages & Status Page
- **Landing Page:** Thiết kế chuẩn SEO, tập trung vào chuyển đổi (Conversion). Nút "Đặt hẹn ngay" phải nổi bật. Giới thiệu các gói Combo bảo dưỡng.
- **Public Status Page:** Một trang web đơn giản, chỉ có một ô nhập liệu. Khách nhập Biển số xe -> Hệ thống trả về trạng thái hiện tại. Trang này không yêu cầu đăng nhập, giúp khách hàng tra cứu nhanh mà không cần cài App.
# TOS/Privacy
## Điều Khoản Dịch Vụ (Terms of Service - TOS)
Cần soạn thảo văn bản TOS tích hợp vào App/Web, bao gồm các điều khoản trọng yếu:
- **Giới hạn trách nhiệm:** NekoTech (đơn vị cung cấp phần mềm) không chịu trách nhiệm về các sai sót kỹ thuật của Gara đối với xe của khách. NekoTech chỉ cung cấp công cụ quản lý.
- **Chính sách bảo quản tài sản**: Quy định rõ Gara không chịu trách nhiệm với tiền bạc, tư trang quý giá để quên trên xe nếu khách hàng không khai báo và niêm phong khi bàn giao (Check-in). Toàn bộ trách nhiệm sẽ không quy về cho NekoTech (đơn vị cung cấp phần mềm), NekoTech chỉ cung cấp công cụ quản lý.
- **Điều khoản thanh toán:** Quy định về việc tạm dừng giấy phép phần mềm/gián đoạn dịch vụ nếu Garage không thanh toán chi phí sử dụng dịch vụ.
## Chính Sách Bảo Mật (Privacy Policy)
- **Thu thập dữ liệu:** Giải thích rõ iCar thu thập: Tên, SĐT, Biển số, Vị trí (cho tính năng cứu hộ).
- **Sử dụng dữ liệu:** Dùng để liên lạc, nhắc bảo dưỡng, gửi ưu đãi. Cam kết không bán dữ liệu cho bên thứ ba.
- **Quyền của chủ thể dữ liệu:** Khách hàng có quyền yêu cầu xóa dữ liệu cá nhân (Right to be forgotten), tuy nhiên dữ liệu lịch sử sửa chữa xe vẫn phải được lưu trữ theo quy định kế toán và an toàn kỹ thuật (nhưng sẽ được ẩn danh tính).
# Giai đoạn xây dựng
## Giai đoạn 1: Xây Dựng Nền Móng (Foundation)
- Tập trung phát triển Backend Core, Database và Web Admin cho quy trình nội bộ.
- **Tính năng:** Tiếp nhận xe, Lập báo giá, Sửa chữa, Thanh toán cơ bản.
## Giai đoạn 2: Mở Rộng Vận Hành (Operation Expansion)
- Triển khai App cho Kỹ thuật viên (KTV) để số hóa quy trình giao việc.
- Triển khai Module Kho chi tiết (Barcode, Vị trí kho).
- Tích hợp Zalo OA/SMS Brandname để gửi thông báo tự động.
## Giai đoạn 3: Trải Nghiệm Khách Hàng (Customer Experience)
- Ra mắt Mobile App cho khách hàng (iOS/Android).
- Ra mắt tính năng đặt lịch Online và tra cứu trạng thái sửa chữa.
- Tích hợp hệ thống CRM nâng cao (Phân tích tâm trạng, CSKH tự động).
## Giai đoạn 4: Hệ Sinh Thái & Chuỗi (Ecosystem & Chain)
- Kích hoạt tính năng Multi-tenancy để mở rộng cho các chi nhánh khác.
- Phát triển các báo cáo quản trị thông minh (Business Intelligence) cho Ban giám đốc.
- **Tích hợp các bên thứ 3:** Bảo hiểm, Đăng kiểm, Cứu hộ 24/7.
# Phân thư mục cho dự án
## NekoGarage-CaddyFile-master (Hạ tầng & Proxy)
- **Tech:** Caddy Server

`Caddyfile`: File cấu hình cho Reverse Proxy (thay thế Nginx). Đây là nơi bạn định nghĩa domain nào trỏ vào service nào.
## NekoGarage-Docker-master (Docker Compose)
- **Tech:** Docker Compose

`docker-compose.yml:` File cấu hình để bật tắt toàn bộ các service (Postgres, Redis, Backend, Web) cùng lúc.
## NekoGarage-Backend-master (Hệ thống Lõi - Core System)
- **Tech:** Golang (hoặc Node.js/NestJS nếu team thạo JS).
- **Các module cần code:**
+ `Auth:` Đăng nhập, phân quyền (Admin/Thợ/Khách).
+ `VehicleService:` CRUD hồ sơ xe, lịch sử sửa chữa (Logic ẩn lịch sử chủ cũ nằm ở đây).
+ `Workflow:` Quản lý trạng thái xe (Đang nhận -> Đang sửa -> Đang rửa -> Xong).
+ `Inventory:` Nhập xuất tồn phụ tùng, dữ liệu ảnh,...
+ `Integration:` Kết nối API Zalo ZNS, SMS Brandname, Hóa đơn điện tử (Misa/VNPT),...
-# update thêm trong quá trình làm
## NekoGarage-Admin-master (Giao diện Quản trị)
- **Tech:** ReactJS (Next.js) + Ant Design (hoặc Tailwind CSS).
- **Các trang (Pages) cần làm:**
+ `/dashboard:` Biểu đồ doanh thu, xe đang nằm xưởng.
- `/reception:` Màn hình tiếp nhận xe (nhập biển số, thông tin khách).
+ `/works:` Bảng kéo thả tiến độ xe (như Trello).
+ `/inventory:` Danh sách phụ tùng, cảnh báo hết hàng.
+ `/accounting:` Xuất hóa đơn, sổ quỹ thu chi.
-# update thêm trong quá trình làm
## NekoGarage-Mobile-master (Ứng dụng Di động)
- **Tech:** Flutter (Code 1 lần ra cả iOS và Android).
- **Phân hệ Thợ (Staff Mode):**
+ Nhận việc (Notification).
+ Check-in xe (Chụp ảnh vết xước, upload lên server).
+ Yêu cầu vật tư (Order phụ tùng từ kho).
- **Phân hệ Khách (Customer Mode):**
+ Đặt lịch hẹn (Booking) & thanh toán (Internet Banking).
+ Sổ bảo dưỡng điện tử (Xem lịch sử xe).
+ Cứu hộ (Gửi tọa độ GPS).
## NekoGarage-Landing-master (Trang Landing, Quảng cáo & Pháp lý)
- **Tech:** Next.js (để SEO tốt).
- Các trang cần làm:
+ **Home:** Giới thiệu dịch vụ, bảng giá cơ bản, nút "Đặt hẹn".
+ **About:** Về Gara, đội ngũ kỹ thuật.
+ **TOS (Terms of Service):** Điều khoản sử dụng dịch vụ (Quy định về để xe qua đêm, trách nhiệm tài sản...).
+ **Privacy:** Chính sách bảo mật (Cam kết không lộ thông tin biển số/chủ xe).
+ **Support:** Trang hướng dẫn, câu hỏi thường gặp (FAQ), form khiếu nại.
-# update thêm trong quá trình làm
## NekoGarage-AIRecognition-master (Module AI riêng biệt)
- **Tech:** Python (FastAPI + YOLOv8) hoặc Node.js wrapper.
- **Chức năng:**
+ Nhận đầu vào là ảnh/video từ Camera cổng vào.
+ Đọc biển số xe -> Trả về text biển số cho backend.
## NekoGarage-StatusPage-master (Trang trạng thái)
- **Tech:** UptimeKuma (Open source).
- **Chức năng:**
+ Hiển thị thông báo bảo trì
+ Hiển thị uptime của các dịch vụ (App, Web, API). Giúp khách hàng đỡ gọi hotline khi không vào được app.
# Tiêu chí dự án
1. **Ưu tiên trải nghiệm Mobile:** Kỹ thuật viên và Cố vấn dịch vụ di chuyển liên tục. App của họ phải cực nhanh và dễ dùng (ít thao tác chạm).
2. **Bảo mật dữ liệu & Thiết kế Database:** Việc thiết kế Database đúng ngay từ đầu (đặc biệt là bảng Lịch sử sở hữu xe) là yếu tố sống còn.
3. **Tự động hóa tối đa:** Tận dụng Zalo/SMS để giảm tải cho bộ phận CSKH. Để Message System nhắc lịch bảo dưỡng thay vì con người.
4. **Tuân thủ quy trình:** Phần mềm chỉ phát huy tác dụng khi con người tuân thủ quy trình. Cần có ngân sách cho việc đào tạo (Training) và quản trị sự thay đổi (Change Management) cho nhân viên.
# Checklist
- [x] Cấu hình Docker, Caddy Server (Proxy, SSL)
- [x] Thiết kế ERD, Quy hoạch bảng User, Vehicle, ServiceOrder
- [x] Code API Authentication (JWT), Phân quyền RBAC
- [-] Dựng khung Dashboard, Trang danh sách khách hàng
- [-] Dựng khung App Flutter, Màn hình Login, Home
- [x] Soạn thảo TOS, Privacy Policy, Biểu mẫu tiếp nhận xe
- [x] Thiết kế Landing Page, Bộ nhận diện thương hiệu App
trình bày chi tiết trong bản kế hoạch riêng của nhóm
# NekoTech Foundation Team
Team tham gia dự án bao gồm
- @akk1to: Lead dự án, thực hiện DevOps, Network, Security, Backend.
- @captainnhwuy: Thực hiện Frontend, Database.
- @Khoasoma: Thực hiện Backend (30%), Database, AI recognition.
- @maiminhdung: Thực hiện Backend (70%), Mobile Application.
> Dự án NekoGarage đã được đăng ký bản quyền dưới quyền của NekoTech Foundation. Mọi hành vi tham khảo, sao chép, sử dụng một phần mã nguồn hoặc toàn bộ mã nguồn mà chưa được sự đồng ý của 3/4 thành viên của Team sẽ được coi là hành vi vi phạm pháp luật.

-# Copyright (C) 2025 - 2026 NekoTech Foundation - Docs written by @akk1to
-# Dự án được thực hiện từ ngày 13/10/2025 - 54% tiến độ đã hoàn thành
