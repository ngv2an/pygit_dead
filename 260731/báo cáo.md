# Báo cáo triển khai 7 task

- Branch: `annguyen-qa`
- Commit cuối: `acb6af51a`
- Phạm vi: `eccf24abc` → `acb6af51a` (10 commit)
- Kết quả kiểm thử: **287/287 unit test pass** (20 suite), lint 0 error

---

#### 1. SYNAPSE_CITY_SCALE — gắn thông tin Scale vào FIELD object

##### 1.1. Đã làm
- Mỗi FIELD tự khai hình học của mình: `map`, `width`, `height`, `sizeMm`.
- Scale **không gõ tay** mà suy ra từ ảnh đã giải mã: `scale = mapPixels / sizeMm`.
- FIELD nào không khai hoặc khai sai thì rơi về `DEFAULT_FIELD_SCALE = 1.0`.
- Mỗi FIELD giữ trạng thái giải mã riêng (`status`, `gray`, `width`, `height`, `scale`), nạp map độc lập, cache riêng.
- Bỏ hằng `PRELOADED_FIELD` (trước đây chỉ hỗ trợ đúng 1 mission). Thêm mission mới giờ là **thay đổi dữ liệu**: thêm một dòng vào `FIELD_DEFS` + thả file map, không sửa logic.
- Map hỏng/thiếu của mission này không còn làm chết cảm biến của mission khác.
- Link code github:
	- Bảng FIELD_DEFS + default scale: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L312-L321
	- Trạng thái từng FIELD: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L326-L345
	- Hàm tính scale: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L351-L356
	- Nạp map theo từng FIELD: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L428-L472

##### 1.2. Số liệu thực tế khác với ví dụ trong task — cần xác nhận
- Task ghi `Scale = 1200 / 1220`, nhưng giá trị đúng hiện nay là **1.0**:
	- File map là `SynapseCity_1220x1220_gray.png`, đọc PNG header xác nhận **1220 × 1220 px**.
	- Sa bàn khai trong `SynapseCity.py` là **1220 mm**.
	- ⇒ `1220 px / 1220 mm = 1.0 px/mm`.
- Ảnh map **từng là 1200 px** đúng như thời điểm giao task, sau đó đã được xuất lại thành 1220 px.
- Vì vậy cách làm hiện tại — suy ra tỉ lệ từ ảnh thật, đồng thời so kích thước ảnh với `FIELD_DEFS` lúc decode — an toàn hơn giá trị cố định: đổi ảnh lần nữa thì scale tự đúng, ảnh sai kích thước thì báo lỗi ra tab Log thay vì âm thầm lấy mẫu lệch.

---

#### 2. Kiểm tra lại chiều quy đổi simXY → pixelXY

##### 2.1. Công thức hiện tại
```js
pixelX = roundHalfEven(state.width  / 2 + simX * state.scale);
pixelY = roundHalfEven(state.height / 2 - simY * state.scale);
```
- Với `scale = mapPixels / sizeMm` thì `pixelXY = simXY / sizeMm * mapPixels` — **đúng chiều task yêu cầu**.
- Dấu trừ ở trục Y do toạ độ ảnh tăng từ trên xuống, còn toạ độ simulation tăng từ dưới lên.
- `SYNAPSE_CITY_SCALE = 61/60` (chiều ngược) đã bị xoá khỏi toàn repo — grep không còn kết quả.
- Link code github: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L484-L494

##### 2.2. Kiểm chứng bằng đo đạc
- Để chắc chắn ảnh sa bàn hiển thị và map cảm biến mô tả cùng một vùng vật lý, đã so vị trí các vạch line trên cùng một scanline của hai ảnh:

| Nguồn | Vị trí vạch (mm) | Sai lệch |
|---|---|---|
| Field map (1220 px / 1220 mm) | −508.5, −339.5, −170.0, −1.0, 169.0, 338.5, 507.5 | mốc chuẩn |
| `board.jpg` nếu phủ **1220 mm** | −508.8, −339.6, −169.8, −0.5, 168.8, 338.6, 507.8 | **0.5 mm** |
| `board.jpg` nếu phủ **1200 mm** | −500.5, −334.0, −167.0, −0.5, 166.0, 333.0, 499.5 | 8.0 mm |

