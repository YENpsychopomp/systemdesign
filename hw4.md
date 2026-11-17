# 小組作業 4：UML 設計

115-1 系統分析與設計 第 5 組｜題目：移動秘書

組長：蔡逸誠｜組員：顏仕祁、林育伸、呂孟澤、施誠軒

## 0. 文件說明

本文件延續作業 3 的需求分析（功能性需求 FR、使用案例 UC），進一步以 UML 描述系統的靜態結構與動態行為。

**本文件涵蓋：**

1. UML 類別圖：分為「領域模型」（系統要保存的資料）與「設計類別」（負責處理流程的控制器、服務與外部服務介面）兩張。
2. 循序圖與活動圖：針對 4 個使用案例，各自繪製一張循序圖與一張活動圖。

| 使用案例 | 選擇原因 | 循序圖 | 活動圖 |
|---|---|---|---|
| UC03 上傳會議音檔（含 UC04、UC05） | 系統核心流程，串接 ASR 與 LLM 兩個外部服務 | 2.1.1 | 2.1.2 |
| UC06 檢視與編輯摘要 | 使用者最常操作的頁面，含權限檢查與資料驗證 | 2.2.1 | 2.2.2 |
| UC08 傳送摘要至 LINE | 自動化整合功能，含綁定檢查與失敗重試 | 2.3.1 | 2.3.2 |
| UC09 任務寫入行事曆 | 自動化整合功能，需逐筆處理多項任務 | 2.4.1 | 2.4.2 |

**設計上的調整：** 作業 3 的處理狀態為「等待中、辨識中、完成、失敗」。設計時在辨識完成後另外加入「摘要中」狀態，讓使用者能分辨目前是 ASR 還是 LLM 在處理，失敗時也能判斷要從哪一步重新處理。

---

## 1. UML 類別圖

### 1.1 領域模型（實體類別）

描述系統需要長期保存的資料，以及資料之間的關係。

```mermaid
classDiagram
    direction LR

    class User {
        -String userId
        -String email
        -String passwordHash
        -DateTime createdAt
        +verifyPassword(String raw) bool
        +hasBinding(Channel channel) bool
    }

    class ExternalBinding {
        <<abstract>>
        -String bindingId
        -String accessToken
        -String refreshToken
        -DateTime expiresAt
        -DateTime boundAt
        +isValid() bool
        +renew() void
        +unbind() void
    }

    class LineBinding {
        -String lineUserId
    }

    class CalendarBinding {
        -String calendarId
        -String timeZone
    }

    class Meeting {
        -String meetingId
        -String title
        -Date meetingDate
        -ProcessStatus status
        -DateTime createdAt
        +updateStatus(ProcessStatus status) void
        +isOwnedBy(String userId) bool
        +canReprocess() bool
    }

    class AudioFile {
        -String fileId
        -String fileName
        -AudioFormat format
        -long sizeBytes
        -int durationSec
        -String storagePath
        -DateTime uploadedAt
        +validate() bool
    }

    class Transcript {
        -String transcriptId
        -String content
        -String language
        -DateTime createdAt
    }

    class Summary {
        -String summaryId
        -List~String~ keyPoints
        -List~String~ decisions
        -DateTime generatedAt
        -DateTime editedAt
        +edit(keyPoints, decisions) void
        +toLineMessage() String
    }

    class Task {
        -String taskId
        -String content
        -String assignee
        -Date dueDate
        -TaskStatus status
        -String calendarEventId
        +edit(content, assignee, dueDate) void
        +updateStatus(TaskStatus status) void
        +hasDueDate() bool
        +isInCalendar() bool
    }

    class DeliveryRecord {
        -String recordId
        -Channel channel
        -bool success
        -int attempts
        -String errorMessage
        -DateTime sentAt
    }

    class ProcessStatus {
        <<enumeration>>
        WAITING
        RECOGNIZING
        SUMMARIZING
        COMPLETED
        FAILED
    }

    class TaskStatus {
        <<enumeration>>
        TODO
        IN_PROGRESS
        DONE
    }

    class AudioFormat {
        <<enumeration>>
        MP3
        WAV
        M4A
    }

    class Channel {
        <<enumeration>>
        LINE
        CALENDAR
    }

    User "1" *-- "0..*" Meeting : 擁有
    User "1" *-- "0..2" ExternalBinding : 綁定
    ExternalBinding <|-- LineBinding
    ExternalBinding <|-- CalendarBinding
    Meeting "1" *-- "1" AudioFile : 包含
    Meeting "1" *-- "0..1" Transcript : 產生
    Meeting "1" *-- "0..1" Summary : 產生
    Meeting "1" *-- "0..*" Task : 擷取
    Summary "1" --> "0..*" DeliveryRecord : LINE 傳送紀錄
    Task "1" --> "0..*" DeliveryRecord : 行事曆寫入紀錄
    Meeting ..> ProcessStatus
    Task ..> TaskStatus
    AudioFile ..> AudioFormat
    DeliveryRecord ..> Channel
```

