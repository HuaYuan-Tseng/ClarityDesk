# Data Model: LINE Integration

**Date**: 2026-01-11\
**Feature**: LINE 整合功能\
**Status**: Design Complete

## Overview

本文件定義 LINE 整合功能所需的新實體、關聯關係、驗證規則和狀態轉換。遵循現有專案的 EF Core + Fluent API Configuration 模式。

## New Entities

### LineBinding (LINE 帳號綁定關係)

**Purpose**: 記錄系統使用者帳號與 LINE 帳號的對應關係

**Entity Definition**:

```csharp
namespace ClarityDesk.Models.Entities
{
    /// <summary>
    /// LINE 帳號綁定關係
    /// </summary>
    public class LineBinding
    {
        /// <summary>
        /// 主鍵
        /// </summary>
        public int Id { get; set; }
        
        /// <summary>
        /// 系統使用者 ID (外鍵)
        /// </summary>
        public int UserId { get; set; }
        
        /// <summary>
        /// LINE User ID (從 LINE Profile API 取得)
        /// </summary>
        public string LineUserId { get; set; } = string.Empty;
        
        /// <summary>
        /// LINE 顯示名稱
        /// </summary>
        public string DisplayName { get; set; } = string.Empty;
        
        /// <summary>
        /// LINE 個人圖片 URL
        /// </summary>
        public string? PictureUrl { get; set; }
        
        /// <summary>
        /// 綁定時間 (UTC)
        /// </summary>
        public DateTime BoundAt { get; set; }
        
        /// <summary>
        /// 是否啟用 (false 代表已解除綁定)
        /// </summary>
        public bool IsActive { get; set; }
        
        /// <summary>
        /// 最後解綁時間 (UTC, nullable)
        /// </summary>
        public DateTime? UnboundAt { get; set; }
        
        /// <summary>
        /// 導覽屬性: 關聯的使用者
        /// </summary>
        public User User { get; set; } = null!;
    }
}
```

**Fluent API Configuration**:

```csharp
// Data/Configurations/LineBindingConfiguration.cs
public class LineBindingConfiguration : IEntityTypeConfiguration<LineBinding>
{
    public void Configure(EntityTypeBuilder<LineBinding> builder)
    {
        builder.ToTable("LineBindings");
        
        builder.HasKey(lb => lb.Id);
        
        builder.Property(lb => lb.LineUserId)
            .IsRequired()
            .HasMaxLength(100);
        
        builder.Property(lb => lb.DisplayName)
            .IsRequired()
            .HasMaxLength(200);
        
        builder.Property(lb => lb.PictureUrl)
            .HasMaxLength(500);
        
        builder.Property(lb => lb.BoundAt)
            .IsRequired();
        
        builder.Property(lb => lb.IsActive)
            .IsRequired()
            .HasDefaultValue(true);
        
        // Unique index: 一個 LINE 帳號只能綁定一個啟用的系統帳號
        builder.HasIndex(lb => lb.LineUserId)
            .IsUnique()
            .HasFilter("[IsActive] = 1")
            .HasDatabaseName("IX_LineBindings_LineUserId_Active");
        
        // Index for lookup by UserId
        builder.HasIndex(lb => lb.UserId)
            .HasDatabaseName("IX_LineBindings_UserId");
        
        // Foreign key relationship
        builder.HasOne(lb => lb.User)
            .WithMany()
            .HasForeignKey(lb => lb.UserId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

**Validation Rules**:

- `LineUserId`: 必填，長度 1-100，格式為 LINE Platform 提供的唯一識別碼
- `DisplayName`: 必填，長度 1-200
- `PictureUrl`: 選填，長度 ≤ 500，需為有效 URL
- `BoundAt`: 必填，UTC 時間
- **Business Rule**: 同一個 `LineUserId` 在 `IsActive = true` 狀態下只能存在一筆記錄 (防止重複綁定)

### ConversationState (對話狀態)

**Purpose**: 記錄 LINE 端回報流程的對話狀態，支援多步驟填寫

**Entity Definition**:

```csharp
namespace ClarityDesk.Models.Entities
{
    /// <summary>
    /// LINE 對話狀態
    /// </summary>
    public class ConversationState
    {
        /// <summary>
        /// 主鍵
        /// </summary>
        public int Id { get; set; }
        
        /// <summary>
        /// LINE User ID
        /// </summary>
        public string LineUserId { get; set; } = string.Empty;
        
        /// <summary>
        /// 當前步驟
        /// </summary>
        public string CurrentStep { get; set; } = string.Empty;
        
        /// <summary>
        /// 已填寫的資料 (JSON 格式)
        /// </summary>
        public string DataJson { get; set; } = "{}";
        
