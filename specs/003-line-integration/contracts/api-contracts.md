# LINE Integration API Contracts

**Date**: 2026-01-11\
**Feature**: LINE 整合功能\
**Version**: 1.0.0

## Overview

本文件定義 LINE 整合功能的 Web API 端點、服務介面合約、資料傳輸物件 (DTOs) 和訊息格式。

## Web API Endpoints

### 1. LINE Webhook Endpoint

**Endpoint**: `POST /api/line/webhook`\
**Purpose**: 接收 LINE Platform 的 Webhook 事件\
**Authentication**: Signature verification (X-Line-Signature header)

**Request Headers**:

```
X-Line-Signature: {HMAC-SHA256 signature}
Content-Type: application/json
```

**Request Body** (LINE Webhook Event Format):

```json
{
  "destination": "U1234567890abcdef1234567890abcdef",
  "events": [
    {
      "type": "message",
      "message": {
        "type": "text",
        "id": "325708",
        "text": "回報問題"
      },
      "timestamp": 1462629479859,
      "source": {
        "type": "user",
        "userId": "U1234567890abcdef1234567890abcdef"
      },
      "replyToken": "b60d432864f44d079f6d8efe86cf404b",
      "mode": "active"
    }
  ]
}
```

**Response**:

```json
{
  "status": "ok"
}
```

**Status Codes**:

- `200 OK`: 事件已成功接收和處理
- `400 Bad Request`: 簽章驗證失敗或請求格式錯誤
- `500 Internal Server Error`: 伺服器處理錯誤

**Implementation Notes**:

- 必須在 3 秒內回應 200 OK，避免 LINE Platform 重試
- 實際事件處理應使用非同步機制 (Background Service)

---

## Service Interface Contracts

### ILineBindingService

**Purpose**: 管理 LINE 帳號綁定與解綁

```csharp
namespace ClarityDesk.Services.Interfaces
{
    /// <summary>
    /// LINE 帳號綁定服務介面
    /// </summary>
    public interface ILineBindingService
    {
        /// <summary>
        /// 建立 LINE 帳號綁定
        /// </summary>
        /// <param name="userId">系統使用者 ID</param>
        /// <param name="lineProfile">LINE 使用者資料</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>綁定結果 DTO</returns>
        /// <exception cref="InvalidOperationException">當 LINE 帳號已被其他使用者綁定時</exception>
        Task<LineBindingDto> CreateBindingAsync(
            int userId, 
            LineUserProfileDto lineProfile, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 解除 LINE 帳號綁定
        /// </summary>
        /// <param name="userId">系統使用者 ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>是否成功解除</returns>
        Task<bool> UnbindAsync(int userId, CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 取得使用者的綁定狀態
        /// </summary>
        /// <param name="userId">系統使用者 ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>綁定資料 DTO，若未綁定則回傳 null</returns>
        Task<LineBindingDto?> GetBindingByUserIdAsync(
            int userId, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 透過 LINE User ID 取得綁定資料
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>綁定資料 DTO，若不存在則回傳 null</returns>
        Task<LineBindingDto?> GetBindingByLineUserIdAsync(
            string lineUserId, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 檢查 LINE 帳號是否已被綁定
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>是否已綁定</returns>
        Task<bool> IsLineUserIdBoundAsync(
            string lineUserId, 
            CancellationToken cancellationToken = default);
    }
}
```

### ILineMessagingService

**Purpose**: 發送 LINE 推送訊息

```csharp
namespace ClarityDesk.Services.Interfaces
{
    /// <summary>
    /// LINE 訊息推送服務介面
    /// </summary>
    public interface ILineMessagingService
    {
        /// <summary>
        /// 發送新回報單通知給處理人員
        /// </summary>
        /// <param name="issueReportId">回報單 ID</param>
        /// <param name="handlerUserIds">處理人員系統使用者 ID 清單</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>推送結果清單</returns>
        Task<List<PushNotificationResultDto>> SendNewIssueNotificationAsync(
            int issueReportId, 
            List<int> handlerUserIds, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 回覆 LINE 使用者訊息
        /// </summary>
        /// <param name="replyToken">LINE Reply Token</param>
        /// <param name="messages">訊息內容清單</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>是否成功</returns>
        Task<bool> ReplyMessageAsync(
            string replyToken, 
            List<LineMessageDto> messages, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 推送訊息給指定 LINE 使用者
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="messages">訊息內容清單</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>是否成功</returns>
        Task<bool> PushMessageAsync(
            string lineUserId, 
            List<LineMessageDto> messages, 
            CancellationToken cancellationToken = default);
    }
}
```

### IConversationService

**Purpose**: 管理 LINE 端對話流程

