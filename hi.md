# LƯU ĐỒ THUẬT TOÁN — HỆ THỐNG GATE IOT 2 TRẠM

> Áp dụng Quy Tắc Vàng: Chuẩn hình khối • Khép kín vòng lặp • Chia nhỏ Module • Bắt buộc diễn giải bằng lời

---

## Hình 1. Giải Thuật Chính Hệ Thống (Tổng Quát)

```mermaid
flowchart TD
    S([BẮT ĐẦU])
    A["Khởi tạo hệ thống — setup()"]
    B["Bắt đầu chu kỳ quét — loop()"]
    C["M1: Kiểm tra sự kiện cảm biến và nút nhấn"]
    D["M2: Cập nhật Loadcell HX711"]
    E["M3: Xử lý giao tiếp UART 2 trạm"]
    F["M4: Xử lý MQTT — thanh toán VietQR"]
    G["M5: Cập nhật State Machine điều phối cầu"]
    H["M6: Xử lý lệnh Serial console"]
    STOP{"Có lệnh\ndừng hệ thống?"}
    END([KẾT THÚC])

    S --> A --> B
    B --> C --> D --> E --> F --> G --> H --> STOP
    STOP -- Không --> B
    STOP -- Có --> END
```

### Mô tả nguyên lý hoạt động

Hệ thống khởi động qua hàm `setup()`, thực hiện tuần tự toàn bộ các bước khởi tạo phần cứng và thiết lập kết nối. Sau khi hoàn tất, chương trình chuyển sang hàm `loop()` chạy liên tục theo chu kỳ quét.

Mỗi chu kỳ, hệ thống tuần tự xử lý 6 module chức năng: **(M1)** kiểm tra sự kiện từ các cảm biến IO16/IO15/IO17 và nút BTN7; **(M2)** đọc giá trị cân Loadcell HX711 định kỳ 300ms; **(M3)** nhận/gửi frame UART vật lý giữa 2 trạm A–B; **(M4)** duy trì kết nối WiFi và MQTT để xử lý callback thanh toán SePay; **(M5)** cập nhật bộ máy trạng thái điều phối quyền ưu tiên qua cầu; **(M6)** xử lý lệnh điều khiển nhập từ Serial console.

Khối điều kiện "Có lệnh dừng hệ thống?" ở cuối vòng lặp trong thực tế luôn trả về **Không** (không có cơ chế dừng phần cứng), tạo thành chu kỳ quét vô hạn, đảm bảo hệ thống phản ứng theo thời gian thực.

---

## Hình 2. Giải Thuật Module Khởi Động — setup()

```mermaid
flowchart TD
    S([BẮT ĐẦU SETUP])
    A["loadConfig: Đọc cấu hình từ NVS\n(bank BIN, STK, stationId, note)"]
    B["Đọc tham số Loadcell từ NVS\n(lc_scale, lc_offset)"]
    C["Khởi tạo phần cứng:\nTFT ST7789, LED RGB WS2812 IO42,\nServo IO2, AceButton IO16/IO15/IO17/IO7"]
    D["initLoadcell: Khởi tạo HX711 trên IO4/IO5"]
    E{"HX711 sẵn\nsàng?"}
    F["loadcellReady = false\nIn lỗi ra Serial"]
    G["Tare 20 mẫu hoặc set scale từ NVS\nloadcellReady = true"]
    H["startupShowWiFiConnection:\nKết nối WiFi SSID cố định — 12 giây"]
    I{"WiFi kết nối\nthành công?"}
    J["Ghi nhận timeout\nSẽ retry trong loop mỗi 6 giây"]
    K["Đồng bộ NTP UTC+7\n(pool.ntp.org, time.google.com)"]
    L["Khởi tạo Bridge UART:\nIO8=RX, IO18=TX, 115200 baud"]
    M["bridgeUartStartupPingCheck:\nGửi PING, chờ PONG từ trạm đối diện"]
    N{"Nhận được PONG\ntừ peer?"}
    O["Tiếp tục gửi PING, cập nhật TFT chờ"]
    P["startupChooseInitialDirection:\nGiữ BTN7 để tranh quyền XANH trước"]
    Q{"BTN7 giữ\nđủ 5 giây?"}
    R["startupLocalFirst = true\nGửi START:stationId cho peer"]
    T["startupLocalFirst = false\nChờ nhận START từ trạm kia"]
    U["bridgeResetRuntime:\nKhởi tạo FIFO, phase=ACTIVE, đặt đèn"]
    V["Vẽ BridgePanel TFT\nIn hướng dẫn Serial\nChạy test LED RGB"]
    END([KẾT THÚC SETUP])

    S --> A --> B --> C --> D --> E
    E -- Không --> F --> H
    E -- Có --> G --> H
    H --> I
    I -- Không --> J --> L
    I -- Có --> K --> L
    L --> M --> N
    N -- Không --> O --> N
    N -- Có --> P --> Q
    Q -- Có --> R --> U
    Q -- Không --> T --> U
    U --> V --> END
```

