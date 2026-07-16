# NguyenVanAn

## NHẬT KÍ NGÀY 14/07/2026

### A. Công việc đã làm

#### 1. Báo cáo lại chi tiết hơn đủ các bước khi bắt đầu chạy Mission này

##### 1.1. Chức năng 
- Brython-worker: Chạy Python ở tốc độ tối đa, tạo các frame `{ simTick, mission, data }`; không đăng nhập và không kết nối MQTT/WebSocket.
- JS-Queue: Nhận từng frame ngay khi Brython sinh ra và lưu theo `runId`.
- JS-Mission: Gọi `waitCmdStart()`, gửi `bootstrap`, đợi `start-bot`, sau đó lấy frame từ JS Queue để gọi `sendTelemetry(mission, data)` theo `speedUp`.
- WS-worker: Gắn auth token vào kết nối WebSocket, gửi 7 API nghiệp vụ và nhận message theo `channel`.
- NestJS backend: Xác thực token, lấy `authUser`, dựng MQTT topic, publish/subscribe và forward message.
- Digital-twins iframe: Đăng ký telemetry/sim, đợi bootstrap, gọi `sendCmdStart()`, nhận dữ liệu sim và render ThreeJS.

##### 1.2. Khởi tạo Digital Twin iframe
- Iframe tạo `LeanbotCar` và `PingObject`. Các đối tượng đăng ký nhận dữ liệu mission và simulation qua `mqttService`. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/experience/leanbot-core/CommonMessageProcess.ts#L187-L235
- Iframe gọi:
    - `startSim()` để nhận dữ liệu render.
    - `startMission()` để nhận bootstrap và tên mission.

- WS-worker của iframe mở WebSocket kèm `?token=....` NestJS lấy `preferred_username` từ token và gắn thành `authUser`. Client không gửi `user` hoặc MQTT topic. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/src/dtt-ws-adapter.ts#L23-L65

- NestJS thực hiện:
    - `startSim()` -> MQTT subscribe `leanbot/<authUser>/sim`.
    - `startMission()` -> MQTT subscribe `leanbot/<authUser>/telemetry/+`.

- Sau khi kết nối và đăng ký hoàn tất, iframe gửi digital-twin-ready cho trang Blockly. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/experience/experience.mjs#L304-L312

##### 1.3. User nhấn RUN
- Trang Blockly gửi cho iframe: `digital-twin-run-status { running: true, speedUp }`
    - Iframe ghi nhận `speedUp` và chuyển sang trạng thái chờ bootstrap. 
        - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/experience/experience.mjs#L125-L145

- Với VirtualLeanbot, JS runtime trực tiếp sinh và gửi telemetry. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L7535-L7568

- Với VirtualLeanbotPython, Brython chạy song song với JS-Mission:
    - Brython-worker sinh từng frame và đẩy ngay sang JS.
    - JS-Queue nhận frame mà không đợi Python simulation kết thúc.
    - JS-Mission bắt đầu xử lý ngay khi thấy bootstrap.
        - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L2929-L2975

##### 1.4. Đăng ký chờ START và gửi bootstrap
- JS-Mission gọi `waitCmdStart`().
- NestJS subscribe tạm thời: `leanbot/<authUser>/cmd`
    - Subscription chỉ nhận payload chính xác `start-bot`; command khác bị bỏ qua. Sau khi forward thành công, backend tự unsubscribe.
        - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/src/app.gateway.ts#L149-L159

- Sau khi backend ACK việc subscribe, JS-Mission gọi `sendTelemetry("msNumberGrid", "00A")`
- NestJS dựng và publish MQTT:
```
Topic:   leanbot/<authUser>/telemetry/msNumberGrid
Payload: 00A
```
- Các telemetry được đưa qua TaskQueueService để publish theo đúng thứ tự. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/src/app.gateway.ts#L64-L77
- Nếu chưa nhận START, bootstrap được thử lại tối đa ba lần, cách nhau 3 giây.

##### 1.5. Digital Twin nhận bootstrap và phát START
- Vì iframe đã gọi startMission(), NestJS forward bootstrap về iframe:
```
{
  "mode": "mqttMessage",
  "channel": "telemetry",
  "mission": "msNumberGrid",
  "data": "00A"
}
```

