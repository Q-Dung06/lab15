================================================================================
LAB 15 - XỬ LÝ LỖI NÂNG CAO, LOGGING, CẤU HÌNH VÀ ĐÓNG GÓI ỨNG DỤNG
Học phần: Lập trình .NET (IT3212)
Sinh viên: Nguyễn Quang Dũng - MSSV 24003351
Trường Đại học Công nghệ Đông Á - Khoa Công nghệ Thông tin
================================================================================

1. GIỚI THIỆU
--------------------------------------------------------------------------------
Project Lab15_ErrorHandling là ứng dụng WinForms quản lý sinh viên (CSDL QLSinhVien)
được xây dựng lại theo mô hình 3 lớp DAL / BUS / GUI để minh họa đầy đủ 10 bài của
Lab 15:

  - BusinessException, mã lỗi nghiệp vụ và thông điệp thân thiện cho người dùng.
  - ValidationResult: kiểm tra dữ liệu một lần, trả về toàn bộ lỗi của từng trường.
  - Logger ghi tệp trong thư mục Logs, log có cấu trúc (thời gian, mức, nhóm, thông
    điệp, chi tiết, ngoại lệ) và xoay vòng theo dung lượng.
  - Xử lý lỗi tập trung tại GUI: phân biệt BusinessException / SqlException / Exception,
    hiển thị hộp thoại phù hợp, ghi log và hướng dẫn khắc phục.
  - Đọc App.config: connection string + ReportFolder, LogFolder, LogMaxSizeKB,
    LogRetentionDays, LogLevel, AppDisplayName.
  - Bắt ngoại lệ toàn cục: Application.ThreadException, AppDomain.UnhandledException,
    TaskScheduler.UnobservedTaskException.
  - Nhật ký xoay vòng theo dung lượng và tự dọn tệp cũ theo số ngày lưu giữ.
  - Bảo vệ chuỗi kết nối: thử ProtectedConfiguration và cung cấp DPAPI
    (Config/connection.protected) làm giải pháp tương đương.
  - Màn hình kiểm tra trước khi chạy (preflight) với kết luận ĐẠT / CẢNH BÁO / LỖI.
  - Đóng gói Publish, kiểm thử trên máy đích và xuất biên bản kiểm thử triển khai.

Bài làm hoàn thiện 10/10 bài tập của Lab 15.

2. MÔI TRƯỜNG THỰC HÀNH
--------------------------------------------------------------------------------
  - Windows 11 (64-bit)
  - .NET SDK 10.0 (target net10.0-windows)
  - Visual Studio 2022 hoặc dotnet CLI
  - SQL Server 2022 Express, instance .\SQLEXPRESS
  - CSDL: QLSinhVien (kế thừa từ Lab 12/Lab 13/Lab 14)
  - Không cần Microsoft Office, không cần SAP Crystal Reports runtime.

3. CẤU TRÚC THƯ MỤC
--------------------------------------------------------------------------------
  Lab15_ErrorHandling.slnx        Solution của Lab 15
  Lab15.DAL\                      DbConnectionFactory, SinhVienRepository,
                                  HealthRepository, ConnectionStringProtector
  Lab15.BUS\                      Models, BusinessException, ValidationResult,
                                  StudentValidator, StudentService, AppConfig,
                                  AppLogger (Logger.cs), StudentExportService,
                                  PreflightService, DeploymentService
  Lab15_ErrorHandling\            WinForms: Program, MainForm (6 tab),
                                  UiTheme, ErrorPresenter, SelfTestRunner,
                                  CaptureRunner, App.config
  Database\KiemTraCauTruc.sql     Truy vấn kiểm tra cấu trúc và dữ liệu CSDL
  Docs\                           Biên bản kiểm thử triển khai
  Publish\                        Gói phát hành Release chạy trên máy đích
  Reports\                        Báo cáo sinh ra khi chạy (CSV, XLSX, biên bản)
  Screenshots\                    15 ảnh chụp minh họa từng bài + DanhMucAnh.txt
  tmp\                            Thư mục tạm của quá trình soạn bài (không nộp)