| 類別 | 中文名稱 | 說明 | 對應需求 |
|---|---|---|---|
| User | 使用者 | 系統帳號，密碼只保存雜湊值。 | FR-01、FR-02、NFR-03 |
| ExternalBinding | 外部帳號綁定（抽象類別） | 保存外部服務的授權資訊，由 LineBinding 與 CalendarBinding 繼承。 | FR-03 |
| LineBinding／CalendarBinding | LINE 綁定／行事曆綁定 | 各服務特有的欄位，例如 LINE 使用者 ID、行事曆 ID 與時區。 | FR-03 |
| Meeting | 會議 | 一次上傳產生一場會議，是音檔、逐字稿、摘要與任務的擁有者。 | FR-04～FR-06 |
| AudioFile | 音檔 | 上傳的原始音檔與儲存位置。 | FR-04 |
| Transcript | 逐字稿 | ASR 辨識結果。 | FR-05 |
| Summary | 會議摘要 | LLM 整理出的討論重點與決議事項，使用者可編輯。 | FR-07、FR-09 |
| Task | 待辦任務 | 由 LLM 擷取或使用者新增，`calendarEventId` 有值表示已寫入行事曆。 | FR-08、FR-10～FR-12 |
| DeliveryRecord | 傳送紀錄 | 記錄每次傳送至 LINE 或寫入行事曆的結果與重試次數，供失敗通知與重送使用。 | FR-13～FR-15 |

**關係說明：**

- **組合（實心菱形）：** 會議、外部綁定屬於使用者；音檔、逐字稿、摘要、任務屬於會議。刪除上層物件時，下層資料一併刪除，對應 NFR-04「刪除後同步刪除相關資料」。
- **一般化（空心三角形）：** LINE 綁定與行事曆綁定共用授權欄位與 `isValid()`、`unbind()` 等方法。
- **多重性：** 一位使用者最多綁定 2 個外部帳號（LINE、行事曆各一）；逐字稿與摘要在處理完成前不存在，因此為 `0..1`。

### 1.2 設計類別（控制器、服務與外部服務介面）

描述處理使用者請求的類別。資料存取一律透過 Repository 類別，為避免圖面過於複雜，Repository 不畫出。

