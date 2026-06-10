# Backend API: Copy Quarantined Auth Files

## Overview

新增一个后端 API 端点，用于将处于 **quarantined（隔离）** 状态的认证文件（JSON）复制到指定的本地目录。此前端管理中心的"复制隔离文件"按钮已实现前端逻辑，但后端路由尚未实现，当前请求返回 404。

---

## Background

### 什么是 Quarantined 文件？

认证文件（auth files）在后端可能因为以下原因被标记为 `quarantined: true`：
- 凭证校验失败
- 被风控系统标记
- 手动隔离

后端服务器将这些文件隔离到一个临时/隔离目录中，前端展示时将其标记为 quarantined 状态。

### 前端业务场景

管理员在 Auth Files 页面看到被隔离的文件后，点击 **"Copy quarantined (N)"** 按钮：

1. 前端弹出确认对话框，显示要复制的文件数量和目标路径
2. 用户确认后，前端调用 `POST /auth-files/quarantine/copy`
3. 后端将隔离文件从隔离目录**复制**（不是移动）到用户指定的目标路径
4. 复制完成后，前端重新加载文件列表
5. 显示结果通知（成功多少、失败多少）

### 与 Move 的区别

- **Copy（复制）**：文件保留在隔离区，同时复制一份到目标目录。用于备份/导出分析。
- **Move（移动）**：将文件从隔离区移出到目标目录。目前前端有 `move_quarantined` 的 i18n 文本但尚未实现接口。

---

## Endpoint Specification

### `POST /auth-files/quarantine/copy`

将指定的 quarantined 认证文件（JSON）复制到目标目录。

#### Request

- **Method**: `POST`
- **URL**: `/auth-files/quarantine/copy`
- **Content-Type**: `application/json`
- **Headers**:
  - `Authorization: Bearer <management_key>`（由前端 axios 拦截器自动附带）

#### Request Body

