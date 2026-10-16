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
| constant-velocity | 1,2488 | 20,0486 | Chờ CP6 |
| IDM | 1,5452 | 0,0694 | (baseline) |
| EgoMLP | 0,7564–0,7646 | 15,4700–22,4072 | Chờ CP6 |
| PercMLP | 0,6187–0,6329 | 0,3122–1,9077 | Chờ CP6 |
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

Lưới dùng cruise 12 m/s, gap 2,7601–59,3123 m, closing 0–12 m/s; 60 × 45 = 2.700 ô. Khai thác được 614 ca phanh, chiếm 6,389% quỹ đạo.  
Trong thời gian phản ứng mất dv × t_react khoảng cách; sau đó cần dv²/(2b_max) để triệt tiêu tốc độ tiến gần. Ô có gap nhỏ hơn tổng này là bất khả thi.  
Tỷ lệ bất khả thi tăng từ 2,222% ở trễ 0 s lên 7,444% ở trễ 0,6 s; vùng đâm IDM tăng tương ứng 5,222 điểm phần trăm.  
FullBrake và biên vật lý lệch 0 ô ngoài vùng sát biên ở cả hai độ trễ; diện tích lỗi quy cho IDM bằng 0, tránh gán lỗi điều khiển cho tình huống ai cũng đâm.  
Giới hạn phanh người lái p99,9 là 3,6248 m/s², thấp hơn khả năng cơ cấu 9 m/s²; 8,556% lưới đòi phanh mạnh hơn mức đã thấy trong log.  
Do đó log không đủ phủ vùng khó; trung bình trên log có thể che thất bại khi cắt vào gấp. Nguồn: [scenarios.json](evidence/scenarios.json), [hình lưới](evidence/scenarios_idm.png).

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Chấm 2.883 quỹ đạo giữ lại, gồm 165 đoạn xe dẫn phanh. Bảng dưới dùng L2 và tỷ lệ va chạm trung bình qua seed để xếp hạng, đồng thời nêu dải để tránh che biến thiên.

| Bộ điều khiển | L2 trung bình (m) / hạng | Va chạm trung bình (%) / hạng | Dải va chạm (%) |
|---|---|---|---|
| constant-velocity | 1,2488 / 3 | 20,0486 / 4 | 20,0486 |
| IDM | 1,5452 / 4 | 0,0694 / 1 | 0,0694 |
| EgoMLP | 0,7591 / 2 | 19,9792 / 3 | 15,4700–22,4072 |
| PercMLP | 0,6251 / 1 | 0,9712 / 2 | 0,3122–1,9077 |

Ba cặp đảo thứ tự là IDM với constant-velocity, EgoMLP và PercMLP. IDM không tối ưu bắt chước quỹ đạo người nên L2 cao, nhưng phản ứng trực tiếp với gap và closing để giữ khoảng cách.  
EgoMLP chỉ nhìn lịch sử tốc độ bản thân nên có thể đoán tốt quán tính trong log, nhưng không thấy xe trước đang gần/phanh; sai số tự điều khiển tích lũy làm trạng thái lệch khỏi tập huấn luyện.  
PercMLP có cảm nhận nên tốt hơn EgoMLP nhưng vẫn chưa bằng baseline vòng kín. Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [hình so sánh](evidence/openloop_vs_closedloop.png).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP trên 5 seed có tỷ lệ va chạm 15,4700%, 20,7076%, 21,8869%, 22,4072%, 19,4242%; dải rộng 6,9372 điểm phần trăm.  
Seed 8667 tốt nhất có 446/2.883 va chạm, Wilson 95% [14,1960%; 16,8359%]; seed 8670 xấu nhất có 646/2.883, khoảng [20,9224%; 23,9654%].  
Hai khoảng không chồng nhau trong phép đo này. Wilson diễn tả bất định lấy mẫu quỹ đạo dưới giả định mẫu phù hợp, còn dải qua seed diễn tả biến thiên huấn luyện; không thay thế được nhau.  
Chỉ chọn seed tốt nhất sẽ báo 15,47%, trông tốt hơn constant-velocity 20,05%; chọn seed xấu nhất sẽ báo 22,41%, trông kém hơn.  
Vì vậy cần báo cả 5 seed và khoảng tin cậy, giữ tập test cố định; không chọn seed theo test. Các quỹ đạo có thể còn phụ thuộc theo bối cảnh nên cũng không diễn giải Wilson là bảo đảm an toàn thực tế.  
Nguồn: mục `logs.EgoMLP.per_seed` trong [closedloop.json](evidence/closedloop.json).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

(trả lời ở đây)

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

