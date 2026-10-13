BÁO CÁO CHẨN ĐOÁN VÀ KHẮC PHỤC SỰ CỐ PIPELINE (SESSION 12 - BÀI 5)
1. Tổng quan
Trong quá trình xây dựng và vận hành hệ thống CI/CD cho ứng dụng Java Spring Boot sử dụng GitHub Actions, tệp kịch bản .github/workflows/broken-pipeline.yml ban đầu gặp phải 3 sự cố đứt gãy quan trọng. Báo cáo này trình bày chi tiết về nguyên nhân gốc rễ (root cause) của từng lỗi và phương án kỹ thuật đã triển khai để khôi phục trạng thái tích xanh thành công cho toàn bộ pipeline.
2. Chi tiết 3 Lỗi và Phương án Khắc phục
Lỗi 1: Permission Denied (Lỗi quyền thực thi Gradle Wrapper)
Triệu chứng lỗi trên Log:
./gradlew: Permission denied
Error: Process completed with exit code 126.

Nguyên nhân gốc rễ:
Khi mã nguồn được checkout lên Linux runner trên môi trường GitHub Actions, quyền thực thi (executable permission) của tệp lệnh gradlew bị mất hoặc chưa được cấp phép (mặc định chỉ có quyền đọc/ghi -rw-r--r--). Do đó, hệ thống Linux không thể trực tiếp thực thi lệnh ./gradlew test hay ./gradlew build.
Phương án khắc phục:
Bổ sung bước cấp quyền thực thi tường minh bằng câu lệnh chmod +x gradlew trước khi tiến hành gọi các tác vụ của Gradle.
- name: Khắc phục Lỗi 1 - Cấp quyền thực thi Gradle Wrapper
  run: |
    chmod +x gradlew
    ./gradlew test

Lỗi 2: Authentication Failed (Lỗi xác thực Container Registry)
Triệu chứng lỗi trên Log:
Error: Username and password required for login-action
Error: Input required and not supplied: password

Nguyên nhân gốc rễ:
Trong bước đăng nhập vào GitHub Container Registry (GHCR) sử dụng docker/login-action, tên biến bí mật (Secret) được truyền vào trường password bị gọi sai chính tả hoặc không khớp với cấu hình thực tế trên kho lưu trữ (ví dụ: gõ nhầm thành ${{ secrets.REGISTRY_PSW }} hoặc thiếu biến).
Phương án khắc phục:
Kiểm tra và chuẩn hóa lại tên biến bí mật khớp chính xác với cấu hình trên GitHub Repository Secrets là ${{ secrets.REGISTRY_TOKEN }}.
- name: Khắc phục Lỗi 2 - Đăng nhập Container Registry an toàn
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.REGISTRY_TOKEN }}

Lỗi 3: Build Context Error (Lỗi ngữ cảnh xây dựng Docker)
Triệu chứng lỗi / Nguyên nhân:
Lệnh xây dựng hình ảnh container (docker/build-push-action) thất bại do runner không tìm thấy tệp Dockerfile tại thư mục làm việc hiện tại, hoặc đường dẫn ngữ cảnh (context) và tệp cấu hình Dockerfile không được định nghĩa rõ ràng.
Phương án khắc phục:
Chỉ định tường minh tham số context trỏ về thư mục gốc (.) và đường dẫn tệp file trỏ chính xác đến ./Dockerfile.
- name: Khắc phục Lỗi 3 - Cấu hình đúng Build Context cho Docker
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    push: false
    tags: ghcr.io/${{ github.actor }}/fixed-app:latest

3. Kết luận
Sau khi áp dụng đồng bộ 3 giải pháp kỹ thuật trên vào tệp kịch bản .github/workflows/broken-pipeline.yml, toàn bộ các điểm nghẽn và sự cố đứt gãy của pipeline đã được xử lý triệt để. Quá trình kiểm thử tự động trên GitHub Actions đã chuyển hoàn toàn sang trạng thái tích xanh (Success), đảm bảo tính sẵn sàng và độ tin cậy cho quy trình phát triển phần mềm tự động (CI/CD).
Bạn chỉ cần lấy file này từ thư mục máy tính nộp kèm ảnh chụp màn hình tích xanh là hoàn thành trọn vẹn!