```csharp
namespace ClarityDesk.Services.Interfaces
{
    /// <summary>
    /// LINE 對話流程服務介面
    /// </summary>
    public interface IConversationService
    {
        /// <summary>
        /// 開始回報問題對話流程
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>初始回應訊息</returns>
        Task<LineMessageDto> StartReportConversationAsync(
            string lineUserId, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 處理使用者輸入並推進對話狀態
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="userInput">使用者輸入內容</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>回應訊息</returns>
        Task<LineMessageDto> ProcessUserInputAsync(
            string lineUserId, 
            string userInput, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 處理使用者 Postback 動作 (按鈕點擊)
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="postbackData">Postback Data</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>回應訊息</returns>
        Task<LineMessageDto> ProcessPostbackAsync(
            string lineUserId, 
            string postbackData, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 取消對話流程
        /// </summary>
        /// <param name="lineUserId">LINE User ID</param>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>是否成功取消</returns>
        Task<bool> CancelConversationAsync(
            string lineUserId, 
            CancellationToken cancellationToken = default);
        
        /// <summary>
        /// 清理過期的對話狀態 (LastInteractionAt > 24 小時)
        /// </summary>
        /// <param name="cancellationToken">取消權杖</param>
        /// <returns>清理的記錄數</returns>
        Task<int> CleanupExpiredConversationsAsync(
            CancellationToken cancellationToken = default);
    }
}
```

---

## Data Transfer Objects (DTOs)

### LineBindingDto

```csharp
namespace ClarityDesk.Models.DTOs
{
    /// <summary>
    /// LINE 綁定資料傳輸物件
    /// </summary>
    public class LineBindingDto
    {
        /// <summary>
        /// 綁定 ID
        /// </summary>
        public int Id { get; set; }
        
        /// <summary>
        /// 系統使用者 ID
        /// </summary>
        public int UserId { get; set; }
        
        /// <summary>
        /// LINE User ID
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
        /// 綁定時間
        /// </summary>
        public DateTime BoundAt { get; set; }
        
        /// <summary>
        /// 是否啟用
        /// </summary>
        public bool IsActive { get; set; }
    }
}
```

### LineUserProfileDto (現有，來自 LINE Login)

```csharp
namespace ClarityDesk.Models.DTOs
{
    /// <summary>
    /// LINE 使用者資料
    /// </summary>
    public class LineUserProfileDto
    {
        /// <summary>
        /// LINE User ID
        /// </summary>
        public string UserId { get; set; } = string.Empty;
        
        /// <summary>
        /// 顯示名稱
        /// </summary>
        public string DisplayName { get; set; } = string.Empty;
        
        /// <summary>
        /// 個人圖片 URL
        /// </summary>
        public string? PictureUrl { get; set; }
        
        /// <summary>
        /// 狀態訊息
        /// </summary>
        public string? StatusMessage { get; set; }
    }
}
```

### ConversationStateDto

```csharp
namespace ClarityDesk.Models.DTOs
{
    /// <summary>
    /// 對話狀態資料傳輸物件
    /// </summary>
    public class ConversationStateDto
    {
        /// <summary>
        /// 對話 ID
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
        /// 已填寫的資料
        /// </summary>
        public Dictionary<string, object> Data { get; set; } = new();
        
        /// <summary>
        /// 對話開始時間
        /// </summary>
        public DateTime StartedAt { get; set; }
        
        /// <summary>
        /// 最後互動時間
        /// </summary>
        public DateTime LastInteractionAt { get; set; }
        
        /// <summary>
        /// 是否啟用
        /// </summary>
        public bool IsActive { get; set; }
    }
}
```

### PushNotificationResultDto

```csharp
namespace ClarityDesk.Models.DTOs
{
    /// <summary>
    /// 推送通知結果
    /// </summary>
    public class PushNotificationResultDto
    {
        /// <summary>
        /// 接收者系統使用者 ID
        /// </summary>
        public int UserId { get; set; }
        
        /// <summary>
        /// 接收者 LINE User ID
        /// </summary>
        public string LineUserId { get; set; } = string.Empty;
        
        /// <summary>
        /// 是否成功
        /// </summary>
        public bool IsSuccess { get; set; }
        
        /// <summary>
        /// 錯誤訊息 (若失敗)
        /// </summary>
        public string? ErrorMessage { get; set; }
        
        /// <summary>
        /// 重試次數
        /// </summary>
        public int RetryCount { get; set; }
    }
}
```

### LineMessageDto

