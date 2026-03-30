---
name: excel-upload-to-s3
description: "将生成 Excel 并上传到 S3 的功能封装为可复用的 skill，便于在项目中抽取、复用和测试。"
scope: workspace
---

# 概要

该 Skill 指导开发者将 `NotUpdateReminder` 任务中“生成 Excel 并上传到 S3，返回预签名 URL”的逻辑封装为可复用模块，并提供实现步骤、示例、测试建议和接入说明，便于在其它任务中复用。
# 适用场景

- 需要把任意结构化数据（struct 切片、map 切片或二维行列）导出为 Excel 并通过 S3 分享下载链接。
- 希望把 Excel 生成与 S3 上传分离，便于单元测试与复用，并支持多种数据源格式（struct、map、[][]interface{}）。

# 输入 / 输出

- 输入（示例）：
  - `headers []string, rows [][]interface{}` — 最直接的表头 + 行数据形式；或
  - `rows []map[string]interface{}` — 字段名到值的映射切片；或
  - `items []T`（任意 struct）配合 `[]ColumnDef`/tag 描述列映射。
- 输出：预签名下载 URL (`string`) 或错误 (`error`)（或仅返回 `[]byte` 由上层上传）。

# 推荐函数签名（建议）

- 通用列定义：

  ```go
  type ColumnDef struct {
      Title  string                 // 列标题
      Field  string                 // struct/map 字段名（可选）
      Width  float64                // 建议列宽（像素）
      Format string                 // 可选格式（如日期格式、数字格式）
      Getter func(item interface{}) interface{} // 可选自定义 getter
  }
  ```

- 按行写入（简单）：
  - `func GenerateExcelFromRows(headers []string, rows [][]interface{}) ([]byte, error)`

- 按 map 写入：
  - `func GenerateExcelFromMaps(headers []string, rows []map[string]interface{}) ([]byte, error)`

- 按 struct 写入（泛型，需 Go 1.18+）：
  - `func GenerateExcelFromStructs[T any](items []T, cols []ColumnDef) ([]byte, error)`

- S3 上传（可注入客户端、可配置过期时间）：
  - `func UploadBytesToS3(ctx context.Context, bucket, key string, data []byte, contentType string, presignExpiry time.Duration) (presignedURL string, err error)`

- 对外封装（示例）：
  - `func ExportDataToS3FromRows(ctx context.Context, headers []string, rows [][]interface{}) (string, error)`
  - `func ExportDataToS3FromStructs[T any](ctx context.Context, items []T, cols []ColumnDef) (string, error)`

# 实现步骤（详细）

1. 设计列定义
   - 使用 `ColumnDef` 支持三种映射方式：按 index 的 rows、按字段名的 map、按 struct 的字段/tag 或自定义 getter。
   - 支持列宽、格式化字符串以及空值处理策略。

2. 提取 Excel 生成逻辑
   - 实现 `GenerateExcelFromRows`/`GenerateExcelFromMaps`/`GenerateExcelFromStructs`，内部统一到一个写表格的实现。
   - 对于大数据量，使用 `excelize.NewStreamWriter` 或分块写入以减少内存占用；对于小数据集可以使用 `WriteToBuffer()` 返回 `[]byte`。

3. 抽象 S3 上传
   - 实现 `UploadBytesToS3`，允许注入或传入可替换的 S3 客户端（接口），并支持设置 `presignExpiry`。
   - 不在代码中硬编码凭证或桶名；通过配置或环境注入。保留 presign 逻辑（返回 GET 的预签名 URL）。

4. 对外封装
   - 提供若干便捷函数（`ExportDataToS3FromRows`、`ExportDataToS3FromStructs`），负责：生成 Excel bytes、生成 key（按日期或应用前缀）、调用 `UploadBytesToS3` 并返回 URL。

5. 单元测试
   - `GenerateExcelFromRows`：构造 headers+rows，断言返回 byte slice 包含 ZIP/PK 标识，并使用 `excelize.OpenReader(bytes.NewReader(b))` 验证关键单元格（如第一行表头）。
   - `GenerateExcelFromStructs`：用简单 struct 验证 tag/字段映射和 getter 的行为。
   - `UploadBytesToS3`：通过注入 mock 的 S3 接口模拟 `PutObject` 与 `PresignGetObject`，验证成功/失败路径。

6. 文档与示例
   - 在 skill 目录添加 README，包含示例调用（struct/map/rows 三种），说明如何注入桶名与凭证，以及如何在 CI 中运行测试。

# 代码组织建议

- internal/exporter/excel：实现 Excel 生成函数与 ColumnDef、StreamWriter 支持。
- internal/exporter/adapter：提供从 struct/map -> [][]interface{} 的适配器与 tag 解析（如 `excel:"Title,width=120"`）。
- internal/exporter/s3uploader：实现 `UploadBytesToS3` 与 S3 客户端注入。
- editor_go/app/editor/task/NotUpdateReminder：保留一个兼容 wrapper（调用新的 exporter），减少改动面。

# 接入示例（BookRemindInfo 为示例）

- 使用 struct 适配器：

  ```go
  cols := []ColumnDef{
    {Title: "作品ID", Field: "BookId"},
    {Title: "作品名称", Field: "BookName"},
    {Title: "近7天金豆收入($)", Field: "RevenueLatest7Day", Format: "%.2f"},
  }
  b, _ := GenerateExcelFromStructs(items, cols)
  url, _ := UploadBytesToS3(ctx, bucket, key, b, "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", 7*24*time.Hour)
  ```

# 验收标准（更新）

- `SKILL.md` 已更新为支持任意结构化数据（本文件）。
- 提出并实现的函数签名能覆盖 rows/map/struct 三类输入。
- `GenerateExcel*` 至少有一个单元测试覆盖表头与单元格验证。
- S3 上传逻辑支持注入 mock 并具备测试用例。

# 示例 Agent 提示（可直接复制给 Copilot/Chat）

- 简短指令：
  "把 `editor_go/app/editor/task/NotUpdateReminder/uploadExcelS3.go` 中的实现抽出通用的导出逻辑：实现 `GenerateExcelFromRows(headers, rows)`、`GenerateExcelFromStructs[T](items, cols)`、`UploadBytesToS3(ctx,bucket,key,data,contentType,expiry)`，并给出示例和单元测试。"

- 详细指令：
  "将 `UploadExcelToS3` 拆为：
  1) `internal/exporter/excel` 实现通用 Excel 生成（支持 rows、maps、structs + ColumnDef）；
  2) `internal/exporter/s3uploader` 实现可注入的 S3 上传与 presign；
  3) 在 `NotUpdateReminder` 下保留兼容 wrapper 并调用新包；
  4) 添加 `GenerateExcelFromRows` 的单元测试并运行 `go test ./...`。"

# 相关文件（建议修改）

- editor_go/app/editor/task/NotUpdateReminder/uploadExcelS3.go
- 新建： internal/exporter/excel/excel.go
- 新建： internal/exporter/adapter/adapter.go
- 新建： internal/exporter/s3uploader/s3uploader.go
- 测试： internal/exporter/excel/excel_test.go

# 注意事项

- 不要把 AWS 密钥直接写入仓库：使用项目配置或环境变量注入。
- 对于大数据集建议使用 `excelize` 的 StreamWriter 避免 OOM。
- 预签名链接有效期默认 7 天，可作为参数传入 `UploadBytesToS3`。

---

如果你确认要我按此设计开始在仓库中实施重构（创建 `internal/exporter` 包并添加单测），我可以直接开始并提交修改、运行 `go test`。请回复“开始重构”或提出额外偏好（如 tag 名称、列标签约定等）。