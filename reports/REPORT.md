# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Phúc Đại

Công cụ gán nhãn đã dùng: CVAT và sửa trực tiếp file nhãn

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Các frame liên tiếp trong cùng video giao thông rất giống nhau (xe gần như đứng yên qua vài frame liền kề). Nếu chia ngẫu nhiên, một frame gần như giống hệt
ảnh đã dùng để train có thể lọt vào tập test → model "nhận diện lại" thay vì dự đoán trên dữ liệu
mới thật sự → điểm AP50/Recall trên test bị TĂNG ẢO (lạc quan giả), không phản ánh đúng khả năng
tổng quát hoá. Vùng đệm ở giữa để đảm bảo khoảng cách thời gian đủ xa, tránh rò rỉ dữ liệu (data
leakage) ở ngay biên giữa 2 tập.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---|---|---|---|---|---|---|---|---|---|---|
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Số liệu cho thấy rõ: **Recall xe nhỏ (0.182) thấp hơn nhiều so với xe vừa (0.547) và xe lớn (0.561)**
— model cold start (huấn luyện gốc trên COCO, chưa thấy ảnh giao thông đêm nào) bỏ sót phần lớn xe
nhỏ/xa trong khung hình.

Dựa vào `outputs/compare_round0.jpg`, model không khớp nhãn tham chiếu ở những loại xe nào?

Cụ thể: xe ở xa/nhỏ, xe bị che khuất, xe ở rìa ảnh, xe chỉ thấy đèn hậu

Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Ví dụ: một box FP (khung đỏ) có thể thực ra là xe thật nhưng nhãn tham chiếu
(do model khác tạo, chưa người rà) bị thiếu sót, không phải do cold start đoán sai

## 3. Chiến lược chọn mẫu

Công thức `score = W_U·U + W_A·A + W_D·D`:
- **U** (Uncertainty): độ bất định của model khi dự đoán trên ảnh đó — ảnh có nhiều box mà model
  "không chắc chắn" (confidence gần 0.5) sẽ có U cao.
- **A**: [Bạn xem lại đúng ý nghĩa cột A trong file cấu hình notebook nếu cần, thường liên quan đến
  mật độ/số lượng đối tượng ambiguous trong ảnh]
- **D**: liên quan đến độ đa dạng/khoảng cách so với các ảnh đã chọn trước đó (giúp tránh chọn
  toàn ảnh giống nhau).
- **MIN_GAP_S**: khoảng thời gian tối thiểu (tính bằng giây) giữa 2 frame được chọn — dùng để lọc
  bớt ảnh gần trùng (near-duplicate), vì các frame quay cách nhau dưới 1-2 giây gần như giống hệt
  nhau về nội dung xe cộ.

Ba frame từ `reports/SELECTION.md` và một frame khác, minh hoạ cách cân nhắc uncertainty/ảnh gần
trùng/công gán nhãn:
frame_0369.jpg (t=147.6s, score=0.932, U=0.932, 43 box) — [ nhiều xe chồng lấn lên nhau kèm với ở góc xa, ở gần trung tâm]
frame_0331.jpg (t=132.4s, score=0.915, U=0.831, 47 box — nhiều box nhất trong 12 ảnh) — [1 số xe có bật xin nhan làm khó cho việc đặt bbox hợp lý]
frame_0099.jpg (t=39.6s, score=0.906, U=0.946, 29 box) — [1 số xe đằng xa góc trên tay trái che khuất đi xe đi trước lộ 1 chút ở đuôi xe, phần đèn]

Điểm bất định (uncertainty) có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?

KHÔNG chắc chắn. Uncertainty cao chỉ cho biết model "không tự tin" ở ảnh
đó (có thể do ảnh nhiễu, ánh sáng khó, nhiều xe chồng lấn), không đảm bảo rằng gán nhãn thêm ảnh đó
sẽ giúp model học được điều mới — nếu độ khó đến từ chất lượng ảnh (mờ, quá tối) chứ không phải từ
thiếu dữ liệu loại xe đó, model có thể vẫn không cải thiện được nhiều dù đã học thêm ảnh này

## 4. Các vòng học chủ động (active learning)

