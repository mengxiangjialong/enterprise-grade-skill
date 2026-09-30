# API 设计：完整接口契约示例

> 说明：本文件为 api-design 技能的辅助资源（references/）。当需要产出完整接口契约、或需要一份"合格契约长什么样"的对照时加载本文件作为示例。日常编写接口字段，遵循 SKILL.md 的模板与规则即可。

## 上下文

该示例对应"任务管理"产品：创建任务、列表筛选、删除任务，带用户鉴权与归属控制。对应需求编号 FR-2/FR-3、业务规则 BR-3。

## 1. 通用约定

- 基址 `/api`，RESTful；鉴权 Bearer JWT；错误体 `{ code, message, details? }`
- 分页 `page`/`pageSize`（默认 20/上限 100），响应 `{ items, page, pageSize, total }`
- 排序 `sort=created_at, order=desc`；过滤 `status=0,1`

## 3. 版本策略

URL 版本 `/api/v1/`；v1 弃用窗口 12 个月。

## 6. 接口定义

### POST /api/v1/tasks  创建任务

Permission: 登录用户

Request:
```json
{ "title": "string(必填,≤200)", "due_date": "date?" }
```

Response 201:
```json
{ "id": 10, "title": "写周报", "status": 0, "created_at": "2026-03-01T10:00:00Z" }
```

Idempotency-Key: 支持，客户端生成，服务端按 key 去重。

Errors:
```
401 -> { code: "UNAUTHORIZED" }
422 -> { code: "TITLE_EMPTY", message: "标题不能为空" }
```

### GET /api/v1/tasks  列表（分页/筛选）

Permission: 登录用户（仅返回自己的；管理员可看全部）

Query: `page, pageSize, status?, sort?, order?`

Response 200:
```json
{ "items": [ "...Task" ], "page": 1, "pageSize": 20, "total": 5 }
```

Errors: `401 -> UNAUTHORIZED; 403 -> FORBIDDEN`

### DELETE /api/v1/tasks/{id}

Permission: 创建者本人或管理员

Response 204 无内容（幂等：重复删除返回 204）

Errors: `401 UNAUTHORIZED; 403 FORBIDDEN(非本人); 404 TASK_NOT_FOUND`

## 7. 错误码表（节选）

| HTTP | 业务码 | 含义 |
| ---- | ------ | ---- |
| 401  | UNAUTHORIZED | 未认证 |
| 403  | FORBIDDEN    | 越权 |
| 404  | TASK_NOT_FOUND | 任务不存在 |
| 422  | TITLE_EMPTY  | 标题为空 |

## 9. 追溯

- `POST /api/v1/tasks` → FR-2 创建任务；tasks 表
- `GET  /api/v1/tasks` → FR-3 列表筛选；tasks 表
- `DELETE /api/v1/tasks/{id}` → BR-3 仅创建者可删；tasks 表