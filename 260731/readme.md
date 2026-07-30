# Bảng tổng hợp message giữa các module

Tài liệu này mô tả **contract message đang chạy** của luồng VirtualLeanbotPython.
Mỗi lần thêm/sửa/xoá message phải cập nhật bảng ở đây trong cùng commit — xem
[Quy tắc cập nhật](#quy-tắc-cập-nhật) ở cuối.

## Phạm vi và cách gọi tên

**Bridge** trong tài liệu này gồm 3 mắt xích nối tiếp, đứng giữa Brython và hai
đích tiêu thụ (Digital Twin, MQTT gateway):

| Thành phần | File |
|---|---|
| Worker bridge | `client/public/VirtualLeanbotPython-Runner/runner-worker.js` |
| Runner frame | `client/public/VirtualLeanbotPython-Runner/runner.html` |
| Parent định tuyến | `client/src/views/blockly-test/BlocklyTestView.ts` |

```
Brython (SynapseCity.py, pyLeanbot2)
   │  (1) gọi hàm JS trên window/self
   ▼
runner-worker.js ──(2) post kind──► runner.html ──(3) type: python-*──► BlocklyTestView
                                                                            │
                                              (4) digital-twin-* ───────────┼──► iframe /experience
                                                                            │
                                              (5) WS-worker ────────────────┴──► MQTT gateway
```

Nguyên tắc: **Brython không bao giờ nói chuyện trực tiếp với Digital Twin hay
MQTT.** Mọi thứ đi qua Bridge, và Bridge quyết định message nào sang đích nào.

---

## Bảng chính

Trống (`—`) nghĩa là message không đi qua chặng đó.

| # | Việc | Brython ⇒ Bridge | Bridge ⇒ DigitalTwin | Bridge ⇒ MQTT |
|---|---|---|---|---|
| 1 | Đăng ký thư viện mission | `__directMessageBridge.registerLibrary(moduleName)` | — | — |
| 2 | Cấu hình sa bàn | `__directMessageBridge.send('scene', 'configure', {profileJSON})` | `target:'scene'` / `operation:'configure'` — payload `{profileJSON}` | — |
| 3 | Tạo object | `__directMessageBridge.send('object', 'create', {objId, modelJSON})` | `object` / `create` — `{objId, modelJSON}` | — |
| 4 | Di chuyển object | `__directMessageBridge.send('object', 'move', {objId, pose})` | `object` / `move` — `{objId, pose{xMm, yMm, headingDeg}}` | — |
| 5 | Bắt đầu / kết thúc run | — (Bridge tự sinh) | `run` / `start` — `{capabilities}`<br>`run` / `end` — `{reason}` | — |
| 6 | Telemetry bootstrap `A` | `__backendBridge.send({mode:'sendTelemetry', user, mission, data:'XXA'})` | `runtime` / `update` — `{state:'A'}` | `sendTelemetry(mission, 'XXA')` — **luôn gửi** |
| 7 | Telemetry vị trí `B` | `...data:'XXB <pos> <time> <ir>'` | `robot` / `pose` — `{mission, pose{xMm, yMm, headingRad}}`<br>`runtime` / `update` — `{state:'B'}` | direct run: **chỉ frame B đầu tiên**<br>legacy run: mọi frame |
| 8 | Telemetry LED RGB | `...data:'XXB L <brightness><21 kênh>'` | `robot` / `rgb` — `{colors[7]}` | direct run: **không gửi**<br>legacy run: gửi |
| 9 | Telemetry gripper | `...data:'XXB G <time><4 góc>'` | `robot` / `gripper` — `{durationMs, leftStartDeg, leftEndDeg, rightStartDeg, rightEndDeg}` | direct run: **không gửi**<br>legacy run: gửi |
| 10 | Telemetry kết thúc `Z` | `...data:'XXZ <pos> <time> 0'` | `robot` / `finish` — `{}`<br>`runtime` / `update` — `{state:'Z'}` | direct run: **đúng 1 frame Z**<br>legacy run: gửi |
| 11 | Lệnh phiên gateway | `__backendBridge.send({mode, ...})` với mode là một trong: `startSim`, `stopSim`, `startMission`, `stopMission`, `sendCmdStart`, `waitCmdStart`, `mqttPublish`, `mqttSubscribe`, `mqttUnsubscribe` | — | — (record-phase **bỏ qua**, xem [Ghi chú](#ghi-chú-quan-trọng) #2) |
| 12 | Đọc vạch line (IR) | `__fieldBridge.begin(missionId)` → trả `0` hoặc `1`<br>`__fieldBridge.readFieldTrack(x, y)` → trả `0..255` | — | — |
| 13 | `print()` của học sinh | `__appendConsole(text, cssClass, simMs)` | — | — |
| 14 | Log nội bộ runner | `__appendLog(text, cssClass, simMs)` | — | — |
| 15 | Toàn màn hình | — | `digital-twin-fullscreen` — `{fullscreen}` | — |
| 16 | Trạng thái chạy | — | `digital-twin-run-status` — `{running, speedUp}` | — |
| 17 | Đổi mission | — | `digital-twin-mission` — `{mission}` | — |

### Envelope cột "Bridge ⇒ DigitalTwin"

Dòng 2–10 dùng chung một envelope `postMessage` (targetOrigin = `window.location.origin`):

```js
{
  source: 'blockly-test',
  type: 'digital-twin-direct-message',
  kind: 'direct-message',
  version: 1,
  runId, seq, simTick,
  target,      // 'scene' | 'object' | 'robot' | 'runtime' | 'run'
  operation,   // 'configure' | 'create' | 'move' | 'pose' | 'rgb' | 'gripper' | 'finish' | 'update' | 'start' | 'end'
  payload,
}
```

Dòng 15–17 là message rời, chỉ có `source` + `type` + field riêng.

> **Lưu ý về đơn vị góc:** `object/move` dùng `headingDeg`, còn `robot/pose` dùng
> `headingRad`. Đây là khác biệt có thật trong code hiện tại, không phải lỗi
> đánh máy của tài liệu.

> **Lưu ý về `robot/pose`:** worker gắn `pose` vào **mọi** item telemetry, và
> `notifyDigitalTwinPoseFrame()` chỉ lọc theo capability chứ không lọc theo dạng
> data. Nên frame `B L` (RGB) và `B G` (gripper) cũng kéo theo một `robot/pose`,
> không riêng frame `B` vị trí.

### Chi tiết cột "Bridge ⇒ MQTT"

| Việc | API gateway | Do ai gọi |
|---|---|---|
| Gửi telemetry | `sendTelemetry(mission, data)` → topic `leanbot/<user>/telemetry/m` | Parent (`sendMissionTelemetry`) |
| `startSim` / `stopSim` / `startMission` / `stopMission` / `sendCmdStart` | qua WS-worker | **iframe Digital Twin**, rồi báo ngược về parent bằng `digital-twin-api` để hiện đủ 7 API trong tab Log |

Việc một telemetry frame có được đẩy lên MQTT hay không do
`classifyPythonTelemetryRoute()` quyết định
(`client/src/views/blockly-test/python-telemetry-route.ts`):

| Loại run | Điều kiện | Route |
|---|---|---|
| Legacy (không import thư viện mission) | mọi frame `sendTelemetry` | `backend` → gửi MQTT |
| Direct, `telemetryPolicy: 'bootstrap-normal-finish'` | frame `A`, frame `B` đầu tiên, frame `Z` cuối | `backend` → gửi MQTT |
| Direct, các frame còn lại | | `direct-only` → chỉ sang Digital Twin |
| `mode` khác `sendTelemetry` hoặc data rỗng | | `invalid` → bỏ, ghi log |

---

## Message hạ tầng của Bridge

Không sang Digital Twin cũng không sang MQTT, nhưng cần cho vòng đời run.

| worker `post(kind)` | runner.html `type` | Nội dung |
|---|---|---|
| `loaded` | `python-runner-loaded` | worker đã nạp |
| `ready` | `python-runner-ready` | Brython sẵn sàng |
| `syntax-result` | `python-syntax-result` | `{checkId, ok, error, internalError}` |
| `console` | `python-console` | `{text, cssClass, simMs}` |
| `log` | `python-log` | `{text, cssClass, simMs}` |
| `queue-item` | `python-telemetry-item` | `{runId, item}` — item là telemetry hoặc direct-message |
| `queue-end` | `python-telemetry-queue-end` | `{runId, count}` |
| `result` | `python-result` | `{runId, ok, elapsedMs, error}` |
| `error` | `python-runner-error` | `{runId, error}` |
| — | `python-runner-stopped` | runner.html tự phát khi bị stop |

Parent → runner.html: `run-python` `{runId, code}`, `check-python-syntax`
`{checkId, code}`, `python-stop` `{reason}`.

## Chiều ngược

| Nguồn | Message | Nội dung |
|---|---|---|
| Digital Twin → parent | `digital-twin-ready` | iframe đã dựng scene xong |
| Digital Twin → parent | `digital-twin-api` | tên API gateway iframe vừa gọi (để ghi Log) |
| Digital Twin → parent | `digital-twin-telemetry-render` | `{robotTime}` — mốc bắt đầu pace output |
| MQTT → parent | message kênh `cmdStart` | tín hiệu Go, dùng trong `waitForPythonTelemetryStart()` |
| MQTT → parent | message kênh `sim` | RobotTime của gateway |
| JS → Brython | `__runPython`, `__checkPythonSyntax`, `__resetDirectLibraries`, `__leanbotSimMillis`, `__leanbotRawSimMillis`, `__leanbotPoseJson`, `__leanbotMissionState` | hàm do harness Brython export cho worker gọi |

---

## Ghi chú quan trọng

1. **Direct message không đi qua MQTT.** Sa bàn và object chỉ tồn tại ở Digital
   Twin; gateway không biết gì về chúng.
2. **Record-phase không nối gateway.** `__backendBridge.connect()` là no-op và
   mọi `mode` khác `sendTelemetry` do Brython gửi đều bị parent phân loại
   `invalid` rồi bỏ. Phiên gateway do parent và iframe tự lo.
3. **Toàn bộ queue dùng chung một trục thứ tự.** `seq` do worker cấp theo độ dài
   queue, chung cho cả telemetry lẫn direct message, nên thứ tự hiển thị giữa
   robot và object luôn khớp với thứ tự chương trình Python.
4. **Capabilities lọc message.** Profile của mission bật/tắt từng nhóm
   (`pose`, `rgb`, `gripper`, `finish`, `runtime`, `objects`); nhóm nào tắt thì
   parent không phát message tương ứng và iframe cũng từ chối nếu nhận được.

## Message đã bị loại bỏ

Giữ lại để đối chiếu với tài liệu cũ — **không dùng nữa**:

| Message cũ | Thay bằng |
|---|---|
| `kind: 'synapse-block'` | `kind: 'direct-message'` + `target`/`operation` |
| `operation: 'set'` và field `slot` | `object/move` (chỉ `objId` + `pose`) |
| `operation: 'set-xy'` | `object/move` với pose đầy đủ (API Python: `setBlockXYA`) |
| `digital-twin-python-runtime-frame` | `runtime/update` |
| `digital-twin-python-pose-frame` | `robot/pose` |
| `digital-twin-python-rgb-frame` | `robot/rgb` |
| `digital-twin-python-gripper-frame` | `robot/gripper` |
| `digital-twin-python-finish-frame` | `robot/finish` |
| `digital-twin-python-synapse-block-frame` | `scene/configure`, `object/create`, `object/move` |

## Quy tắc cập nhật

Khi sửa message, đối chiếu lại đúng các file sau rồi cập nhật bảng:

| Chặng | File nguồn |
|---|---|
| Brython ⇒ Bridge | `SynapseCity.py`, `pyLeanbot2/LbDigitalTwin.py`, `pyLeanbot2/MQTT.py`, `backend.py` |
| Bridge nội bộ | `runner-worker.js` (`__directMessageBridge`, `__backendBridge`, `__fieldBridge`, `post()`), `runner.html` |
| Bridge ⇒ DigitalTwin | `BlocklyTestView.ts` (`postDigitalTwinDirectMessage`, `notifyDigitalTwin*`), `python-direct-message.ts` |
| Bridge ⇒ MQTT | `python-telemetry-route.ts`, `BlocklyTestView.ts` (`sendMissionTelemetry`) |
| Phía nhận | `experience.mjs` (`applyDigitalTwinDirectMessage`), `direct-scene-profile.ts`, `direct-object-model-json.ts` |

Message bị xoá thì chuyển xuống mục [Message đã bị loại bỏ](#message-đã-bị-loại-bỏ)
thay vì xoá hẳn, để người đọc tài liệu cũ biết nó đã thành cái gì.
