# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Trọng Phúc |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602552` |
| GitHub | Wrxhard |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8162 / 8667, 8668, 8669, 8670, 8671 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | | | |
| IDM | | | (baseline) |
| EgoMLP | | | |
| PercMLP | | | |
| PercMLP+ (sau khi sửa) | | | |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu có 992.679 dòng, 9.993 quỹ đạo ban đầu, lấy mẫu 0,1 s và độ dài 70–109 bước (trung vị 106).  
Ba điểm cần xử lý là gia tốc xe dẫn không đáng tin, quỹ đạo trùng và nguy cơ rò rỉ khi chia theo khung hình.  
Tương quan Acc_LV với đạo hàm tốc độ chỉ 0,04535, trong khi ở xe theo sau là 0,92909; thay bằng đạo hàm Speed_LV đã làm mượt để khai thác phanh.  
Loại 383 quỹ đạo trùng hệt, còn 9.610; ID khác nhau không bảo đảm là hai mẫu độc lập.  
Chia theo quỹ đạo sau loại trùng giữ các khung tương quan của cùng một đoạn trong cùng tập, tránh đưa gần như cùng tình huống vào cả train và test.  
Nguồn: [data_audit.json](evidence/data_audit.json).

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Cùng mã chạy lại cho 0/600 đoạn khác nhau; float32 đổi thứ tự phép toán khác bit ở cả 600 đoạn nhưng 0 đoạn vượt dung sai atol = rtol = 0,001 (sai khác lớn nhất khoảng 0,0004 m/s²).  
Dung sai kết hợp tuyệt đối và tương đối tránh coi sai số làm tròn là thay đổi hành vi. AEB làm lệch 7/600 đoạn, nên đây là thay đổi có chọn lọc.  
Đã mở đoạn 50: ở khung 98, gap = 4,3033 m, closing = 2,9189 m/s, TTC = 1,4743 s < ngưỡng 1,5 s; lệnh đổi từ −5,4363 sang −6 m/s².  
Các khung 99–101 cũng có TTC dưới 1,5 s và lệnh gốc phanh nhẹ hơn −6, nên AEB can thiệp đúng điều kiện. Các giá trị này đọc từ replay_log được sinh bởi bước replay, không sửa JSON.  
Fastest không có trễ/rớt; realtime có 582 khung trễ, 578 khung rớt trên 59.757 khung, làm 372 đoạn đổi lệnh.  
Chi phí xử lý mô phỏng có spike vượt deadline 100 ms; phát nhanh không đặt deadline nên che mất lỗi thời gian. Nguồn: [replay.json](evidence/replay.json).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

(trả lời ở đây)

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

(trả lời ở đây)

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

(trả lời ở đây)

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

(trả lời ở đây)

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

(trả lời ở đây)

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(trả lời ở đây)