```csharp
namespace ClarityDesk.Models.DTOs
{
    /// <summary>
    /// LINE 訊息資料傳輸物件
    /// </summary>
    public class LineMessageDto
    {
        /// <summary>
        /// 訊息類型
        /// </summary>
        public LineMessageType Type { get; set; }
        
        /// <summary>
        /// 文字訊息內容 (Type = Text)
        /// </summary>
        public string? Text { get; set; }
        
        /// <summary>
        /// Flex Message 內容 (Type = Flex)
        /// </summary>
        public string? FlexJson { get; set; }
        
        /// <summary>
        /// 快速回覆選項 (Type = Text 時使用)
        /// </summary>
        public List<QuickReplyItemDto>? QuickReply { get; set; }
    }
    
    /// <summary>
    /// LINE 訊息類型
    /// </summary>
    public enum LineMessageType
    {
        Text = 0,
        Flex = 1
    }
    
    /// <summary>
    /// 快速回覆項目
    /// </summary>
    public class QuickReplyItemDto
    {
        /// <summary>
        /// 顯示文字
        /// </summary>
        public string Label { get; set; } = string.Empty;
        
        /// <summary>
        /// 動作類型 (message / postback)
        /// </summary>
        public string ActionType { get; set; } = "message";
        
        /// <summary>
        /// 動作資料
        /// </summary>
        public string Data { get; set; } = string.Empty;
    }
}
```

---

## LINE Message Formats

### 1. 新回報單推送通知 (Flex Message)

**Template Name**: `NewIssueNotification`\
**Format**: Flex Message (Bubble)

```json
{
  "type": "bubble",
  "header": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "text",
        "text": "🔔 新的問題回報",
        "weight": "bold",
        "size": "lg",
        "color": "#1DB446"
      }
    ],
    "backgroundColor": "#F0F0F0"
  },
  "body": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "text",
        "text": "🆔 回報單編號：{{IssueNumber}}",
        "wrap": true,
        "margin": "md"
      },
      {
        "type": "separator",
        "margin": "md"
      },
      {
        "type": "text",
        "text": "📌 問題標題：{{Title}}",
        "wrap": true,
        "weight": "bold",
        "margin": "md"
      },
      {
        "type": "text",
        "text": "⚡ 緊急程度：{{PriorityLevel}}",
        "wrap": true,
        "margin": "sm"
      },
      {
        "type": "text",
        "text": "🏢 所屬單位：{{Department}}",
        "wrap": true,
        "margin": "sm"
      },
      {
        "type": "text",
        "text": "👤 聯絡人：{{ContactPerson}}",
        "wrap": true,
        "margin": "sm"
      },
      {
        "type": "text",
        "text": "📞 連絡電話：{{ContactPhone}}",
        "wrap": true,
        "margin": "sm"
      },
      {
        "type": "text",
        "text": "📅 紀錄日期：{{RecordedDate}}",
        "wrap": true,
        "margin": "sm"
      },
      {
        "type": "text",
        "text": "✍️ 回報人：{{Reporter}}",
        "wrap": true,
        "margin": "sm"
      }
    ]
  },
  "footer": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "button",
        "action": {
          "type": "uri",
          "label": "查看回報單詳情",
          "uri": "{{DetailUrl}}"
        },
        "style": "primary",
        "color": "#1DB446"
      }
    ]
  }
}
```

**Placeholder Mapping**:

- `{{IssueNumber}}`: 回報單編號
- `{{Title}}`: 問題標題
- `{{PriorityLevel}}`: 緊急程度 (高 / 中 / 低)
- `{{Department}}`: 問題所屬單位名稱
- `{{ContactPerson}}`: 聯絡人
- `{{ContactPhone}}`: 連絡電話
- `{{RecordedDate}}`: 紀錄日期 (格式: yyyy-MM-dd HH:mm)
- `{{Reporter}}`: 回報人名稱
- `{{DetailUrl}}`: 回報單詳細頁面 URL

### 2. 對話流程訊息範例

#### 開始回報 (START → TITLE)

```json
{
  "type": "text",
  "text": "歡迎使用 ClarityDesk 問題回報系統！\n\n請輸入問題標題：",
  "quickReply": {
    "items": [
      {
        "type": "action",
        "action": {
          "type": "message",
          "label": "取消",
          "text": "取消"
        }
      }
    ]
  }
}
```

#### 選擇單位 (CONTENT → DEPARTMENT)

```json
{
  "type": "text",
  "text": "請選擇問題所屬單位：",
  "quickReply": {
    "items": [
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "客服部",
          "data": "DEPARTMENT:1"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "技術部",
          "data": "DEPARTMENT:2"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "業務部",
          "data": "DEPARTMENT:3"
        }
      }
    ]
  }
}
```

#### 選擇緊急程度 (DEPARTMENT → PRIORITY)

```json
{
  "type": "text",
  "text": "請選擇緊急程度：",
  "quickReply": {
    "items": [
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "🔴 高",
          "data": "PRIORITY:High"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "🟡 中",
          "data": "PRIORITY:Medium"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "🟢 低",
          "data": "PRIORITY:Low"
        }
      }
    ]
  }
}
```