4. CHUẨN BỊ CƠ SỞ DỮ LIỆU
--------------------------------------------------------------------------------
Bước 1. Mở SQL Server Management Studio, kết nối instance .\SQLEXPRESS.
Bước 2. Chạy Database\KiemTraCauTruc.sql để kiểm tra CSDL QLSinhVien đã có đủ:
          - Bảng SinhVien (MaSV, HoTen, NgaySinh, GioiTinh, MaKhoa, DiemTB,
            Email, IsActive), khóa chính PK_SinhVien, unique UX_SinhVien_Email.
          - Bảng Khoa (MaKhoa, TenKhoa), HocPhan (MaHocPhan, TenHocPhan),
            DangKy (MaSV, MaHocPhan, ...), khóa ngoại FK_DangKy_SinhVien.
          - Dữ liệu mẫu: 30 sinh viên, 3 khoa, 5 học phần, 15 lượt đăng ký.
Bước 3. Kiểm tra chuỗi kết nối trong Lab15_ErrorHandling\App.config:
        Data Source=.\SQLEXPRESS;Initial Catalog=QLSinhVien;Integrated Security=True;
        Encrypt=True;TrustServerCertificate=True;Connect Timeout=8

5. CÁCH BUILD VÀ CHẠY
--------------------------------------------------------------------------------
Mở terminal tại thư mục lab15 và chạy:

  dotnet build .\Lab15_ErrorHandling.slnx
  dotnet run --project .\Lab15_ErrorHandling\Lab15_ErrorHandling.csproj

Hoặc mở Lab15_ErrorHandling.slnx bằng Visual Studio, chọn Lab15_ErrorHandling làm
Startup Project và nhấn F5.

Chế độ dòng lệnh phục vụ kiểm thử và tạo sản phẩm:

  dotnet run --project .\Lab15_ErrorHandling\Lab15_ErrorHandling.csproj -- --test
  dotnet run --project .\Lab15_ErrorHandling\Lab15_ErrorHandling.csproj -- --capture .\Screenshots
  dotnet run --project .\Lab15_ErrorHandling\Lab15_ErrorHandling.csproj -- --protocol

  --test      Chạy 14 kiểm thử (cấu hình, validation, CRUD, nghiệp vụ, SQL,
              logging, bảo mật, xuất báo cáo, preflight, triển khai).
              Kết quả ghi vào Reports\KetQuaKiemThuTuDong.txt, mã thoát 0 nếu đạt.
  --capture   Sinh 15 ảnh giao diện vào thư mục truyền vào (mặc định Screenshots).
  --protocol  Chạy preflight và xuất biên bản kiểm thử triển khai vào Reports.

