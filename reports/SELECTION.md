# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: 

Bảng 5 frame ưu tiên (nếu chỉ đủ ngân sách rà 5 ảnh):

Thứ tự	File	t (s)	Score	U (uncertainty)	Số box
1	frame_0182.jpg	72.8	0.959	0.918	28
2	frame_0369.jpg	147.6	0.932	0.932	43
3	frame_0380.jpg	152.0	0.917	0.934	40
4	frame_0326.jpg	130.4	0.916	0.931	39
5	frame_0331.jpg	132.4	0.915	0.831	47

Lí do em chọn rà các ảnh trên là dựa vào số box nhiều/phức tạp. Cũng vì là cũng tập có uncertainty cao nhất
Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: 
frame_0369.jpg (t=147.6s, score=0.932, U=0.932, 43 box) — [ nhiều xe chồng lấn lên nhau kèm với ở góc xa, ở gần trung tâm]
frame_0331.jpg (t=132.4s, score=0.915, U=0.831, 47 box — nhiều box nhất trong 12 ảnh) — [1 số xe có bật xin nhan làm khó cho việc đặt bbox hợp lý]
frame_0099.jpg (t=39.6s, score=0.906, U=0.946, 29 box) — [1 số xe đằng xa góc trên tay trái che khuất đi xe đi trước lộ 1 chút ở đuôi xe, phần đèn]

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: 
Bằng chứng khách quan về ảnh gần trùng (near-duplicate):

frame_0372.jpg (rank 6, t=148.8s, score=0.910) — không được chọn dù điểm rất cao, vì chỉ cách frame_0369.jpg (đã chọn, t=147.6s) đúng 1.2 giây.
frame_0368.jpg (rank 9, t=147.2s, score=0.900) — cũng không chọn, cách frame_0369.jpg chỉ 0.4 giây.
frame_0330.jpg (rank 12, t=132.0s, score=0.890) — không chọn, nằm giữa frame_0326.jpg (130.4s) và frame_0331.jpg (132.4s), cả hai đều đã được chọn.

Lý do: những frame này quay cùng một khoảnh khắc giao thông với frame đã chọn (cách nhau dưới 1.5 giây), nên gần như trùng nội dung — rà thêm sẽ tốn công mà không thêm nhiều thông tin mới cho model.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: 
Chiến lược "uncertainty" chỉ chọn ảnh model không chắc chắn, không đảm bảo đó là ảnh model sai nhiều nhất hay đại diện cho toàn bộ tập dữ liệu; điểm uncertainty cao có thể chỉ do ảnh có quá nhiều xe/box (n_boxes lớn) chứ chưa chắc model dự đoán sai.
