# Implementation Plan: LINE 整合功能

**Branch**: `003-line-integration` | **Date**: 2026-01-11 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/003-line-integration/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

**主要需求**：在現有的 ClarityDesk 問題回報追蹤系統中，新增 LINE 官方帳號整合功能，實現三個核心功能：(1) 處理人員可綁定 LINE 帳號建立身份關聯，(2) 新增回報單時自動推送 LINE 通知給已綁定的處理人員，(3) 使用者可直接在 LINE 端透過對話式介面回報問題，無需登入網頁系統。

**技術方案**：使用 `LineDevelopers.MessagingApi` (2.0.0) SDK 整合 LINE Messaging API，新增三個實體表儲存綁定關係、對話狀態和推送日誌，使用 Polly (8.4.0) 實作指數退避重試策略，使用 Extension Methods 實作 DTO 映射 (不使用 AutoMapper)，使用 Hosted Service 實作每小時清理過期對話狀態，使用 Flex Message 格式提供豐富的視覺推送通知。

## Technical Context

**Language/Version**: C# / .NET 8.0\
**Primary Dependencies**: ASP.NET Core 8.0 Razor Pages, Entity Framework Core 8.0, LineDevelopers.MessagingApi 2.0.0, Polly 8.4.0\
**Storage**: SQL Server (現有專案已使用，透過 EF Core 存取)\
**Testing**: xUnit (單元測試), Playwright (E2E 測試) - 現有專案已建立測試框架\
**Target Platform**: Web Server (Windows Server / Linux with ASP.NET Core Runtime)\
**Project Type**: Web Application (ASP.NET Core Razor Pages 單體架構)\
**Performance Goals**:

- LINE Webhook 回應時間 <3 秒 (LINE Platform 要求)
- 推送通知發送延遲 < 10 秒
- 支援至少 100 位使用者同時進行對話流程
- 資料庫查詢回應時間 <200ms (p95)

**Constraints**:

- 不使用 AutoMapper (使用 Extension Methods 手動映射)
- 不使用 Redis (使用 EF Core + SQL Server 儲存對話狀態)
- 必須遵守憲章：最小化變更，不修改現有正常運作的程式碼
- LINE Messaging API Rate Limit: 100 requests/sec (需實作 Circuit Breaker)

**Scale/Scope**:

- 預估使用者數：100-500 人
- 預估回報單數量：每月 500-1000 筆
- LINE 推送訊息：每月 2000-5000 則
- 新增程式碼：約 15 個新檔案 (實體 3 + 配置 3 + 服務 3 + DTOs 5 + Controller 1)

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

### Pre-Design Check (Phase 0)

✅ **尊重棕地專案**：

- 不修改現有 `User`, `Department`, `IssueReport` 實體結構
- 僅新增獨立的 `LineBinding`, `ConversationState`, `PushNotificationLog` 實體
- 在 `User` 實體僅新增導覽屬性 (不影響資料庫結構)

✅ **最小化變更原則**：

- 新功能完全封裝在新的服務層 (`ILineBindingService`, `ILineMessagingService`, `IConversationService`)
- 對 `IssueReportService` 的修改僅新增推送通知呼叫 (<10 行程式碼)
- 對 `Program.cs` 的修改僅新增服務註冊 (<15 行程式碼)

✅ **需要明確許可**：

- 新增依賴套件：`LineDevelopers.MessagingApi`, `Polly` - ✅ 已於用戶指示中明確
- 新增 API Controller (`LineWebhookController`) - ✅ 為功能需求必要
- 新增資料庫表 - ✅ 為功能需求必要

✅ **測試紀律**：

- 現有測試不受影響 (功能獨立)
- 新增測試檔案：
  - `ClarityDesk.UnitTests/Services/LineBindingServiceTests.cs`
  - `ClarityDesk.UnitTests/Services/LineMessagingServiceTests.cs`
  - `ClarityDesk.UnitTests/Services/ConversationServiceTests.cs`
  - `ClarityDesk.IntegrationTests/Controllers/LineWebhookControllerTests.cs`

### Post-Design Check (Phase 1)

✅ **架構一致性**：

- 遵循現有 Service Interface + Implementation 模式
- 遵循現有 Entity + DTO + Extension Methods 模式
- 遵循現有 Fluent API Configuration 模式

✅ **資料庫設計**：

- 新增三個獨立表，無外鍵指向現有表 (除了 `LineBinding.UserId` 和 `PushNotificationLog.IssueReportId`)
- 使用 `DeleteBehavior.Restrict` 防止級聯刪除
- 適當的索引策略 (詳見 `data-model.md`)