        /// <summary>
        /// 對話開始時間 (UTC)
        /// </summary>
        public DateTime StartedAt { get; set; }
        
        /// <summary>
        /// 最後互動時間 (UTC)
        /// </summary>
        public DateTime LastInteractionAt { get; set; }
        
        /// <summary>
        /// 是否啟用 (false 代表已完成或已取消)
        /// </summary>
        public bool IsActive { get; set; }
        
        /// <summary>
        /// 完成或取消時間 (UTC, nullable)
        /// </summary>
        public DateTime? CompletedAt { get; set; }
    }
}
```

**Fluent API Configuration**:

```csharp
// Data/Configurations/ConversationStateConfiguration.cs
public class ConversationStateConfiguration : IEntityTypeConfiguration<ConversationState>
{
    public void Configure(EntityTypeBuilder<ConversationState> builder)
    {
        builder.ToTable("ConversationStates");
        
        builder.HasKey(cs => cs.Id);
        
        builder.Property(cs => cs.LineUserId)
            .IsRequired()
            .HasMaxLength(100);
        
        builder.Property(cs => cs.CurrentStep)
            .IsRequired()
            .HasMaxLength(50);
        
        builder.Property(cs => cs.DataJson)
            .IsRequired()
            .HasColumnType("nvarchar(max)");
        
        builder.Property(cs => cs.StartedAt)
            .IsRequired();
        
        builder.Property(cs => cs.LastInteractionAt)
            .IsRequired();
        
        builder.Property(cs => cs.IsActive)
            .IsRequired()
            .HasDefaultValue(true);
        
        // Composite index for lookup and cleanup
        builder.HasIndex(cs => new { cs.LineUserId, cs.IsActive, cs.LastInteractionAt })
            .HasDatabaseName("IX_ConversationStates_Lookup");
    }
}
```

**Validation Rules**:

- `LineUserId`: 必填，長度 1-100
- `CurrentStep`: 必填，長度 1-50，允許值見「對話流程步驟」章節
- `DataJson`: 必填，有效 JSON 格式
- `LastInteractionAt`: 必填，用於超時檢查 (15 分鐘無互動則視為逾時)
- **Business Rule**: 系統每小時自動清理 `LastInteractionAt` 超過 24 小時的記錄 (FR-031)

**Data JSON Schema**:

```json
{
  "title": "問題標題",
  "content": "問題內容",
  "departmentId": 1,
  "priorityLevel": "Medium",
  "contactPerson": "聯絡人姓名",
  "contactPhone": "0912345678"
}
```

### PushNotificationLog (推送通知日誌)

**Purpose**: 記錄推送訊息的發送歷史，用於除錯和監控

**Entity Definition**:

```csharp
namespace ClarityDesk.Models.Entities
{
    /// <summary>
    /// 推送通知日誌
    /// </summary>
    public class PushNotificationLog
    {
        /// <summary>
        /// 主鍵
        /// </summary>
        public int Id { get; set; }
        
        /// <summary>
        /// 回報單 ID (外鍵)
        /// </summary>
        public int IssueReportId { get; set; }
        
        /// <summary>
        /// 接收者 LINE User ID
        /// </summary>
        public string ReceiverLineUserId { get; set; } = string.Empty;
        
        /// <summary>
        /// 發送時間 (UTC)
        /// </summary>
        public DateTime SentAt { get; set; }
        
        /// <summary>
        /// 發送狀態
        /// </summary>
        public PushNotificationStatus Status { get; set; }
        
        /// <summary>
        /// 錯誤訊息 (若失敗)
        /// </summary>
        public string? ErrorMessage { get; set; }
        
        /// <summary>
        /// 重試次數
        /// </summary>
        public int RetryCount { get; set; }
        
        /// <summary>
        /// 導覽屬性: 關聯的回報單
        /// </summary>
        public IssueReport IssueReport { get; set; } = null!;
    }
    
    /// <summary>
    /// 推送通知狀態
    /// </summary>
    public enum PushNotificationStatus
    {
        [Display(Name = "發送成功")]
        Success = 0,
        
        [Display(Name = "發送失敗")]
        Failed = 1,
        