```mermaid
classDiagram
    direction LR

    class AccountController {
        +register(email, password) User
        +login(email, password) Token
        +logout() void
        +bind(Channel channel, authCode) ExternalBinding
        +unbind(Channel channel) void
    }

    class MeetingController {
        +createMeeting(title, date, file) Meeting
        +getStatus(meetingId) ProcessStatus
        +reprocess(meetingId) void
        +searchMeetings(keyword, date) List~Meeting~
        +deleteMeeting(meetingId) void
    }

    class SummaryController {
        +getSummary(meetingId) SummaryView
        +updateSummary(meetingId, SummaryForm form) Summary
        +sendToLine(meetingId) DeliveryRecord
    }

    class TaskController {
        +listTasks(TaskFilter filter) List~Task~
        +updateTask(taskId, TaskForm form) Task
        +addToCalendar(List~String~ taskIds) CalendarResult
    }

    class AuthService {
        +hashPassword(raw) String
        +issueToken(User user) Token
        +currentUser() User
    }

    class AudioStorageService {
        +save(file) String
        +delete(path) void
    }

    class MeetingProcessingService {
        +process(meetingId) void
        -runAsr(Meeting meeting) Transcript
        -runSummary(Transcript transcript) SummaryResult
    }

    class IntegrationService {
        +sendSummaryToLine(userId, summaryId) DeliveryRecord
        +addTasksToCalendar(userId, taskIds) CalendarResult
    }

    class RetryExecutor {
        -int maxAttempts = 3
        -int backoffSec
        +execute(action) Result
    }

    class SpeechRecognizer {
        <<interface>>
        +transcribe(AudioFile audio) Transcript
    }

    class Summarizer {
        <<interface>>
        +summarize(Transcript transcript) SummaryResult
    }

    class LineGateway {
        <<interface>>
        +createNote(LineBinding binding, String text) void
    }

    class CalendarGateway {
        <<interface>>
        +createEvent(CalendarBinding binding, String title, Date due) String
    }

    class IAIAsrAdapter {
        -String apiKey
        -String endpoint
    }

    class LlmAdapter {
        -String model
        -String promptVersion
    }

    class LineAdapter {
        -String channelToken
    }

    class CalendarAdapter {
        -String clientId
    }

    class SummaryResult {
        +Summary summary
        +List~Task~ tasks
    }

    class CalendarResult {
        +List~String~ succeeded
        +List~String~ failed
        +List~String~ missingDueDate
    }

    AccountController --> AuthService
    MeetingController --> AudioStorageService
    MeetingController --> MeetingProcessingService
    SummaryController --> IntegrationService
    TaskController --> IntegrationService
    MeetingProcessingService --> SpeechRecognizer
    MeetingProcessingService --> Summarizer
    MeetingProcessingService --> RetryExecutor
    IntegrationService --> LineGateway
    IntegrationService --> CalendarGateway
    IntegrationService --> RetryExecutor
    SpeechRecognizer <|.. IAIAsrAdapter
    Summarizer <|.. LlmAdapter
    LineGateway <|.. LineAdapter
    CalendarGateway <|.. CalendarAdapter
    Summarizer ..> SummaryResult
    IntegrationService ..> CalendarResult
```

| 層級 | 類別 | 職責 |
|---|---|---|
| 控制器 | AccountController、MeetingController、SummaryController、TaskController | 接收前端 API 請求、檢查登入與資料擁有者，再交給服務處理。 |
| 服務 | AuthService | 密碼雜湊、發放登入憑證、取得目前登入的使用者。 |
| 服務 | AudioStorageService | 儲存與刪除音檔。 |
| 服務 | MeetingProcessingService | 背景執行「辨識 → 摘要」流程並更新會議狀態。 |
| 服務 | IntegrationService | 傳送摘要至 LINE、將任務寫入行事曆，並記錄 DeliveryRecord。 |
| 服務 | RetryExecutor | 統一處理外部服務呼叫失敗時的重試，最多 3 次（NFR-07）。 |
| 介面 | SpeechRecognizer、Summarizer、LineGateway、CalendarGateway | 外部服務的抽象介面，服務只依賴介面，之後更換 ASR 或 LLM 供應商時只需新增 Adapter（NFR-08）。 |
| 轉接器 | IAIAsrAdapter、LlmAdapter、LineAdapter、CalendarAdapter | 實作介面，負責實際呼叫 iAI ASR、LLM、LINE、行事曆的 API。 |

---

## 2. 循序圖與活動圖

- **循序圖** 的參與者與第 1.2 節的類別名稱一致；「資料庫」代表各 Repository 與資料庫。`loop` 表示重試或逐筆處理，`alt` 表示條件分支，`par` 表示同時進行。
- **活動圖** 以顏色區分執行者：綠色為使用者、藍色為移動秘書系統、橘色為外部服務；菱形為判斷，雙框方塊為呼叫其他使用案例。

### 2.1 UC03 上傳會議音檔（含 UC04、UC05）

#### 2.1.1 循序圖

上傳完成後系統立即回應，辨識與摘要在背景執行；前端同時定期查詢處理狀態。