- Kết luận: `board.jpg` tuy là ảnh 1200 px nhưng **nội dung phủ đúng 1220 mm**; Digital Twin trải texture theo UV 0..1 trên cả mặt sa bàn nên hiển thị hiện tại chính xác.
- Ba nguồn 1220 mm khớp nhau: profile trong `SynapseCity.py`, field map IR, và nội dung ảnh sa bàn.

---

#### 3. Đổi tên kind `synapse-block` ⇒ `direct-message`

##### 3.1. Contract mới
- Message không còn mang tên mission, tách thành `target` + `operation`:
```
{
  kind: "direct-message",
  version: 1,
  seq,
  simTick,
  target,      // scene | object | robot | runtime | run
  operation,   // configure | create | move | pose | rgb | gripper | finish | update | start | end
  payload
}
```
- Link code github:
	- Định nghĩa + decoder: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/python-direct-message.ts#L107-L197
	- Bridge phát message trong worker: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L177-L235
	- Envelope gửi sang iframe: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L5942-L5968
	- Iframe định tuyến theo target/operation: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/experience.mjs#L592-L636

##### 3.2. Đã xoá hẳn
- 4 module + 3 spec mang tên SynapseCity: `python-synapse-block-frame.ts`, `python-synapse-block-state.ts`, `SynapseCityBlockLayer.ts`, `synapse-city-model-json.ts`.
- Các message cũ: `operation: "set"`, field `slot`, `operation: "set-xy"`, và 6 loại `digital-twin-python-*-frame`.

##### 3.3. Bảng tổng hợp message
- Đã lập tài liệu `digital-twins/docs/MESSAGE_CONTRACT.md` với đúng 3 cột **Brython ⇒ Bridge / Bridge ⇒ DigitalTwin / Bridge ⇒ MQTT** (17 dòng), kèm:
	- Envelope chi tiết và điều kiện một telemetry frame có được đẩy lên MQTT hay không.
	- Mục "Message đã bị loại bỏ" — map từ tên cũ sang tên mới, để đối chiếu với tài liệu cũ.
	- Quy tắc cập nhật: liệt kê đúng file nguồn phải đối chiếu cho từng chặng.

---

#### 4. Iframe Digital Twin bị refresh ⇒ refresh toàn bộ VirtualLeanbot

##### 4.1. Cơ chế
```ts
handleIframeLoad() {
  if (this.digitalTwinFrameEverReady && this.allowDigitalTwinPageReload()) {
    window.location.reload();
    return;
  }
  ...
}
```
- Chỉ tính là "refresh" khi tài liệu **trước** đã bắt tay ready xong. Tài liệu trung gian (`about:blank`, hoặc iframe tự reload vì `/auth/user` trả 401) chưa từng post ready nên không kéo cả trang vào vòng lặp reload.
- Phanh chống lặp: bỏ qua 15 giây đầu sau khi trang khởi tạo + cooldown 10 giây lưu trong `sessionStorage` (cờ trong bộ nhớ không sống qua reload nên không tự phát hiện lặp được). Cả hai nhánh chặn đều ghi log cảnh báo.
- Link code github:
	- `handleIframeLoad`: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L6392-L6405
	- Phanh chống lặp: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L6354-L6390
	- Hằng số cấu hình: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L942-L948

##### 4.2. Đơn giản hoá code
- Bỏ hỗ trợ refresh iframe độc lập kéo theo xoá **~112 dòng** logic khâu lại trạng thái:
	- `refreshDigitalTwinFrame()`, `resetDigitalTwinReadyState()`, `digitalTwinFrameKey` và `:key` trên iframe.
	- `restoreDigitalTwinDirectMessages()`, `restoreDigitalTwinRuntimeFrames()`, `getPythonDirectMessageRestoreFrames()`.
	- State chỉ phục vụ replay: `pythonRuntimeFirstBFrame`, `pythonRuntimeLatestFrame`, type `PythonRuntimeFrameSnapshot`.
- Phần state còn giữ lại (`pythonDirectMessageSnapshots`) **không phải** code restore mà là guard cho luồng live: chặn `configure` lần hai, chặn `move` khi object chưa `create`, giới hạn 128 object/run.
	- https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L6036-L6046