        [Display(Name = "重試中")]
        Retrying = 2
    }
}
```

**Fluent API Configuration**:

```csharp
// Data/Configurations/PushNotificationLogConfiguration.cs
public class PushNotificationLogConfiguration : IEntityTypeConfiguration<PushNotificationLog>
{
    public void Configure(EntityTypeBuilder<PushNotificationLog> builder)
    {
        builder.ToTable("PushNotificationLogs");
        
        builder.HasKey(pnl => pnl.Id);
        
        builder.Property(pnl => pnl.ReceiverLineUserId)
            .IsRequired()
            .HasMaxLength(100);
        
        builder.Property(pnl => pnl.SentAt)
            .IsRequired();
        
        builder.Property(pnl => pnl.Status)
            .IsRequired();
        
        builder.Property(pnl => pnl.ErrorMessage)
            .HasMaxLength(2000);
        
        builder.Property(pnl => pnl.RetryCount)
            .IsRequired()
            .HasDefaultValue(0);
        
        // Index for querying by IssueReportId
        builder.HasIndex(pnl => pnl.IssueReportId)
            .HasDatabaseName("IX_PushNotificationLogs_IssueReportId");
        
        // Index for monitoring failed notifications
        builder.HasIndex(pnl => new { pnl.Status, pnl.SentAt })
            .HasDatabaseName("IX_PushNotificationLogs_StatusSentAt");
        
        // Foreign key relationship
        builder.HasOne(pnl => pnl.IssueReport)
            .WithMany()
            .HasForeignKey(pnl => pnl.IssueReportId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

**Validation Rules**:

- `ReceiverLineUserId`: 必填，長度 1-100
- `SentAt`: 必填，UTC 時間
- `Status`: 必填，列舉值
- `ErrorMessage`: 選填，長度 ≤ 2000
- `RetryCount`: 必填，預設 0，最大 3 (根據 FR-014)

## Modified Entities

### User (現有實體 - 新增導覽屬性)

**Changes**:

```csharp
// Models/Entities/User.cs (僅新增導覽屬性，不修改現有欄位)
public class User
{
    // ... 現有屬性保持不變 ...
    
    /// <summary>
    /// 導覽屬性: LINE 綁定關係 (新增)
    /// </summary>
    public LineBinding? LineBinding { get; set; }
}
```

**Rationale**:

- 避免修改 `Users` 表結構 (最小化變更原則)
- 使用獨立的 `LineBindings` 表管理綁定關係
- 導覽屬性僅用於 EF Core 查詢便利性

## Entity Relationships

```
User (1) ←→ (0..1) LineBinding
  ↓
  已存在關聯: DepartmentUser, IssueReport

IssueReport (1) ←→ (0..*) PushNotificationLog

ConversationState (獨立實體，透過 LineUserId 關聯 LineBinding)
```

**Notes**:

- `LineBinding` 與 `User` 為一對一關係 (一個使用者最多一個啟用的綁定)
- `ConversationState` 不直接關聯 `User`，而是透過 `LineUserId` 查詢
- `PushNotificationLog` 與 `IssueReport` 為多對一關係 (一個回報單可有多筆推送記錄)

## Conversation Flow Steps

LINE 端回報流程的步驟定義 (`ConversationState.CurrentStep` 允許值)：

| Step Code        | Description | Next Step              | User Input Type                        |
| ---------------- | ----------- | ---------------------- | -------------------------------------- |
| `START`          | 初始狀態        | `TITLE`                | 關鍵字 "回報問題"                             |
| `TITLE`          | 等待輸入問題標題    | `CONTENT`              | 文字                                     |
| `CONTENT`        | 等待輸入問題內容    | `DEPARTMENT`           | 文字                                     |
| `DEPARTMENT`     | 等待選擇所屬單位    | `PRIORITY`             | Postback Action (DepartmentId)         |
| `PRIORITY`       | 等待選擇緊急程度    | `CONTACT_PERSON`       | Postback Action (High/Medium/Low)      |
| `CONTACT_PERSON` | 等待輸入聯絡人     | `CONTACT_PHONE`        | 文字                                     |
| `CONTACT_PHONE`  | 等待輸入連絡電話    | `CONFIRM`              | 文字 (需驗證格式)                             |
| `CONFIRM`        | 顯示摘要等待確認    | `COMPLETED` or `TITLE` | Postback Action (Confirm/Retry/Cancel) |
| `COMPLETED`      | 已完成         | -                      | -                                      |
| `CANCELLED`      | 已取消         | -                      | -                                      |

**State Transition Rules**:

1. 使用者可在任何步驟輸入「取消」跳轉至 `CANCELLED` 狀態
2. 在 `CONFIRM` 步驟選擇「重新填寫」時，清空 `DataJson` 並回到 `TITLE` 步驟
3. 在 `CONFIRM` 步驟選擇「確認送出」後，建立回報單並標記為 `COMPLETED`
4. 超過 15 分鐘未互動自動標記為 `CANCELLED`

## Database Migration Strategy

**Migration Order**:

1. **AddLineBindingsTable**: 建立 `LineBindings` 表
2. **AddConversationStatesTable**: 建立 `ConversationStates` 表
3. **AddPushNotificationLogsTable**: 建立 `PushNotificationLogs` 表

**Rollback Strategy**:

- 各表獨立建立，可個別 Rollback
- 無修改現有表，Rollback 風險低

**Data Seeding**:

- 不需要種子資料 (使用者自行綁定)

## Performance Considerations

### Indexing Strategy

已在 Fluent API Configuration 中定義的索引：

1. **LineBindings**:
   - `IX_LineBindings_LineUserId_Active` (Unique, Filtered): 快速查詢啟用的綁定
   - `IX_LineBindings_UserId`: 支援由使用者查詢綁定狀態

2. **ConversationStates**:
   - `IX_ConversationStates_Lookup` (Composite): 支援查詢啟用對話和清理過期記錄

3. **PushNotificationLogs**:
   - `IX_PushNotificationLogs_IssueReportId`: 查詢特定回報單的推送歷史
   - `IX_PushNotificationLogs_StatusSentAt`: 監控失敗通知

### Query Optimization

**常見查詢模式**:

```csharp
// 1. 檢查使用者是否已綁定 LINE
var binding = await _context.LineBindings
    .FirstOrDefaultAsync(lb => lb.UserId == userId && lb.IsActive);

// 2. 取得使用者的啟用對話狀態
var conversation = await _context.ConversationStates
    .FirstOrDefaultAsync(cs => cs.LineUserId == lineUserId && cs.IsActive);

// 3. 查詢特定回報單的推送日誌
var logs = await _context.PushNotificationLogs
    .Where(pnl => pnl.IssueReportId == issueId)
    .OrderByDescending(pnl => pnl.SentAt)
    .ToListAsync();
```

### Data Retention Policy

- **ConversationStates**: 保留 24 小時後自動清理 (Background Service 實作)
- **PushNotificationLogs**: 建議保留 90 天 (需手動實作清理任務，不在本次範圍)
- **LineBindings**: 不自動清理，由使用者手動解綁

## Data Model Diagram

```
┌─────────────────┐
│      User       │
│  (現有實體)      │
├─────────────────┤
│ Id (PK)         │
│ LineUserId      │
│ DisplayName     │
│ Email           │
│ Role            │
│ IsActive        │
└────────┬────────┘
         │ 1
         │
         │ 0..1
┌────────▼────────┐
│  LineBinding    │
├─────────────────┤
│ Id (PK)         │
│ UserId (FK)     │◄────┐
│ LineUserId (UK) │     │ Unique when IsActive = true
│ DisplayName     │     │
│ PictureUrl      │     │
│ BoundAt         │     │
│ IsActive        │     │
│ UnboundAt       │     │
└─────────────────┘     │
                        │
┌─────────────────┐     │
│IssueReport      │     │
│  (現有實體)      │     │
├─────────────────┤     │
│ Id (PK)         │     │
│ Title           │     │
│ Content         │     │
│ Status          │     │
│ PriorityLevel   │     │
└────────┬────────┘     │
         │ 1            │
         │              │
         │ 0..*         │
┌────────▼────────┐     │
│PushNotification │     │
│      Log        │     │
├─────────────────┤     │
│ Id (PK)         │     │
│ IssueReportId   │◄────┘
│   (FK)          │
│ ReceiverLine    │
│   UserId        │
│ SentAt          │
│ Status          │
│ ErrorMessage    │
│ RetryCount      │
└─────────────────┘

┌─────────────────┐
│Conversation     │
│    State        │
├─────────────────┤
│ Id (PK)         │
│ LineUserId      │ (Not FK, loose coupling)
│ CurrentStep     │
│ DataJson        │
│ StartedAt       │
│ LastInteraction │
│   At            │
│ IsActive        │
│ CompletedAt     │
└─────────────────┘
```

## Enum Definitions

### PushNotificationStatus

```csharp
namespace ClarityDesk.Models.Enums
{
    /// <summary>
    /// 推送通知狀態
    /// </summary>
    public enum PushNotificationStatus
    {
        /// <summary>
        /// 發送成功
        /// </summary>
        [Display(Name = "發送成功")]
        Success = 0,
        
        /// <summary>
        /// 發送失敗
        /// </summary>
        [Display(Name = "發送失敗")]
        Failed = 1,
        
        /// <summary>
        /// 重試中
        /// </summary>
        [Display(Name = "重試中")]
        Retrying = 2
    }
}
```

## Summary

本資料模型設計符合以下原則：

1. ✅ **最小化變更**：不修改現有表結構，僅新增獨立表和導覽屬性
2. ✅ **遵循現有模式**：使用 EF Core + Fluent API Configuration + `IEntityTypeConfiguration<T>`
3. ✅ **支援所有功能需求**：涵蓋 FR-001 至 FR-031 的資料需求
4. ✅ **效能優化**：適當的索引策略和查詢優化
5. ✅ **資料完整性**：外鍵約束、唯一性約束、預設值
6. ✅ **可維護性**：清晰的實體關聯、狀態轉換規則
