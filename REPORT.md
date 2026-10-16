# REPORT — Lab Day 12

> Kết quả chạy local với MSSV `2A202602552`, 3 seed, không dùng `--quick`. Số liệu lấy từ các file pipeline sinh trong `evidence/`; phần 3D chỉ để quan sát bổ sung.

| | |
|---|---|
| Họ và tên | Nguyễn Trọng Phúc |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602552` |
| GitHub | Wrxhard |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8162 / 8667, 8668, 8669 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Nguồn: `evidence/closedloop.json`, `openloop.json`, `gate.json`. Với mô hình học, báo **dải qua các seed**; bảng xếp hạng ở Câu 4 dùng trung bình. Cổng dùng seed xấu nhất theo quy tắc gốc.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,2488 | 20,0486 | BLOCK: R2, R3, R4, R6 |
| IDM | 1,5452 | 0,0694 | (baseline) |
| EgoMLP | 0,7570–0,7602 | 13,9438–23,1009 | BLOCK: R2, R3, R4, R5, R6 |
| PercMLP | 0,6197–0,6266 | 0,3469–1,6302 | BLOCK: R2, R3, R4, R5, R6 |
| PercMLP+ | 0,8219–0,8710 | 0,2775 | RELEASE |

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

Các khung 99–101 cũng có TTC dưới 1,5 s và lệnh gốc phanh nhẹ hơn −6, nên AEB can thiệp đúng điều kiện. Các giá trị được tính lại từ replay_log và lưu vào [replay_inspection.json](evidence/replay_inspection.json).

Fastest không có trễ/rớt; realtime có 582 khung trễ, 578 khung rớt trên 59.757 khung, làm 372 đoạn đổi lệnh.

Chi phí xử lý mô phỏng có spike vượt deadline 100 ms; phát nhanh không đặt deadline nên che mất lỗi thời gian. Nguồn: [replay.json](evidence/replay.json).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới dùng cruise 12 m/s, gap 2,7601–59,3123 m, closing 0–12 m/s; 60 × 45 = 2.700 ô. Khai thác được 614 ca phanh, chiếm 6,389% quỹ đạo.

Trong thời gian phản ứng mất dv × t_react khoảng cách; sau đó cần dv²/(2b_max) để triệt tiêu tốc độ tiến gần. Ô có gap nhỏ hơn tổng này là bất khả thi.

Tỷ lệ bất khả thi tăng từ 2,222% ở trễ 0 s lên 7,444% ở trễ 0,6 s; vùng đâm IDM tăng tương ứng 5,222 điểm phần trăm.

FullBrake và biên vật lý lệch 0 ô ngoài vùng sát biên ở cả hai độ trễ; diện tích lỗi quy cho IDM bằng 0, tránh gán lỗi điều khiển cho tình huống ai cũng đâm.

Giới hạn phanh người lái p99,9 là 3,6248 m/s², thấp hơn khả năng cơ cấu 9 m/s²; 8,556% lưới đòi phanh mạnh hơn mức đã thấy trong log.

Hình lưới cho thấy các chấm log tập trung ở closing thấp, thưa ở vùng closing lớn và gap nhỏ. Do đó log không đủ phủ vùng khó; trung bình trên log có thể che thất bại khi cắt vào gấp. Nguồn: [scenarios.json](evidence/scenarios.json), [hình lưới](evidence/scenarios_idm.png).

## Câu 4 · Vòng hở và vòng kín

Đánh giá vòng hở: 167553 cửa sổ train, 71580 cửa sổ test; lát cắt phanh 4,3546%. const-vel: L2 trung bình/lát cắt 1,2488/1,8660 m; IDM: L2 trung bình/lát cắt 1,5452/1,6373 m; EgoMLP: L2 trung bình/lát cắt 0,7586/1,0257 m; PercMLP: L2 trung bình/lát cắt 0,6239/0,8223 m. Nguồn: [openloop.json](evidence/openloop.json).

Chấm 2883 quỹ đạo giữ lại, gồm 165 đoạn xe dẫn phanh. Bảng dùng trung bình qua seed để xếp hạng, đồng thời nêu dải va chạm.

| Bộ điều khiển | L2 trung bình (m) / hạng | Va chạm trung bình (%) / hạng | Dải va chạm (%) |
|---|---|---|---|
| const-vel | 1,2488 / 3 | 20,0486 / 4 | 20,0486 |
| IDM | 1,5452 / 4 | 0,0694 / 1 | 0,0694 |
| EgoMLP | 0,7586 / 2 | 19,1120 / 3 | 13,9438–23,1009 |
| PercMLP | 0,6239 / 1 | 0,8440 / 2 | 0,3469–1,6302 |

Các cặp đảo thứ tự: const-vel ↔ IDM; IDM ↔ EgoMLP; IDM ↔ PercMLP.

IDM không tối ưu bắt chước quỹ đạo người nên L2 cao, nhưng phản ứng trực tiếp với gap và closing để giữ khoảng cách. EgoMLP chỉ nhìn lịch sử tốc độ bản thân: có thể đoán tốt quán tính trên log nhưng không biết xe trước đang gần/phanh; sai số tự lái còn đưa nó ra khỏi phân bố train.

PercMLP có thêm cảm nhận nên giảm lỗi, nhưng điểm dự báo tốt vẫn chưa bảo đảm vòng kín an toàn bằng baseline. Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [hình so sánh](evidence/openloop_vs_closedloop.png).

## Câu 5 · Một lần chạy chưa đủ

EgoMLP trên 3 seed có tỷ lệ va chạm 13,9438%; 20,2914%; 23,1009%; dải rộng 9,1571 điểm phần trăm.

Tốt nhất: seed 8667, 402/2883 va chạm, Wilson 95% [12,7272%; 15,2563%].

Xấu nhất: seed 8669, 666/2883 va chạm, Wilson 95% [21,5988%; 24,6747%].

Hai khoảng không chồng nhau. Wilson biểu diễn bất định lấy mẫu quỹ đạo dưới giả định mẫu phù hợp; dải seed biểu diễn biến thiên huấn luyện, hai thứ không thay thế nhau.

Chọn một seed có thể báo 13,9438% hoặc 23,1009%, trong khi constant-velocity là 20,0486%; chọn seed theo test sẽ làm lệch kết luận. Cần báo mọi seed và CI; phụ thuộc giữa các quỹ đạo còn có thể khiến CI lạc quan. Nguồn: [closedloop.json](evidence/closedloop.json).

## Câu 6 · Tác nhân phản ứng

Hiệu chỉnh T = 1,4 s, W1 = 0,3455, so với 0,8594 của xe không phản ứng; headway người lái trung vị 1,8602 s. Đây chỉ là khớp một phân bố, chưa chứng minh tác nhân giống người ở mọi hành vi.

| Bộ điều khiển | Trễ ego 0 s: replay / reactive (%) | Trễ ego 0,6 s: replay / reactive (%) |
|---|---|---|
| const-vel | 77,630 / 77,630 | 77,630 / 77,630 |
| IDM | 75,259 / 2,222 | 75,741 / 7,444 |
| EgoMLP | 76,815 / 76,815 | 77,000 / 77,000 |
| PercMLP | 81,852 / 15,148 | 78,815 / 29,815 |

Bảng là tỷ lệ ô có bất kỳ va chạm, lấy seed xấu nhất theo thế giới reactive; xe sau cách 18,8 m, trễ 0,6 s.

IDM phanh tránh xe cắt vào nhưng xe sau replay không biết phản ứng, nên vẫn lao vào đuôi. Va chạm này là sai lệch của giả định thế giới khi dùng để quy trách nhiệm cho ego.

Va chạm phía sau của IDM (replay → reactive) ở trễ ego 0 s và 0,6 s lần lượt: 73,037% → 0,000%; 68,296% → 0,000%; do đó trong thế giới reactive lợi thế của IDM trước constant-velocity thể hiện rõ hơn nhiều. Trong bộ số này IDM vẫn đứng trước các mô hình học ở cả hai thế giới, nhưng chênh lệch thay đổi mạnh; không được đồng nhất va chạm replay với lỗi của bộ điều khiển. Nguồn: [agents.json](evidence/agents.json).

## Câu 7 · Cổng phát hành và sửa

PercMLP bị chặn bởi R2, R3, R4, R5, R6. Các điều kiện không đạt dưới đây trích từ cổng cuối (rate/area thập phân theo thang 0–1):

R2: upper 95% 0.021610 <= baseline 0.000694 + margin 0.010000.

R3: 7.9 % (n=165, upper 13.0 %) vs baseline 0.6 % + 5 points.

R4: avoidable collision area 0.223704 <= baseline 0.000000 + margin 0.020000.

R5: 0.3 .. 1.6 % over 3 run(s).

R6: 29.8 % of the grid vs baseline 7.4 %.

Hai nguyên nhân là thiếu kịch bản cắt vào khó (R4/R6), và sai số bắt chước tích lũy khi tự lái (lỗi log/lát cắt, biến thiên seed). Cửa sổ cắt vào gắn nhãn IDM phủ tình huống thiếu; DAgger hỏi IDM dự báo từ đúng trạng thái mô hình tự đi vào để học hồi phục.

Đã thêm 20000 tình huống cắt vào và 2 vòng DAgger. Sau sửa, va chạm trên log đổi từ 0,3469–1,6302% thành 0,2775%; diện tích lưới tránh được nhưng vẫn đâm (trễ 0,6 s) từ 22,3704% thành 0,0741%.

Các quy tắc chuyển rớt → qua: R2, R3, R4, R5, R6. Quyết định cuối PercMLP+: RELEASE. L2 trung bình đổi từ 0,6239 thành 0,8423 m, cho thấy tối ưu L2 khác mục tiêu điều khiển.

Ở tốc độ ngoài tập sửa 20.0 m/s (cửa sổ tổng hợp chỉ tới 16 m/s), vùng lỗi quy cho PercMLP+ là 0,3704–0,4815%, IDM 0,0000%.

Qua cổng chỉ là không kém hơn theo các biên đã chọn, không chứng minh tốt hơn IDM; mô hình sửa còn học nhãn từ IDM. Chưa đề xuất phát hành thực tế: cần kiểm tra ngoài phân bố độc lập, mở rộng tương tác/cảm nhận, đánh giá thoải mái và hiệu quả, rồi thử có kiểm soát. Nguồn: [gate.json](evidence/gate.json), [fix.json](evidence/fix.json), [openloop_fixed_per_seed.json](evidence/openloop_fixed_per_seed.json).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, thế giới chỉ có chuyển động dọc 1 chiều: không kiểm tra đánh lái, giao lộ hay người đi bộ, nên không thể suy ra an toàn của hệ thống lái hoàn chỉnh. Cần thêm kịch bản đa tác nhân 2D/3D.

Thứ hai, xe dẫn vẫn phát lại log; nó không phản ứng với xe thử. Kết quả thay đổi khi xe sau được phản ứng cho thấy giả định thế giới có thể đổi kết luận, cần hiệu chỉnh và kiểm chứng nhiều loại tác nhân bằng log độc lập.

Thứ ba, cảm nhận chủ yếu lý tưởng; nhiễu gap 1 m và tốc độ xe dẫn 0,5 m/s chỉ là vài mức mô phỏng, chưa có mất dấu, sai đồng bộ hay lỗi cảm biến kéo dài. Cần dữ liệu cảm biến và kiểm thử lỗi có tương quan theo thời gian.

Thứ tư, 2.700 ô và 3 seed chưa bảo đảm gặp sự kiện hiếm; các ô cùng mô hình vật lý cũng không đại diện cho mọi phân bố giao thông. Cần tập tình huống độc lập, tìm phản ví dụ và phân tầng theo điều kiện vận hành.

Cuối cùng, ngưỡng cổng là tiêu chí của lab và nhãn sửa đến từ IDM; qua cổng chỉ hỗ trợ kết luận trong phạm vi mô phỏng. Trước phát hành cần kiểm định ngoài phân bố, đo độ thoải mái/tiến triển và thử nghiệm có kiểm soát, có giám sát.

Các quy mô và mức nhiễu đối chiếu trong [scenarios.json](evidence/scenarios.json) và [closedloop.json](evidence/closedloop.json).