---

#### 5. Chuẩn hoá ThreeJS model và tiêu chí validation

##### 5.1. Đối chiếu 5 yêu cầu

| Yêu cầu | Hiện trạng |
|---|---|
| Cho phép object lồng nhau | Lồng tối đa 12 cấp; **Mesh cũng được có child**; `children` là optional |
| Chỉ 1 Group root | `object.type` bắt buộc là `Group`; layer từ chối nếu model load ra không phải `Group` |
| Cho phép Geometry bất kỳ | Bỏ whitelist 5 loại — nhận mọi geometry THREE dựng được, kể cả `BufferGeometry` với attribute thô |
| Không giới hạn kích thước | Bỏ hết điều kiện `> 0`, cap segment, cap 610 — chỉ còn yêu cầu **số hữu hạn** |
| Cho phép transform | `position` / `rotation` / `scale` / `matrix` trên mọi node |

##### 5.2. Cách validate "geometry bất kỳ" mà vẫn an toàn
- Thay 5 validator theo loại bằng 2 lớp kiểm tra:
	- **Loại geometry**: tra thẳng namespace THREE, chấp nhận khi constructor là `BufferGeometry` hoặc kế thừa `BufferGeometry`. Loại lạ bị chặn ngay tại validator, nhằm tránh để `ObjectLoader` ném lỗi giữa chừng và bỏ lại geometry/material đã tạo.
	- **Dữ liệu**: mọi key ngoài `uuid`/`type` chỉ cần là dữ liệu thuần (số hữu hạn, boolean, chuỗi ngắn, mảng, record). Nhờ vậy `data.attributes.position.array` của BufferGeometry đi qua được mà không cần schema riêng.
- Link code github:
	- Kiểm tra loại geometry: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-object-model-json.ts#L339-L349
	- Kiểm tra dữ liệu: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-object-model-json.ts#L306-L331
	- Cây node + object lồng nhau: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-object-model-json.ts#L471-L518

##### 5.3. Guardrail giữ lại — đổi từ giới hạn hình dạng sang giới hạn tài nguyên
- Payload ≤ 256 KiB, node ≤ 256, độ sâu ≤ 12, geometries/materials ≤ 256, mảng ≤ 65 536 phần tử, tổng phần tử ≤ 131 072.
- Vẫn chặn: `NaN`/`Infinity`, chuỗi trỏ tài nguyên ngoài (`://`, `data:`, `blob:`, `javascript:`, `file:`), key `__proto__`/`constructor`/`prototype`, `textures`/`images`/`animations` ở top-level, node Light/Camera.
- Link code github: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-object-model-json.ts#L5-L15

##### 5.4. Kiểm chứng
- `direct-object-model-json.spec.ts`: 32 test, gồm BufferGeometry có attribute thật, geometry tham số bất kỳ với kích thước/segment lớn, kích thước 0/âm/rất lớn được chấp nhận, mesh lồng mesh, và các ca bị chặn (type lạ, chuỗi URL/data URI, key reserved, tràn budget).
- `direct-object-layer.spec.ts`: thêm 2 test **đi qua `THREE.ObjectLoader` thật** — model BufferGeometry và model lồng Group→Group→Mesh→Mesh dựng được, transform của group lồng đúng.
- Model SynapseCity hiện tại vẫn pass nguyên vẹn.

---

#### 6. `setBlockXYA` và `headingDeg` — cập nhật code mẫu

##### 6.1. API
- `setBlockXYA(objId, x, y, headingDeg)` nhận đủ 3 tham số rời, thay cho tuple pose.
- Link code github: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/SynapseCity.py#L337-L352

##### 6.2. Cập nhật 4 file mẫu

| File | Thay đổi |
|---|---|
| `1.Containment.py` | Bỏ `moveObject(obj, (x,y,a))` → `setBlockXYA` (dòng 45); dời lời gọi vào đúng bước `PushPollutionB1()` thay vì nằm rời trong `loop()` |
| `2.Neutralize.py` | Bổ sung `setBlock` Blue@E2 + Neutralizer@S3 (dòng 22–23); `setBlockXYA` khi đẩy khối vào Containment E và khi thả Neutralize (dòng 58, 63) |
| `3.Analys.py` | Bổ sung `setBlock` Red@F4 (dòng 19); `setBlockXYA` khi thả tại CRL (dòng 59) |
| `4.Codegom.py` | Khôi phục 4 lời gọi `setBlock` (dòng 28–31); `setBlockXYA` ở cả 4 bước (dòng 72, 77, 92, 118) |