### Mô tả nguyên lý hoạt động

Quá trình khởi động diễn ra tuần tự theo từng lớp từ phần cứng đến mạng và giao thức.

Đầu tiên, hệ thống đọc toàn bộ cấu hình từ bộ nhớ NVS (Non-Volatile Storage) gồm thông tin tài khoản VietQR, ID trạm (A hoặc B), và tham số calibration của cân Loadcell. Tiếp theo, tất cả phần cứng được khởi tạo: màn hình TFT, dải LED RGB, servo barie, và các AceButton cho 4 đầu vào cảm biến.

Với HX711 Loadcell: nếu chip **không sẵn sàng** (sai dây hoặc mất nguồn), hệ thống đặt cờ `loadcellReady = false` và in lỗi, nhưng **không dừng khởi động** — hệ thống vẫn tiếp tục, cho phép sử dụng lệnh `LC REINIT` sau. Nếu **sẵn sàng**, hệ thống tare hoặc áp dụng scale đã lưu.

Kết nối WiFi được thử trong 12 giây: nếu **thành công**, đồng bộ đồng hồ NTP để có timestamp cho log Google Sheets; nếu **thất bại**, hệ thống tiếp tục và tự retry mỗi 6 giây trong vòng lặp chính.

Giai đoạn quan trọng nhất là **kiểm tra kết nối 2 trạm**: trạm A gửi PING định kỳ, trạm B lắng nghe và phản hồi PONG. Vòng lặp `N -- Không --> O --> N` thể hiện hệ thống **block tại đây cho đến khi 2 trạm nhận ra nhau**, đảm bảo không có trạm nào hoạt động độc lập khi cấu hình 2 trạm.

Cuối cùng, người vận hành **giữ BTN7 đủ 5 giây** để tranh quyền bật đèn XANH trước. Trạm nào nhấn trước sẽ gửi `START:<stationId>`, trạm còn lại nhận và nhường đèn ĐỎ. Sau khi phân vai xong, `bridgeResetRuntime()` khởi tạo toàn bộ state machine và hệ thống chuyển sang vòng lặp chính.

---

## Hình 3. Giải Thuật Vòng Lặp Chính — loop()

```mermaid
flowchart TD
    S([BẮT ĐẦU CHU KỲ QUÉT])
    A["AceButton.check() cho 4 đầu vào:\nIO16 Loadcell, IO15 Barie, IO17 Exit, IO7 BTN"]
    B{"Có sự kiện\ncảm biến?"}
    C["handleInputEvent: Xử lý sự kiện\n(PRESSED / RELEASED / LONG_PRESSED)"]
    D["updateLoadcell: Đọc HX711 nếu đủ 300ms"]
    E{"Trọng lượng >=\nngưỡng phát hiện?"}
    F["lcTriggered = true\nGọi bridgeTryEntry - LOADCELL"]
    G["bridgeUartTick: Nhận ký tự từ UART peer"]
    H{"Nhận đủ frame\nUART mới?"}
    I["bridgeUartHandleLine:\nXử lý PING/PONG/EXIT/WAIT/SWITCH"]
    J["mqttTick: ensureWiFi + ensureMQTT + mqttClient.loop()"]
    K{"MQTT có\nmessage mới?"}
    L["mqttCallback → handleMqttPaymentPayload"]
    M{"Pending payment\n> 120 giây?"}
    N["Hủy pendingPayment\nVẽ lại BridgePanel"]
    O{"IO16 timer đủ\n700ms?"}
    P["pendingBarieEntryMs = 0\nGọi bridgeTryEntry - IO16"]
    Q["bridgeTick: Kiểm tra hết window N giây\nwaitingEntry auto-commit\nVẽ TFT định kỳ"]
    R{"Có ký tự\ntrên Serial?"}
    T["processSerialIfAvailable → handleCommandLine"]
    STOP{"Có tín hiệu\ndừng?"}
    END([KẾT THÚC])

    S --> A --> B
    B -- Có --> C --> D
    B -- Không --> D
    D --> E
    E -- Có --> F --> G
    E -- Không --> G
    G --> H
    H -- Có --> I --> J
    H -- Không --> J
    J --> K
    K -- Có --> L --> M
    K -- Không --> M
    M -- Có --> N --> O
    M -- Không --> O
    O -- Có --> P --> Q
    O -- Không --> Q
    Q --> R
    R -- Có --> T --> STOP
    R -- Không --> STOP
    STOP -- Không --> S
    STOP -- Có --> END
```

