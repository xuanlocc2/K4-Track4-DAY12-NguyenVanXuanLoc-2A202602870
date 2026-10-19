# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Nguyễn Văn Xuân Lộc |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602870 |
| GitHub | xuanlocc2 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 7780 / model seed 6675 (các seed huấn luyện: 6675, 6676, 6677) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 21.6 | BLOCK |
| IDM | 1.526 | 0.2 | (baseline) |
| EgoMLP | 0.767 ± 0.004 | 17.2 .. 25.1 (TB 21.6) | BLOCK |
| PercMLP | 0.625 ± 0.004 | 0.7 .. 2.6 (TB 1.5) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.802 ± 0.007 | 0.24 .. 0.28 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Tập có 992.679 dòng, 9.993 quỹ đạo thô; có 383 quỹ đạo trùng tuyệt đối bị loại, còn 9.610 quỹ đạo sạch (`data_audit.json`). Các bản trùng nằm ở id cách xa nhau, nên nếu chia ngẫu nhiên theo cửa sổ hoặc theo id thì cùng một quỹ đạo có thể rơi vào cả train lẫn test và gây rò rỉ; vì vậy phải chia theo quỹ đạo (đã loại trùng trước khi chia). Cột `Acc_LV` có tương quan với d(speed)/dt chỉ 0,045 (trong khi gia tốc xe sau là 0,929), tức là cảm biến ghi sẵn gần như là nhiễu. Tôi thay bằng gia tốc xe dẫn `a_lead` tính từ `Speed_LV` đã làm mượt. Ngoài ra gap = pos_diff − chiều dài xe (lệch 4,5 m, sai số 3.6e-11) nên tính khoảng cách nhất quán.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác, 0 đoạn khác bit. Bản refactor float32 khác bit ở 600/600 đoạn nhưng 0 đoạn vượt dung sai (max diff 2.5e-05 so với atol=rtol=1e-3), nên so sánh `==` sẽ báo lỗi giả ở 100% đoạn; (0,1+0,2)+0,3 = 0,6000000000000001 ≠ 0,6 minh hoạ tính không kết hợp. Thêm AEB làm lệch 7/600 đoạn (57, 138, 204, 318, 370, 504, 573; max diff 1,59 m/s²): chỉ lệch ở những đoạn có khung xe dẫn phanh gấp làm AEB kích hoạt, còn các đoạn khác lệnh giữ nguyên. Phát theo thời gian thật lộ ra 583 khung trễ và 576 khung bị rơi (trên 59.865 khung), khiến 378/600 đoạn đổi lệnh (max diff 9,99), điều mà chế độ nhanh nhất (0 trễ, 0 rơi) che đi vì ở đó không có hạn chót.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới gồm 60 khoảng cách × 45 tốc độ tiến gần = 2.700 ô, cruise 12 m/s, gap từ 2,76 đến 59,3 m (p2..p98 của log), closing từ 0 đến 12 m/s (xe cắt đầu đứng yên). Với t_react=0 có 2,2% ô không ai tránh được; với 0,6 s là 7,4%. Vùng đâm của IDM tăng từ 2,2% lên 7,4% (gấp ~3,3 lần) và toàn bộ phần đó là bất khả kháng (attributable = 0%), mô phỏng khớp biên vật lý với 0 ô lệch ở cả hai độ trễ. Log không phủ vùng khó: người lái trong log phanh mạnh nhất chỉ 3,62 m/s² (p99.9) trong khi cơ cấu chấp hành tới 9 m/s²; 8,6% ô của lưới đòi hỏi phanh mạnh hơn thế. Khai phá được 614 sự kiện phanh gấp (6,4% quỹ đạo, peak decel p50 −2,53, p99 −4,21 m/s²).

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 (m) | hạng L2 | Va chạm vòng kín | hạng |
|---|---|---|---|---|
| const-vel | 1.253 | 3 | 21.6% | 3 |
| IDM | 1.526 | 4 | 0.2% | 1 |
| EgoMLP | 0.767 | 2 | 21.6% | 4 |
| PercMLP | 0.625 | 1 | 1.5% | 2 |

