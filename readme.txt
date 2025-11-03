Tại sao code này thoả mãn yêu cầu bài toán ?
Trả lời:
MEK: Master Encryption Key
1. Yêu cầu: "Dùng Key-Value Store" và "API"

    Bài toán yêu cầu: Dùng một đối tượng (object) làm KVS (Key-Value Store) cho tên miền và mật khẩu. Phải triển khai các hàm init, load, dump, get, set, remove.

    Chương trình của bạn:

        Bạn dùng this.data.kvs = {} để lưu trữ dữ liệu.

        Bạn đã triển khai tất cả các hàm API được yêu cầu.

    Bằng chứng (từ Test): Toàn bộ nhóm describe('functionality', ...) đã kiểm tra các hàm này.

        'can set and retrieve a password' (✓ PASS)

        'can set and retrieve multiple passwords' (✓ PASS)

        'returns null for non-existent passwords' (✓ PASS)

        'can remove a password' (✓ PASS)

        'can dump and restore the database' (✓ PASS) => Nhóm test này xác nhận API và KVS của bạn hoạt động đúng chức năng.

2. Yêu cầu: "Bảo vệ bằng Mật khẩu chủ" & "Không lưu Mật khẩu"

    Bài toán yêu cầu: Được bảo vệ bằng mật khẩu chủ và không được lưu mật khẩu chủ (hoặc thông tin rò rỉ về nó) trong file dump.

    Chương trình của bạn:

        Bạn không bao giờ lưu password vào this.data.

        Bạn chỉ dùng nó một lần trong init hoặc load để tạo ra Khóa Mã hóa Chính (MEK).

        MEK được lưu trong this.secrets (không bao giờ bị dump).

        Bạn lưu salt (là public) để có thể tạo lại MEK từ mật khẩu chủ khi load.

    Bằng chứng (từ Test):

        "doesn't store domain names and passwords in the clear" (✓ PASS): Test này kiểm tra expect(contents).not.to.contain(password); và nó đã pass, chứng tỏ mật khẩu chủ không có trong file dump.

        'returns false if trying to load with an incorrect password' (✓ PASS): Đây là test quan trọng. Nó xác nhận rằng nếu không có mật khẩu đúng, không thể load. Chúng ta đã làm điều này bằng cách tạo một verifier (một chuỗi đã mã hóa) trong hàm init, và hàm load phải giải mã nó thành công để xác thực mật khẩu.

3. Yêu cầu: "Dùng PBKDF2"

    Bài toán yêu cầu: Phải dùng PBKDF2 để tạo khóa từ mật khẩu.

    Chương trình của bạn:

        Hàm _deriveMEK của bạn sử dụng subtle.deriveKey với các tham số { name: "PBKDF2", ... } và iterations: 100000.

    Bằng chứng (từ Test):

        Chính là thời gian chạy test (màu đỏ và vàng). Như bạn đã hỏi, các test (81ms), (114ms), (96ms) chạy "chậm" chính là vì chúng đang phải thực thi 100,000 vòng lặp PBKDF2. Đây là bằng chứng rõ ràng nhất cho thấy PBKDF2 đang hoạt động chính xác.

4. Yêu cầu: "Chống Rollback Attack"

    Bài toán yêu cầu: Bảo vệ chống lại tấn công "quay lui" (kẻ tấn công thay thế file keychain của bạn bằng một phiên bản cũ hơn).

    Chương trình của bạn:

        Hàm dump() của bạn trả về [repr, checksum], trong đó checksum là một hash SHA-256 của repr.

        Hàm load(password, repr, trustedDataCheck) của bạn so sánh trustedDataCheck (từ nơi lưu trữ tin cậy) với hash mới tính toán của repr.

    Bằng chứng (từ Test):

        'fails to restore the database if checksum is wrong' (✓ PASS): Test này đã thử load với một fakeChecksum. Chương trình của bạn đã phát hiện ra sự không khớp và throw new Error(...), khiến await expectReject(...) thành công.

5. Yêu cầu: "Chống Swap Attack" (và Phải độc lập)

    Bài toán yêu cầu: Chống tấn công "tráo đổi" (kẻ tấn công sao chép một mục đã mã hóa từ keychain A sang keychain B).

    Chương trình của bạn:

        Trong init, bạn tạo một keychainID duy nhất và lưu nó (public) vào this.data.

        Khi mã hóa (set) và giải mã (get), bạn sử dụng Additional Authenticated Data (AAD) của AES-GCM.

        AAD của bạn là this.data.keychainID + name (tên miền).

        Điều này "ràng buộc" mật mã với cả keychainID VÀ name. Nếu kẻ tấn công sao chép một mục từ keychain A sang B, keychainID của B sẽ sai, AAD không khớp, và phép giải mã (subtle.decrypt) sẽ thất bại.

    Bằng chứng (từ Test):

        Mặc dù file test.js này không có một bài test tường minh tên là "Swap Attack", nhưng thiết kế (sử dụng AAD) của bạn đã thoả mãn yêu cầu này.

        Yêu cầu này cũng độc lập với Rollback Attack, vì nó hoạt động ngay cả khi trustedDataCheck là null.

6. Yêu cầu Bổ sung (từ Test): "Không lưu tên miền rõ"

    Bài toán yêu cầu: (Ngụ ý từ test) test.js có một bài test bảo mật rất nghiêm ngặt.

    Chương trình của bạn:

        Bạn không dùng name (ví dụ: www.stanford.edu) làm key cho KVS.

        Thay vào đó, bạn dùng const key = await _hash(name); và lưu vào this.data.kvs[key].

    Bằng chứng (từ Test):

        "doesn't store domain names and passwords in the clear" (✓ PASS): Test này đã kiểm tra expect(contents).not.to.contain(url); và nó đã pass. Đây là lý do chúng ta phải hash các key (tên miền).

Tóm lại, chương trình của bạn đã vượt qua tất cả các bài test vì bạn đã triển khai chính xác các kỹ thuật mật mã tiên tiến: PBKDF2 (để tạo khóa), AES-GCM (để mã hóa), AAD (để chống swap attack), Hashing (SHA-256) (để chống rollback và ẩn tên miền), và một Verifier (để kiểm tra mật khẩu).