- File 2 và 3 trước đó có `import SynapseCity` nhưng chưa gọi API nào nên khối không hiện trên sa bàn dù phần mô tả mission có nêu.
- File 4 bị mất 4 dòng `setBlock` khi tune lại tham số motion; nay đã khôi phục.
- Link code github: https://github.com/PTV-TechHub/digitaltwins-service/tree/acb6af51a/digital-twins/client/public/PyDemo/SynapseCity

##### 6.3. Nguồn toạ độ
- Containment B `(-334, 0)`: tâm 4 slot B1–B4 trong `_POSITION_POSES`.
- Containment E `(0, 0)`: tâm 4 slot E1–E4.
- CRL `(260, -260, 135°)`: lấy từ `getStartPose('SynapseCity.01')`.
- Heading giữ nguyên góc của slot xuất phát (`±44.56`).

##### 6.4. Hai giá trị cần xác nhận bằng mắt
- Vị trí thả Neutralize `(0, -80)` — đặt phía nam khối Blue, chỗ Leanbot đứng khi mở gripper.
- Vị trí thả Red tại CRL `(260, -260)` — trùng tâm robot, có thể cần nhích ra trước mũi ~30–40 mm.
- Hai giá trị này không có số đã tune sẵn nên là ước lượng theo hình học sa bàn.

---

#### 7. Tổng quát hoá iframe Digital Twin

##### 7.1. Mission tự mô tả sa bàn của mình
- Toàn bộ thông tin riêng của SynapseCity nằm trong `_DIRECT_PROFILE` của `SynapseCity.py`: kích thước sa bàn, độ dày, màu, đường texture, capabilities, telemetryPolicy.
- Iframe chỉ dựng lại theo profile nhận được, không định nghĩa và không validate cho riêng SynapseCity.
- Nhận diện mission theo `missionFamily` trong profile, không hardcode tên mission.
- Bridge trong worker là mission-agnostic: bất kỳ thư viện Python nào cũng `registerLibrary` rồi `send(target, operation, payloadJSON)`.
- Link code github:
	- Schema profile: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-scene-profile.ts#L126-L168
	- Đường asset chỉ cần nằm dưới `mission-assets/<tên>/`: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-scene-profile.ts#L86-L98
	- Khớp mission theo family: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/direct-scene-profile.ts#L100-L110
	- Bridge mission-agnostic: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/runner-worker.js#L197-L235

##### 7.2. Vòng đời sa bàn trong iframe
- `MissionSceneRuntime` là owner trung lập cho sa bàn + object, thiết kế theo ba nguyên tắc phòng ngừa:
	- **Dựng sa bàn đồng bộ, texture gắn sau**: nhằm tránh việc các message object phía sau phải xếp hàng chờ tải ảnh qua mạng, hoặc bị chặn khi ảnh không về.
	- **Quyền sở hữu texture thuộc về chính sa bàn đó, đánh dấu bằng số đếm thế hệ**: ai thay sa bàn thì texture cũ bị huỷ, còn run kết thúc mà sa bàn vẫn hiển thị thì texture về muộn vẫn được gắn. Dùng số thay cho so sánh identity object là nhằm tránh sai lệch khi runtime được giữ trong state Vue — reactive proxy làm `===` không còn tin được sau `await`.
	- **`markRaw` khi tạo runtime**: nhằm tránh Vue bọc reactive proxy lên cả cây scene ThreeJS, vừa an toàn về identity vừa bỏ được chi phí proxy khi render.
- Toàn bộ vòng đời sa bàn đều ghi log (`Board configured` / `texture applied` / `texture dropped` / `beginRun: dropping`) để mọi sự cố hiển thị đều truy được nguyên nhân từ Console.
- Link code github:
	- Dựng sa bàn: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/MissionSceneRuntime.ts#L67-L105
	- Gắn texture theo thế hệ sa bàn: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/MissionSceneRuntime.ts#L109-L143
	- `markRaw`: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/experience.mjs#L283