```mermaid
sequenceDiagram
    autonumber
    actor U as 使用者
    participant UI as 前端網頁
    participant MC as MeetingController
    participant ST as AudioStorageService
    participant PS as MeetingProcessingService
    participant DB as 資料庫
    participant ASR as iAI ASR 服務
    participant LLM as LLM 服務

    U->>UI: 輸入會議名稱、日期並選擇音檔
    UI->>UI: 檢查副檔名與檔案大小
    alt 格式不支援或檔案過大
        UI-->>U: 顯示錯誤原因
    else 檢查通過
        UI->>MC: POST /meetings（音檔、名稱、日期）
        UI-->>U: 顯示上傳進度
        MC->>MC: AudioFile.validate()
        MC->>ST: save(file)
        ST-->>MC: storagePath
        MC->>DB: 新增 Meeting（WAITING）與 AudioFile
        MC-)PS: process(meetingId)（非同步）
        MC-->>UI: 202 Accepted（meetingId）
        UI-->>U: 顯示「等待中」

        par 背景處理
            PS->>DB: 更新狀態為 RECOGNIZING
            loop RetryExecutor：失敗時重試，最多 3 次
                PS->>ASR: transcribe(audio)
                ASR-->>PS: 逐字稿或錯誤
            end
            alt 辨識成功
                PS->>DB: 儲存 Transcript，狀態改為 SUMMARIZING
                loop RetryExecutor：失敗時重試，最多 3 次
                    PS->>LLM: summarize(transcript)
                    LLM-->>PS: 摘要與待辦任務（JSON）
                end
                alt 摘要成功
                    PS->>DB: 儲存 Summary 與 Task，狀態改為 COMPLETED
                else 重試 3 次仍失敗
                    PS->>DB: 狀態改為 FAILED（保留逐字稿）
                end
            else 重試 3 次仍失敗
                PS->>DB: 狀態改為 FAILED（保留音檔）
            end
        and 前端輪詢狀態
            loop 每 5 秒，直到 COMPLETED 或 FAILED
                UI->>MC: GET /meetings/{id}/status
                MC->>DB: 查詢 Meeting.status
                DB-->>MC: status
                MC-->>UI: status
                UI-->>U: 更新狀態顯示
            end
        end

        alt 狀態為 COMPLETED
            UI-->>U: 通知完成，顯示摘要頁連結
            U->>UI: 點選連結進入摘要頁面
        else 狀態為 FAILED
            UI-->>U: 通知失敗，顯示「重新處理」按鈕
        end
    end
```

#### 2.1.2 活動圖

```mermaid
flowchart TD
    S(("開始")) --> A1["點選「新增會議」"]
    A1 --> A2["輸入會議名稱、日期<br/>並選擇音檔"]
    A2 --> B1{"格式為 mp3／wav／m4a<br/>且未超過大小限制？"}
    B1 -->|否| B2["顯示錯誤原因"]
    B2 --> A2
    B1 -->|是| B3["上傳音檔並顯示進度"]
    B3 --> B4{"上傳完成？"}
    B4 -->|否，中斷| B5["保留已填資料<br/>提示重新上傳"]
    B5 --> A2
    B4 -->|是| B6["建立會議<br/>狀態：等待中"]

    B6 --> B7["呼叫 iAI ASR<br/>狀態：辨識中"]
    B7 --> X1["iAI ASR 將音檔<br/>轉為逐字稿"]
    X1 --> C1{"辨識成功？"}
    C1 -->|否| C2{"已重試 3 次？"}
    C2 -->|否| B7
    C2 -->|是| F1["狀態：失敗<br/>保留音檔"]
    C1 -->|是| B8["儲存逐字稿<br/>狀態：摘要中"]

    B8 --> B9["呼叫 LLM 服務"]
    B9 --> X2["LLM 產生摘要<br/>並擷取待辦任務"]
    X2 --> D1{"產生成功？"}
    D1 -->|否| D2{"已重試 3 次？"}
    D2 -->|否| B9
    D2 -->|是| F2["狀態：失敗<br/>保留逐字稿"]
    D1 -->|是| B10["儲存摘要與任務<br/>狀態：完成"]

    B10 --> B11["通知使用者處理完成"]
    B11 --> A3["點選連結進入摘要頁面"]
    A3 --> E((("結束")))

    F1 --> N1["通知使用者處理失敗"]
    F2 --> N1
    N1 --> A4{"是否重新處理？"}
    A4 -->|否| E
    A4 -->|是| R1{"已有逐字稿？"}
    R1 -->|是| B9
    R1 -->|否| B7

    classDef user fill:#e3f4e1,stroke:#3d8b37,color:#000
    classDef sys fill:#dbe7f5,stroke:#4a78b5,color:#000
    classDef ext fill:#fde9d4,stroke:#d07a1f,color:#000
    classDef term fill:#333,stroke:#333,color:#fff
    class A1,A2,A3,A4 user
    class B1,B2,B3,B4,B5,B6,B7,B8,B9,B10,B11,C1,C2,D1,D2,F1,F2,N1,R1 sys
    class X1,X2 ext
    class S,E term
```

