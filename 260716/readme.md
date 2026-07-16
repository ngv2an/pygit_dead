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

### B. Công việc tiếp theo
- Xin anh giao việc cho em!