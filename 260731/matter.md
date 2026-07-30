# Báo cáo: Tìm hiểu khả năng dùng matter.js mô phỏng Leanbot đẩy object

- Branch: `digital_twins_pythaverse_v1.1-qa`
- Thư mục kết quả: `digital-twins/client/public/matter-lab/` (chưa commit)
- Thư viện: matter.js **0.20.0**, vendor sẵn trong repo
- Kết luận ngắn: **Khả thi**, chi phí CPU không đáng kể, có 4 điểm cần chốt trước khi tích hợp

---

#### 1. Kết quả bàn giao

- Một webpage **độc lập hoàn toàn** với ứng dụng chính: không module nào của app import vào đây, và trang này cũng không gọi API nào của app.
- Mở tại `/matter-lab/index.html` trên server đang chạy.

| File | Vai trò |
|---|---|
| `sim-core.js` | Toàn bộ phần vật lý, **không đụng DOM** — nhờ vậy chạy lại được bằng Node để đo |
| `lab.js` | Renderer, bàn phím, chuột, số liệu hiển thị |
| `index.html` | Giao diện + panel chỉnh tham số |
| `probe.node.js` | Kịch bản đo tự động: `node probe.node.js` |
| `vendor/matter.min.js` | matter.js 0.20.0 |
| `README.md` | Tóm tắt thiết lập, số đo, kết luận |

- **Vì sao vendor thư viện thay vì dùng CDN:** server đặt CSP `script-src 'self'`, mọi script từ CDN đều bị chặn. Cách này cũng đúng với thông lệ sẵn có của repo (thư viện three.js cũng nằm trong `public/common/js/lib/`).

---

#### 2. Thiết lập 2D top-down

##### 2.1. Không trọng lực
```js
engine.gravity.x = 0;
engine.gravity.y = 0;
engine.gravity.scale = 0;
```
- `sim-core.js:57-59`
- Đặt cả `gravity.scale = 0` chứ không chỉ hai trục, để chắc chắn không còn thành phần trọng lực nào tác động.

##### 2.2. Không quán tính xoay cho robot
```js
inertia: Infinity,
```
- `sim-core.js:85`
- Va chạm với khối **không** làm robot tự xoay: hướng robot chỉ do lệnh điều khiển quyết định, đúng như Leanbot thật quay bằng hai bánh chứ không bị khối đẩy lệch.

##### 2.3. Robot điều khiển bằng đặt vận tốc, không dùng lực
```js
Body.setVelocity(robot, {
  x: Math.cos(heading) * forwardPx,
  y: Math.sin(heading) * forwardPx,
});
```
- `sim-core.js:135-146`
- Lý do không dùng `applyForce`: robot phải giữ **đúng** tốc độ lệnh và **buông phím là dừng ngay**, giống hành vi tắt động cơ. Dùng lực thì robot còn trôi theo quán tính.
- Khối vẫn nhận đủ xung lực va chạm nên vẫn bị đẩy đi bình thường.

##### 2.4. Robot nặng hơn khối
- `density` robot `0.05` so với khối `0.001` — nặng gấp ~50 lần, nên đẩy được cả cụm khối mà không bị dội ngược.
- `sim-core.js:33-45`

##### 2.5. Kích thước
| Vật thể | Kích thước | Nguồn |
|---|---|---|
| Leanbot | 170 × 110 mm | frame base `createBox(85*2, …, 56)` trong `createbot.mjs`, cộng phần bánh |
| Khối | **25 × 15 mm** | đúng footprint khối SynapseCity hiện tại |
| Sa bàn | 750 × 750 mm | đủ rộng để thử đẩy, gốc toạ độ ở tâm giống hệ toạ độ sim |

- `sim-core.js:29-45`

##### 2.6. Thang đơn vị
- Vật lý chạy ở thang **pixel** (`pxPerMm = 1.2`), còn API public của `sim-core.js` nhận và trả **mm + độ**, giống contract direct-message của Digital Twin.
- Lý do: các hằng số mặc định của matter.js (`slop`, giới hạn vận tốc) được tinh chỉnh theo thang pixel; chạy thẳng ở mm cho kết quả sai.

---

#### 3. Điều khiển bằng bàn phím + chuột

| Phím | Tác dụng |
|---|---|
| `W` / `↑` | tiến |
| `S` / `↓` | lùi |
| `A` `D` / `←` `→` | xoay tại chỗ |
| `Shift` | đi chậm 35% (căn khối cho chính xác) |
| `Space` | phanh |
| Chuột | kéo robot hoặc khối |

- Đọc phím: `lab.js:96-104`; áp lệnh lái mỗi bước vật lý: `lab.js:189-198`
- Chuột dùng `MouseConstraint` như demo `#mixed`: `lab.js:106-115`
- Một chi tiết phải xử lý: gốc toạ độ world nằm ở **tâm** sa bàn còn chuột đọc theo góc trên-trái canvas, nên phải bù `Mouse.setOffset` theo `render.bounds`; thiếu bước này thì kéo chuột sẽ "bắt" nhầm vật thể cách xa con trỏ.

