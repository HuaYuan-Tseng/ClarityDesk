# Quickstart Guide: LINE Integration

**Date**: 2026-01-11\
**Feature**: LINE 整合功能\
**Audience**: 開發團隊

## Overview

本指南提供 LINE 整合功能的快速開發指引，包含環境設定、開發步驟、測試方法和部署注意事項。

## Prerequisites

### LINE Platform Setup

1. **建立 LINE Official Account**
   - 前往 [LINE Official Account Manager](https://manager.line.biz/)
   - 建立新的官方帳號 (建議建立測試用和正式環境各一個)

2. **建立 LINE Messaging API Channel**
   - 前往 [LINE Developers Console](https://developers.line.biz/console/)
   - 建立新的 Provider (或使用現有的)
   - 建立 Messaging API Channel
   - 記錄以下資訊：
     - **Channel ID**
     - **Channel Secret**
     - **Channel Access Token** (Long-lived)

3. **設定 Webhook URL**
   - 在 Messaging API Settings 中設定 Webhook URL
   - 測試環境可使用 [ngrok](https://ngrok.com/) 建立臨時 HTTPS 端點
   - 正式環境需使用有效的 SSL 憑證網域

### Development Environment

- .NET 8.0 SDK
- SQL Server (LocalDB 或完整版)
- Visual Studio 2022 或 Rider
- Postman 或類似的 API 測試工具
- LINE Official Account (測試用)

## Installation Steps

### Step 1: 安裝 NuGet 套件

在專案根目錄執行：

```bash
dotnet add package LineDevelopers.MessagingApi --version 2.0.0
dotnet add package Polly --version 8.4.0
```

或編輯 `ClarityDesk.csproj`：

```xml
<PackageReference Include="LineDevelopers.MessagingApi" Version="2.0.0" />
<PackageReference Include="Polly" Version="8.4.0" />
```

### Step 2: 設定配置檔案

編輯 `appsettings.json`(**不要提交到 Git**)：

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ClarityDesk;Trusted_Connection=true;MultipleActiveResultSets=true"
  },
  "LineLogin": {
    "ChannelId": "[現有 LINE Login Channel ID]",
    "ChannelSecret": "[現有 LINE Login Channel Secret]"
  },
  "LineMessaging": {
    "ChannelAccessToken": "[Messaging API Channel Access Token]",
    "ChannelSecret": "[Messaging API Channel Secret]",
    "WebhookUrl": "https://yourdomain.com/api/line/webhook",
    "ConversationTimeout": 900,
    "RetryPolicy": {
      "MaxRetryCount": 3,
      "RetryIntervals": [1, 5, 15]
    }
  }
}
```

建立 `appsettings.Development.json` 用於本地開發：

```json
{
  "LineMessaging": {
    "WebhookUrl": "https://your-ngrok-url.ngrok.io/api/line/webhook"
  }
}
```

**注意**：將 LINE 相關設定加入 `.gitignore` (或使用 User Secrets)。

### Step 3: 執行資料庫 Migration

```bash
# 建立 Migration
dotnet ef migrations add AddLineIntegrationTables

# 更新資料庫
dotnet ef database update
```

**預期產生的 Migration**:

- `LineBindings` 表
- `ConversationStates` 表
- `PushNotificationLogs` 表
- 相關索引和外鍵

### Step 4: 註冊服務至 DI 容器

編輯 `Program.cs`，在現有服務註冊之後新增：

```csharp
// 註冊 LINE Messaging Options
builder.Services.Configure<LineMessagingOptions>(
    builder.Configuration.GetSection("LineMessaging"));

// 註冊 LINE 整合服務
builder.Services.AddScoped<ILineBindingService, LineBindingService>();
builder.Services.AddScoped<ILineMessagingService, LineMessagingService>();
builder.Services.AddScoped<IConversationService, ConversationService>();

// 註冊 Background Service (對話狀態清理)
builder.Services.AddHostedService<ConversationCleanupService>();

// 註冊 LINE Messaging API Client
builder.Services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<IOptions<LineMessagingOptions>>().Value;
    return new LineDevelopers.MessagingApi.MessagingApiClient(options.ChannelAccessToken);
});
```

### Step 5: 建立實體和配置

**Entity 檔案位置**：

- `Models/Entities/LineBinding.cs`
- `Models/Entities/ConversationState.cs`
- `Models/Entities/PushNotificationLog.cs`
- `Models/Enums/PushNotificationStatus.cs`

**Configuration 檔案位置**：

- `Data/Configurations/LineBindingConfiguration.cs`
- `Data/Configurations/ConversationStateConfiguration.cs`
- `Data/Configurations/PushNotificationLogConfiguration.cs`

**更新 `ApplicationDbContext.cs`**：

```csharp
public DbSet<LineBinding> LineBindings { get; set; }
public DbSet<ConversationState> ConversationStates { get; set; }
public DbSet<PushNotificationLog> PushNotificationLogs { get; set; }

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);
    
    // 現有配置...
    modelBuilder.ApplyConfiguration(new LineBindingConfiguration());
    modelBuilder.ApplyConfiguration(new ConversationStateConfiguration());
    modelBuilder.ApplyConfiguration(new PushNotificationLogConfiguration());
}
```

### Step 6: 實作服務層

**服務檔案位置**：

- `Services/Interfaces/ILineBindingService.cs`
- `Services/Interfaces/ILineMessagingService.cs`
- `Services/Interfaces/IConversationService.cs`
- `Services/LineBindingService.cs`
- `Services/LineMessagingService.cs`
- `Services/ConversationService.cs`
- `Infrastructure/Services/ConversationCleanupService.cs` (Background Service)

**實作要點**：

- 所有服務注入 `ILogger<T>`
- `LineMessagingService` 使用 Polly 實作重試邏輯
- `ConversationService` 處理狀態轉換和資料驗證

### Step 7: 建立 DTOs 和 Extension Methods

**DTOs 檔案位置**：

- `Models/DTOs/LineBindingDto.cs`
- `Models/DTOs/ConversationStateDto.cs`
- `Models/DTOs/PushNotificationResultDto.cs`
- `Models/DTOs/LineMessageDto.cs`

**Extension Methods 檔案位置**：

- `Models/Extensions/LineBindingExtensions.cs`
- `Models/Extensions/ConversationStateExtensions.cs`

**範例 Extension Method**：

```csharp
// Models/Extensions/LineBindingExtensions.cs
public static class LineBindingExtensions
{
    public static LineBindingDto ToDto(this LineBinding entity)
    {
        return new LineBindingDto
        {
            Id = entity.Id,
            UserId = entity.UserId,
            LineUserId = entity.LineUserId,
            DisplayName = entity.DisplayName,
            PictureUrl = entity.PictureUrl,
            BoundAt = entity.BoundAt,
            IsActive = entity.IsActive
        };
    }
    