6. BẢN ĐỒ BÀI TẬP
--------------------------------------------------------------------------------
  Bài 1  BusinessException và lỗi nghiệp vụ
         Xóa sinh viên còn đăng ký học phần ném BusinessException mã
         SV_HAS_REGISTRATION; GUI hiển thị thông điệp thân thiện kèm gợi ý.
         Ảnh: 03_LoiNghiepVu_BusinessException.png

  Bài 2  ValidationResult - kiểm tra dữ liệu một lần
         Form nhập sai mã, họ tên, ngày sinh, khoa, điểm, email: tất cả lỗi được
         gom vào ValidationResult, ErrorProvider đánh dấu từng ô và panel đỏ liệt kê
         chi tiết thay vì dừng ở lỗi đầu tiên.
         Ảnh: 02_KiemTraDuLieu_NhieuLoi.png, 07_KiemTraNhieuLoi.png

  Bài 3  Logger ghi tệp trong thư mục Logs
         AppLogger ghi lab15.log với định dạng có cấu trúc:
         thời gian | mức | nhóm | thông điệp | chi tiết | ngoại lệ.
         Tab Nhật ký xem, lọc theo mức, xem chi tiết và xuất CSV.
         Ảnh: 08_NhatKy_Logging.png

  Bài 4  Xử lý lỗi tập trung tại GUI
         ErrorPresenter phân loại:
           - BusinessException  -> hộp thoại cảnh báo, mã lỗi + gợi ý.
           - SqlException       -> hộp thoại lỗi CSDL, ánh xạ mã 2601/2627/547/208/53.
           - Exception          -> hộp thoại lỗi hệ thống, ghi log đầy đủ stack trace.
         Ảnh: 03, 04_LoiCSDL_KhoaNgoai.png, 05_LoiCSDL_BangKhongTonTai.png,
              06_LoiHeThong_Exception.png

  Bài 5  Đọc cấu hình từ App.config
         AppConfig đọc connectionStrings và appSettings (ReportFolder, LogFolder,
         LogMaxSizeKB, LogRetentionDays, LogLevel, AppDisplayName). Tab Cấu hình
         hiển thị giá trị đang dùng, đường dẫn tệp và chuỗi kết nối đã che.
         Ảnh: 10_CauHinh_AppConfig.png

  Bài 6  Bắt ngoại lệ toàn cục
         Program.cs gắn Application.ThreadException, AppDomain.UnhandledException,
         TaskScheduler.UnobservedTaskException; mọi ngoại lệ chưa xử lý đều được ghi
         log và hiển thị hộp thoại lỗi thân thiện trước khi thoát.
         Minh họa: 06_LoiHeThong_Exception.png (kịch bản Exception trong tab Kiểm thử lỗi)

  Bài 7  Nhật ký xoay vòng
         Khi tệp vượt LogMaxSizeKB (mặc định 2048 KB), AppLogger đổi tên tệp hiện tại
         thành lab15-yyyyMMdd-HHmmss-mmm.log, tạo tệp mới và xóa tệp cũ hơn
         LogRetentionDays ngày. Nút "Xoay vòng ngay" minh họa trực tiếp.
         Ảnh: 09_NhatKy_XoayVong.png

  Bài 8  Mã hóa chuỗi kết nối
         - Thử ProtectedConfiguration: gỡ bảo vệ cũ (nếu có), mã hóa section
           connectionStrings bằng DataProtectionConfigurationProvider và lưu lại.
           Trên máy thực hành thao tác này chạy thành công.
         - DPAPI: mã hóa chuỗi kết nối bằng ProtectedData (CurrentUser), ghi
           Config\connection.protected; DbConnectionFactory ưu tiên đọc tệp này,
           nếu không có thì quay về App.config.
         Ảnh: 11_BaoVe_ProtectedConfiguration.png, 12_BaoVe_DPAPI.png

  Bài 9  Màn hình kiểm tra trước khi chạy
         PreflightService kiểm tra: tệp cấu hình, chuỗi kết nối, thư mục Logs,
         thư mục Reports, kết nối SQL Server, CSDL, bảng bắt buộc, dữ liệu mẫu,
         bảo vệ cấu hình, gói triển khai. Kết luận SẴN SÀNG / CẢNH BÁO / LỖI.
         Ảnh: 13_KiemTraTruocKhiChay.png

  Bài 10 Đóng gói và kiểm thử triển khai
         - dotnet publish -c Release -o Publish tạo gói chạy độc lập.
         - Tab Triển khai hiển thị thông tin máy đích, thư mục Publish, số tệp,
           trạng thái bảo vệ chuỗi kết nối và danh mục 12 kiểm thử TC01-TC12.
         - Nút Xuất biên bản sinh Docs/Reports\BienBanKiemThuTrienKhai-*.txt.
         Ảnh: 14_TrienKhai.png, 15_BienBanKiemThuTrienKhai.png

7. CẤU HÌNH VÀ NHẬT KÝ
--------------------------------------------------------------------------------
App.config (bản mẫu, không chứa mật khẩu thật):

  <connectionStrings>
    <add name="QLSinhVien" providerName="System.Data.SqlClient"
         connectionString="Data Source=.\SQLEXPRESS;Initial Catalog=QLSinhVien;
         Integrated Security=True;Encrypt=True;TrustServerCertificate=True;
         Connect Timeout=8;Application Name=Lab15_ErrorHandling" />
  </connectionStrings>
  <appSettings>
    <add key="ReportFolder" value="Reports" />
    <add key="LogFolder" value="Logs" />
    <add key="LogMaxSizeKB" value="2048" />
    <add key="LogRetentionDays" value="30" />
    <add key="LogLevel" value="INFO" />
  </appSettings>

Thứ tự xác định chuỗi kết nối của DbConnectionFactory:
  1. Config\connection.protected (DPAPI, nếu giải mã được)
  2. App.config connectionStrings["QLSinhVien"]
  3. Chuỗi mặc định .\SQLEXPRESS;Initial Catalog=QLSinhVien;Integrated Security=True

Tệp nhật ký: <thư mục chạy>\Logs\lab15.log (và lab15-*.log sau khi xoay vòng).
Mức log hỗ trợ: DEBUG, INFO, WARN, ERROR, FATAL. Nhóm log: App, UI, Validation,
StudentService, Config, Security, Deployment, Database.