### Mô tả nguyên lý hoạt động

Vòng lặp `loop()` là trung tâm điều hành của toàn bộ hệ thống, chạy liên tục không ngắt nghỉ với tốc độ hàng trăm lần mỗi giây.

**Giai đoạn 1 — Cảm biến:** Bốn đối tượng AceButton được polling. Khi phát hiện sự kiện thay đổi trạng thái (nhấn/nhả/giữ lâu), `handleInputEvent()` được gọi ngay lập tức để xử lý đúng loại tín hiệu.

**Giai đoạn 2 — Loadcell:** `updateLoadcell()` chỉ đọc HX711 khi đủ 300ms từ lần đọc trước (tránh blocking). Nếu trọng lượng vượt ngưỡng `minDetectWeight` và đã ổn định 1 giây (lcSettled), hệ thống tự động kích hoạt luồng xe vào.

**Giai đoạn 3 — UART:** Đọc từng ký tự từ `bridgeUart` và tích lũy vào buffer. Khi gặp ký tự `\n` (kết thúc frame), toàn bộ frame được phân tích để xử lý lệnh PING/PONG/EXIT/WAIT/SWITCH từ trạm đối diện.

**Giai đoạn 4 — MQTT:** Gọi `mqttClient.loop()` để xử lý các message đến. Khi SePay gửi webhook xác nhận thanh toán, callback `mqttCallback()` được kích hoạt ngay trong giai đoạn này.

**Giai đoạn 5 — Bộ định thời:** Kiểm tra hai timer: (a) nếu QR đang chờ quá 120 giây mà chưa được quét — hủy giao dịch; (b) nếu IO16 đã phát hiện xe đủ 700ms — đọc cân và tiến hành entry.

**Giai đoạn 6 — State Machine:** `bridgeTick()` kiểm tra xem cửa sổ thời gian N giây đã hết chưa, có xe đã thanh toán đang chờ vào không, và cập nhật màn hình TFT.

Mũi tên `STOP -- Không --> S` thể hiện chu trình quét vô tận, đặc trưng của hệ thống nhúng thời gian thực.

---

## Hình 4. Giải Thuật Phát Hiện Xe và Thanh Toán VietQR

