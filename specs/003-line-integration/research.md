# Technical Research: LINE Integration

**Date**: 2026-01-11\
**Feature**: LINE 整合功能\
**Status**: Completed

## Overview

本文件記錄為實現 LINE 整合功能 (帳號綁定、推送通知、對話式回報) 所做的技術研究和決策。專案限制：不使用 AutoMapper、不使用 Redis。

## Technology Stack Analysis

### LINE Messaging API SDK

**Decision**: 使用官方 `LineDevelopers.MessagingApi` NuGet 套件

**Rationale**:

- LINE 官方維護，與 LINE Platform API 版本同步
- 支援完整的 Messaging API 功能 (Push Message, Flex Message, Webhook)
- 強型別 API，減少執行時期錯誤
- 已整合至 .NET 8.0 環境

**Alternatives Considered**:

- **手動 HTTP 呼叫**：不考慮，維護成本高且容易出錯
- **第三方封裝庫**：不考慮，官方支援優先

**Implementation Notes**:

```bash
dotnet add package LineDevelopers.MessagingApi --version 2.0.0
```

### LINE Login Integration

**Decision**: 繼續使用現有 OAuth 流程 (已實作於 `Program.cs`)，擴充支援 LINE Profile API

**Rationale**:

- 專案已有 LINE Login OAuth 實作 (使用 `AddOAuth("LINE", ...)` )
- 需額外呼叫 LINE Profile API 獲取 LINE User ID 以建立綁定關係
- 符合最小化變更原則 (Constitution 第二條)

**Alternatives Considered**:

- **重新實作 LINE Login**：違反憲章第一條 (尊重棕地專案)
- **使用 JWT 自訂流程**：過度設計，現有 OAuth 已足夠

**Implementation Notes**:

- 在 OAuth `OnCreatingTicket` 事件中取得 LINE User ID
- 儲存至新的 `LineBinding` 實體

### Webhook Handling for LINE Bot

**Decision**: 建立新的 API Controller 專門處理 LINE Webhook，驗證簽章並路由至對應服務

**Rationale**:

- LINE Bot 透過 Webhook 接收使用者訊息
- 需驗證 `X-Line-Signature` 確保請求來自 LINE Platform
- 使用 ASP.NET Core Minimal API 或 Controller 處理 POST 端點

**Alternatives Considered**:

- **Azure Functions / Serverless**：不適用，專案為 Razor Pages 單體架構
- **SignalR / WebSocket**：不必要，Webhook 已滿足需求

**Implementation Notes**:

```csharp
[ApiController]
[Route("api/line/webhook")]
public class LineWebhookController : ControllerBase
{
    // Verify signature, parse events, dispatch to ConversationService
}
```

### Conversation State Management

**Decision**: 使用 **Entity Framework Core + SQL Server** 儲存對話狀態 (`ConversationState` 實體)

**Rationale**:

- 專案禁用 Redis (根據用戶指示)
- EF Core 已是專案主要資料存取方式，保持一致性
- 對話狀態需持久化 24 小時 (根據 FR-031)，SQL Server 可靠性足夠
- 可使用 Indexed Column (`LineUserId` + `LastInteractionAt`) 優化查詢

**Alternatives Considered**:

- **Redis / Distributed Cache**：用戶明確禁用
- **In-Memory Cache (`IMemoryCache`)**：不可靠，應用程式重啟後遺失
- **Session Storage**：不適用於跨裝置的 LINE 對話

**Performance Consideration**:

- 對話並發量預估 <100 (根據 SC-008)，SQL Server 可承受
- 建立 Composite Index: `(LineUserId, IsActive, LastInteractionAt)`
- 使用 Background Service 每小時清理過期對話狀態

**Implementation Notes**:

```csharp
public class ConversationState
{
    public int Id { get; set; }
    public string LineUserId { get; set; }
    public string CurrentStep { get; set; } // "TITLE", "CONTENT", "DEPARTMENT", ...
    public string DataJson { get; set; } // JSON serialized partial data
    public DateTime LastInteractionAt { get; set; }
    public bool IsActive { get; set; }
}
```

### DTO Mapping without AutoMapper

**Decision**: 使用 **Extension Methods** 實作手動映射 (現有專案模式)

**Rationale**:

- 用戶明確要求使用 POCO 而非 AutoMapper
- 專案已在 `Models/Extensions/` 使用 Extension Methods (例如 `ToDto()`, `ToEntity()`)
- 明確性高，易於除錯，符合團隊慣例

**Alternatives Considered**:

- **AutoMapper**：用戶明確禁用
- **Mapster**：引入新依賴，違反最小化變更原則

**Implementation Notes**:

- 建立 `LineBindingExtensions.cs` 提供 `ToDto()`, `ToEntity()`, `UpdateFromDto()`
- 建立 `ConversationStateExtensions.cs` 處理 JSON 序列化 / 反序列化

### Push Notification Retry Strategy