- Panel bên phải chỉnh **trực tiếp khi đang chạy**: tốc độ tiến, tốc độ xoay, ma sát sàn, ma sát tiếp xúc (`lab.js:151-175`), kèm số liệu FPS, số khối đang chuyển động, pose robot và thời gian khối dừng sau khi buông phím.

---

#### 4. Yêu cầu "ngừng đẩy thì khối dừng lại"

- Đây là điểm mất công nhất và cũng là phát hiện đáng chú ý nhất.
- Chỉ đặt `frictionAir` là **chưa đủ**: vận tốc giảm theo cấp số nhân nên khối trôi mỗi lúc một chậm nhưng không bao giờ về 0 — nhìn trên màn hình là khối cứ nhích mãi.
- Giải pháp: `frictionAir = 0.35` (mô phỏng ma sát mặt sàn) **cộng** ngưỡng ép vận tốc về 0 khi đã đủ chậm:
```js
if (block.speed < cfg.stopSpeed && block.angularSpeed < cfg.stopAngularSpeed) {
  Body.setVelocity(block, { x: 0, y: 0 });
  Body.setAngularVelocity(block, 0);
}
```
- `sim-core.js:148-160`, ngưỡng khai báo tại `sim-core.js:47-48`
- Kết quả đo: khối dừng hẳn sau **150 ms** kể từ lúc ngừng đẩy, và khối không ai đụng thì trôi **0.000 mm** sau 3 giây.

---

#### 5. Kiểm chứng bằng số đo, không nhìn bằng mắt

- Phần vật lý tách khỏi DOM nên chạy lại được bằng Node với chính file matter.js đã vendor: `node probe.node.js`.
- Chạy trên matter.js 0.20.0, sa bàn 750 mm, bước 1/60 s:

| Phép đo | Kết quả |
|---|---|
| Đẩy 1 khối trong 1.5 s ở 260 mm/s | khối dịch **174.9 mm** |
| Ngừng đẩy → khối dừng hẳn | **150 ms** |
| Đẩy cụm 6 khối | robot đi 295.8 mm, **4/6 khối** bị đẩy (2 khối rìa trượt ra ngoài mép robot) |
| Khối không ai đụng, sau 3 s | trôi **0.000 mm** |
| 12 khối | **0.01 ms** mỗi bước |
| 60 khối | **0.10 ms** mỗi bước |
| 200 khối | **0.34 ms** mỗi bước |

- Ngân sách một khung hình 60 fps là 16.7 ms, nên 200 khối mới dùng khoảng **2%**.

##### 5.1. Ngưỡng tốc độ trước khi lọt vật thể

| Tốc độ robot | Kết quả |
|---|---|
| 200 – 2000 mm/s | ok, không lọt |
| 3000 mm/s | khối bị ép lọt qua tường |

- Leanbot thật chạy khoảng 200–400 mm/s nên biên an toàn rất rộng.

---

#### 6. Kết luận và các điểm cần chốt

**Khả thi.** matter.js xử lý đúng bài toán đẩy khối top-down, đáp ứng đủ các yêu cầu của task, chi phí CPU không đáng kể.

Bốn điểm cần lưu ý nếu quyết định đưa vào sản phẩm:

1. **Phải có ngưỡng dừng, không chỉ ma sát** (mục 4). Đây là chi tiết dễ bỏ sót nhất khi triển khai.
2. **Phải có lớp quy đổi đơn vị** giữa mm của Digital Twin và thang pixel mà matter.js được tinh chỉnh theo.
3. **Nếu cho phép `speedUp` lớn** thì phải chia nhỏ bước thời gian, không tăng quãng đường mỗi bước, nếu không sẽ lọt vật thể.
4. **Cần chốt nguồn sự thật của pose khối.** Hiện Digital Twin nhận pose khối từ Python qua message `object/move`. Nếu dùng matter.js thì vị trí khối do vật lý quyết định, khác hẳn mô hình hiện tại — đây là quyết định kiến trúc, cần chốt trước khi viết code tích hợp. Ba hướng có thể cân nhắc:
	- Vật lý chạy trong Digital Twin, Python chỉ gửi pose robot.
	- Vật lý chạy trong runner Python-side, vẫn gửi `object/move` như hiện nay.
	- Vật lý chỉ dùng cho một số mission, khai báo trong scene profile.

##### 6.1. Việc tiếp theo nếu được duyệt
- Chốt hướng ở điểm 4, sau đó mới ước lượng được khối lượng công việc tích hợp.
- Nếu muốn đưa `probe.node.js` vào CI thì cần thêm `matter-js` làm devDependency (hiện chưa đụng tới `package.json` để giữ spike tách biệt hoàn toàn).
