# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Phạm Thị Ngọc Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602831 |
| GitHub | ngocanhpham-hust |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 7014 / model seeds 9266, 9267, 9268, 9269, 9270 (cruise speed 12.0 m/s) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.246 | 20.3 (CI95 [18.9, 21.8]) | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1.522 | 0.14 (CI95 [0.05, 0.36]) | (baseline) |
| EgoMLP | 0.757 .. 0.761 (TB 0.760 ± 0.002) | 21.0 .. 31.8 (seed xấu nhất CI95 [30.1, 33.5]) | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP | 0.615 .. 0.621 (TB 0.619 ± 0.003) | 0.2 .. 0.7 (seed xấu nhất CI95 [0.5, 1.1]) | BLOCK (rớt R3, R4, R6) |
| PercMLP+ (sau khi sửa) | TB 0.825 ± 0.026 | 0.2 .. 0.2 (0.21 %, CI95 [0.10, 0.45]) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 phát hiện ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan tới gia tốc thật của xe dẫn: tương quan với đạo hàm vận tốc `acc_lead_column_corr_with_dv_dt` = 0.045, trong khi cột của xe theo sau đạt 0.929. Vì vậy mình bỏ `Acc_LV` và dùng `a_lead` tính từ `Speed_LV` đã làm mượt (phân vị p0.1/p1 của gia tốc làm mượt là −2.77 / −1.95 m/s²). (2) Có quỹ đạo trùng lặp tuyệt đối ở các ID cách xa nhau: 9993 quỹ đạo thô, loại 383 bản trùng, còn 9610. (3) `Spatial_Gap` bằng hiệu vị trí trừ chiều dài xe 4.5 m (sai lệch tối đa 3.6e-11 m), nên phải hiểu đúng định nghĩa gap trước khi tính TTC. Chia train/test theo quỹ đạo chứ không theo cửa sổ, vì các cửa sổ 0.1 s liên tiếp của cùng một quỹ đạo (hoặc của bản trùng ở ID khác) gần như giống hệt nhau. Chia theo cửa sổ sẽ để lộ dữ liệu test vào train, làm điểm test đẹp giả tạo.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Trên 600 đoạn log: chạy lại cùng mã cho 0/600 đoạn khác. Bản refactor float32 đổi thứ tự phép toán khác bit ở cả 600/600 đoạn, nhưng 0/600 đoạn vượt dung sai (max diff 1.2e-04 so với atol = rtol = 1e-3). Đây là lý do phải so có dung sai: phép `==` sẽ báo lỗi giả ở 100 % đoạn. Thay đổi AEB (phanh −6 m/s² khi TTC < 1.5 s) chỉ làm lệch 10/600 đoạn (62, 77, 84, 244, 333, …). Mở đoạn 62 ra xem: chỉ 4 khung 98–101 bị lệch, đúng các khung có TTC 1.43–1.49 s < 1.5 s, còn các khung khác đều có TTC ≥ 1.52 s. Lệnh IDM tại đó là −5.44 .. −5.91 m/s², bị AEB kẹp xuống −6.0. Đoạn 77 cũng vậy: các khung 73–79 có TTC 1.35–1.44 s. Phát nhanh nhất cho 0 khung trễ và 0 khung bị bỏ vì không có khái niệm hạn chót. Phát theo thời gian thật làm lộ ra 585 khung trễ và 578 khung bị bỏ trên 59952 khung, khiến 385/600 đoạn có lệnh thay đổi. Như vậy lỗi trễ hạn chỉ hiện ra khi phát lại theo thời gian thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của mình: cruise 12.0 m/s, khoảng cách 2.76–59.31 m (p2..p98 của log), tốc độ tiến gần 0–12 m/s, 60 × 45 = 2700 ô. Có 2.2 % số ô không bộ điều khiển nào tránh được khi t_react = 0 s, và 7.4 % khi t_react = 0.6 s. Bộ mô phỏng phanh hết cỡ khớp đúng với biên `unavoidable` (0 ô lệch ở xa biên, ở cả hai độ trễ). Vùng đâm của IDM tăng từ 2.2 % lên 7.4 % khi độ trễ tăng 0 → 0.6 s, nhưng diện tích "tránh được mà vẫn đâm" của IDM là 0.0 % ở cả hai độ trễ. Nghĩa là mọi va chạm của IDM đều nằm trong vùng ai cũng đâm, không phải lỗi của bộ điều khiển. Log không phủ vùng khó: mức phanh mạnh nhất của người lái (phân vị 99.9) chỉ là 3.6 m/s², trong khi cơ cấu chấp hành phanh được 9 m/s², và 8.6 % lưới đòi phanh mạnh hơn mức đó. Khai thác được 614 sự kiện xe dẫn phanh (6.4 % số quỹ đạo, đỉnh p99 = −4.2 m/s²), nhưng gần như không có tình huống cắt đầu ở cự ly gần.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| const-vel | 1.246 | 3 | 20.3 % | 3 |
| IDM | 1.522 | 4 | 0.1 % | 1 |
| EgoMLP | 0.760 | 2 | 26.9 % | 4 |
| PercMLP | 0.619 | 1 | 0.5 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất vì nó không bắt chước người: tham số T = 1.2 s của nó khác thói quen của người lái (headway trung vị 1.86 s), nên dự báo 3 s lệch với quỹ đạo thật. Nhưng IDM là một vòng phản hồi trên gap và tốc độ tiến gần: khoảng cách co lại thì nó phanh. Vì thế trong vòng kín nó tự sửa sai số của chính mình và chỉ đâm 0.14 %. EgoMLP chỉ nhìn lịch sử tốc độ của chính xe, không thấy gap hay xe dẫn. Nó đạt L2 0.760 nhờ ngoại suy mượt hồ sơ tốc độ của người, và ở vòng hở lịch sử đó vốn đã chứa phản ứng của người với xe dẫn. Trong vòng kín, lịch sử ấy là của chính nó, nên sai số tích luỹ mà không có tín hiệu nào kéo về. Kết quả là va chạm 21.0..31.8 %, và L2 trên lát cắt phanh tăng lên 1.050.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động từ 21.0 % đến 31.8 % va chạm qua 5 seed (21.0, 24.9, 25.5, 31.3, 31.8 %), tức dải 10.8 điểm phần trăm, dù L2 vòng hở gần như không đổi (0.757..0.761). Khoảng tin cậy Wilson của seed xấu nhất là [30.1, 33.5] %, chỉ rộng khoảng 3.4 điểm với n = 2883 quỹ đạo. Như vậy độ bất định do số mẫu nhỏ hơn nhiều so với độ bất định do seed huấn luyện. Nếu chỉ chạy đúng một seed, có thể báo 21.0 % hoặc 31.8 %, và mỗi con số đi kèm một CI hẹp trông rất "chắc chắn" nhưng không chứa con số của seed kia. Điều đó nguy hiểm vì vô tình chọn được seed đẹp sẽ làm mô hình trông an toàn hơn thực tế. Ngay cả PercMLP cũng dao động 0.2..0.7 %, ngang cỡ biên 1 điểm của quy tắc R2.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Hiệu chỉnh tài xế IDM xe sau bằng khoảng cách Wasserstein-1 giữa phân phối headway mô phỏng và log: T = 1.4 s cho W1 = 0.36, gần với người thật (headway trung vị 1.86 s). Một xe không phản ứng có W1 = 0.88, kém chân thực hơn rõ rệt. Bảng t_react = 0.6 s (tỷ lệ lưới có va chạm bất kỳ, kèm va chạm sau replay / react):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau (replay / react) |
|---|---|---|---|
| const-vel | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| IDM | 75.7 % | 7.4 % | 68.3 % / 0.0 % |
| EgoMLP | 79.4 % | 79.4 % | 0.0 % / 0.0 % |
| PercMLP | 78.9 % | 23.5 % | 56.2 % / 0.0 % |