```mermaid
flowchart TD
    S([BẮT ĐẦU — Kích hoạt Entry])
    SRC{"Nguồn kích\nhoạt?"}
    IO16["IO16 kích hoạt:\nSet pendingBarieEntryMs\nChờ thêm 700ms để xe ổn định trên cân"]
    LC_AUTO["Loadcell auto-detect:\nSmooth value >= minDetectWeight\nVà đã ổn định >= 1 giây"]
    BTN7["BTN7 giữ 5 giây:\nNgười vận hành xác nhận thủ công"]

    CHECK["bridgeTryEntry: Kiểm tra điều kiện vào cầu"]
    CK1{"localGreen\n== true?"}
    BLK1["BLOCK: Đèn ĐỎ\nIn log ENTRY BLOCK"]
    CK2{"phase ==\nBRIDGE_ACTIVE?"}
    BLK2["BLOCK: Đang xả xe\nIn log ENTRY BLOCK"]
    CK3{"Loadcell\nsẵn sàng?"}
    BLK3["BLOCK: Cảm biến lỗi\nIn log ENTRY BLOCK"]
    RD["Đọc trọng lượng xe — 5 mẫu\nvehicleWeight = readLoadcellUnits(5)"]
    CK4{"weight >=\nminDetectWeight?"}
    BLK4["BLOCK: Không có xe thực sự\nIn log ENTRY BLOCK"]
    CK5{"bridgeWeight +\nweight <= maxWeight?"}
    BLK5["sensorOverloadActive = true\nHiển thị cảnh báo QUÁ TẢI trên TFT"]

    PAY["startPendingPayment:\nSinh mã DH########\nTính giá theo bảng cước 3 mức"]
    QR["showQrForAmount:\nTạo payload VietQR EMVCo\nVẽ mã QR lên màn hình TFT 240x320"]
    WAIT_PAY{"Nhận MQTT\ncallback thanh toán?"}
    TIMEOUT{"Chờ > 120s\nmà chưa có?"}
    CANCEL["Hủy pendingPayment\nVẽ lại BridgePanel"]

    CK_TYPE{"transferType\n== in?"}
    CK_CODE{"content chứa\nmã giao dịch?"}
    CK_AMT{"amount >=\nsố tiền yêu cầu?"}
    CK_CAP{"bridgeWeight +\nweight <= maxWeight?"}
    WAIT_CAP["waitingEntry = true\nHiển thị màn hình CHỜ TRỌNG TẢI\nTự động vào khi xe khác ra"]
    COMMIT["bridgeCommitEntry:\nPush FIFO queue\nMở barie — servo 90°\nbridgeWeight += vehicleWeight"]
    PUB["mqttPublishStatus: enter\nGửi trạng thái lên broker"]
    LOG["logToSheets: Ghi log\nGoogle Sheets qua HTTPS\n(FreeRTOS task riêng — non-blocking)"]
    IO15{"IO15 sensor:\nXe qua barie?"}
    CLOSE["setBarrierOpen false:\nĐóng barie — servo 0°"]
    IGNORE([Bỏ qua — Không hành động])
    END([KẾT THÚC — Xe đã vào cầu])

    S --> SRC
    SRC -- IO16 --> IO16 --> CHECK
    SRC -- Loadcell --> LC_AUTO --> CHECK
    SRC -- BTN7 --> BTN7 --> CHECK

    CHECK --> CK1
    CK1 -- Không --> BLK1 --> IGNORE
    CK1 -- Có --> CK2
    CK2 -- Không --> BLK2 --> IGNORE
    CK2 -- Có --> CK3
    CK3 -- Không --> BLK3 --> IGNORE
    CK3 -- Có --> RD --> CK4
    CK4 -- Không --> BLK4 --> IGNORE
    CK4 -- Có --> CK5
    CK5 -- Không --> BLK5 --> IGNORE
    CK5 -- Có --> PAY --> QR --> WAIT_PAY

    WAIT_PAY -- Chưa có --> TIMEOUT
    TIMEOUT -- Chưa đủ --> WAIT_PAY
    TIMEOUT -- Đã đủ --> CANCEL --> IGNORE

    WAIT_PAY -- Có --> CK_TYPE
    CK_TYPE -- Không --> IGNORE
    CK_TYPE -- Có --> CK_CODE
    CK_CODE -- Không --> IGNORE
    CK_CODE -- Có --> CK_AMT
    CK_AMT -- Không --> IGNORE
    CK_AMT -- Có --> CK_CAP
    CK_CAP -- Không --> WAIT_CAP --> IO15
    CK_CAP -- Có --> COMMIT --> PUB --> LOG --> IO15
    IO15 -- Có --> CLOSE --> END
    IO15 -- Không --> IO15
```

### Mô tả nguyên lý hoạt động

Luồng phát hiện xe và thanh toán là luồng nghiệp vụ trung tâm, được kích hoạt từ 3 nguồn khác nhau.

**Nguồn 1 — IO16 (tự động):** Khi cảm biến vật lý tại vị trí cân phát hiện xe, hệ thống không đọc cân ngay mà chờ thêm 700ms để xe ổn định trên bàn cân trước khi đọc giá trị chính xác.

**Nguồn 2 — Loadcell auto-detect:** Module `updateLoadcell()` liên tục giám sát giá trị cân. Khi trọng lượng smooth vượt ngưỡng `minDetectWeight` (50g) và đã ổn định liên tục >= 1 giây, hệ thống tự động kích hoạt. Cơ chế này bổ sung cho IO16 trong trường hợp cảm biến vật lý gặp sự cố.

