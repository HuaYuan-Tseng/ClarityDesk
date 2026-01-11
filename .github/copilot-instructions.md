# ClarityDesk - 問題回報系統

## 專案概述

ClarityDesk 是一個基於 ASP.NET Core 8.0 Razor Pages 的問題回報與追蹤系統，使用 LINE Login 進行身份驗證，支援多單位協作處理客戶回報問題。

## 架構模式

### 分層架構

- **Presentation**: Razor Pages (`Pages/`) - 使用 PageModel 模式
- **Service**: 業務邏輯層 (`Services/`) - 所有服務必須實作 `Interfaces/` 中的介面
- **Data**: 資料存取層 (`Data/`) - Entity Framework Core 搭配 Repository Pattern
- **Models**: 資料模型 (`Models/`)
  - `Entities/`: EF Core 實體
  - `DTOs/`: 資料傳輸物件 (與外部交互使用)
  - `ViewModels/`: 視圖模型 (Razor Pages 專用)
  - `Enums/`: 列舉類型，使用 `[Display(Name = "")]` 屬性
  - `Extensions/`: 實體映射擴充方法 (Entity ↔ DTO 轉換)

### 核心領域

- **IssueReport**: 問題回報主實體，支援多單位指派 (透過 `DepartmentAssignment`)
- **Department**: 單位/部門管理，包含處理人員關聯 (`DepartmentUser`)
- **User**: 使用者管理，支援 LINE Login OAuth，角色分為 `Admin` 和 `User`

## 開發規範

### Entity Framework Core

- **DbContext 位置**: `Data/ApplicationDbContext.cs`
- **Fluent API 配置**: 必須在 `Data/Configurations/` 建立獨立的 `IEntityTypeConfiguration<T>` 類別
- **時間戳記**: 所有實體的 `CreatedAt`/`UpdatedAt` 由 `ApplicationDbContext.SaveChanges` 自動管理
- **連線字串**: 在 `appsettings.json` 的 `ConnectionStrings:DefaultConnection` 配置
- **Migration 指令**:
  ```bash
  dotnet ef migrations add <MigrationName>
  dotnet ef database update
  ```

### DTO 映射模式

使用 Extension Methods 進行實體轉換 (位於 `Models/Extensions/`)：

```csharp
// Entity → DTO
var dto = entity.ToDto();

// CreateDto → Entity
var entity = createDto.ToEntity();

// UpdateDto → Entity (返回是否有變更)
bool hasChanges = entity.UpdateFromDto(updateDto);
```

### 服務層規範

- 所有服務必須注入 `ILogger<T>` 並記錄關鍵操作
- 使用 `IMemoryCache` 快取統計資料或常用查詢，快取 key 定義為 const
- 拋出例外時讓 `ExceptionHandlingMiddleware` 統一處理
- 服務方法支援 `CancellationToken` 參數

### 身份驗證與授權

- **LINE Login**: OAuth 配置在 `Program.cs`，流程：LINE OAuth → `AuthenticationService.LoginOrRegisterWithLineAsync` → 建立/更新本地 User 記錄
- **Claims**:
  - `ClaimTypes.NameIdentifier`: LINE User ID
  - `"UserId"`: 本地 User.Id
  - `ClaimTypes.Role`: UserRole (Admin/User)
- **授權策略**:
  - 頁面層級授權在 `Program.cs` 的 `RazorPagesOptions.Conventions` 配置
  - Admin 頁面需要 `"Admin"` 政策

### Razor Pages 慣例

- PageModel 類別位於 `.cshtml.cs` 檔案
- 使用 `[BindProperty(SupportsGet = true)]` 綁定查詢參數（如篩選條件）
- 分頁參數使用 `Page` 和 `PageSize` 屬性
- 載入列表數據時同時載入相關的下拉選單選項 (Departments, Users)
- Excel 匯出使用 EPPlus 套件

## 關鍵檔案

- **[Program.cs](../Program.cs)**: 應用程式啟動與 DI 容器配置，包含 LINE Login OAuth 設定
- **[ApplicationDbContext.cs](../Data/ApplicationDbContext.cs)**: DbContext 定義與自動時間戳記邏輯
- **[ApplicationDbContextSeed.cs](../Data/ApplicationDbContextSeed.cs)**: 種子資料初始化（預設 Admin 與單位）
- **[ExceptionHandlingMiddleware.cs](../Infrastructure/Middleware/ExceptionHandlingMiddleware.cs)**: 全域例外處理

## 開發工作流程

### 本地開發

```bash
# 執行應用程式
dotnet run

# 監聽模式 (自動重新載入)
dotnet watch run

# 執行測試
dotnet test
```

### 新增功能步驟

1. 定義 Entity 於 `Models/Entities/`
2. 建立 `IEntityTypeConfiguration` 於 `Data/Configurations/`
3. 更新 `ApplicationDbContext.cs` 加入 `DbSet<T>`
4. 執行 Migration: `dotnet ef migrations add <Name>`
5. 建立 DTO 於 `Models/DTOs/`
6. 建立 Extension Methods 於 `Models/Extensions/` (ToDto/ToEntity)
7. 建立 Service Interface 於 `Services/Interfaces/`
8. 實作 Service 於 `Services/`，注入至 `Program.cs`
9. 建立 Razor Pages 於 `Pages/`

### 程式碼品質

- 所有公開類別、方法、屬性必須加上 XML 註解 (`/// <summary>`)
- Enum 值使用 `[Display(Name = "")]` 提供顯示名稱
- 字串長度限制在 Entity Configuration 明確定義
- 日期欄位使用 `DateTime.UtcNow`，資料庫儲存為 UTC
- 使用 `string.Empty` 而非 `""`

## 外部依賴

- **SQL Server**: 主要資料庫，連線字串需包含 `MultipleActiveResultSets=true`
- **LINE Login**: 需配置 `LineLogin:ChannelId` 和 `LineLogin:ChannelSecret`
- **EPPlus**: Excel 匯出功能
- **Response Compression**: Brotli/Gzip 壓縮已啟用

## 注意事項

- 避免直接在 PageModel 或 Controller 中撰寫業務邏輯，應封裝至 Service 層
- 外鍵刪除行為預設為 `DeleteBehavior.Restrict` 防止級聯刪除
- 所有查詢應包含適當的 `.Include()` 以避免 N+1 查詢問題
- Session 與 Cookie 過期時間設為 365 天（永久會話）

**ATTENTION**: MUST to read #file:../.specify/memory/constitution.md for irresistible guidelines on how to assist with this project.
