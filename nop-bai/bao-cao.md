# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

<!--
HƯỚNG DẪN - đọc rồi XÓA TOÀN BỘ các khối chú thích này sau khi điền xong:

  - Giới hạn: KHÔNG QUÁ 1 TRANG A4, tương đương khoảng 450 - 550 từ nội dung.
  - Chỉ điền vào các chỗ ___ và các ô trong bảng. Không thêm mục mới.
  - Viết bằng câu hoàn chỉnh, không gạch đầu dòng cụt lủn.
  - Kiểm tra độ dài sau khi đã xóa hết chú thích:
        wc -w nop-bai/bao-cao.md
    và xem trước bản in bằng cách mở file trên GitHub rồi Ctrl+P / Cmd+P.
-->

| | |
|---|---|
| Họ và tên | ___ |
| MSSV | ___ |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/___/___ |
| Ngày nộp | ___ |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

<!-- Khoảng 120 - 150 từ. Điền kết quả thật từ MLflow UI ở Bước 1, tối thiểu 3 lần chạy. -->

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Ba lần chạy được thực hiện trên cùng tập train_batch1 và holdout để so sánh công bằng. Cấu hình 200 cây, learning_rate 0.1 và max_depth 5 đạt F1 cao nhất, 0.7149, nên được chọn cho Bước 2 vì vượt ngưỡng 0.65. Cấu hình mặc định 100 cây, depth 3 có accuracy cao nhất là 0.8780 nhưng F1 chỉ 0.7109. Như vậy accuracy cao nhất không trùng với F1 cao nhất; với dữ liệu mất cân bằng, accuracy có thể bị ảnh hưởng nhiều bởi lớp thu nhập thấp. Cấu hình 50 cây với learning_rate 0.05 đạt F1 thấp nhất, 0.6051, cho thấy tốc độ học nhỏ cần nhiều cây hơn để học đủ. Tăng số cây lên 200 khi giữ learning_rate 0.1 cải thiện F1 0.0040 so với cấu hình mặc định, dù accuracy giảm nhẹ 0.0040.

<!--
Trả lời trong phần Lý do:
  - Vì sao bộ này tốt hơn các bộ còn lại (dựa trên f1_score, không phải accuracy)?
  - Lần chạy có accuracy cao nhất có trùng với lần có f1_score cao nhất không?
    Nếu không, điều đó nói lên điều gì?
  - Bạn quan sát thấy đánh đổi nào giữa n_estimators và learning_rate?
-->

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

<!-- Khoảng 120 - 150 từ. -->

Tập Adult mất cân bằng lớp: chỉ khoảng 24,8% mẫu có thu nhập trên 50K. Vì vậy, một mô hình luôn dự đoán "thu nhập thấp" vẫn có accuracy khoảng 0,752, nhưng không phát hiện được bất kỳ mẫu dương nào nên F1 bằng 0. Accuracy chỉ cho biết tỷ lệ dự đoán đúng chung và dễ bị lớp đa số làm cho có vẻ tốt. Trong khi đó, F1 của lớp dương kết hợp precision và recall, nên phản ánh tốt hơn khả năng tìm đúng nhóm thu nhập cao mà bài toán quan tâm. Do đó quality gate dùng `f1_score >= 0.65`. Khi tính F1, em dùng mặc định cho nhãn dương `1`, không dùng `average="weighted"` hoặc `average="macro"`; hai cách trung bình này làm kết quả bị ảnh hưởng bởi lớp thu nhập thấp và không còn phản ánh trực tiếp chất lượng dự đoán lớp thu nhập cao.

<!--
Cần nêu được:
  - Phân bố lớp của tập dữ liệu (tỷ lệ lớp thu nhập > 50K) và hệ quả của nó.
  - Accuracy của một mô hình luôn trả lời "thu nhập thấp" là bao nhiêu, vì sao con số
    đó gây hiểu nhầm.
  - F1 của lớp dương đo điều gì mà accuracy không đo được.
  - Vì sao KHÔNG dùng average="weighted" hay average="macro" khi gọi f1_score.
-->

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

<!-- Nêu 2 - 3 khó khăn thật, mỗi ô một câu ngắn. -->

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| GitHub Runner không SSH được vào EC2 để chạy job Release. | Security group ban đầu chỉ cho phép IP cá nhân; đồng thời public deploy key chưa có trong `authorized_keys` của user `ubuntu`. | Thêm deploy key vào EC2, kiểm tra SSH bằng private key tương ứng, mở tạm port 22 cho runner trong lúc pipeline chạy rồi gỡ rule sau khi deploy xong. |
| MLflow 2.13 lỗi khi dùng SQLAlchemy phiên bản mới. | API pool của SQLAlchemy mới không còn tương thích với code của MLflow đang ghim trong lab. | Ghim `sqlalchemy==2.0.31` trong `requirements.txt`, tạo lại môi trường và chạy lại ba test trước khi huấn luyện. |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

<!-- Lấy số liệu từ bảng ở mục 3.6 của tasks/buoc-3.md. -->

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

**Nhận xét:** Khi thêm 22.361 mẫu từ cùng phân phối, F1 tăng từ 0.7149 lên 0.7354 và accuracy tăng từ 0.8740 lên 0.8820. Mức thay đổi không lớn vì dữ liệu mới cùng nguồn Adult, nhưng pipeline đã tự nạp DVC data, huấn luyện lại, vượt quality gate và triển khai model mới lên EC2.

<!--
Một câu trả lời trung thực kiểu "f1 giảm 0,01 vì dữ liệu mới cùng phân phối, không mang
thêm thông tin mới" được đánh giá cao hơn kết luận sai rằng thêm dữ liệu luôn tốt hơn.
-->

---

## 5. Phần Bonus Đã Thực Hiện (nếu có)

<!-- Xóa cả mục 5 nếu không làm bonus. Mỗi bonus tối đa 1 dòng. -->

- [ ] Bonus 1 - Tracking MLflow từ xa với DagsHub: ___
- [ ] Bonus 2 - Điều chỉnh ngưỡng quyết định: ___
- [ ] Bonus 3 - Báo cáo precision / recall tự động: ___
- [ ] Bonus 4 - Hoàn trả về phiên bản trước: ___
- [ ] Bonus 5 - Cảnh báo lệch lạc dữ liệu: ___