**Nguồn 3 — BTN7 thủ công:** Người vận hành giữ nút 5 giây để xác nhận thủ công, dùng trong trường hợp test hoặc xe không lên cân đúng cách.

**Chuỗi kiểm tra điều kiện** trong `bridgeTryEntry()` thực hiện 4 kiểm tra lồng nhau: (1) đèn phải XANH; (2) đang ở pha ACTIVE không phải xả xe; (3) cảm biến cân hoạt động; (4) trọng lượng đủ lớn để nhận diện là xe thật. Nếu **bất kỳ điều kiện nào sai** — luồng kết thúc với BLOCK, không tạo giao dịch.

**Kiểm tra tải trọng:** Nếu cầu đã đầy tải, hệ thống **không từ chối xe** mà chuyển sang chế độ `waitingEntry`, tức là xe đã được phép thanh toán nhưng barie chưa mở — hệ thống sẽ tự động mở barie khi có xe khác ra khỏi cầu giải phóng tải trọng.

**Thanh toán VietQR:** Mã QR được tạo theo chuẩn EMVCo với payload chứa thông tin ngân hàng, số tài khoản, số tiền và mã giao dịch DH########. Hệ thống chờ webhook từ SePay qua MQTT. Khi nhận được, 4 lớp xác thực được kiểm tra: loại giao dịch, mã khớp, số tiền đủ, và tải trọng cầu. Sau khi thành công, FIFO được cập nhật, barie mở, và log được ghi bất đồng bộ lên Google Sheets qua FreeRTOS task riêng.

---

## Hình 5. Giải Thuật State Machine Điều Phối Cầu

```mermaid
flowchart TD
    S([BẮT ĐẦU — bridgeResetRuntime])
    INIT["Khởi tạo:\nphase = BRIDGE_ACTIVE\nlocalGreen = startupLocalFirst\nFIFO = rỗng, bridgeWeight = 0"]

    CHK_GREEN{"localGreen\n== true?"}
    GREEN_STATE["ĐÈN XANH — Nhận xe\nsetRgbColor: 0, 255, 0\nChờ cảm biến hoặc Loadcell"]

    CHK_WIN{"Cửa sổ N giây\nđã hết?"}
    TO_DRAIN["phase = BRIDGE_WAIT_DRAIN\nĐổi đèn sang ĐỎ\nGửi WAIT:peerId qua UART"]
    RESEND_WAIT{"Đủ 2 giây\nchưa gửi lại?"}
    SEND_WAIT_AGAIN["Gửi lại WAIT:peerId\n(retry tránh mất gói UART)"]

    CHK_DRAIN{"phase ==\nWAIT_DRAIN và\nfifoCount == 0?"}
    SWITCH_DIR["localGreen = false\nGửi SWITCH:peerId qua UART\nĐèn ĐỎ — không nhận xe"]

    RED_STATE["ĐÈN ĐỎ — Chờ peer\nsetRgbColor: 255, 0, 0"]
    RCV_WAIT{"Nhận WAIT\ntừ peer?"}
    YELLOW_STATE["ĐÈN VÀNG — Chuẩn bị\nphase = BRIDGE_WAIT_DRAIN\nsetRgbColor: 255, 180, 0"]
    RCV_SWITCH{"Nhận SWITCH\ntừ peer?"}
    GET_GREEN["localGreen = true\nphase = BRIDGE_ACTIVE\nReset phaseStartMs\nĐèn XANH"]

    EXIT_EVT["Nhận EXIT từ peer:\nbridgeTryExit\nFIFO pop, bridgeWeight giảm"]
    CHK_EMPTY{"fifoCount == 0\nvà phase == WAIT_DRAIN?"}
    DO_SWITCH["Thực hiện chuyển hướng\nbridgeSwitchDirectionIfNeeded"]

    STOP{"Có lệnh\ndừng?"}
    END([KẾT THÚC])

    S --> INIT --> CHK_GREEN

    CHK_GREEN -- Có --> GREEN_STATE --> CHK_WIN
    CHK_WIN -- Chưa hết --> RESEND_WAIT
    RESEND_WAIT -- Không --> CHK_DRAIN
    RESEND_WAIT -- Có --> SEND_WAIT_AGAIN --> CHK_DRAIN
    CHK_WIN -- Đã hết --> TO_DRAIN --> CHK_DRAIN

    CHK_DRAIN -- Có --> SWITCH_DIR --> RED_STATE
    CHK_DRAIN -- Không --> STOP

    CHK_GREEN -- Không --> RED_STATE
    RED_STATE --> RCV_WAIT
    RCV_WAIT -- Có --> YELLOW_STATE --> RCV_SWITCH
    RCV_WAIT -- Không --> EXIT_EVT
    RCV_SWITCH -- Có --> GET_GREEN --> GREEN_STATE
    RCV_SWITCH -- Không --> EXIT_EVT

    EXIT_EVT --> CHK_EMPTY
    CHK_EMPTY -- Có --> DO_SWITCH --> STOP
    CHK_EMPTY -- Không --> STOP

    STOP -- Không --> CHK_GREEN
    STOP -- Có --> END
```