    public static LineBinding ToEntity(this LineUserProfileDto dto, int userId)
    {
        return new LineBinding
        {
            UserId = userId,
            LineUserId = dto.UserId,
            DisplayName = dto.DisplayName,
            PictureUrl = dto.PictureUrl,
            BoundAt = DateTime.UtcNow,
            IsActive = true
        };
    }
}
```

### Step 8: 建立 Webhook Controller

**檔案位置**：`Controllers/LineWebhookController.cs`

```csharp
[ApiController]
[Route("api/line/webhook")]
public class LineWebhookController : ControllerBase
{
    private readonly ILineMessagingService _messagingService;
    private readonly IConversationService _conversationService;
    private readonly ILogger<LineWebhookController> _logger;
    private readonly IConfiguration _configuration;
    
    public LineWebhookController(
        ILineMessagingService messagingService,
        IConversationService conversationService,
        ILogger<LineWebhookController> logger,
        IConfiguration configuration)
    {
        _messagingService = messagingService;
        _conversationService = conversationService;
        _logger = logger;
        _configuration = configuration;
    }
    
    [HttpPost]
    public async Task<IActionResult> Webhook()
    {
        // 1. 驗證簽章
        var signature = Request.Headers["X-Line-Signature"].FirstOrDefault();
        var body = await new StreamReader(Request.Body).ReadToEndAsync();
        
        if (!VerifySignature(body, signature))
        {
            return Unauthorized(new { error = "Invalid signature" });
        }
        
        // 2. 解析事件
        var webhookEvent = JsonSerializer.Deserialize<LineWebhookEvent>(body);
        
        // 3. 非同步處理事件 (避免 Webhook 超時)
        _ = ProcessEventsAsync(webhookEvent);
        
        // 4. 立即回應 200 OK
        return Ok(new { status = "ok" });
    }
    
    private bool VerifySignature(string body, string? signature)
    {
        // 實作 HMAC-SHA256 簽章驗證
        // 參考：https://developers.line.biz/en/docs/messaging-api/receiving-messages/#verifying-signatures
    }
    
    private async Task ProcessEventsAsync(LineWebhookEvent webhookEvent)
    {
        // 處理各類事件：message, postback, follow, unfollow
    }
}
```

### Step 9: 建立 Razor Pages (LINE 綁定頁面)

**檔案位置**：

- `Pages/Account/LineBinding.cshtml`
- `Pages/Account/LineBinding.cshtml.cs`

```csharp
public class LineBindingModel : PageModel
{
    private readonly ILineBindingService _lineBindingService;
    