- Iframe nhận biết bootstrap khi payload kết thúc bằng A, lấy mission từ metadata và đổi sa bàn tương ứng. Iframe không tự phân tích hoặc tạo MQTT topic.

- Do RUN đã được kích hoạt, iframe gọi tự động:
```
autoStartBot(speedUp)
sendCmdStart()
```
- https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/experience/experience.mjs#L262-L271
- https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/components/start-bot/start-bot.ts#L307-L399

- NestJS publish MQTT với QoS 1:
```
Topic:   leanbot/<authUser>/cmd
Payload: start-bot
```
- Sau PUBACK, backend forward ngay cho JS-Mission với `channel: "cmdStart"`.
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/src/app.gateway.ts#L80-L98

##### 1.6. JS-Mission gửi dần telemetry
- Sau khi nhận đúng `cmdStart/start-bot`:
    - `VirtualLeanbot` bắt đầu countdown, chạy block và sinh telemetry theo thời gian thực.
    - `VirtualLeanbotPython` lấy lần lượt các frame tiếp theo từ `JS Queue`.
    - `simTick` chỉ dùng tại JS để tính thời điểm gửi:
    ```
    targetElapsed = (simTick - baseSimTick) / speedUp
    ```
    - Dữ liệu gửi qua API chỉ gồm `mission` và `data`, không gửi `simTick`, `user` hoặc `MQTT topic`.
    - Phần đợi START và streaming 
        - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L2629-L2715

##### 1.7. Backend và iframe render ThreeJS
- Mỗi frame được NestJS publish tới:
```
leanbot/<authUser>/telemetry/<mission>
```
- Bộ xử lý simulation phía MQTT/backend chuyển telemetry hex thành dữ liệu JSON và publish tới:
```
leanbot/<authUser>/sim
```

- NestJS đã subscribe topic này từ `startSim()`, nên forward về iframe với `channel: "sim"`.

- Iframe parse JSON, sắp xếp theo `RobotTime`, phát theo `RobotTime` / `speedUp`, rồi cập nhật vị trí, góc quay, gripper, LED và trạng thái ThreeJS. 
    - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/experience/leanbot-core/CommonMessageProcess.ts#L24-L168

- Iframe đồng thời báo mốc render về trang cha bằng `digital-twin-telemetry-render`.

##### 1.8. Kết thúc
- JS-Mission gửi frame trạng thái cuối `Z`.
- Iframe nhận bản tin sim có `RobotStatus: "Finish"` và hoàn tất hiển thị.
- `waitCmdStart` đã tự unsubscribe ngay sau `start-bot`.
- Khi các subscription iframe được giải phóng, client gọi `stopMission()` và `stopSim()`. Nếu WebSocket đóng trước thì backend tự dọn subscription trong `handleDisconnect()`.

#### 2. Chức năng LeanbotIoT trên VirtualLeanbotPython hiện đang như thế nào?

##### 2.1. Đã gen code python chưa?
- Chưa gen được code Python cho 2 block LbIoT