### Mô tả nguyên lý hoạt động

State Machine điều phối cầu là thuật toán quyết định **ai được phép cho xe vào** tại mỗi thời điểm, đảm bảo mutual exclusion giữa hai chiều lưu thông.

**Giai đoạn XANH (BRIDGE_ACTIVE + localGreen=true):** Trạm đang có quyền ưu tiên, đèn XANH, barie có thể mở. Trong giai đoạn này, bộ đếm thời gian cửa sổ N giây (mặc định 120s) đang chạy. Hệ thống tiếp tục nhận xe miễn là còn tải trọng và còn thời gian.

**Chuyển sang xả xe:** Khi cửa sổ N giây hết, hệ thống chuyển `phase = BRIDGE_WAIT_DRAIN`, đèn đổi sang ĐỎ, và gửi lệnh `WAIT:<peerId>` cho trạm đối diện. Lệnh WAIT được gửi lại mỗi 2 giây như cơ chế retry, tránh trường hợp gói UART bị mất khiến trạm đối diện bị kẹt ĐỎ mãi.

**Điều kiện chuyển hướng:** Hệ thống chỉ thực sự nhường quyền khi `fifoCount == 0` — tức là **không còn xe nào đang trên cầu**. Khi đó `SWITCH:<peerId>` được gửi đi, trạm này về ĐỎ hoàn toàn.

**Giai đoạn ĐỎ (localGreen=false):** Trạm đang chờ. Khi nhận `WAIT`, đổi sang VÀNG (báo hiệu sắp được chuyển). Khi nhận `SWITCH`, chuyển về XANH và bắt đầu cửa sổ N giây mới.

**Sự kiện EXIT:** Bất cứ khi nào nhận được `EXIT` từ peer (xe đã sang đến trạm đối diện), FIFO được pop và tải trọng cầu giảm. Nếu sau EXIT mà cầu vừa trống và đang ở pha WAIT_DRAIN — chuyển hướng ngay lập tức mà không cần chờ tiếp.

---

## Hình 6. Giải Thuật Giao Tiếp UART Giữa 2 Trạm

```mermaid
flowchart TD
    S([BẮT ĐẦU — bridgeUartTick])
    READ["Đọc từng ký tự từ bridgeUart\nTích lũy vào bridgeRxLine"]
    CHK_EOL{"Gặp ký tự\nnewline?"}
    PARSE["bridgeUartHandleLine:\nPhân tích frame nhận được"]
    CHK_FRAME{"Loại\nframe?"}

    PING["PING: Cập nhật bridgePeerAlive\nPhản hồi PONG:stationId"]
    PONG["PONG: Cập nhật bridgePeerAlive\nbridgeLastPeerRxMs = millis()"]
    START_F["START:X: Phân vai khởi động\nstartupRoleLocked = true"]
    EXIT_F["EXIT:X: Xe từ trạm X đã sang\nbridgeTryExit — FIFO pop — giảm tải trọng"]
    WAIT_F["WAIT:stationId: Peer đang xả xe\nphase = WAIT_DRAIN, đèn VÀNG"]
    SWITCH_F["SWITCH:stationId: Nhận quyền XANH\nlocalGreen = true\nReset phaseStartMs"]

    CHK_A{"stationId == A?"}
    SEND_PING{"Đủ 1 giây\ntừ lần gửi trước?"}
    DO_PING["Gửi PING:A qua UART\nbridgeLastPingMs = millis()"]

    CHK_ALIVE{"bridgePeerAlive == true\nvà > 5s không nhận gói?"}
    SET_DOWN["bridgePeerAlive = false\nIn log: peer link DOWN"]

    STOP{"Có tín hiệu\ndừng?"}
    END([KẾT THÚC])

    S --> READ --> CHK_EOL
    CHK_EOL -- Không --> CHK_A
    CHK_EOL -- Có --> PARSE --> CHK_FRAME

    CHK_FRAME -- PING --> PING --> CHK_A
    CHK_FRAME -- PONG --> PONG --> CHK_A
    CHK_FRAME -- START --> START_F --> CHK_A
    CHK_FRAME -- EXIT --> EXIT_F --> CHK_A
    CHK_FRAME -- WAIT --> WAIT_F --> CHK_A
    CHK_FRAME -- SWITCH --> SWITCH_F --> CHK_A

    CHK_A -- Không --> CHK_ALIVE
    CHK_A -- Có --> SEND_PING
    SEND_PING -- Không --> CHK_ALIVE
    SEND_PING -- Có --> DO_PING --> CHK_ALIVE

    CHK_ALIVE -- Có --> SET_DOWN --> STOP
    CHK_ALIVE -- Không --> STOP

    STOP -- Không --> S
    STOP -- Có --> END
```