(trả lời ở đây)

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, thế giới chỉ có chuyển động dọc 1 chiều: không kiểm tra đánh lái, giao lộ hay người đi bộ, nên không thể suy ra an toàn của hệ thống lái hoàn chỉnh. Cần thêm kịch bản đa tác nhân 2D/3D.  
Thứ hai, xe dẫn vẫn phát lại log; nó không phản ứng với xe thử. Kết quả thay đổi khi xe sau được phản ứng cho thấy giả định thế giới có thể đổi kết luận, cần hiệu chỉnh và kiểm chứng nhiều loại tác nhân bằng log độc lập.  
Thứ ba, cảm nhận chủ yếu lý tưởng; nhiễu gap 1 m và tốc độ xe dẫn 0,5 m/s chỉ là vài mức mô phỏng, chưa có mất dấu, sai đồng bộ hay lỗi cảm biến kéo dài. Cần dữ liệu cảm biến và kiểm thử lỗi có tương quan theo thời gian.  
Thứ tư, 2.700 ô và 5 seed chưa bảo đảm gặp sự kiện hiếm; các ô cùng mô hình vật lý cũng không đại diện cho mọi phân bố giao thông. Cần tập tình huống độc lập, tìm phản ví dụ và phân tầng theo điều kiện vận hành.  
Cuối cùng, ngưỡng cổng là tiêu chí của lab và nhãn sửa đến từ IDM; qua cổng chỉ hỗ trợ kết luận trong phạm vi mô phỏng. Trước phát hành cần kiểm định ngoài phân bố, đo độ thoải mái/tiến triển và thử nghiệm có kiểm soát, có giám sát.  
Các quy mô và mức nhiễu đối chiếu trong [scenarios.json](evidence/scenarios.json) và [closedloop.json](evidence/closedloop.json).

## Chạy local và chụp màn hình

CP3: 167.553 cửa sổ train, 71.580 cửa sổ test; lát cắt phanh chiếm 4,3546%. L2 trung bình/lát cắt lần lượt: constant-velocity 1,2488/1,8660 m; IDM 1,5452/1,6373 m; EgoMLP 0,7591/1,0310 m; PercMLP 0,6251/0,8318 m. Lát cắt đều khó hơn trung bình. Nguồn: [openloop.json](evidence/openloop.json).

Chạy PowerShell trong thư mục chứa `run_all.py`. Dữ liệu `data/`, model `work/` và hai file xuất cho 3D được giữ local theo `.gitignore`.

```powershell
$env:PYTHONUTF8 = '1'
$env:OPENBLAS_NUM_THREADS = '1'
python -m pip install -r requirements.txt
python run_all.py --student-id 2A202602552 --seeds 5
python -m unittest discover -s tests -v
python scripts/check_submission.py --save-verify
python scripts/export_viz.py
python viz/serve.py
```

Mở <http://localhost:8765> sau lệnh cuối; giữ cửa sổ terminal chạy server. Lần mở lại trên máy hiện tại chỉ cần hai lệnh `export_viz.py` và `viz/serve.py` nếu model đã có. Trang 3D cần internet để nạp three.js.

Rubric chấm JSON và hình do pipeline sinh, không bắt buộc ảnh chụp màn hình. Nếu cần nộp thêm screenshot, dùng `Win + Shift + S`, lưu ảnh thật vào `evidence/screenshots/`; không dùng hình 3D để thay số liệu JSON.

| CP | Nội dung cần hiện rõ khi chụp | Evidence chính |
|---|---|---|
| 0 | Terminal `python run_all.py data`: số quỹ đạo, tương quan và số trùng | `data_audit.json` |
| 1 | Terminal bước `replay`: 600 đoạn, float32, AEB, realtime | `replay.json` |
| 2 | Mở `evidence/scenarios_idm.png`: đủ hai panel và đường biên | `scenarios.json`, `scenarios_idm.png` |
| 3 | Terminal bước `openloop`: bảng L2 và lát cắt | `openloop.json` |
| 4 | Mở `evidence/openloop_vs_closedloop.png`, kèm bảng hạng trong terminal | `closedloop.json` |
| 5 | Terminal bước `agents`: T hiệu chỉnh, hai bảng phát lại/phản ứng | `agents.json` |
| 6 | Terminal bước `gate`: các quy tắc BLOCK của PercMLP | `gate.json` |
| 7 | Terminal bước `fix`: trước/sau, kết quả cổng và thử tốc độ 20 m/s | `fix.json` |
| 8 | Trình duyệt localhost: chọn PercMLP+, seed, cảnh cắt vào và bảng cổng cuối trang | Chỉ minh họa bổ sung |

Để chụp lại một bước, dùng `python run_all.py --student-id 2A202602552 --seeds 5 <tên-bước>`. Bước `openloop` và `fix` huấn luyện lại; sau khi chạy lại `closedloop`/`agents` cần chạy `fix` để cập nhật đủ PercMLP+ trước khi kiểm tra nộp bài. Sau khi thêm ảnh: `git add evidence/screenshots`, `git commit -m "docs: add local checkpoint screenshots"`, `git push origin main`.