**Decision**: 使用 **Polly** 實作指數退避重試 (1s, 5s, 15s)

**Rationale**:

- Polly 是 .NET 生態系統標準的韌性庫
- 符合 FR-014 需求 (最多重試 3 次)
- 可設定 Circuit Breaker 防止 LINE API 持續失敗時的資源耗損

**Alternatives Considered**:

- **手動實作重試邏輯**：容易出錯，不建議
- **Azure Service Bus / Queue**：過度設計，推送通知非關鍵任務

**Implementation Notes**:

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(() => _lineMessagingClient.PushMessageAsync(...));
```

### Flex Message Template Design

**Decision**: 使用 **Flex Message Simulator** 設計 JSON 模板，儲存為嵌入資源或 `appsettings.json`

**Rationale**:

- Flex Message 提供豐富的視覺呈現 (根據用戶需求)
- 使用 LINE 官方 Simulator 確保格式正確：https://developers.line.biz/flex-simulator/
- 模板參數化 (`{{IssueNumber}}`, `{{Title}}` 等) 便於動態替換

**Alternatives Considered**:

- **程式碼硬編碼 Flex Message JSON**：難以維護
- **使用第三方樣板引擎**：不必要，字串替換已足夠

**Implementation Notes**:

```json
// appsettings.json -> LineMessaging:FlexTemplates:NewIssueNotification
{
  "type": "bubble",
  "header": { "type": "box", "layout": "vertical", "contents": [...] },
  "body": {
    "type": "box",
    "contents": [
      { "type": "text", "text": "🆔 回報單編號：{{IssueNumber}}", "wrap": true },
      { "type": "text", "text": "📌 問題標題：{{Title}}", "wrap": true },
      ...
    ]
  },
  "footer": {
    "type": "box",
    "contents": [
      {
        "type": "button",
        "action": { "type": "uri", "label": "查看回報單詳情", "uri": "{{DetailUrl}}" }
      }
    ]
  }
}
```

### Background Jobs for State Cleanup

**Decision**: 使用 **Hosted Service (`IHostedService`)**   實作每小時清理過期對話狀態

**Rationale**:

- .NET 8.0 內建支援，無需額外依賴
- 符合 FR-031 需求 (24 小時後清理)
- 使用 `Timer` 或 `PeriodicTimer` 實作定時任務

**Alternatives Considered**:

- **Hangfire / Quartz.NET**：過度設計，簡單定時任務不需要
- **Azure Functions Timer Trigger**：專案非雲端原生架構

**Implementation Notes**:

```csharp
public class ConversationCleanupService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            // Delete ConversationState where LastInteractionAt < DateTime.UtcNow.AddHours(-24)
        }
    }
}
```

## Architecture Decisions

### Service Layer Structure

**Decision**: 建立以下新服務

| Service                 | Responsibility               |
| ----------------------- | ---------------------------- |
| `ILineBindingService`   | 管理 LINE 帳號綁定 / 解綁，驗證綁定狀態     |
| `ILineMessagingService` | 發送推送訊息 (Flex Message)，處理重試邏輯 |
| `IConversationService`  | 管理 LINE 對話流程，儲存 / 讀取對話狀態     |
| `LineWebhookController` | 接收 LINE Webhook，驗證簽章，分派事件    |

**Rationale**:

- 符合現有專案的 Service Interface + Implementation 模式
- 各服務職責單一，易於測試
- 避免 `IssueReportService` 過度膨脹

### Database Schema Changes

**Decision**: 新增三個實體表

| Entity                 | Key Columns                                                                    | Purpose |
| ---------------------- | ------------------------------------------------------------------------------ | ------- |
| `LineBindings`         | `UserId` (FK), `LineUserId` (unique), `DisplayName`, `BoundAt`, `IsActive`     | 儲存綁定關係  |
| `ConversationStates`   | `LineUserId`, `CurrentStep`, `DataJson`, `LastInteractionAt`, `IsActive`       | 對話狀態    |
| `PushNotificationLogs` | `IssueReportId` (FK), `ReceiverLineUserId`, `SentAt`, `Status`, `ErrorMessage` | 推送日誌    |

**Rationale**:

- 避免修改現有 `Users` 表 (最小化變更原則)
- `LineBindings.LineUserId` 設定為 Unique Index 防止重複綁定 (FR-007)
- `ConversationStates.DataJson` 使用 JSON 儲存彈性資料 (避免頻繁 Schema 變更)

### API Endpoints Design

**Decision**: 新增以下端點

| Endpoint               | Method | Purpose               |
| ---------------------- | ------ | --------------------- |
| `/Account/LineBinding` | GET    | 顯示綁定狀態頁面 (Razor Page) |
| `/Account/LineBinding` | POST   | 執行綁定操作                |
| `/Account/LineUnbind`  | POST   | 解除綁定                  |
| `/api/line/webhook`    | POST   | 接收 LINE Webhook 事件    |

**Rationale**:

- Razor Pages 架構維持一致性
- Webhook 端點使用 API Controller (非 Razor Page)，符合 RESTful 慣例

## Security Considerations

### Webhook Signature Verification

**Decision**: 在 `LineWebhookController` 驗證 `X-Line-Signature` 標頭

**Implementation**:

```csharp
var signature = Request.Headers["X-Line-Signature"];
var body = await new StreamReader(Request.Body).ReadToEndAsync();
var isValid = VerifySignature(body, signature, _channelSecret);
if (!isValid) return Unauthorized();
```

**Reference**: https://developers.line.biz/en/docs/messaging-api/receiving-messages/#verifying-signatures

### CSRF Protection for LINE Binding

**Decision**: 使用 `state` 參數於 LINE Login OAuth 流程 (已於 FR-002 定義)

**Implementation**:

- 生成隨機 `state` 值並儲存於 Session
- LINE 回傳時驗證 `state` 是否匹配

### Data Privacy

**Decision**:

- `LineBindings.LineUserId` 使用索引但不加密 (查詢效能考量)
- `ConversationStates.DataJson` 不儲存敏感資訊 (如密碼)，僅儲存回報單資料
- 依 FR-031 自動清理 24 小時後的對話狀態

## Performance Optimization

### Database Indexes

```sql
-- LineBindings
CREATE UNIQUE INDEX IX_LineBindings_LineUserId ON LineBindings(LineUserId) WHERE IsActive = 1;
CREATE INDEX IX_LineBindings_UserId ON LineBindings(UserId);

