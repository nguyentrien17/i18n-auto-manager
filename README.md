# i18n Auto Manager

Extension cho Visual Studio Code giúp quốc tế hóa (i18n) dự án có sẵn một cách an toàn, có thể dự đoán được và hoạt động hoàn toàn offline, không cần AI hay dịch vụ đám mây.

> **Trạng thái:** giai đoạn thiết kế. Repo hiện chứa tài liệu đặc tả, chưa có mã nguồn extension.

## Ý tưởng

Extension quét codebase, nhận diện chuỗi hiển thị cho người dùng bằng AST theo từng ngôn ngữ, chuyển chúng thành lời gọi localization bằng các chỉnh sửa theo khoảng ký tự (giữ nguyên định dạng mã nguồn), sinh và gộp file dịch, rồi kiểm tra tính nhất quán giữa các locale.

Luồng sử dụng:

1. Mở dự án trong VS Code và chạy quét workspace.
2. Xem các chuỗi phát hiện được, mức độ tin cậy và vị trí chính xác trên Activity Bar.
3. Cấu hình ngôn ngữ nguồn, ngôn ngữ đích và chiến lược adapter.
4. Xem trước thay đổi bằng trình diff gốc của VS Code.
5. Áp dụng các thay đổi đã duyệt, có Undo và bảo vệ khi file đã bị sửa.
6. Kiểm tra key thiếu, key thừa, biến nội suy và tính nhất quán tài nguyên.
7. Tiếp tục phát hiện chuỗi mới khi dự án phát triển.

## Nguyên tắc

- **Local-first, offline:** mã nguồn không rời khỏi máy.
- **Xác định (không phụ thuộc AI):** dùng compiler/AST thay vì mô hình xác suất.
- **An toàn, giữ định dạng:** tìm khoảng ký tự bằng AST rồi thay thế theo khoảng, không tuần tự hóa lại toàn bộ file.
- **Có thể xem lại:** mọi thay đổi đều xem được diff trước khi ghi.
- **Idempotent:** chạy hai lần không tạo key hay import trùng.
- **Trung thực:** tách rõ mức tin cậy `high`, `medium`, `low`, `unknown`.

## Hỗ trợ phiên bản 1.0 (kế hoạch)

| Công nghệ | Tích hợp i18n |
|---|---|
| JSON resources (lồng và phẳng) | File locale JSON chuẩn |
| HTML | Thuộc tính template tùy chỉnh |
| JavaScript / TypeScript | `i18next` / `t()` chung |
| Vue 3 SFC | `vue-i18n` |
| React (`.jsx`, `.tsx`) | `react-i18next` |

Dự kiến từ v1.1: Laravel PHP/Blade, YAML, Angular, Svelte.

## Tài liệu

Thư mục [tailieu/](tailieu/):

- [PROJECT_BLUEPRINT_I18N_AUTO_MANAGER.md](tailieu/PROJECT_BLUEPRINT_I18N_AUTO_MANAGER.md): yêu cầu sản phẩm, kiến trúc kỹ thuật, quy trình triển khai.
- [requirements_and_constraints_i18n_auto_manager.md](tailieu/requirements_and_constraints_i18n_auto_manager.md): ràng buộc và tiêu chí nghiệm thu (dùng quy ước MUST/SHOULD/MAY theo RFC 2119).