Khi xe sau chỉ phát lại tốc độ trong log, nó không biết xe mình vừa phanh để tránh xe cắt đầu, nên tông vào đuôi. IDM phanh đúng nên lại bị phạt nặng nhất (68.3 % va chạm sau). Đó là "va chạm giả" do thế giới không phản ứng. Với xe sau phản ứng, va chạm sau về 0.0 % và IDM chỉ còn 7.4 %, bằng đúng vùng bất khả thi. Nếu dùng thế giới phát lại, IDM (75.7 %) trông gần tệ như const-vel (77.6 %), dẫn tới kết luận sai rằng phanh là xấu. Thế giới phản ứng đảo lại kết luận đó.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi ba quy tắc. R3: va chạm trên lát cắt phanh 2.7 % (n = 182, cận trên 6.3 %), lớn hơn ngưỡng 1.1 % + 5 điểm. R4: diện tích tránh được mà vẫn đâm 16.1 % lưới, so với 0.0 % + 2 điểm của IDM. R6: 23.5 % lưới, so với 7.4 % + 2 điểm. R2 (0.73 %, cận trên 1.11 % ≤ 0.14 % + 1) và R5 (0.2..0.7 %) vẫn qua với seed của mình. Nguyên nhân gốc: log tự nhiên gần như không có cắt đầu cự ly gần hay phanh mạnh (8.6 % lưới vượt mức phanh 3.6 m/s² của người, lát cắt phanh chỉ chiếm 4.2 % cửa sổ). Ngoài ra mô hình chỉ học trên trạng thái người đã đi qua, nên dễ trôi khỏi phân phối đó (covariate shift). Mình sửa bằng 197220 cửa sổ từ 20000 tình huống cắt đầu mô phỏng (chữa vùng log thiếu) cộng 2 vòng DAgger với khoảng 892k–895k cửa sổ, trong đó IDM gán nhãn tại chính các trạng thái mô hình tự đi vào (chữa covariate shift). PercMLP+ qua cả 7 quy tắc: R2 0.21 % (cận trên 0.45 %), R4 0.1 %, R6 7.5 % so với 7.4 %. Tuy vậy PercMLP+ **không tốt hơn** IDM, mà chỉ **không kém hơn trong biên cho phép**: IDM vẫn bằng hoặc tốt hơn ở mọi chỉ số (0.14 % so với 0.21 %, 0.0 % so với 0.1 %). Ở cruise 20 m/s chưa từng thấy khi huấn luyện, PercMLP+ còn đâm 0.1..3.3 % lưới trong vùng tránh được, so với 0.0 % của IDM; seed xấu nhất đã vượt biên 2 điểm. Vì vậy mình chưa phát hành. Trước khi quyết định nên chạy chế độ shadow, thêm lưới ở nhiều tốc độ cruise vào cổng, và chứng minh một lợi ích cụ thể so với IDM.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) Thế giới chỉ có một chiều dọc và xe dẫn phát lại log, không phản ứng. Không có chuyển làn, người đi bộ hay giao lộ, nên 0.21 % va chạm chỉ đúng cho bài toán bám xe. Đây không phải bằng chứng an toàn cho cả một hệ lái. (2) Chuyên gia gán nhãn là IDM, và cổng cũng so với IDM. PercMLP+ học bắt chước IDM nên tối đa chỉ "không kém" IDM, và mọi điểm mù của IDM (tham số T = 1.2 s, cảm biến hoàn hảo) cũng được kế thừa. (3) Phân phối dữ liệu hẹp: một tập Argoverse 2 đô thị, tốc độ xe theo sau p99 = 17.8 m/s. Thử ở 20 m/s, PercMLP+ đã đâm tới 3.3 % lưới, nghĩa là kết luận không tổng quát ra ngoài vùng đã thấy. (4) Thống kê còn mỏng: IDM chỉ có 4/2883 va chạm (CI95 [0.05, 0.36] %) và lát cắt phanh chỉ có n = 182, nên các biên 1 và 5 điểm khá thô. 5 seed cũng chưa đủ để ước lượng đuôi phân phối. (5) Nhiễu cảm biến là Gaussian nhân tạo (±1 m, ±0.5 m/s), còn động lực học xe là lý tưởng (phanh tức thì tới 9 m/s²). Bước tiếp theo nên là: mô phỏng 2D có tác nhân phản ứng và nhiều họ kịch bản hơn, mô hình cảm biến thực tế, chạy shadow mode trên xe thật để so quyết định với tài xế, rồi thử trên sân thử trước khi phát hành dần.