    public LineBindingDto? Binding { get; set; }
    
    public async Task<IActionResult> OnGetAsync()
    {
        var userId = int.Parse(User.FindFirst("UserId")!.Value);
        Binding = await _lineBindingService.GetBindingByUserIdAsync(userId);
        return Page();
    }
    
    public async Task<IActionResult> OnPostUnbindAsync()
    {
        var userId = int.Parse(User.FindFirst("UserId")!.Value);
        await _lineBindingService.UnbindAsync(userId);
        return RedirectToPage();
    }
}
```

### Step 10: 整合推送通知至 IssueReportService

編輯 `Services/IssueReportService.cs`，在建立回報單後新增：

```csharp
public async Task<IssueReportDto> CreateAsync(
    CreateIssueReportDto createDto, 
    CancellationToken cancellationToken = default)
{
    // ... 現有建立邏輯 ...
    
    var issueReport = createDto.ToEntity();
    _context.IssueReports.Add(issueReport);
    await _context.SaveChangesAsync(cancellationToken);
    
    // 新增：發送 LINE 推送通知給處理人員
    var handlerUserIds = issueReport.DepartmentAssignments
        .Select(da => da.AssignedUserId)
        .Distinct()
        .ToList();
    
    if (handlerUserIds.Any())
    {
        await _lineMessagingService.SendNewIssueNotificationAsync(
            issueReport.Id, 
            handlerUserIds, 
            cancellationToken);
    }
    
    return issueReport.ToDto();
}
```

## Development Workflow

### Local Development with ngrok

1. **啟動應用程式**：
   ```bash
   dotnet run
   ```

2. **啟動 ngrok** (另一個終端)：
   ```bash
   ngrok http https://localhost:7001
   ```

3. **複製 ngrok URL** 並更新：
   - `appsettings.Development.json` 的 `LineMessaging:WebhookUrl`
   - LINE Developers Console 的 Webhook URL

4. **測試 Webhook**：
   - 在 LINE Developers Console 點擊「Verify」按鈕
   - 應看到「Success」訊息

### Testing Strategy

#### 1. Unit Tests (xUnit)

**測試範例**：

```csharp
public class LineBindingServiceTests
{
    [Fact]
    public async Task CreateBindingAsync_WithValidData_ReturnsBindingDto()
    {
        // Arrange
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
        
        var context = new ApplicationDbContext(options);
        var service = new LineBindingService(context, Mock.Of<ILogger<LineBindingService>>());
        
        var lineProfile = new LineUserProfileDto
        {
            UserId = "U1234567890",
            DisplayName = "Test User",
            PictureUrl = "https://example.com/pic.jpg"
        };
        
        // Act
        var result = await service.CreateBindingAsync(1, lineProfile);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal("U1234567890", result.LineUserId);
    }
    
