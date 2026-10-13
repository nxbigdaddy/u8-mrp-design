# 06　API 设计

## 1. 约定

- 前缀：`/api/v1`
- JSON字段使用 `snake_case`；时间采用 ISO 8601。
- 认证使用企业 OIDC/AD；所有写操作校验角色和数据范围。
- 每个响应返回或透传 `trace_id`。
- OpenAPI是接口契约，但关键接口仍需版本化契约测试。

## 2. 核心资源

```text
POST /api/v1/snapshots
GET  /api/v1/snapshots/{snapshot_id}
POST /api/v1/scenarios
POST /api/v1/runs
GET  /api/v1/runs/{run_id}
POST /api/v1/runs/{run_id}/cancel
GET  /api/v1/runs/{run_id}/exceptions
GET  /api/v1/runs/{run_id}/planned-orders
GET  /api/v1/pegging?demand_id=...
PATCH /api/v1/planned-orders/{plan_id}
POST /api/v1/runs/{run_id}/freeze
POST /api/v1/approval-batches
POST /api/v1/writeback-batches
GET  /api/v1/writeback-batches/{batch_id}
```

## 3. 创建运行

```json
{
  "snapshot_id": "01J...",
  "scenario_id": "base-v3",
  "scope": {
    "organization_codes": ["001"],
    "warehouse_codes": ["01", "02"],
    "item_class_prefixes": ["A"]
  },
  "cutoff_date": "2026-07-30",
  "requested_by": "user-id"
}
```

返回 `202 Accepted` 和 `run_id/status_url`。相同客户端请求通过 `Idempotency-Key` 防止重复创建。

## 4. 查询与分页

大结果集采用游标分页：`limit <= 500`、`next_cursor`。排序字段使用白名单，不接受任意 SQL 字段。导出超过 10,000 行转后台任务并返回下载任务ID。

## 5. 并发编辑

计划建议带 `version`。调整时客户端提交旧版本：

```json
{
  "version": 3,
  "quantity": "120.000000",
  "due_date": "2026-08-15",
  "reason_code": "SUPPLIER_MOQ",
  "comment": "供应商最小起订量"
}
```

版本不一致返回 `409 PLAN_VERSION_CONFLICT`，禁止后写覆盖先写。

## 6. 错误模型

```json
{
  "code": "SNAPSHOT_VALIDATION_FAILED",
  "message": "数据快照未通过质量门禁",
  "details": [{"rule": "BOM_CYCLE", "object_id": "..."}],
  "trace_id": "..."
}
```

业务错误使用稳定错误码；不得向前端返回SQL、连接串、堆栈或U8密码。422用于请求结构错误，409用于状态/版本冲突，503用于依赖暂不可用。

## 7. 授权

除角色外还需数据范围授权：组织、仓库、物料分类和计划组。服务端强制过滤；不能依赖前端隐藏按钮。