### Mô tả nguyên lý hoạt động

Module giao tiếp UART là dây thần kinh kết nối 2 trạm A–B, sử dụng cổng `HardwareSerial(1)` trên chân IO8 (RX) và IO18 (TX) với tốc độ 115200 baud.

**Cơ chế nhận frame:** Hệ thống đọc từng ký tự và tích lũy vào buffer chuỗi `bridgeRxLine`. Khi gặp ký tự xuống dòng `\n`, toàn bộ chuỗi được đưa vào hàm phân tích `bridgeUartHandleLine()`. Buffer bị giới hạn 120 ký tự để tránh tràn bộ nhớ.

**Xử lý từng loại frame:**
- `PING` — Trạm A gửi mỗi 1 giây; trạm B nhận và phản hồi `PONG` ngay lập tức, đồng thời cập nhật timestamp `bridgeLastPeerRxMs`.
- `PONG` — Trạm A nhận, cập nhật trạng thái `bridgePeerAlive = true`.
- `START:X` — Dùng trong giai đoạn khởi động để phân vai ai được XANH trước.
- `EXIT:X` — Báo xe từ trạm X đã sang trạm đối diện. Trạm nhận thực hiện `bridgeTryExit()`: pop FIFO, giảm `bridgeWeight`, kiểm tra xem có nên chuyển hướng không.
- `WAIT:X` — Trạm đang XANH thông báo sắp nhường quyền. Trạm X nhận tín hiệu, chuyển đèn VÀNG, chuẩn bị nhận `SWITCH`.
- `SWITCH:X` — Cầu đã trống, trạm X được nhận quyền XANH. Trạm X reset bộ đếm cửa sổ thời gian và bắt đầu chu kỳ mới.

**Heartbeat và phát hiện lỗi:** Chỉ **trạm A** chủ động gửi PING mỗi 1 giây. Nếu hệ thống không nhận bất kỳ gói nào từ peer trong vòng **5 giây liên tiếp**, cờ `bridgePeerAlive` được hạ xuống `false`. Trạng thái này hiển thị trên TFT và Serial để người vận hành biết link đã bị mất — tuy nhiên hiện tại firmware **chưa có cơ chế fail-safe tự động** (đặt đèn ĐỎ cả 2 trạm) khi link down, đây là điểm cần hoàn thiện.

---

## Tổng Kết Phân Tầng Lưu Đồ

| Hình | Tên module | Mức độ chi tiết |
|------|-----------|----------------|
| Hình 1 | Giải thuật tổng quát | Kiến trúc hệ thống |
| Hình 2 | Module khởi động setup() | Trình tự khởi tạo |
| Hình 3 | Vòng lặp chính loop() | Chu kỳ quét thời gian thực |
| Hình 4 | Phát hiện xe và thanh toán VietQR | Luồng nghiệp vụ chính |
| Hình 5 | State Machine điều phối cầu | Logic phân quyền |
| Hình 6 | Giao tiếp UART 2 trạm | Giao thức truyền thông |