    [Fact]
    public async Task CreateBindingAsync_WithDuplicateLineUserId_ThrowsException()
    {
        // ... 測試重複綁定邏輯
    }
}
```

#### 2. Integration Tests

**測試 Webhook 端點**：

```csharp
public class LineWebhookControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public LineWebhookControllerTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task Webhook_WithValidSignature_ReturnsOk()
    {
        // Arrange
        var webhookBody = JsonSerializer.Serialize(new
        {
            destination = "U1234567890",
            events = new[] { /* ... */ }
        });
        
        var signature = GenerateSignature(webhookBody, "channel-secret");
        var content = new StringContent(webhookBody, Encoding.UTF8, "application/json");
        content.Headers.Add("X-Line-Signature", signature);
        
        // Act
        var response = await _client.PostAsync("/api/line/webhook", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

#### 3. Manual Testing with LINE

**測試綁定流程**：

1. 登入系統
2. 前往個人設定頁面
3. 點擊「綁定 LINE 官方帳號」
4. 掃描 QR Code 加入官方帳號
5. 驗證綁定狀態顯示

**測試推送通知**：

1. 確保處理人員已綁定 LINE
2. 建立新的回報單並指派給該處理人員
3. 驗證 LINE 收到推送訊息
4. 檢查資料庫 `PushNotificationLogs` 表

**測試對話流程**：

1. 在 LINE 官方帳號中輸入「回報問題」
2. 依序填寫各項資訊
3. 驗證確認摘要正確
4. 確認送出後檢查資料庫 `IssueReports` 表

## Common Issues & Troubleshooting

### Webhook 無法接收事件

**症狀**：LINE Developers Console 顯示 Webhook 驗證失敗

**解決方法**：

1. 檢查 ngrok URL 是否正確設定
2. 確認應用程式正在執行
3. 檢查防火牆設定
4. 驗證 Webhook URL 使用 HTTPS

### 簽章驗證失敗

**症狀**：Webhook 端點回傳 401 Unauthorized

**解決方法**：

1. 確認 `appsettings.json` 的 `ChannelSecret` 正確
2. 檢查簽章演算法實作 (HMAC-SHA256)
3. 確保 Request Body 未被修改

### 推送訊息失敗

**症狀**：`PushNotificationLogs` 表中 `Status = Failed`

**解決方法**：

1. 檢查 `ChannelAccessToken` 是否正確
2. 確認處理人員已綁定 LINE
3. 檢查 Flex Message JSON 格式
4. 查看 `ErrorMessage` 欄位的詳細錯誤

### 對話流程中斷

**症狀**：使用者輸入後無回應

**解決方法**：

1. 檢查 `ConversationStates` 表的 `CurrentStep`
2. 驗證 Postback Data 格式
3. 查看應用程式 Log
4. 確認對話未超時 (15 分鐘)

## Deployment Checklist

### Pre-Deployment

- [ ] 更新 `appsettings.Production.json` 的 LINE 配置
- [ ] 設定正式環境 Webhook URL (必須為 HTTPS)
- [ ] 執行所有測試 (Unit + Integration)
- [ ] 檢查資料庫 Migration 狀態
- [ ] 驗證 Polly 重試策略配置
- [ ] 確認 Background Service 正常運作

### Deployment

- [ ] 執行 `dotnet ef database update` 於正式環境
- [ ] 部署應用程式至伺服器
- [ ] 更新 LINE Developers Console Webhook URL
- [ ] 驗證 Webhook 連線 (點擊 Verify 按鈕)
- [ ] 測試綁定流程
- [ ] 測試推送通知

### Post-Deployment

- [ ] 監控 `PushNotificationLogs` 表的失敗記錄
- [ ] 檢查應用程式 Log 是否有異常
- [ ] 驗證對話流程完整性
- [ ] 確認 Background Service 清理過期對話
- [ ] 設定監控告警 (推送失敗率、Webhook 錯誤)

## Performance Tuning

### Database Optimization

```sql
-- 定期清理過期對話狀態 (手動執行或排程)
DELETE FROM ConversationStates
WHERE IsActive = 0 AND LastInteractionAt < DATEADD(DAY, -1, GETUTCDATE());

-- 定期清理舊的推送日誌 (建議保留 90 天)
DELETE FROM PushNotificationLogs
WHERE SentAt < DATEADD(DAY, -90, GETUTCDATE());

-- 更新統計資訊
UPDATE STATISTICS LineBindings;
UPDATE STATISTICS ConversationStates;
UPDATE STATISTICS PushNotificationLogs;
```

### Caching Strategy

在 `LineMessagingService` 快取 Flex Message 模板：

```csharp
private readonly IMemoryCache _cache;
private const string FlexTemplateKey = "FlexTemplate:NewIssueNotification";

private async Task<string> GetFlexTemplateAsync()
{
    return await _cache.GetOrCreateAsync(FlexTemplateKey, async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24);
        return _configuration["LineMessaging:FlexTemplates:NewIssueNotification"] ?? "";
    });
}
```

### Async Processing

使用 Background Task 處理推送通知 (避免阻塞請求)：

```csharp
// 在 IssueReportService 中
_ = Task.Run(async () =>
{
    await _lineMessagingService.SendNewIssueNotificationAsync(issueId, handlerUserIds);
});
```

## Security Best Practices

1. **Webhook 簽章驗證**：絕對不要跳過簽章驗證
2. **Channel Secret 保護**：使用 User Secrets 或環境變數，不要提交到 Git
3. **HTTPS Only**：正式環境必須使用 HTTPS
4. **Rate Limiting**：實作 API Rate Limiting 防止濫用
5. **Input Validation**：驗證使用者輸入 (電話、標題長度等)
6. **Error Handling**：不要在錯誤訊息中洩露敏感資訊

## Next Steps

完成本 Quickstart 後，繼續進行：

1. **實作任務拆解**：參考 `specs/003-line-integration/tasks.md` (由 `/speckit.tasks` 生成)
2. **撰寫測試**：完成所有 Unit Tests 和 Integration Tests
3. **效能測試**：使用 JMeter 或 k6 測試 Webhook 端點
4. **文件補充**：更新使用者手冊，說明如何綁定 LINE 帳號

## Resources

- [LINE Messaging API Documentation](https://developers.line.biz/en/docs/messaging-api/)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [LineDevelopers.MessagingApi NuGet](https://www.nuget.org/packages/LineDevelopers.MessagingApi/)
- [Polly Documentation](https://github.com/App-vNext/Polly)
- [專案內部文件](../../README.md)

---

**版本**: 1.0.0\
**最後更新**: 2026-01-11\
**維護者**: ClarityDesk 開發團隊