### 2.2 UC06 檢視與編輯摘要

#### 2.2.1 循序圖

```mermaid
sequenceDiagram
    autonumber
    actor U as 使用者
    participant UI as 前端網頁
    participant SC as SummaryController
    participant AS as AuthService
    participant DB as 資料庫

    U->>UI: 從會議列表選擇一場會議
    UI->>SC: GET /meetings/{id}/summary
    SC->>AS: currentUser()
    AS-->>SC: user
    SC->>DB: 查詢 Meeting、Transcript、Summary、Task
    DB-->>SC: 會議資料
    SC->>SC: meeting.isOwnedBy(user.userId)<br/>且 status == COMPLETED

    alt 非本人或尚未處理完成
        SC-->>UI: 403 / 409
        UI-->>U: 顯示無權限或「尚未處理完成」
    else 檢查通過
        SC-->>UI: 逐字稿、摘要、任務
        UI-->>U: 顯示內容
        U->>UI: 點選「編輯」，修改摘要或任務

        alt 放棄修改
            U->>UI: 點選「取消」
            UI-->>U: 還原為原內容（不送出請求）
        else 儲存
            U->>UI: 點選「儲存」
            UI->>SC: PUT /meetings/{id}/summary（摘要、任務）
            SC->>SC: 驗證欄位（期限格式、必填）
            alt 驗證失敗
                SC-->>UI: 400（錯誤欄位）
                UI-->>U: 標示錯誤欄位，保留輸入內容
            else 驗證通過
                SC->>DB: Summary.edit()、Task.edit() 後更新
                alt 更新成功
                    DB-->>SC: OK
                    SC-->>UI: 200（更新後資料）
                    UI-->>U: 顯示「儲存成功」
                else 資料庫錯誤
                    DB-->>SC: 錯誤
                    SC-->>UI: 500
                    UI-->>U: 提示儲存失敗，保留輸入內容
                end
            end
        end
    end
```

#### 2.2.2 活動圖

```mermaid
flowchart TD
    S(("開始")) --> A1["從會議列表選擇會議"]
    A1 --> B1{"為本人的會議<br/>且狀態為完成？"}
    B1 -->|否| B2["顯示無權限或<br/>尚未處理完成"]
    B2 --> E((("結束")))
    B1 -->|是| B3["顯示逐字稿、<br/>摘要與待辦任務"]
    B3 --> A2["閱讀內容"]
    A2 --> A3{"需要修改？"}
    A3 -->|否| A7
    A3 -->|是| A4["點選「編輯」<br/>修改摘要或任務"]
    A4 --> A5{"儲存或放棄？"}
    A5 -->|放棄| B4["還原為原內容"]
    B4 --> A7
    A5 -->|儲存| B5{"欄位驗證通過？<br/>（期限格式、必填）"}
    B5 -->|否| B6["標示錯誤欄位"]
    B6 --> A4
    B5 -->|是| B7["更新摘要與任務"]
    B7 --> B8{"儲存成功？"}
    B8 -->|否| B9["提示錯誤<br/>保留已輸入內容"]
    B9 --> A4
    B8 -->|是| B10["顯示「儲存成功」"]
    B10 --> A7{"要傳送至 LINE？"}
    A7 -->|否| E
    A7 -->|是| U8[["執行 UC08<br/>傳送摘要至 LINE"]]
    U8 --> E

    classDef user fill:#e3f4e1,stroke:#3d8b37,color:#000
    classDef sys fill:#dbe7f5,stroke:#4a78b5,color:#000
    classDef subuc fill:#fff,stroke:#4a78b5,stroke-width:2px,color:#000
    classDef term fill:#333,stroke:#333,color:#fff
    class A1,A2,A3,A4,A5,A7 user
    class B1,B2,B3,B4,B5,B6,B7,B8,B9,B10 sys
    class U8 subuc
    class S,E term
```

### 2.3 UC08 傳送摘要至 LINE

#### 2.3.1 循序圖