##### 2.2. Code Python đã chạy được với Brython chưa?
- Code Python chưa chạy được với Brython
- So sánh: 
    - trang JS (VirtualLeanbot) chạy IoT qua REST, interpreter gọi thẳng `axios.post('/iot/read' | '/iot/write')`
        - https://github.com/PTV-TechHub/digitaltwins-service/blob/a866d8207b7f6a36066d03360650cb85971cce05/digital-twins/client/src/views/blockly-test/BlocklyTestView.ts#L7733-L7761
    - server NestJS forward sang OpenRemote qua [iot.controller](https://github.com/PTV-TechHub/digitaltwins-service/blob/annguyen-qa/digital-twins/src/controllers/iot.controller.ts) + [iot.service](https://github.com/PTV-TechHub/digitaltwins-service/blob/annguyen-qa/digital-twins/src/services/iot.service.ts). Đường REST này dùng được cho cả Python.

##### 2.3 Đề xuất cách thực hiện
- gọi thẳng REST `/iot/*` bằng sync XHR từ trong worker (dùng lại nguyên endpoint + auth của trang JS)
    - Code  gen: 
        - `generatePythonBlockCode` + case `lb_iot_write` -> `leanbot.iotWrite(<asset>, <attr>, <value>)`
        - `generatePythonExpressionCode` + case `lb_iot_read` -> `leanbot.iotRead(<asset>, <attr>)`

- Với code LbIoT-Test, code kỳ vọng sẽ là:
```
iotCounter = (iotCounter + 1)
leanbot.iotWrite("33khwmcNaidrPJ2oZpcgde", "notes", (str("Hello ") + str(iotCounter)))
print(leanbot.iotRead("33khwmcNaidrPJ2oZpcgde", "notes"))
```

- Runtime trong runner: thêm `iotWrite`/`iotRead` vào [leanbot.py](https://github.com/PTV-TechHub/digitaltwins-service/blob/annguyen-qa/digital-twins/client/public/VirtualLeanbotPython-Runner/leanbot.py) (và sửa shim `LeanbotIoT.py` dùng chung đường này), gọi `POST /iot/write`, `POST /iot/read` bằng XHR đồng bộ (Brython `ajax` với `blocking=True`):
    - Sync XHR bị hạn chế trên main thread nhưng hợp lệ trong Web Worker, mà Brython chạy đúng trong worker và bản thân chương trình Python là đồng bộ.
    - Request same-origin nên cookie phiên đăng nhập tự đính kèm, đúng cách trang JS đang auth với `/iot/*` (nếu cần Bearer, [runner](https://github.com/PTV-TechHub/digitaltwins-service/blob/annguyen-qa/digital-twins/client/public/VirtualLeanbotPython-Runner/runner.html#L35-L41) đã đọc sẵn `token` cookie và truyền vào worker qua message `init`).
    - Lỗi HTTP -> raise `RuntimeError` để chương trình Python báo lỗi rõ ràng như bên JS.

#### 3. LeanbotIoT trên VirtualLeanbotPython
- Đường JS hiện tại đang chạy: `LbIoT.read/write` (JS) => `axios.post('/iot/read' | '/iot/write')` => NestJS `IotController` => `IotService` => OpenRemote.
- Bước tạo URL tới OpenRemote: 
    - Kiểm tra thực tế cho thấy URL OpenRemote đã nằm trong NestJS rồi:
        - Chỗ duy nhất ghép URL: `iot.service.ts` → `${IOT_BASE_URL}/api/${IOT_REALM}/asset/query` và `/asset/attributes`.
        - Client JS không biết OpenRemote, chỉ gọi `/iot/read`, `/iot/write`.
    
- Chỗ còn biết URL là phía Python: shim `LeanbotIoT.py` vẫn giữ `LbIoT.config(server=…, realm=…, port=…)`

=> đề xuất: khoá cứng OpenRemote trong NestJS, dựng cặp API `iotRead`/`iotWrite` ở tầng `backend`, và xoá sạch khái niệm `server`/`realm`/`port` khỏi Python.

- Kiến trúc đề xuất:

```
Blockly /VirtualLeanbotPython            Blockly /VirtualLeanbot (JS)
        │ codegen                                │ interpreter
        V                                        V
  leanbot.iotWrite / iotRead          services/iot-api.ts : iotWrite / iotRead
        |                                        |
        V                                        │
  pyLeanbot2.LbIoT   <---- Không còn server/realm/port
        |                                        |
        V                                        │
  backend.iotWrite / iotRead   <- API Python, cùng dáng sendTelemetry/waitCmdStart
        |                                        |
        V                                        │
  __backendBridge.iotWrite / iotRead   <- nơi duy nhất giấu transport (HTTP)
        └──────────────┬─────────────────────────┘
                       V
         POST /iot/write | /iot/read      (NestJS, AuthGuard)
                       |
                       V
                  IotService              <- Duy nhất biết OpenRemote
                       |
                       V   ${IOT_BASE_URL}/api/${IOT_REALM}/asset/…
                  OpenRemote
```



### B. Công việc tiếp theo
#### 1. Xử lý Mission name string trước khi gen ra code python
#### 2. Bỏ bớt và giảm Countdown
#### 3. Gen code đang thừa dòng
#### 4. Thêm các dòng log vào minh họa cho mô tả các bước chạy