-- ConversationStates
CREATE INDEX IX_ConversationStates_Lookup ON ConversationStates(LineUserId, IsActive, LastInteractionAt);
```

### Caching Strategy

**Decision**: 使用 `IMemoryCache` 快取啟用的單位清單 (對話流程中使用)

**Implementation**:

```csharp
var departments = await _cache.GetOrCreateAsync("ActiveDepartments", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
    return await _context.Departments.Where(d => d.IsActive).ToListAsync();
});
```

### Message Batching

**Decision**: 當多位處理人員需接收通知時，使用 `Multicast Message` API 批次發送

**Rationale**:

- 減少 API 呼叫次數
- LINE Messaging API 支援一次發送給最多 500 位使用者

**Implementation**:

```csharp
await _lineMessagingClient.MulticastAsync(new MulticastRequest
{
    To = lineUserIds, // List<string>
    Messages = new[] { flexMessage }
});
```

## Testing Strategy

### Unit Tests (xUnit)

- `LineBindingService` 測試：綁定邏輯、重複綁定驗證
- `ConversationService` 測試：狀態轉換、資料持久化
- `LineMessagingService` 測試：Mock LINE API，驗證重試邏輯

### Integration Tests

- Webhook 端點：驗證簽章、事件分派
- 端對端流程：綁定 → 建立回報單 → 驗證推送訊息日誌

### E2E Tests (Playwright)

- LINE 綁定流程 (使用 LINE Sandbox 環境)
- 推送訊息接收驗證 (需要 LINE Bot Simulator)

## Dependencies to Add

```xml
<PackageReference Include="LineDevelopers.MessagingApi" Version="2.0.0" />
<PackageReference Include="Polly" Version="8.4.0" />
```

## Configuration Required

**appsettings.json**:

```json
{
  "LineMessaging": {
    "ChannelAccessToken": "[從 LINE Developers Console 取得]",
    "ChannelSecret": "[從 LINE Developers Console 取得]",
    "WebhookUrl": "https://yourdomain.com/api/line/webhook",
    "FlexTemplates": {
      "NewIssueNotification": "{ ... Flex Message JSON ... }"
    }
  }
}
```

## Risk Mitigation

| Risk                   | Mitigation                              |
| ---------------------- | --------------------------------------- |
| LINE API Rate Limiting | 實作 Circuit Breaker，超過限制時暫停發送並記錄         |
| Webhook 超時             | 非同步處理事件 (儲存至佇列或背景任務)，立即回應 200 OK 給 LINE |
| 對話狀態遺失                 | 加入 `ConversationStates` 資料庫備份策略         |
| Flex Message 格式錯誤      | 使用 LINE Flex Message Simulator 預先驗證     |

## Open Questions (Resolved)

- ✅ 對話狀態保留時間：24 小時 (FR-031)
- ✅ 推送失敗重試策略：3 次指數退避 (FR-014)
- ✅ LINE 帳號綁定驗證：OAuth 2.0 + state 參數 (FR-002)
- ✅ 併發操作處理：最後寫入優先 + 操作日誌 (FR-008a)
- ✅ 回報流程修正：確認摘要提供「重新填寫」選項 (FR-022a)

## References

- [LINE Messaging API Documentation](https://developers.line.biz/en/docs/messaging-api/)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [Polly Documentation](https://github.com/App-vNext/Polly)
- [ASP.NET Core Hosted Services](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