Các cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất vì L2 so quỹ đạo dự báo với hành vi con người (người giữ khoảng cách ~1,9 s, IDM giữ khoảng cách khác), nhưng IDM là luật phản hồi: mỗi bước nó nhìn gap và closing speed thật nên không tích luỹ sai số và chỉ va chạm 0,2%. EgoMLP chỉ thấy lịch sử tốc độ của chính xe mình, không thấy xe dẫn; nó dự báo trung bình tốt (L2 0,767 < 1,253 của const-vel) nhưng không có tín hiệu để phanh khi xe dẫn phanh, nên va chạm 21.6% (50%+ ở lát cắt xe dẫn phanh: 54.0%). L2 trung bình bị chi phối bởi 95% cửa sổ bình thường (lát phanh chỉ 4,9%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động từ 17,2% đến 25,1% giữa 3 seed (biên độ ~8 điểm), trong khi L2 vòng hở gần như không đổi (±0,004 m). Wilson 95% của seed xấu nhất là [23,6; 26,7]% và khoảng của từng seed không trùng nhau hoàn toàn, nghĩa là độ lệch giữa seed lớn hơn nhiễu lấy mẫu trong một seed. PercMLP cũng dao động 0,7% đến 2,6%. Nếu chạy duy nhất seed 6676 tôi có thể báo EgoMLP "25,1%" hoặc seed 6677 báo "17,2%" — hai kết luận khác nhau cho cùng một mô hình; chọn seed đẹp là cherry-picking. Vì vậy cổng có R5 (độ rộng qua seed ≤ 1 điểm) và R2 lấy seed xấu nhất.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Với t_react=0,6, tỷ lệ ô lưới có va chạm bất kỳ (trước hoặc sau): 

| Bộ điều khiển | xe sau phát lại | xe sau phản ứng | va chạm sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 77.6% | 77.6% | 0 / 0 |
| IDM | 75.7% | 7.4% | 68.3% / 0 |
| EgoMLP | 77.9% | 77.9% | 0 / 0 |
| PercMLP | 78.2% | 28.0% | 52.0% / 0 |
| PercMLP+ | 75.5% | 7.7% | 67.8% / 0 |

T hiệu chỉnh = 1,4 s (khoảng cách Wasserstein-1 giữa phân phối headway mô phỏng và log là 0,35; xe không phản ứng được 0,87; log có headway trung vị 1,89 s). Khi xe sau chỉ phát lại, nó không biết xe thử phanh gấp nên đâm đuôi: IDM bị 68,3% ô là va chạm sau giả, dù IDM phanh đúng. Khi xe sau phản ứng, va chạm sau về 0 và vùng đâm của IDM chỉ còn 7,4% (đúng bằng vùng bất khả kháng). Nếu không hiệu chỉnh, ta sẽ kết luận sai rằng IDM kém an toàn ngang các mô hình khác.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn ở R2 (worst seed 2,6%, cận trên 3,3% so với baseline 0,2% + 1 điểm), R3 (lát phanh 9,6%, cận trên 14,5% so với 1,0% + 5), R4 (diện tích tránh được nhưng đâm 20,5% so với 0,0% + 2), R5 (0,7 .. 2,6% qua seed) và R6 (28,0% so với 7,4% + 2). Nguyên nhân gốc: log tự nhiên không chứa cắt đầu ở cự ly hẹp (R4, R6), và sai số bắt chước tích luỹ trong vòng kín do covariate shift (R2, R5). Tôi sửa bằng 20.000 cửa sổ cắt đầu mô phỏng do IDM lái, cộng 2 vòng DAgger: cho mô hình tự lái trên log train, ghi các trạng thái nó tự đi vào và gán nhãn bằng dự báo 3 s của IDM tại chính các trạng thái đó. PercMLP+ qua cả 7 quy tắc: va chạm 0,24 .. 0,28%, R4 0,3%, R6 7,7% so với 7,4%. Nhưng nó chưa tốt hơn IDM (0,28% so với 0,17%, vùng lưới 7,7% so với 7,4%), chỉ không kém hơn trong biên; vì nó được dạy bắt chước IDM nên tối đa chỉ bằng IDM, và L2 vòng hở còn xấu đi (0,625 → 0,802). Chưa nên phát hành lên xe thật; trước hết cần thử trên đa dạng tốc độ (ngoài 12 m/s: ở 20 m/s PercMLP+ còn 0 .. 0,9% so với PercMLP 34 .. 39%), nhiễu cảm biến và thử trên mô phỏng độc lập.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) Phạm vi dữ liệu: chỉ 9.610 quỹ đạo Argoverse với gap, tốc độ ≤ ~18 m/s, người lái phanh tối đa 3,6 m/s²; không có phanh khẩn, đường trơn, mưa, hay tốc độ cao nên kết luận không ngoại suy được tới các vùng này. (2) Thế giới mô phỏng đơn giản: động học một chiều, xe sau là IDM với T=1,4 s hiệu chỉnh bằng headway nên chưa chắc là hành vi thật, xe dẫn chỉ phát lại log, cắt đầu là kịch bản tổng hợp (cut-in lưới 2.700 ô), và độ trễ phản ứng cố định 0 hoặc 0,6 s; mọi con số đúng chỉ trong mô phỏng này. (3) Thống kê: chỉ 3 seed và 198 quỹ đạo lát phanh (cận trên Wilson 3,6% ngay cả khi đạt 1,0%), nên chưa chứng minh được "an toàn hơn IDM" và các ngưỡng 1/2/5 điểm là quy ước, không phải bằng chứng rủi ro thực. (4) PercMLP+ học nhãn từ IDM nên bị chặn trên bởi IDM. Cần làm tiếp: thu thêm dữ liệu cảnh khó, mô phỏng đa tác nhân và đa tốc độ, nhiễu và trễ cảm biến, tăng số seed, chạy shadow mode trên xe thật trước khi phát hành.