```mermaid
sequenceDiagram
    autonumber
    actor U as 使用者
    participant UI as 前端網頁
    participant SC as SummaryController
    participant IS as IntegrationService
    participant DB as 資料庫
    participant LINE as LINE 服務

    U->>UI: 在摘要頁點選「傳送至 LINE」
    UI->>SC: POST /meetings/{id}/summary/line
    SC->>IS: sendSummaryToLine(userId, summaryId)
    IS->>DB: 查詢使用者的 LineBinding
    DB-->>IS: binding（可能不存在）

    alt 尚未綁定或授權已失效
        IS-->>SC: NotBound
        SC-->>UI: 409 尚未綁定 LINE
        UI-->>U: 提示並導向綁定頁面（UC02）
    else 已綁定
        IS->>DB: 查詢 Summary
        DB-->>IS: summary
        IS->>IS: summary.toLineMessage()
        loop RetryExecutor：失敗時重試，最多 3 次
            IS->>LINE: createNote(binding, 摘要內容)
            LINE-->>IS: 成功或錯誤
        end
        IS->>DB: 新增 DeliveryRecord（LINE、結果、重試次數）

        alt 傳送成功
            IS-->>SC: DeliveryRecord（success）
            SC-->>UI: 200
            UI-->>U: 顯示「已傳送」與傳送時間
        else 重試 3 次仍失敗
            IS-->>SC: DeliveryRecord（failed）
            SC-->>UI: 502（錯誤原因）
            UI-->>U: 顯示失敗通知與「重新傳送」按鈕
        end
    end
```

#### 2.3.2 活動圖

```mermaid
flowchart TD
    S(("開始")) --> A1["在摘要頁點選<br/>「傳送至 LINE」"]
    A1 --> B1{"已綁定 LINE<br/>且授權有效？"}
    B1 -->|否| B2["提示尚未綁定<br/>導向綁定頁面"]
    B2 --> U2[["執行 UC02<br/>綁定 LINE"]]
    U2 --> A1
    B1 -->|是| B3["組合要傳送的摘要內容"]
    B3 --> B4["呼叫 LINE 服務"]
    B4 --> X1["LINE 建立記事"]
    X1 --> C1{"傳送成功？"}
    C1 -->|否| C2{"已重試 3 次？"}
    C2 -->|否| B4
    C2 -->|是| B5["記錄失敗結果"]
    B5 --> B6["通知失敗<br/>顯示「重新傳送」按鈕"]
    B6 --> A2{"是否重新傳送？"}
    A2 -->|是| B3
    A2 -->|否| E((("結束")))
    C1 -->|是| B7["記錄傳送時間與結果"]
    B7 --> B8["顯示「已傳送」"]
    B8 --> E

    classDef user fill:#e3f4e1,stroke:#3d8b37,color:#000
    classDef sys fill:#dbe7f5,stroke:#4a78b5,color:#000
    classDef ext fill:#fde9d4,stroke:#d07a1f,color:#000
    classDef subuc fill:#fff,stroke:#4a78b5,stroke-width:2px,color:#000
    classDef term fill:#333,stroke:#333,color:#fff
    class A1,A2 user
    class B1,B2,B3,B4,B5,B6,B7,B8,C1,C2 sys
    class X1 ext
    class U2 subuc
    class S,E term
```

### 2.4 UC09 任務寫入行事曆

#### 2.4.1 循序圖

系統逐筆處理勾選的任務：沒有期限的任務不建立事件；已寫入過的任務直接略過，避免重複建立（對應 UC09 例外流程 4a）。

