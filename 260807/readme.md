SynapseCity.setBlockXYA() thêm tham số duration
Commit: 5314979 · 10 file, +377 / −25 · nhánh annguyen-qa

1. Yêu cầu và kết quả
Yêu cầu	Kết quả
Thêm tham số duration = 0.0 (giây)	Đã có trên setBlockXYA() và moveObject(); mặc định 0.0
Thời gian tween từ vị trí cũ sang vị trí mới	Digital Twin nội suy vị trí + góc quay trong đúng số giây đó
Không làm delay Python code	Lệnh chỉ đẩy message vào hàng đợi rồi trả về; không thêm sleep nào

SynapseCity.setBlockXYA(SynapseCity.POLUTION_BLUE, 0, 0, -44.56)        # đặt thẳng, như cũ
SynapseCity.setBlockXYA(SynapseCity.POLUTION_BLUE, 0, 0, -44.56, 2.5)   # trượt trong 2.5 giây
Các dòng Python phía sau chạy tiếp ngay trong lúc khối vẫn đang di chuyển trên màn hình. setBlock(objId, slot) giữ nguyên đặt tức thời.

2. Định dạng message trên dây
Khoá durationS chỉ xuất hiện khi > 0:


// duration = 0 (hoặc không truyền) — payload y hệt trước khi có task này
{"objId":"POLUTION_BLUE","pose":{"xMm":0.0,"yMm":0.0,"headingDeg":-44.56}}

// duration = 2.5
{"objId":"POLUTION_BLUE","pose":{"xMm":0.0,"yMm":0.0,"headingDeg":-44.56},"durationS":2.5}
Đây là lựa chọn có chủ đích: mọi lệnh đặt tức thời — tức toàn bộ code đang chạy hôm nay — đi qua đường truyền với đúng hình dạng payload cũ, không byte nào khác.

3. Các tầng đã sửa
Tầng	File	Việc
Thư viện Python	SynapseCity.py	_move_duration_seconds() kiểm tra đầu vào; _MAX_MOVE_DURATION_S = 60.0; gắn durationS vào payload
Validator hàng đợi	python-direct-message.ts	Nhận durationS như khoá tuỳ chọn; thêm hasAllowedKeys() vì hasExactKeys cũ khoá cứng bộ khoá
Cầu sang iframe	BlocklyTestView.ts:4992	Chuyển tiếp durationS khi dựng lại payload phẳng cho postMessage
Router của DT	experience.mjs:634	Truyền xuống runtime
Runtime cảnh	MissionSceneRuntime.ts:171	Xuyên tham số
Chỗ chạy tween	DirectObjectLayer.moveObject()	Toàn bộ logic nội suy
Tween nằm trong group mặc định của @tweenjs/tween.js, vốn đã được TWEEN.update() gọi mỗi frame trong vòng animate của createbot.mjs:863 — cùng cơ chế camera đang dùng, không phải đụng vào render loop.

4. Bốn quyết định thiết kế
Lần đặt vị trí đầu tiên luôn tức thời, kể cả khi truyền duration. Trước đó khối chưa hiện (visible = false) và đang nằm ở gốc toạ độ; tween ở đây sẽ thành cảnh khối trượt từ tâm sa bàn ra chỗ đứng đầu — không phải thứ ai muốn. Trong setBlock thì create và move đi liền nhau nên luật này áp đúng chỗ.

Lệnh move mới huỷ tween cũ và đi tiếp từ chỗ khối đang đứng, không xếp hàng. Cảnh phải khớp với lệnh vừa nhận, không phải lệnh trước đó — giống cách camera xử lý khi bấm View mới giữa chừng.

Xoay theo cung ngắn nhất: từ 170 sang −170 độ là quay 20 độ, không quay ngược 340 độ. Dùng atan2(sin Δ, cos Δ).

Easing tuyến tính, khác camera (Quadratic.Out). Khối bị robot đẩy hoặc mang đi thì chạy đều theo robot; easing vào/ra sẽ làm nó trông như một vật tự tăng giảm tốc.

5. Ràng buộc và xử lý lỗi
Trần 60 giây, kiểm ở cả hai phía. Duration dài hơn thế gần như chắc chắn là lỗi đơn vị (nhầm mili-giây với giây), mà hậu quả là khối treo lơ lửng giữa đường suốt phần còn lại của bài.

Python ném ValueError với thông báo rõ ràng — lỗi hiện ngay cho người viết chương trình.
Digital Twin từ chối cả message nếu durationS không phải số hữu hạn hoặc nằm ngoài [0, 60].
Đã xác nhận 7 đầu vào hỏng đều bị chặn: -1, 61, nan, inf, 'abc', None, [1].

6. Tương thích ngược
Không có breaking change. duration là tham số có giá trị mặc định nên mọi lời gọi cũ giữ nguyên hành vi; payload của lệnh tức thời không đổi; các demo trong PyDemo/SynapseCity/ chạy y như trước.

7. Kiểm chứng
316/316 test pass (20 suite), trong đó 15 test mới: 6 cho tween ở tầng DirectObjectLayer (đi đúng quãng đường theo thời gian, xoay cung ngắn, lần đầu đặt thẳng, huỷ và nối lại giữa chừng, dừng tween khi clear run, từ chối duration hỏng) và 9 cho validator.
tsc --noEmit sạch.
Phía Python: chạy trực tiếp module với bridge giả, xác nhận payload không-duration giữ nguyên dạng cũ, duration=2.5 sinh "durationS":2.5, duration=0 không sinh khoá, setBlock vẫn tức thời.
Lưu ý về giới hạn kiểm chứng: experience.mjs là JS thuần không được type-check và không có test tự động, nên một dòng sửa ở file này chỉ được xác nhận bằng đọc code.

8. Ngoài phạm vi
Chưa cập nhật các demo trong PyDemo/SynapseCity/ — 2.Neutralize.py vẫn đặt khối tức thời khi đẩy và khi thả. Đây là chỗ dùng tween tự nhiên nhất nếu muốn demo đẹp hơn.
setBlock(objId, slot) chưa có duration — chỉ setBlockXYA và moveObject có, đúng theo yêu cầu.
Thay đổi hạ tầng test: để spec import được @tweenjs/tween.js (gói này là dependency của client/, không phải của digital-twins/), đã thêm một paths trong tsconfig.json và một moduleNameMapper trong package.json — cùng lý do và cùng kiểu với alias @/ vốn đã có sẵn ở đó.
