# Tự Động Hóa Nhận Cập Nhật Blog Microsoft Dev qua Email Outlook 365 với GitHub Actions

Dự án này cung cấp một giải pháp tự động để theo dõi các bài viết mới từ một blog cụ thể của Microsoft Developer (ví dụ: Microsoft 365 Developer Blog) và gửi thông báo qua email trực tiếp đến hộp thư Outlook 365 của bạn.

## Mục Lục

1.  [Mục Tiêu](#mục-tiêu)
2.  [Các Thành Phần Chính Tham Gia](#các-thành-phần-chính-tham-gia)
3.  [I. Bước Chuẩn Bị](#i-bước-chuẩn-bị)
    *   [1. Fork Kho Lưu Trữ](#1-fork-kho-lưu-trữ)
    *   [2. Đăng ký Ứng dụng trong Azure AD](#2-đăng-ký-ứng-dụng-trong-azure-ad-tạo-định-danh-cho-workflow-của-bạn)
    *   [3. Thiết lập GitHub Secrets](#3-thiết-lập-github-secrets-bảo-mật-thông-tin)
    *   [4. Xác định RSS Feed URL](#4-xác-định-rss-feed-url-nguồn-dữ-liệu)
4.  [II. Triển Khai và Cấu Hình Workflow](#ii-triển-khai-và-cấu-hình-workflow)
    *   [1. Clone Kho Lưu Trữ Đã Fork](#1-clone-kho-lưu-trữ-đã-fork)
    *   [2. Cấu Hình Tệp Workflow](#2-cấu-hình-tệp-workflow)
    *   [3. Commit và Đẩy Thay Đổi](#3-commit-và-đẩy-thay-đổi)
5.  [III. Giải Thích Mã Workflow](#iii-giải-thích-mã-workflow)
6.  [IV. Chạy và Kiểm Tra Workflow](#iv-chạy-và-kiểm-tra-workflow)
7.  [V. Lưu Ý Quan Trọng và Khắc Phục Sự Cố](#v-lưu-ý-quan-trọng-và-khắc-phục-sự-cố)

---

## Mục Tiêu

Thiết lập một hệ thống tự động:
- Định kỳ kiểm tra các bài viết mới trên một blog cụ thể của Microsoft Dev (sử dụng RSS Feed).
- Nếu có bài viết mới, tự động soạn một email chứa thông tin tóm tắt và đường dẫn đến bài viết.
- Gửi email đó đến hộp thư Outlook 365 của bạn.

## Các Thành Phần Chính Tham Gia

-   **Microsoft 365/Azure Active Directory (Azure AD)**: Nơi chúng ta đăng ký một "ứng dụng" để có thể xác thực và gửi email qua Microsoft Graph API.
-   **Microsoft Graph API**: Giao diện lập trình ứng dụng (API) của Microsoft 365. Chúng ta sẽ dùng nó để gửi email.
-   **GitHub Workflow (GitHub Actions)**: Nền tảng tự động hóa của GitHub. Nơi chúng ta định nghĩa các bước chạy để lấy dữ liệu và gửi email.
-   **RSS Feed**: Định dạng phổ biến để các trang web blog cung cấp nội dung cập nhật. Đây là nguồn dữ liệu chính của chúng ta.
-   **GitHub Secrets**: Cơ chế bảo mật của GitHub để lưu trữ các thông tin nhạy cảm (như mật khẩu, khóa API).
-   **`curl`**: Công cụ dòng lệnh để gửi các yêu cầu HTTP (tải RSS, gọi API).
-   **`jq`**: Công cụ dòng lệnh để xử lý dữ liệu JSON.
-   **`xmlstarlet`**: Công cụ dòng lệnh để xử lý dữ liệu XML (phân tích RSS Feed).

---

## I. Bước Chuẩn Bị

Để GitHub Workflow có thể gửi email qua Outlook 365, bạn cần "cho phép" nó thông qua Azure AD.

### 1. Fork Kho Lưu Trữ

Bắt đầu bằng cách tạo một bản sao của kho lưu trữ này vào tài khoản GitHub của bạn:

1.  Truy cập trang kho lưu trữ gốc: [https://github.com/king1x32/Microsoft-Dev-Notification](https://github.com/king1x32/Microsoft-Dev-Notification)
2.  Nhấn nút **Fork** ở góc trên bên phải hoặc liên kết này: [https://github.com/king1x32/Microsoft-Dev-Notification/fork](https://github.com/king1x32/Microsoft-Dev-Notification/fork)
3.  Thực hiện theo các bước trên màn hình để hoàn tất quá trình fork. Điều này sẽ tạo một bản sao (fork) của kho lưu trữ này dưới tài khoản của bạn (ví dụ: `your-username/Microsoft-Dev-Notification`).

### 2. Đăng ký Ứng dụng trong Azure AD (Tạo định danh cho Workflow của bạn)

Đây là bước bạn tạo một "định danh" (identity) cho GitHub Workflow của mình trong hệ thống Microsoft 365 để nó có quyền gửi email.

**Yêu cầu**: Bạn cần có quyền quản trị viên trên một tenant Microsoft 365/Azure AD để thực hiện bước này.

1.  Mở trình duyệt và truy cập **Azure Portal**: [portal.azure.com](https://portal.azure.com/).
2.  Đăng nhập bằng tài khoản quản trị viên của bạn.
3.  Trong thanh tìm kiếm ở trên cùng, gõ "Azure Active Directory" và chọn nó.
4.  Trong menu bên trái của Azure Active Directory, chọn **App registrations**.
5.  Nhấn nút **+ New registration**.
6.  Điền thông tin:
    *   **Name**: Đặt tên dễ nhớ cho ứng dụng của bạn, ví dụ: `GitHubOutlookEmailSender`.
    *   **Supported account types**: Chọn **"Accounts in this organizational directory only (Default Directory only - Single tenant)"**.
    *   **Redirect URI (optional)**: Để trống hoặc chọn "Web" và nhập bất kỳ URL nào (ví dụ: `https://localhost`), nó không quan trọng cho kịch bản này.
    *   Nhấn nút **Register**.
7.  Sau khi ứng dụng được tạo, bạn sẽ được đưa đến trang tổng quan của ứng dụng đó. **Ghi lại các thông tin sau (cực kỳ quan trọng!)**:
    *   **Application (client) ID**: Mã định danh của ứng dụng.
    *   **Directory (tenant) ID**: Mã định danh của tổ chức/tenant Microsoft 365 của bạn.
8.  **Tạo Client Secret (Mật khẩu cho ứng dụng của bạn)**:
    *   Trong menu bên trái của ứng dụng vừa tạo, chọn **Certificates & secrets**.
    *   Chọn tab **Client secrets**.
    *   Nhấn nút **+ New client secret**.
    *   **Description**: Đặt tên cho secret này, ví dụ: `GitHubWorkflowSecret`.
    *   **Expires**: Chọn thời gian hết hạn (ví dụ: `24 months`). Hãy nhớ rằng bạn sẽ cần tạo secret mới khi cái này hết hạn.
    *   Nhấn nút **Add**.
    *   **NGAY LẬP TỨC SAO CHÉP GIÁ TRỊ (Value)** của Client Secret. Giá trị này sẽ chỉ hiển thị một lần duy nhất khi bạn tạo. Nếu bạn rời khỏi trang này, bạn sẽ không thể lấy lại được nó và sẽ phải tạo một cái mới. **Lưu trữ giá trị này ở nơi an toàn**.
9.  **Cấp quyền API (Cho phép ứng dụng gửi email)**:
    *   Trong menu bên trái của ứng dụng, chọn **API permissions**.
    *   Nhấn nút **+ Add a permission**.
    *   Trong tab "Microsoft APIs", chọn **Microsoft Graph**.
    *   Chọn **Application permissions** (vì ứng dụng sẽ gửi email mà không cần người dùng đăng nhập tương tác).
    *   Trong phần "Select permissions", tìm kiếm và mở rộng nhóm **Mail**.
    *   Chọn quyền **Mail.Send**. (Quyền này cho phép ứng dụng gửi thư dưới dạng bất kỳ người dùng nào).
    *   Nhấn nút **Add permissions**.
    *   **RẤT QUAN TRỌNG**: Sau khi thêm quyền, bạn sẽ thấy một cảnh báo "Not granted for..." hoặc "Not granted admin consent for...". Bạn cần nhấn nút **Grant admin consent for <Your Organization Name>** để xác nhận rằng quản trị viên đồng ý cho ứng dụng này có quyền `Mail.Send`. Xác nhận khi được hỏi. Bạn sẽ thấy trạng thái chuyển thành "Granted for...".

### 3. Thiết lập GitHub Secrets (Bảo mật thông tin)

Chúng ta sẽ lưu trữ các thông tin nhạy cảm đã thu thập được từ Azure AD và các địa chỉ email trong GitHub Secrets để chúng không bị lộ ra trong mã nguồn.

1.  Mở trình duyệt và truy cập **kho lưu trữ GitHub đã fork của bạn** (ví dụ: `https://github.com/your-username/Microsoft-Dev-Notification`).
2.  Đi tới tab **Settings** (Cài đặt) của kho lưu trữ.
3.  Trong menu bên trái, chọn **Secrets and variables** > **Actions**.
4.  Nhấn nút **New repository secret**.
5.  Tạo các secret sau với các giá trị bạn đã ghi lại hoặc muốn sử dụng:
    *   `AZURE_AD_CLIENT_ID`: Dán giá trị `Application (client) ID` từ Azure AD.
    *   `AZURE_AD_TENANT_ID`: Dán giá trị `Directory (tenant) ID` từ Azure AD.
    *   `AZURE_AD_CLIENT_SECRET`: Dán giá trị `Value` của Client Secret mà bạn đã sao chép ngay lập tức khi tạo.
    *   `SENDER_EMAIL_ADDRESS`: Địa chỉ email Outlook 365 mà bạn muốn các email tự động được gửi đi từ đó (ví dụ: `no-reply@yourcompany.com`). Địa chỉ này **phải là một hộp thư tồn tại** trong tenant Microsoft 365 của bạn.
    *   `RECIPIENT_EMAIL_ADDRESS`: **Một hoặc nhiều địa chỉ email Outlook 365 để nhận thông báo.** Phân tách nhiều địa chỉ bằng dấu phẩy (`,`). **Khoảng trắng xung quanh dấu phẩy sẽ được tự động loại bỏ.**
        *   **Ví dụ:** `email1@example.com,email2@example.com,team@example.com`
        *   **Ví dụ:** `email1@example.com, email2@example.com , email3@example.com` (cũng hoạt động)

#### Về SENDER_EMAIL_ADDRESS:

Địa chỉ này **phải là một hộp thư có thật, đang tồn tại và hoạt động trong tổ chức Microsoft 365 của bạn** (cái mà bạn đã tạo Azure AD App cho nó).

**Khuyến nghị:** Sử dụng một [Hộp thư chia sẻ (Shared Mailbox)](https://learn.microsoft.com/en-us/microsoft-365/admin/email/create-shared-mailbox?view=o365-worldwide) chuyên dụng cho thông báo tự động (ví dụ: `no-reply@yourcompany.com`).

**Cách tạo Hộp thư chia sẻ (nếu bạn là quản trị viên M365):**
1.  Đăng nhập [admin.microsoft.com](https://admin.microsoft.com/).
2.  Trong menu bên trái, chọn **Teams & groups** > **Shared mailboxes** (Hộp thư chia sẻ).
3.  Nhấn nút **+ Add a shared mailbox** (Thêm hộp thư chia sẻ).
4.  Điền **Name** và **Email address** bạn muốn sử dụng (ví dụ: `no-reply@yourdomain.com`).
5.  Nhấn **Create**. (Bạn không cần thêm thành viên vào hộp thư này cho mục đích tự động gửi).

### 4. Xác định RSS Feed URL (Nguồn dữ liệu)

Xác định URL RSS Feed của blog Microsoft Dev mà bạn muốn theo dõi.

*   Để theo dõi **Microsoft 365 Developer Blog**, URL RSS Feed là:
    `https://devblogs.microsoft.com/microsoft365dev/feed/`
*   Bạn có thể kiểm tra URL này bằng cách mở nó trong trình duyệt của mình; bạn sẽ thấy nội dung XML của RSS Feed.

---

## II. Triển Khai và Cấu Hình Workflow

Các bước này sẽ hướng dẫn bạn cách thiết lập và cấu hình GitHub Workflow trong kho lưu trữ của bạn.

### 1. Clone Kho Lưu Trữ Đã Fork

Nếu bạn chưa clone kho lưu trữ đã fork của mình về máy cục bộ, hãy làm điều đó bây giờ:

```bash
git clone https://github.com/<your-username>/Microsoft-Dev-Notification.git
cd Microsoft-Dev-Notification
```
(Thay thế `<your-username>` bằng tên người dùng GitHub của bạn)

### 2. Cấu Hình Tệp Workflow

Workflow YAML đã được định nghĩa trong tệp `.github/workflows/monitor-microsoft-dev.yml`. Bạn cần mở tệp này và thực hiện một số cấu hình:

1.  Mở tệp `monitor-microsoft-dev.yml` (nằm trong thư mục `.github/workflows/` trong kho lưu trữ cục bộ của bạn) bằng trình soạn thảo mã yêu thích của bạn.
2.  **Cập nhật RSS Feed URL**: Tìm phần `env` ở đầu tệp và thay đổi giá trị của `RSS_FEED_URL` thành URL RSS Feed bạn đã chọn ở Bước I.4.

    ```yaml
    # Nội dung của .github/workflows/monitor-microsoft-dev.yml

    # ... (các phần khác) ...

    env:
      # Thay đổi URL RSS Feed này thành URL bạn muốn theo dõi
      RSS_FEED_URL: 'https://devblogs.microsoft.com/microsoft365dev/feed/' # <--- CẬP NHẬT TẠI ĐÂY
      # Tệp này sẽ lưu trữ ngày/giờ của bài viết mới nhất đã được xử lý để tránh gửi trùng lặp
      LAST_PROCESSED_FILE: 'last_processed_date.txt' # Khuyến nghị đặt ở thư mục gốc để tránh lỗi quyền

    # ... (phần còn lại của mã workflow) ...
    ```

    **Lưu ý quan trọng**: Đảm bảo rằng `LAST_PROCESSED_FILE` được đặt ở vị trí ngoài thư mục `.github/workflows/` để tránh các vấn đề về quyền. Ví dụ, `last_processed_date.txt` (nếu bạn muốn nó ở thư mục gốc của repo).

### 3. Commit và Đẩy Thay Đổi

Sau khi bạn đã cấu hình tệp `monitor-microsoft-dev.yml` và lưu lại, hãy commit và đẩy thay đổi đó lên kho lưu trữ GitHub đã fork của bạn:

```bash
git add .github/workflows/monitor-microsoft-dev.yml
git commit -m "Configure RSS feed URL and apply latest fixes for email sending"
git push
```

---

## III. Chạy và Kiểm Tra Workflow

1.  **Kích hoạt Workflow**:
    *   **Thủ công (để kiểm tra ngay lập tức)**:
        *   Trên GitHub, đi tới **kho lưu trữ đã fork của bạn**.
        *   Nhấp vào tab **Actions**.
        *   Trong danh sách các workflow bên trái, chọn "Theo dõi Microsoft Dev Blog và Gửi Email".
        *   Nhấn nút **Run workflow** ở góc trên bên phải. Nhấn **Run workflow** lần nữa để xác nhận.
    *   **Tự động**: Workflow sẽ tự động chạy theo lịch trình bạn đã đặt trong `cron` (ví dụ: 9 AM UTC mỗi ngày).

2.  **Kiểm tra kết quả**:
    *   Sau khi kích hoạt, bạn sẽ thấy một lần chạy workflow mới xuất hiện trong tab "Actions".
    *   Nhấp vào lần chạy đó để xem chi tiết. Bạn có thể theo dõi từng bước.
    *   Nếu mọi thứ thành công, bạn sẽ thấy thông báo "Email đã được gửi thành công!" trong log của bước "Gửi Email với các bài viết mới".
    *   Kiểm tra hộp thư đến của `RECIPIENT_EMAIL_ADDRESS` và thư mục "Sent Items" của `SENDER_EMAIL_ADDRESS` trong Outlook 365 của bạn. Bạn sẽ thấy email với các bài viết mới nhất.
    *   Kiểm tra xem tệp `last_processed_date.txt` (hoặc vị trí bạn đã đặt nó) đã được tạo/cập nhật và commit lên GitHub hay chưa.

---

## IV. Lưu Ý Quan Trọng và Khắc Phục Sự Cố

*   **Quyền Admin Azure AD là Mấu Chốt**: Lỗi phổ biến nhất là quên "Grant admin consent" cho quyền `Mail.Send` trong Azure AD. Nếu bạn thấy lỗi liên quan đến "Insufficient privileges" hoặc "Access token is invalid", hãy kiểm tra lại bước này.
*   **Địa chỉ Email Người Gửi**: Đảm bảo `SENDER_EMAIL_ADDRESS` là một hộp thư tồn tại và có thể gửi email trong tenant Microsoft 365 của bạn.
*   **Bảo mật Client Secret**: Không bao giờ đưa Client Secret trực tiếp vào mã YAML. Luôn sử dụng GitHub Secrets.
*   **Kiểm tra Log**: Nếu workflow thất bại, hãy luôn kiểm tra log chi tiết của từng bước trong tab "Actions" để xác định lỗi. Các thông báo `::error::` và `::warning::` được đưa vào để giúp bạn gỡ lỗi.
*   **Giờ Quốc tế Phối hợp (UTC)**: Lịch trình `cron` trong GitHub Actions luôn sử dụng UTC. Hãy chuyển đổi múi giờ của bạn sang UTC khi thiết lập lịch trình.
*   **RSS Feed Có Thể Thay Đổi**: Cấu trúc của RSS Feed có thể thay đổi theo thời gian, điều này có thể làm hỏng logic `xmlstarlet` của bạn. Nếu workflow ngừng hoạt động, hãy kiểm tra lại RSS Feed URL và cấu trúc của nó.
*   **Phân Tích Ngày**: Đảm bảo rằng `date -u -d "$PUB_DATE"` có thể phân tích được định dạng ngày từ RSS Feed của bạn. Hầu hết các RSS Feed chuẩn đều hoạt động tốt.

---