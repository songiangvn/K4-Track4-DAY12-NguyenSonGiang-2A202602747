# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Nguyễn Sơn Giang` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602747` |
| GitHub | `songiangvn` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `split_seed 7022 / model_seed 1066 (5 seed: 1066–1070), cruise 12 m/s` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5 (mặc định)` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,243 | 19,4 (tất định, 1 lần chạy; 559/2883) | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1,526 | 0,1 (4/2883, CI95 [0,05; 0,36]) | (baseline) |
| EgoMLP | 0,751 ± 0,002 | 14,0 .. 23,0 (5 seed, TB 18,2) | BLOCK (rớt R2–R6) |
| PercMLP | 0,616 ± 0,003 | 0,6 .. 3,6 (5 seed, TB 1,4) | BLOCK (rớt R2–R6) |
| PercMLP+ (sau khi sửa) | 0,818 ± 0,004 | 0,2 .. 0,3 (5 seed, TB 0,30; seed xấu nhất 0,35, CI95 trên 0,64) | RELEASE (qua cả R1–R7) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Có ba vấn đề trong `data_audit.json`. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm của `Speed_LV`: tương quan chỉ 0,045, trong khi cột tương ứng của xe theo sau đạt 0,929. Vì vậy không thể dùng `Acc_LV` để khai thác ca phanh. Tôi dùng `a_lead`, gia tốc tính từ tốc độ xe dẫn đã làm mượt (`v_lead_s`); với nó, phân vị p0.1 của giảm tốc là −2,77 m/s². (2) Có 383 quỹ đạo trùng hệt nhau nằm ở các ID cách xa nhau: 9993 quỹ đạo thô còn 9610 sau khi làm sạch. Nếu không loại, một bản sao có thể rơi vào train và bản kia rơi vào test, làm điểm test đẹp giả. (3) `Spatial_Gap` đúng bằng hiệu vị trí trừ 4,5 m (sai lệch tối đa 3,6e−11 m), nên khoảng cách đáng tin. Tôi chia theo **quỹ đạo**, không theo khung hình, vì các khung 0,1 s liên tiếp trong cùng một quỹ đạo gần như giống nhau: chia theo khung sẽ để cửa sổ gần như y hệt nằm ở cả train và test, gây rò rỉ.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho 0/600 đoạn khác (giống từng bit). Bản refactor float32 khác bit ở **600/600** đoạn nhưng **0/600** đoạn vượt dung sai (atol = rtol = 1e−3), sai lệch lớn nhất chỉ 2,2e−5 m/s². Phép cộng số thực không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6), nên so sánh `==` sẽ báo lỗi giả. Ngược lại, AEB làm lệch **12/600** đoạn. Tôi mở đoạn 48: chỉ 4/106 khung bị lệch (khung 98–101), tất cả có TTC 1,43–1,49 s, tức dưới ngưỡng AEB 1,5 s. Ở đó lệnh bị ép từ khoảng −5,4..−5,9 xuống −6,0 m/s². Đoạn 59 cũng tương tự: 3/78 khung, TTC 1,44–1,48 s. AEB chỉ kích hoạt khi TTC thấp, nên những khung khác giữ nguyên. Ở chế độ phát nhanh nhất, trễ hạn không tồn tại (0 khung trễ, 0 khung rớt). Phát theo thời gian thật cho thấy 583 khung trễ và 579 khung rớt trên 59852 khung, làm 365/600 đoạn đổi lệnh. Lỗi về thời gian thực này bị phát nhanh nhất che mất hoàn toàn.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của tôi có cruise 12 m/s, khoảng cách 2,76–59,31 m (p2–p98 của log), tốc độ tiến gần 0–12 m/s, tổng 60 × 45 = 2700 ô. Vùng bất khả thi chiếm 2,2 % ô khi t_react = 0 và 7,4 % khi t_react = 0,6 s. Bộ mô phỏng phanh hết cỡ khớp đúng biên này: 0 ô lệch ở cả hai độ trễ. Vùng đâm của IDM tăng từ 2,2 % lên 7,4 % (+5,2 điểm), nhưng diện tích "tránh được mà vẫn đâm" của IDM là 0,0 % ở cả hai độ trễ. Nói cách khác, mọi va chạm của IDM đều nằm trong vùng không ai tránh được, và lỗi không thuộc về bộ điều khiển. Log **không phủ** vùng khó. Người lái trong log chỉ phanh mạnh nhất 3,6 m/s² (p99.9), trong khi bộ chấp hành làm được 9 m/s²; 8,6 % lưới đòi phanh mạnh hơn mức đó. Tôi khai thác được 614 ca xe dẫn phanh (6,4 % quỹ đạo), nhưng đa số nhẹ: giảm tốc đỉnh p50 là −2,5 và p99 là −4,2 m/s². Vì vậy mô hình học từ log chưa từng thấy các ca cắt đầu gấp ở khoảng cách hẹp.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| PercMLP | 0,616 | 1 | 1,4 % | 2 |
| EgoMLP | 0,751 | 2 | 18,2 % | 3 |
| const-vel | 1,243 | 3 | 19,4 % | 4 |
| IDM | 1,526 | 4 | 0,1 % | 1 |

Có ba cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP) và (IDM, PercMLP). IDM có L2 tệ nhất (1,526 m) vì nó không bắt chước người lái. Nó là một luật giữ khoảng cách theo thời gian an toàn T của chính nó, nên dự báo 3 s của nó lệch có hệ thống so với quỹ đạo người thật. Nhưng trong vòng kín, IDM là một bộ điều khiển phản hồi: mỗi bước nó đọc lại khoảng cách và tốc độ tiến gần rồi phanh tương ứng, nên sai số không tích luỹ, và nó chỉ đâm 4/2883 log. L2 chỉ đo việc giống người trên những trạng thái người đã đi qua, không đo hậu quả khi bộ điều khiển tự lái. EgoMLP chỉ nhìn lịch sử tốc độ của chính nó, không thấy khoảng cách hay tốc độ xe dẫn. Nó đoán tốt quán tính tốc độ trung bình của người (L2 0,751 < 1,243), nhưng không thể biết xe trước đang phanh: trên lát xe dẫn phanh, nó đâm 52,9 % (seed xấu nhất), và trên lưới cắt đầu, 69,3 % ô là "tránh được mà vẫn đâm".

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed, va chạm của EgoMLP dao động từ 14,0 % (seed 1070) đến 23,0 % (seed 1066), tức 9 điểm phần trăm. Khoảng Wilson 95 % (n = 2883) của seed tốt nhất là [12,8; 15,3] % và của seed xấu nhất là [21,5; 24,6] %. Hai khoảng **không chồng nhau**. Khoảng Wilson chỉ phản ánh độ bất định do lấy mẫu quỹ đạo trong *một* mô hình; nó không tính đến độ ngẫu nhiên khi huấn luyện, mà ở đây phần đó còn lớn hơn. Nếu chỉ chạy một seed, tôi có thể báo "14,0 %" hoặc "23,0 %" với khoảng tin cậy hẹp trông rất chắc chắn, nhưng cả hai đều sai lệch. Với PercMLP còn nguy hiểm hơn: dải là 0,6..3,6 %. Seed 1067 một mình có 0,6 % (CI trên 0,90 %), thấp hơn ngưỡng R2 là 0,14 % + 1 điểm, nên nếu chỉ chạy seed đó thì mô hình đã qua R2. Chính vì vậy cổng lấy **seed xấu nhất** (3,6 %, CI trên 4,3 %) và có thêm R5 về độ ổn định qua seed.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng ở t_react = 0,6 s (tỷ lệ ô của lưới cắt đầu có bất kỳ va chạm nào; trong ngoặc là va chạm phía sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng |
|---|---|---|
| const-vel | 77,6 % (sau 0,0 %) | 77,6 % (sau 0,0 %) |
| IDM | 75,7 % (sau 68,3 %) | 7,4 % (sau 0,0 %) |
| EgoMLP | 76,7 % (sau 0,0 %) | 76,7 % (sau 0,0 %) |
| PercMLP | 73,0 % (sau 43,2 %) | 31,6 % (sau 0,0 %) |

T hiệu chỉnh là **1,4 s**. Đây là giá trị cho khoảng cách Wasserstein-1 nhỏ nhất (0,35) giữa phân bố thời gian bám đuôi mô phỏng và phân bố của người lái trong log (trung vị 1,88 s). Một xe không bao giờ phản ứng có khoảng cách 0,89, tức kém chân thực hơn khoảng 2,5 lần. Khi xe sau chỉ phát lại tốc độ trong log, nó không biết xe thử vừa phanh. Vì vậy IDM, chính vì phanh đúng lúc để tránh xe cắt đầu, lại bị xe sau tông vào ở 68,3 % ô (73,0 % khi t_react = 0). Đó là va chạm giả do thế giới không phản ứng tạo ra. Trong thế giới phát lại, IDM (75,7 %) trông gần tệ bằng const-vel (77,6 %), một bộ điều khiển không bao giờ phanh. Khi xe sau biết phản ứng, va chạm sau của IDM về 0,0 % và tổng chỉ còn 7,4 %, đúng bằng vùng bất khả thi. Kết luận vì thế đảo hẳn: phanh để tránh va chạm là hành vi đúng, không phải lỗi.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP rớt 5 quy tắc. R2: seed xấu nhất 3,6 %, CI trên 4,3 % > 0,1 % + 1. R3: 14,1 % trên lát phanh (CI trên 20,1 %) > 1,8 % + 5. R4: 24,2 % lưới "tránh được mà vẫn đâm" > 0,0 % + 2. R5: 0,6..3,6 % qua seed. R6: 31,6 % > 7,4 % + 2. Có hai nguyên nhân gốc. (a) **Log không chứa kịch bản cắt đầu ở cự ly hẹp** (R4, R6): người lái trong log phanh tối đa 3,6 m/s², và ở tốc độ chưa thấy (20 m/s) PercMLP đâm 27,1..35,8 % lưới. (b) **Sai số tích luỹ / covariate shift** (R2, R3, R5): L2 vòng hở tốt nhất (0,616 m) nhưng khi tự lái, nó đi vào các trạng thái người chưa từng đi qua. Tôi sửa (a) bằng 20000 kịch bản cắt đầu mô phỏng (197340 cửa sổ), gán nhãn bởi IDM; sửa (b) bằng DAgger 2 vòng, gán nhãn IDM tại chính các trạng thái mô hình tự đi vào (~896k cửa sổ). Kết quả: R4 còn 0,3 %, R6 7,7 %, R2 0,3 % (CI trên 0,6 %), R5 0,2..0,3 %, R3 2,4 % (CI trên 5,9 %), nên **RELEASE**. L2 lại tăng từ 0,616 lên 0,818 m: điểm vòng hở xấu đi trong khi an toàn vòng kín tốt lên, một lần nữa cho thấy L2 không phải thước đo an toàn. Tuy vậy, PercMLP+ **không tốt hơn** IDM, chỉ **không kém hơn** trong biên cho phép. Trung bình nó đâm 0,30 % so với 0,14 % của IDM, lưới 0,3 % so với 0,0 %, phản ứng 7,7 % so với 7,4 %, và ở 20 m/s là 0,1..1,1 % so với 0,0 %. Nó học từ chính IDM nên khó vượt thầy. Tôi không thay IDM ngay. Trước khi quyết định, cần chạy ở chế độ shadow bên cạnh IDM, thêm họ kịch bản (xe dẫn phanh gấp, nhiều tốc độ cruise), dùng tập log lớn hơn để thu hẹp CI, và chỉ phát hành nếu có lợi ích rõ ràng ngoài an toàn (ví dụ độ êm, giống người).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Thế giới mô phỏng là giả định của chúng ta.** Xe là chất điểm 1 chiều, gia tốc giới hạn 3 / 9 m/s², không trễ chấp hành, không mặt đường trơn. Ngay cả "va chạm giả" ở câu 6 cũng phụ thuộc vào một tài xế IDM với T = 1,4 s. Nếu mô hình thế giới sai, cả 7 quy tắc đều đo sai. (2) **Không có cảm biến thật.** Khoảng cách và tốc độ xe dẫn được cho gần như chính xác; phép thử nhiễu chỉ là Gauss (±1 m, ±0,5 m/s, PercMLP+ 0,2..0,4 %). Các lỗi thật như mất dấu, nhận nhầm xe hay trễ khung hình (583 khung trễ ở câu 2) chưa được kiểm thử trong vòng kín. (3) **Xe dẫn trong log không phản ứng, và chỉ có chiều dọc.** Không có chuyển làn, người đi bộ hay giao lộ, nên bài lab không nói gì về các tình huống đó. (4) **Độ phủ và cỡ mẫu.** 2883 quỹ đạo test chỉ cho CI trên khoảng 0,6 %, quá thô để chứng nhận các sự kiện hiếm cỡ 1e−6 mỗi giờ. Lưới chỉ ở cruise 12 m/s, và dữ liệu là đô thị (p99 tốc độ xe dẫn 15,6 m/s). (5) **IDM vừa là thầy vừa là baseline**, nên "không kém hơn IDM" mang tính vòng tròn; ngưỡng cổng cũng chỉ là lựa chọn của đội. Bước tiếp theo: dùng mô phỏng có cảm biến và động lực học xe, cho agent phản ứng ở mọi xe, thêm kịch bản 2D, tăng cỡ mẫu, chạy shadow mode / HIL trên xe thật, rồi mới xét phát hành.