```mermaid
sequenceDiagram
    autonumber
    actor U as 使用者
    participant UI as 前端網頁
    participant TC as TaskController
    participant IS as IntegrationService
    participant DB as 資料庫
    participant CAL as 行事曆服務

    U->>UI: 勾選任務，點選「加入行事曆」
    UI->>TC: POST /tasks/calendar（taskIds）
    TC->>IS: addTasksToCalendar(userId, taskIds)
    IS->>DB: 查詢使用者的 CalendarBinding
    DB-->>IS: binding（可能不存在）

    alt 尚未綁定或授權已失效
        IS-->>TC: NotBound
        TC-->>UI: 409 尚未綁定行事曆
        UI-->>U: 提示並導向綁定頁面（UC02）
    else 已綁定
        IS->>DB: 查詢選取的 Task
        DB-->>IS: tasks
        loop 每一項任務
            alt 沒有期限（!task.hasDueDate()）
                IS->>IS: 加入 missingDueDate 清單
            else 已寫入過（task.isInCalendar()）
                IS->>IS: 略過，避免重複建立
            else 可建立事件
                loop RetryExecutor：失敗時重試，最多 3 次
                    IS->>CAL: createEvent(binding, 任務內容, dueDate)
                    CAL-->>IS: eventId 或錯誤
                end
                alt 建立成功
                    IS->>DB: 更新 Task.calendarEventId，新增 DeliveryRecord
                else 重試 3 次仍失敗
                    IS->>DB: 新增 DeliveryRecord（failed）
                    IS->>IS: 加入 failed 清單
                end
            end
        end
        IS-->>TC: CalendarResult（成功、失敗、缺期限）
        TC-->>UI: 200 CalendarResult
        UI-->>U: 顯示結果：成功任務標記「已加入行事曆」
        opt 有失敗或缺期限的任務
            UI-->>U: 提示補上期限，失敗任務顯示「重新加入」按鈕
        end
    end
```

#### 2.4.2 活動圖

```mermaid
flowchart TD
    S(("開始")) --> A1["在任務清單勾選<br/>一或多項任務"]
    A1 --> A2["點選「加入行事曆」"]
    A2 --> B1{"已綁定行事曆<br/>且授權有效？"}
    B1 -->|否| B2["提示尚未綁定<br/>導向綁定頁面"]
    B2 --> U2[["執行 UC02<br/>綁定行事曆"]]
    U2 --> A2
    B1 -->|是| B3["取出下一項任務"]

    B3 --> B4{"任務有期限？"}
    B4 -->|否| B5["列入「需補期限」清單"]
    B4 -->|是| B6{"已加入過行事曆？"}
    B6 -->|是| B7["略過，避免重複建立"]
    B6 -->|否| B8["呼叫行事曆服務"]
    B8 --> X1["行事曆服務建立事件<br/>（標題：任務內容，時間：期限）"]
    X1 --> C1{"建立成功？"}
    C1 -->|否| C2{"已重試 3 次？"}
    C2 -->|否| B8
    C2 -->|是| B9["列入「失敗」清單"]
    C1 -->|是| B10["記錄事件 ID<br/>標記「已加入行事曆」"]

    B5 --> B11{"還有未處理的任務？"}
    B7 --> B11
    B9 --> B11
    B10 --> B11
    B11 -->|是| B3
    B11 -->|否| B12["顯示處理結果"]

    B12 --> B13{"有缺期限或<br/>失敗的任務？"}
    B13 -->|否| E((("結束")))
    B13 -->|是| A3{"是否補上期限<br/>或重新加入？"}
    A3 -->|否| E
    A3 -->|是| A4["補上期限後<br/>重新勾選任務"]
    A4 --> A2

    classDef user fill:#e3f4e1,stroke:#3d8b37,color:#000
    classDef sys fill:#dbe7f5,stroke:#4a78b5,color:#000
    classDef ext fill:#fde9d4,stroke:#d07a1f,color:#000
    classDef subuc fill:#fff,stroke:#4a78b5,stroke-width:2px,color:#000
    classDef term fill:#333,stroke:#333,color:#fff
    class A1,A2,A3,A4 user
    class B1,B2,B3,B4,B5,B6,B7,B8,B9,B10,B11,B12,B13,C1,C2 sys
    class X1 ext
    class U2 subuc
    class S,E term
```

---

## 3. 圖表與需求對照

| 圖表 | 涵蓋使用案例 | 涵蓋需求 |
|---|---|---|
| 1.1 領域模型 | 全部 | FR-01～FR-15、NFR-03、NFR-04 |
| 1.2 設計類別 | 全部 | NFR-07（RetryExecutor）、NFR-08（介面與 Adapter） |
| 2.1 循序圖／活動圖 | UC03、UC04、UC05 | FR-04、FR-05、FR-07、FR-08、NFR-02、NFR-07 |
| 2.2 循序圖／活動圖 | UC06（擴充 UC08） | FR-09、NFR-04 |
| 2.3 循序圖／活動圖 | UC08（前置 UC02） | FR-13、FR-15、NFR-07 |
| 2.4 循序圖／活動圖 | UC09（前置 UC02） | FR-14、FR-15、NFR-07 |