Bảng đầy đủ từ `rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 0 | yolov8n cold start | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 346 | 0.574 | **-0.197** | 1.000 | 0.097 | 0.176 | 0.000 | 0.112 | 0.146 |

**Mức độ sửa nhãn gợi ý ở vòng 1** (từ `outputs/round1_diff.md`, đã in ra khi chạy `pack_labels.py`):
- Model đề xuất ban đầu: **169 box**
- Giữ nguyên (accepted): **78**
- Sửa lại (edited): **67**
- Xoá bỏ (deleted): **24**
- Thêm mới hoàn toàn (added): **201**
- Tổng nhãn cuối: **346 box** trên 12 ảnh

→ Hơn 58% tổng số box cuối cùng (201/346) là do bạn **tự thêm mới hoàn toàn** — model pre-label ban
đầu bỏ sót rất nhiều xe.

**AP50 thay đổi:** giảm **0.197 điểm** (từ 0.771 xuống 0.574) so với cold start — **không cải thiện,
mà tệ đi rõ rệt**.

**Nhóm xe theo Recall:** cả 3 nhóm đều giảm mạnh, đặc biệt:
- Recall xe nhỏ: **0.182 → 0.000** (giảm 100%, model không còn phát hiện được xe nhỏ nào)
- Recall xe vừa: 0.547 → 0.112 (giảm ~80%)
- Recall xe lớn: 0.561 → 0.146 (giảm ~74%)

Dựa vào `outputs/compare_round1.jpg`: đối chiếu 4 frame mẫu (0050, 0150, 0250, 0350), model vòng 1
đều cho **FP = 0** nhưng **FN tăng vọt** so với cold start (ví dụ frame_0050: cold start FN=7 →
round1 FN=16; frame_0350: cold start FN=14 → round1 FN tăng theo cùng xu hướng) — mô hình trở nên
**quá thận trọng, gần như không dám đoán box nào**, khớp đúng với Precision=1.0 nhưng Recall cực
thấp trong bảng số liệu.

Chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu đi), cùng lý do có thể kiểm:

[reference frame 0250 compare_round0.jpg/compare_round1.jpg, mô tả rõ: ở vị
trí trung tâm làn phải trong ảnh, cold start đoán đúngvị trí xe nhưng sai kích thước. Nguyên nhân:
 model chỉ học từ 12 ảnh/346 box — quá ít so với việc fine-tune 50 epoch, có thể
gây overfit nặng vào đúng 12 ảnh đó và "quên" kiến thức tổng quát ban đầu học từ COCO (hiện tượng
catastrophic forgetting)]

Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt quan sát độc lập, lỗi pre-label
đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline:

round	frame_id	object	action	rule_or_reason
1	frame_0099.jpg	xe oto g?n mép trái gi?a màn hình	Edited	bbox du?c gán tràn ra ngoài
2	frame_0331	xe oto v?n có th? th?y du?c ? trên cùng bên trái	added	Th?y thân xe và d? ranh gi?i theo GUIDELINE_LABEL.md
2	frame_0380	xe oto v?n có th? th?y du?c ? trên cùng bên trái	added	Th?y thân xe và d? ranh gi?i theo GUIDELINE_LABEL.md


## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục?

AP50 và Recall đều giảm mạnh sau vòng 1, đặc biệt Recall xe nhỏ
về 0. Bạn có thể lập luận: kết quả này cho thấy fine-tune trên tập quá nhỏ (12 ảnh) với epoch=50 có
thể đã gây overfit/catastrophic forgetting, cần cân nhắc trước khi tiếp tục vòng 2 — ví dụ giảm số
epoch, hoặc gán nhãn thêm nhiều ảnh hơn trước khi train lại

Đề xuất hai ca còn yếu hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng:

Xe nhỏ ở xa chỉ thấy đèn đỏ đuôi (frame_0150): Chi phí rà cao vì mỗi ảnh có nhiều xe nhỏ sát nhau
Xe bị cắt ở mép ảnh (frame_0250): cả cold start và vòng 1 đều sót nhưng rà được nhanh

Tập kiểm thử chỉ 20 ảnh, có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà
thủ công; các giới hạn đó ảnh hưởng thế nào đến kết luận?

20 ảnh là mẫu rất nhỏ, một vài case sai có thể làm lệch AP50 đáng kể; nhãn
tham chiếu do model khác tạo (chưa người rà) có thể tự nó đã sai/sót ở một số xe, khiến điểm
Precision/Recall không phản ánh chính xác 100% hiệu năng thật; luật bỏ 14 box quá nhỏ (<16px) làm
bộ nhãn tham chiếu không đại diện đầy đủ cho toàn bộ số xe thật trong ảnh

Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

 (1) xem lại nhãn round1 mới thêm có lỗi hệ thống nào không (VD toạ độ box sai, nhầm class)
 (2) kiểm tra learning rate/epoch có quá cao gây overfit không 
 (3) so sánh trực quan thêm nhiều frame trong compare_round1.jpg để xác nhận xu hướng "quá
thận trọng" có nhất quán không hay chỉ ở vài ảnh