8. DỮ LIỆU THỬ
--------------------------------------------------------------------------------
  Sinh viên:        30 (CNTT 12, Kinh tế 9, Công nghệ kỹ thuật ô tô 9)
  Khoa:             3 (CNTT, KT, OTO)
  Học phần:         5 (HP001 - HP005)
  Lượt đăng ký:     15 (học kỳ 1: 8, học kỳ 2: 7)
  Sinh viên mẫu:    SV001 Nguyễn Văn An (có 2 đăng ký - dùng cho lỗi nghiệp vụ)
  Dữ liệu kiểm thử tự động dùng mã ZZT... và tự xóa sau khi chạy.

9. ĐÓNG GÓI VÀ CHẠY TRÊN MÁY ĐÍCH
--------------------------------------------------------------------------------
Tạo gói phát hành:

  dotnet publish .\Lab15_ErrorHandling\Lab15_ErrorHandling.csproj -c Release -o .\Publish

Chạy trên máy đích:
  1. Sao chép thư mục Publish sang máy đích.
  2. Kiểm tra App.config nằm cạnh Lab15_ErrorHandling.exe và trỏ đúng SQL Server.
  3. Chạy Lab15_ErrorHandling.exe.
  4. Kiểm thử tự động: Lab15_ErrorHandling.exe --test
  5. Xuất biên bản: Lab15_ErrorHandling.exe --protocol

Gói Publish kèm theo đã được kiểm tra: --test đạt 14/14, --protocol xuất biên bản
với kết luận SẴN SÀNG (CÓ CẢNH BÁO) do chuỗi kết nối đang đọc trực tiếp từ
App.config (chưa bật DPAPI trên gói mới).

10. KIỂM THỬ ĐÃ THỰC HIỆN
--------------------------------------------------------------------------------
  [OK] dotnet build Lab15_ErrorHandling.slnx           -> 0 cảnh báo, 0 lỗi
  [OK] --test (Debug)                                  -> 14/14 PASS, exit code 0
  [OK] --test (Publish)                                -> 14/14 PASS, exit code 0
  [OK] --protocol (Publish)                            -> biên bản + báo cáo preflight
  [OK] --capture Screenshots                           -> 15 ảnh PNG 1256 x 708
  [OK] Kiểm tra dữ liệu sau test                       -> 30 sinh viên, SV001 còn nguyên,
                                                          không còn bản ghi ZZT
  [OK] Mở lại XLSX bằng openpyxl                       -> sheet SinhVien, 10 cột, đúng kiểu
  [OK] ProtectedConfiguration                          -> mã hóa connectionStrings thành công
  [OK] DPAPI                                           -> tạo/gỡ Config\connection.protected

Chi tiết từng hạng mục xem tại Docs\BienBanKiemThuTrienKhai.md.

11. GHI CHÚ KỸ THUẬT
--------------------------------------------------------------------------------
  - Trên máy thực hành, ProtectedConfiguration hoạt động (nhờ gói
    System.Configuration.ConfigurationManager). Bài làm vẫn cài thêm DPAPI để có
    giải pháp tương đương khi chạy trên nền tảng không hỗ trợ.
  - Ứng dụng đặt APP_CONFIG_FILE = AppContext.BaseDirectory\App.config trước khi
    dùng ConfigurationManager, nhờ đó gói Publish đọc đúng tệp cạnh file exe.
  - SQL Server dùng Integrated Security, không lưu mật khẩu trong tệp cấu hình.
  - Ứng dụng không phụ thuộc Office/COM: XLSX được sinh trực tiếp theo chuẩn OpenXML.
  - Giao diện tự co theo vùng làm việc của màn hình, tối thiểu 1180 x 690.

12. SẢN PHẨM KÈM THEO
--------------------------------------------------------------------------------
  Lab15_ErrorHandling.slnx + 3 project     Source code đầy đủ, build được
  Publish\                                 Gói phát hành Release đã kiểm thử
  Screenshots\*.png                        15 ảnh minh họa + DanhMucAnh.txt
  Docs\BienBanKiemThuTrienKhai.md          Biên bản kiểm thử triển khai
  Reports\                                 Báo cáo sinh khi chạy (nếu có)
  24003351_NguyenQuangDung_Lab15.docx      Báo cáo Word
  README.txt                               Tệp này

13. LIÊN KẾT
--------------------------------------------------------------------------------
  GitHub: https://github.com/Q-Dung06/lab15
  (Nếu tài khoản dùng tên repository khác, cập nhật lại liên kết này và trong Word.)

================================================================================
Hết.