#### 確認摘要 (CONTACT\_PHONE → CONFIRM)

```json
{
  "type": "text",
  "text": "📋 請確認回報內容：\n\n📌 問題標題：系統登入異常\n📝 問題內容：無法使用 LINE Login 登入系統\n🏢 所屬單位：技術部\n⚡ 緊急程度：高\n👤 聯絡人：王小明\n📞 連絡電話：0912345678\n📅 紀錄日期：2026-01-11 14:30\n✍️ 回報人：王小明\n📊 處理狀態：待處理\n👨‍💼 指派處理人員：張工程師",
  "quickReply": {
    "items": [
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "✅ 確認送出",
          "data": "CONFIRM:YES"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "postback",
          "label": "🔄 重新填寫",
          "data": "CONFIRM:RETRY"
        }
      },
      {
        "type": "action",
        "action": {
          "type": "message",
          "label": "❌ 取消",
          "text": "取消"
        }
      }
    ]
  }
}
```

#### 完成回報 (CONFIRM → COMPLETED)

```json
{
  "type": "text",
  "text": "✅ 回報成功！\n\n回報單編號：#12345\n\n您可以點擊下方連結查看詳情：\nhttps://claritydesk.example.com/Issues/Details/12345"
}
```

---

## Error Responses

### Webhook Signature Verification Failed

```json
{
  "error": "Invalid signature",
  "message": "Webhook signature verification failed"
}
```

### LINE User Not Bound

```json
{
  "error": "User not bound",
  "message": "請先完成帳號綁定。請登入網頁系統並前往個人設定頁面綁定 LINE 帳號。"
}
```

### Conversation Timeout

```json
{
  "type": "text",
  "text": "⏰ 回報流程已逾時(超過 15 分鐘未回應)\n\n請重新輸入「回報問題」開始新的回報流程。"
}
```

### Invalid Input Format

```json
{
  "type": "text",
  "text": "❌ 電話號碼格式不正確！\n\n請輸入有效的電話號碼(例如：0912345678)"
}
```

---

## Postback Data Format

LINE Postback 動作的 `data` 欄位格式：

| Action | Data Format                 | Example         |
| ------ | --------------------------- | --------------- |
| 選擇單位   | `DEPARTMENT:{DepartmentId}` | `DEPARTMENT:1`  |
| 選擇緊急程度 | `PRIORITY:{Level}`          | `PRIORITY:High` |
| 確認送出   | `CONFIRM:YES`               | `CONFIRM:YES`   |
| 重新填寫   | `CONFIRM:RETRY`             | `CONFIRM:RETRY` |

**Parsing Logic**:

```csharp
var parts = postbackData.Split(':');
var action = parts[0]; // DEPARTMENT, PRIORITY, CONFIRM
var value = parts[1];  // DepartmentId, Level, YES/RETRY
```

---

## Configuration Schema

### appsettings.json

```json
{
  "LineMessaging": {
    "ChannelAccessToken": "[從 LINE Developers Console 取得]",
    "ChannelSecret": "[從 LINE Developers Console 取得]",
    "WebhookUrl": "https://yourdomain.com/api/line/webhook",
    "FlexTemplates": {
      "NewIssueNotification": "[Flex Message JSON - 見上方範例]"
    },
    "ConversationTimeout": 900,
    "RetryPolicy": {
      "MaxRetryCount": 3,
      "RetryIntervals": [1, 5, 15]
    }
  }
}
```

**Configuration Model**:

```csharp
public class LineMessagingOptions
{
    public string ChannelAccessToken { get; set; } = string.Empty;
    public string ChannelSecret { get; set; } = string.Empty;
    public string WebhookUrl { get; set; } = string.Empty;
    public Dictionary<string, string> FlexTemplates { get; set; } = new();
    public int ConversationTimeout { get; set; } = 900; // 秒
    public RetryPolicyOptions RetryPolicy { get; set; } = new();
}

public class RetryPolicyOptions
{
    public int MaxRetryCount { get; set; } = 3;
    public List<int> RetryIntervals { get; set; } = new() { 1, 5, 15 }; // 秒
}
```

---

## Summary

本合約文件定義了：

1. ✅ **Web API 端點**：LINE Webhook 處理
2. ✅ **服務介面**：3 個核心服務介面 (`ILineBindingService`, `ILineMessagingService`, `IConversationService`)
3. ✅ **DTOs**：6 個資料傳輸物件 (綁定、對話、推送、訊息)
4. ✅ **訊息格式**：Flex Message 範本和對話流程範例
5. ✅ **錯誤處理**：標準錯誤回應格式
6. ✅ **配置結構**：`appsettings.json` 結構定義

所有合約遵循現有專案慣例 (Service Interface + Implementation，DTO 命名，Extension Methods 映射)。
