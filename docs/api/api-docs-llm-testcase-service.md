# 接口文档（自动生成）

生成时间：2026-11-18T15:58:00+08:00
项目：llm-testcase-service
扫描方式：api-doc-generator（源码静态扫描，Python / FastAPI）
扫描范围：项目根目录全部 `*.py`（排除 `tests/`、`.venv/`、`__pycache__/`）

> ⚠️ 本文档由源码扫描生成，**只反映接口结构**（路径、方法、参数、字段类型）。
> 业务语义（什么情况返回 404 / 422、字段取值范围背后的业务原因）需人工补充，扫描器无法从装饰器推断。

## 接口总览

| # | 方法 | 完整路径 | 说明 | 来源 |
|---|------|---------|------|------|
| 1 | GET | `/` | 健康检查 | `main.py` · `root` |
| 2 | POST | `/api/v1/generate` | 生成测试用例并存入历史 | `routers/testcase.py` · `generate` |
| 3 | GET | `/api/v1/history` | 查询全部生成历史 | `routers/testcase.py` · `history` |
| 4 | GET | `/api/v1/history/{record_id}` | 查询单条生成记录 | `routers/testcase.py` · `get_record` |
| 5 | DELETE | `/api/v1/history/{record_id}` | 删除单条生成记录 | `routers/testcase.py` · `delete_record` |
| 6 | GET | `/api/v1/history/{record_id}/export` | 导出该记录的 Markdown 文本 | `routers/testcase.py` · `export_markdown` |

**前缀来源说明**：`/api/v1` 定义在 `routers/testcase.py` 的 `APIRouter(prefix="/api/v1")` 中；
`main.py` 的 `app.include_router(router)` **未附加额外前缀**。

---

## testcase-service 测试用例服务

### GET /

**接口说明**
健康检查，确认服务已启动。

**所属控制器**
`main.py` · `root`

**响应（Responses）**

- `200 OK` · `application/json`

  | 字段 | 类型 | 必填 | 说明 |
  | ---- | ---- | ---- | ---- |
  | message | string | 是 | 固定值「服务运行中」 |
  | status | string | 是 | 固定值 `ok` |

---

### POST /api/v1/generate

**接口说明**
输入功能名与平台，调用大模型生成结构化测试用例，并把结果作为一条历史记录保存。

**所属控制器**
`routers/testcase.py` · `generate`

**请求体（Request Body）**
Content-Type: `application/json`
模型：`models.GenerateRequest`

| 字段 | 类型 | 必填 | 约束 | 说明 |
| ---- | ---- | ---- | ---- | ---- |
| feature | string | 是 | `min_length=1` | 功能名称，如 `login`、`扫码支付`；空串触发 422 |
| platform | string | 否 | 默认 `Android` | 平台：`Android` / `iOS` / `Web` |
| test_types | string[] | 否 | 默认四类全选 | 用例类型：`normal` / `exception` / `boundary` / `security`；传 `[]` 则不调用模型 |
| case_count | integer | 否 | 默认 `2`，范围 `1~3` | 每类生成的用例条数；越界触发 422 |

**响应（Responses）**

- `200 OK` · `application/json` · 模型 `HistoryRecord`

  | 字段 | 类型 | 必填 | 说明 |
  | ---- | ---- | ---- | ---- |
  | id | integer | 是 | 记录自增 ID |
  | feature | string | 是 | 功能名 |
  | platform | string | 是 | 平台 |
  | test_types | string[] | 是 | 本次生成的用例类型 |
  | case_count | integer | 是 | 每类条数 |
  | cases | object | 是 | 键为类型名，值为该类的 Markdown 表格文本 |
  | created_at | string | 是 | 格式 `YYYY-MM-DD HH:MM:SS` |

- `422 Unprocessable Entity` · 参数校验失败（`feature` 空串 / `case_count` 越界 / 缺必填）

---

### GET /api/v1/history

**接口说明**
查询全部生成历史（内存存储，服务重启后清空）。

**所属控制器**
`routers/testcase.py` · `history`

**参数**
无

**响应（Responses）**

- `200 OK` · `application/json` · `HistoryRecord` 数组

  | 字段 | 类型 | 必填 | 说明 |
  | ---- | ---- | ---- | ---- |
  | [].id | integer | 是 | 记录 ID |
  | [].feature | string | 是 | 功能名 |
  | [].platform | string | 是 | 平台 |
  | [].test_types | string[] | 是 | 用例类型 |
  | [].case_count | integer | 是 | 每类条数 |
  | [].cases | object | 是 | 各类 Markdown 文本 |
  | [].created_at | string | 是 | 创建时间 |

> 空库时返回 `[]`，不会报错。

---

### GET /api/v1/history/{record_id}

**接口说明**
按 ID 查询单条生成记录。

**所属控制器**
`routers/testcase.py` · `get_record`

**路径参数**

| 名称 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| record_id | integer | 是 | 记录 ID |

**响应（Responses）**

- `200 OK` · `application/json` · `HistoryRecord`（字段同 `POST /api/v1/generate` 响应）
- `404 Not Found` · `{ "detail": "记录 {record_id} 不存在" }`
- `422 Unprocessable Entity` · `record_id` 不是整数（如传 `abc`）

---

### DELETE /api/v1/history/{record_id}

**接口说明**
按 ID 删除单条生成记录。

**所属控制器**
`routers/testcase.py` · `delete_record`

**路径参数**

| 名称 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| record_id | integer | 是 | 记录 ID |

**响应（Responses）**

- `200 OK` · `application/json`

  | 字段 | 类型 | 必填 | 说明 |
  | ---- | ---- | ---- | ---- |
  | message | string | 是 | 固定值「删除成功」 |

- `404 Not Found` · `{ "detail": "记录 {record_id} 不存在" }`
- `422 Unprocessable Entity` · `record_id` 不是整数

---

### GET /api/v1/history/{record_id}/export

**接口说明**
把某条记录的用例拼接为完整 Markdown 文本返回。

**所属控制器**
`routers/testcase.py` · `export_markdown`

**路径参数**

| 名称 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| record_id | integer | 是 | 记录 ID |

**响应（Responses）**

- `200 OK` · `text/plain`（FastAPI 按返回类型推断为字符串）
  返回体为一整段 Markdown 文本，结构为 `# {feature} {platform} 测试用例` + 每类一个 `## {类型}` 小节。
- `404 Not Found` · `{ "detail": "记录 {record_id} 不存在" }`
- `422 Unprocessable Entity` · `record_id` 不是整数

---

## 扫描能力说明（本次实测）

| 检查项 | 结果 |
|--------|------|
| 路由装饰器识别 | ✅ 6/6 全部识别 |
| `APIRouter(prefix=...)` 前缀合并 | ✅ 已合并进完整路径 |
| `include_router` 额外前缀 | ✅ 识别为「无」，未重复拼接 |
| 路径参数类型（`record_id: int`） | ✅ 识别为 integer |
| Pydantic 请求体字段与约束 | ✅ 含 default / ge / le / min_length |
| 响应模型展开（HistoryRecord） | ✅ 已展开为字段表 |
| 业务错误码（404 / 422 触发条件） | ⚠️ 部分推断（422 由约束可推，404 需读 `raise HTTPException`） |
| 响应 Content-Type（export 为纯文本） | ⚠️ 按返回类型推断，未在代码中显式声明 |