✅ **API 設計**：

- Webhook 端點遵循 RESTful 慣例
- 使用現有的認證機制 (Cookie Authentication)
- Razor Pages 命名和結構與現有頁面一致

**結論**：✅ 通過憲章檢查，可進入實作階段

## Project Structure

### Documentation (this feature)

```text
specs/003-line-integration/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command) - 技術研究與決策
├── data-model.md        # Phase 1 output (/speckit.plan command) - 資料模型設計
├── quickstart.md        # Phase 1 output (/speckit.plan command) - 開發指引
├── contracts/           # Phase 1 output (/speckit.plan command)
│   └── api-contracts.md # API 端點與服務介面合約
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

**選擇的架構**：Web Application (ASP.NET Core Razor Pages 單體架構)

**理由**：專案已採用 Razor Pages 架構，LINE 整合功能為新增功能模組，遵循現有架構模式。

```text
ClarityDesk/
├── Models/
│   ├── Entities/
│   │   ├── LineBinding.cs                    # 新增
│   │   ├── ConversationState.cs              # 新增
│   │   ├── PushNotificationLog.cs            # 新增
│   │   └── User.cs                            # 修改(僅新增導覽屬性)
│   ├── DTOs/
│   │   ├── LineBindingDto.cs                 # 新增
│   │   ├── ConversationStateDto.cs           # 新增
│   │   ├── PushNotificationResultDto.cs      # 新增
│   │   ├── LineMessageDto.cs                 # 新增
│   │   └── LineUserProfileDto.cs             # 現有(來自 LINE Login)
│   ├── Enums/
│   │   └── PushNotificationStatus.cs         # 新增
│   └── Extensions/
│       ├── LineBindingExtensions.cs          # 新增
│       └── ConversationStateExtensions.cs    # 新增
│
├── Data/
│   ├── ApplicationDbContext.cs               # 修改(新增 DbSet)
│   └── Configurations/
│       ├── LineBindingConfiguration.cs       # 新增
│       ├── ConversationStateConfiguration.cs # 新增
│       └── PushNotificationLogConfiguration.cs # 新增
│
├── Services/
│   ├── Interfaces/
│   │   ├── ILineBindingService.cs            # 新增
│   │   ├── ILineMessagingService.cs          # 新增
│   │   └── IConversationService.cs           # 新增
│   ├── LineBindingService.cs                 # 新增
│   ├── LineMessagingService.cs               # 新增
│   ├── ConversationService.cs                # 新增
│   └── IssueReportService.cs                 # 修改(新增推送通知呼叫)
│
├── Infrastructure/
│   └── Services/
│       └── ConversationCleanupService.cs     # 新增(Background Service)
│
├── Controllers/
│   └── LineWebhookController.cs              # 新增(API Controller)
│
├── Pages/
│   └── Account/
│       ├── LineBinding.cshtml                # 新增
│       └── LineBinding.cshtml.cs             # 新增
│
├── Migrations/
│   └── [Timestamp]_AddLineIntegrationTables.cs  # 新增(Migration)
│
├── Program.cs                                # 修改(註冊服務)
├── appsettings.json                          # 修改(新增 LineMessaging 配置)
└── ClarityDesk.csproj                        # 修改(新增 NuGet 套件)

Tests/
├── ClarityDesk.UnitTests/
│   └── Services/
│       ├── LineBindingServiceTests.cs        # 新增
│       ├── LineMessagingServiceTests.cs      # 新增
│       └── ConversationServiceTests.cs       # 新增
└── ClarityDesk.IntegrationTests/
    └── Controllers/
        └── LineWebhookControllerTests.cs     # 新增
```

**新增檔案統計**：

- 實體 (Entities): 3 個新增，1 個修改
- DTOs: 4 個新增，1 個現有
- Extension Methods: 2 個新增
- Enums: 1 個新增
- Configurations: 3 個新增
- Services: 6 個新增，1 個修改
- Controllers: 1 個新增
- Pages: 2 個新增
- Infrastructure: 1 個新增
- 測試: 4 個新增
- **總計**：約 **28 個新檔案**，**4 個修改檔案**

## Complexity Tracking

> **本專案無違反憲章的設計決策，此章節留空**

本功能設計完全符合憲章規範：

- ✅ 尊重棕地專案：僅新增獨立模組，不修改現有核心邏輯
- ✅ 最小化變更：對現有檔案的修改總計 < 50 行程式碼
- ✅ 明確許可：所有新增依賴和架構決策已於用戶指示中明確
- ✅ 測試紀律：新增測試覆蓋新功能，現有測試不受影響

**無需填寫複雜度追蹤表**。