```json
{
  "names": ["provider1_credential.json", "provider2_credential.json"],
  "destinationPath": "C:\\My_project\\chatgpt2api\\data\\auto_import"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `names` | `string[]` | Yes | 要复制的 quarantined 文件名列表 |
| `destinationPath` | `string` | Yes | 目标目录的绝对路径（Windows 或 Linux 格式） |

#### Success Response (200)

**全部成功：**

```json
{
  "status": "ok",
  "copied": 2,
  "files": ["provider1_credential.json", "provider2_credential.json"],
  "failed": [],
  "destinationPath": "C:\\My_project\\chatgpt2api\\data\\auto_import"
}
```

**部分成功：**

```json
{
  "status": "partial",
  "copied": 1,
  "files": ["provider1_credential.json"],
  "failed": [
    {
      "name": "provider2_credential.json",
      "error": "Destination directory does not exist"
    }
  ],
  "destinationPath": "C:\\My_project\\chatgpt2api\\data\\auto_import"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | `string` | `"ok"` 全部成功，`"partial"` 部分失败 |
| `copied` | `number` | 成功复制的文件数量 |
| `files` | `string[]` | 成功复制的文件名列表 |
| `failed` | `object[]` | 失败的文件列表及其错误信息 |
| `failed[].name` | `string` | 失败的文件名 |
| `failed[].error` | `string` | 错误描述 |
| `destinationPath` | `string` | 实际复制到的目标路径（可能与请求中的一致或由后端规范化处理） |

#### Error Responses

| Status | Description |
|--------|-------------|
| `400` | 请求体缺少必填字段（`names` 或 `destinationPath`） |
| `404` | 指定的某些文件在隔离区中不存在 |
| `500` | 文件系统操作失败（权限、磁盘空间等） |

---

## Business Logic Requirements

### 1. 输入校验

```
- names: 非空数组，每个元素为有效的文件名字符串
- destinationPath: 非空字符串，必须是一个有效的且存在的目录路径
```

### 2. 文件查找

- 在后端的 quarantined 文件存储目录中，按 `names` 列表查找对应的文件
- quarantined 文件的存储路径由后端实现决定（如 `{data_dir}/quarantined/{filename}`）
- **不存在的文件**：列入 `failed` 列表，error 信息写明文件不存在

### 3. 文件复制

- 将文件从 quarantined 目录**复制**到 `destinationPath` 目录
- 使用原始文件名复制（不做重命名）
- 如果目标路径已存在同名文件：**覆盖**（或按后端策略处理，前端不感知）

### 4. 权限校验

- 确保后端进程对 quarantined 源目录有**读取**权限
- 确保后端进程对 `destinationPath` 有**写入**权限
- 权限不足时，将对应文件列入 `failed` 列表并给出明确错误信息

### 5. 路径安全

- 对 `destinationPath` 做基本的路径遍历防护（防止 `../` 逃逸到预期之外）
- 确保复制操作不会影响 quarantined 隔离区的源文件（仅读取，不删除/移动）

### 6. 返回值

- 必须返回 `status`、`copied`、`files`、`failed`、`destinationPath` 五个字段
- `status` 使用 `"ok"` 或 `"partial"`（不是 `"success"` / `"error"` — 注意前端解析逻辑）
- `destinationPath` 建议返回后端的规范化路径（方便前端展示）
- 如果全部成功，`failed` 返回空数组 `[]`

---

## Frontend Integration Reference

### API Call Site

`src/services/api/authFiles.ts:520-541`

```typescript
copyQuarantined: async (
  names: string[],
  destinationPath: string
): Promise<AuthFileBatchCopyResult> => {
  const requestedNames = normalizeRequestedAuthFileNames(names);
  const normalizedDestinationPath = String(destinationPath ?? '').trim();

  const payload = await apiClient.post<AuthFileBatchCopyResponse>(
    '/auth-files/quarantine/copy',
    {
      names: requestedNames,
      destinationPath: normalizedDestinationPath,
    }
  );
  return normalizeBatchCopyResponse(payload, requestedNames, normalizedDestinationPath);
},
```

### Response Type Used by Frontend

```typescript
// Raw response from backend (flexible field names)
type AuthFileBatchCopyResponse = {
  status?: string;
  copied?: number;
  files?: unknown;         // string[] expected
  failed?: unknown;        // Array<{name, error}> expected
  destinationPath?: string; // takes priority
  destination_path?: string; // fallback
};

// Normalized result used in UI
export type AuthFileBatchCopyResult = {
  status: string;
  copied: number;
  files: string[];
  failed: AuthFileBatchFailure[];
  destinationPath: string;
};

type AuthFileBatchFailure = {
  name: string;
  error: string;
};
```

### UI Button Location

`src/pages/AuthFilesPage.tsx` — header actions bar:

```tsx
<Button
  variant="secondary"
  size="sm"
  onClick={handleCopyQuarantined}
  disabled={/* quarantinedCopyableFiles.length === 0 */}
  loading={copyingQuarantined}
>
  {t('auth_files.copy_quarantined_button', { count: quarantinedCopyableFiles.length })}
</Button>
```

### User Flow

1. 页面加载后，`quarantinedCopyableFiles` 通过 `useMemo` 筛选出所有 `quarantined === true` 且非 runtime-only 的文件
2. 用户点击按钮 → 弹出确认框（显示文件数量和目标路径）
3. 用户确认 → 调用 API → 按钮显示 loading 状态
4. API 返回 → 刷新文件列表 (`loadFiles()`)
5. 显示成功/部分成功/失败通知

---

## Existing Similar Endpoints (for Reference)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/auth-files` | `POST` (multipart) | 上传认证文件 |
| `/auth-files` | `DELETE` | 批量删除认证文件 |
| `/auth-files/status` | `PATCH` | 启用/禁用认证文件 |
| `/auth-files/unquarantine` | `PATCH` | 解除文件的隔离状态 |
| `/auth-files/unfreeze` | `PATCH` | 解除文件的冻结状态 |
| `/auth-files/quarantine/copy` | `POST` | **【新】复制隔离文件到指定目录** |

---

## Implementation Checklist

- [ ] 添加路由 `POST /auth-files/quarantine/copy`
- [ ] 请求体解析与校验（`names` 非空数组，`destinationPath` 非空字符串）
- [ ] 目标目录存在性检查
- [ ] 源文件存在性检查（不存在的列入 `failed`）
- [ ] 文件复制操作（读取源文件 → 写入目标目录）
- [ ] 权限检查与错误处理
- [ ] 路径遍历防护
- [ ] 返回符合规范的 JSON 响应（`status` / `copied` / `files` / `failed` / `destinationPath`）
- [ ] 日志记录（记录复制操作的源、目标、结果）
