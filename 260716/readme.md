# NguyenVanAn

## NHẬT KÍ NGÀY 16/07/2026

### A. Công việc đã làm

#### 1. Khi chạy code Demo chưa thấy Finish trên Digtial Twin?
- Luồng hiển thị:
```
telemetry Z
-> MQTT
-> simulator tạo /sim với RobotStatus: Finish
-> hàng đợi render của client
-> drawcar()
-> leanbotStatus
-> badge Digital Twin
```

- Trong luồng Python `record -> replay`, các frame cuối được gửi sát nhau. Sau frame `Finish`, simulator vẫn có thể phát frame nội suy đuôi:
```
Finish @ t
Go @ t + vài mili-giây
```

- Status thiếu cũng được chuẩn hóa thành `Go`
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/be1b4f842320add3c41244ea44e5d589d4e17134/digital-twins/client/src/views/experience/leanbot-core/CommonMessageProcess.ts#L293
- Bộ lọc cũ chỉ bỏ frame có `RobotTime` đi lùi. Vì frame `Go` đuôi có thời gian lớn hơn frame `Finish`, nó vẫn hợp lệ và ghi đè `leanbotStatus`. Header bind trực tiếp với biến này 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/annguyen-qa/digital-twins/client/src/views/experience/ExperienceView.vue#L190
- Nguyên nhân gốc: `Finish` chưa được coi là trạng thái kết thúc cố định của mission hiện tại, nên bị frame `Go` nội suy đến sau ghi đè.

- Bổ sung cơ chế giữ trạng thái kết thúc:
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/be1b4f842320add3c41244ea44e5d589d4e17134/digital-twins/client/src/views/experience/experience.mjs#L638

```
if (this.leanbotStatus === "Finish" && robotStatus !== "Finish"
    && Number.isFinite(robotTime)
    && robotTime >= ROBOT_TIME_RESET_THRESHOLD_S) {
  robotStatus = "Finish";
}
```

- Nguyên tắc:
    - Khi UI đã nhận `Finish`, các frame khác trạng thái nhưng vẫn thuộc mission hiện tại không được đổi badge về `Go`.
    - Mission mới được nhận diện khi `RobotTime < 0.002s`.
    - Khi thời gian reset, trạng thái `Ready/Go` của mission mới được phép hiển thị bình thường.
    - Bộ lọc frame cũ có thời gian đi lùi vẫn hoạt động như trước.

- Kết quả: Digital Twin giữ ổn định `Status: Finish` khi mission kết thúc.

#### 2. Ví dụ cụ thể từng frame
- Ở cuối mission, client gửi các frame thô (raw serial), mỗi frame mở đầu bằng ký tự trạng thái:

| Frame thô client gửi   | Ký tự status | Nghĩa           |
|------------------------|--------------|-----------------|
| `...27B 0c06H 000b0 0` | B            | Go (đang chạy)  |
| `28Z 0c06H 000eo 0`    | Z            | Finish          |

- Simulator .212 nhận, giải mã, và tự sinh thêm frame nội suy để làm mượt chuyển động. Ở tầng sim JSON (log `DT-recv`) có 3 loại:

| Frame sim JSON            | RobotStatus | Nguồn                       |
|---------------------------|-------------|-----------------------------|
| `status=Go t=3.2`         | "Go"        | từ frame B                  |
| `status=Finish t=3.2`     | "Finish"    | từ frame Z                  |
| `status=undefined t=3.21` | Không có    | frame nội suy .212 tự sinh  |

- Frame gây lật badge không có `Status Z`, cũng không có `Status B`, nó thiếu hẳn `RobotStatus`. Nó là frame nội suy vị trí do simulator sinh ra, `robotTime = 3.21` (nhích hơn `Finish` 3.20 vài ms).

- Lý do lật badge: render queue sắp xếp theo `robotTime`. `Finish` (3.20) render trước -> badge "Finish". Rồi frame nội suy (3.21) render sau. Đáng lẽ frame thiếu status không nên đổi gì, nhưng code tự gán mặc định "Go" cho nó -> lật badge về "Go". 

#### 2. Cải tiến
- 2 chỗ tự gán `RobotStatus = "Go"` khi thiếu:
    - `RenderBotQueue.push`: https://github.com/PTV-TechHub/digitaltwins-service/blob/a391e925c6fdc5e7c8958d584b12bcd0ee914809/digital-twins/client/src/views/experience/leanbot-core/CommonMessageProcess.ts#L144-L150
    - `enqueueMessage`: https://github.com/PTV-TechHub/digitaltwins-service/blob/a391e925c6fdc5e7c8958d584b12bcd0ee914809/digital-twins/client/src/views/experience/leanbot-core/CommonMessageProcess.ts#L306-L312

- Bỏ cả 2 chỗ gán "Go".
- Thêm carry-forward tại `RenderBotQueue.handleMessage`, nơi frame được lấy ra render theo đúng thứ tự `robotTime` đã sort: nếu thiếu `RobotStatus` thì giữ `lastRenderedStatus` (status hợp lệ gần nhất). Reset về rỗng khi mission mới (`robotTime < ngưỡng`).




### B. Công việc tiếp theo
- Xin anh giao việc cho em!