##### 7.3. Loại bỏ kiểm tra 610
- Không còn ở bất kỳ đâu trong luồng direct:
	- Profile chỉ yêu cầu kích thước sa bàn là số dương hữu hạn.
	- Pose chỉ cần số hữu hạn; layer ThreeJS nhận mọi toạ độ.
- Giá trị `610` còn lại duy nhất trong repo là sa bàn Leanbot mặc định `1220×610` của luồng legacy, không phải một phép kiểm tra.
- Link code github: https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/src/views/experience/leanbot-core/DirectObjectLayer.ts#L97-L107

##### 7.4. Sẵn sàng cho mission tương lai
- Test fixture đã dùng một mission giả định khác (`OceanRescue.v1`, sa bàn 1800×900, texture riêng) chạy trọn pipeline mà **không sửa dòng code nào** của iframe.
- Test `switches to a second mission across runs without leaking the first scene`: mission A → `beginRun()` → mission B với sa bàn 900×900×8, khẳng định sa bàn A đã dispose và object A biến mất.
- Thư viện mission tương lai tự đăng ký điểm xuất phát qua `registerStartPose()`, không phải sửa `pyLeanbot2`:
	- https://github.com/PTV-TechHub/digitaltwins-service/blob/acb6af51a/digital-twins/client/public/VirtualLeanbotPython-Runner/pyLeanbot2/Leanbot_config.py#L82-L105

##### 7.5. Hai phần còn nằm ngoài `SynapseCity.py` — có lý do kỹ thuật
- **Field map IR** trong `runner-worker.js`: ảnh xám bắt buộc phải nằm trong worker để Brython đọc đồng bộ, không đẩy qua Python được. Đã tổng quát hoá thành bảng tra theo tên mission, thêm mission = thêm một dòng dữ liệu.
- **Điểm xuất phát CRL** trong `Leanbot_config.py`: `LbMission.begin()` gọi `getStartPose()` cho **mọi** chương trình, mà code sinh từ Blockly và các sample có sẵn chỉ có `LbMission.begin("SynapseCity.01")` chứ không `import SynapseCity`. Chuyển hẳn giá trị sang `SynapseCity.py` sẽ làm các chương trình đó xuất phát ở `(0,0,0)`. Vì vậy đã thêm cơ chế đăng ký cho mission mới và ghi rõ lý do dòng SynapseCity ở lại ngay trong file.
	- Muốn dứt điểm thì cần cho Blockly generator sinh thêm `import SynapseCity` — đề xuất tách thành task riêng vì đụng tới code sinh cho học sinh.

---

#### 8. Tổng kết kiểm chứng

| Hạng mục | Kết quả |
|---|---|
| Unit test | **287/287 pass**, 20 suite |
| Test bổ sung trong đợt này | model JSON (32 test), MissionSceneRuntime (10 test), DirectObjectLayer (+2 test chạy qua `ObjectLoader` thật), direct-message state |
| Lint | `vue-cli-service lint` — 0 error trên toàn bộ file đã sửa |
| Cú pháp Python | `py_compile` 4 file demo — OK |
| Tài liệu | `digital-twins/docs/MESSAGE_CONTRACT.md`, có link từ `digital-twins/README.md` |

---

#### 9. Các điểm cần ý kiến

1. **Scale của SynapseCity là `1.0`, không phải `1200/1220`** — vì file map đã được xuất lại thành 1220 px trên sa bàn 1220 mm (mục 1.2). Code đang suy ra tỉ lệ từ ảnh thật nên tự đúng nếu ảnh đổi lần nữa.
2. **Hai toạ độ trong code mẫu là ước lượng** (mục 6.4), cần chạy thử và xác nhận bằng mắt.
3. **`object/move` dùng `headingDeg` còn `robot/pose` dùng `headingRad`** — khác biệt có thật trong contract hiện tại. Nếu muốn thống nhất thì nên làm thành task riêng vì đụng cả decoder lẫn phía iframe.
4. **Điểm xuất phát cho chương trình Blockly** (mục 7.5) — cần quyết định có cho generator sinh `import SynapseCity` hay không.
