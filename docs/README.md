# U8 外挂 MRP 系统设计文档

状态：方案设计稿　　技术栈：Python　　更新时间：2026-07-30

## 1. 文档目标

本目录描述一个独立于 U8 运算、以 U8 数据为主要输入的 MRP 计划平台。系统负责数据快照、质量校验、BOM 展开、时段净需求、供需挂接、计划协同和受控写回；U8 继续作为基础档案、正式单据、库存和财务核算系统。

## 2. 设计原则

1. 不在 `UFDATA_001_2020` 中建立外挂业务表，不直接写 U8 业务表。
2. 每次运算绑定不可变快照、参数版本、代码版本和 Run ID，结果可重放。
3. MRP 引擎只依赖标准领域模型，不依赖 U8 字段名。
4. 批量计算与 Web API 进程分离，长任务支持取消、检查点和安全重跑。
5. 第一阶段只读计算；经过并行验证后才启用人工审核与受控写回。
6. 第一版采用模块化单体，不为“微服务”而拆分。

## 3. 文档目录

| 文档 | 内容 |
|---|---|
| [01-业务需求与范围](01-business-requirements.md) | 角色、范围、业务流程和边界 |
| [02-系统架构](02-system-architecture.md) | 技术栈、模块、部署和代码结构 |
| [03-数据架构](03-data-architecture.md) | 数据分层、核心模型、维度键和保留策略 |
| [04-MRP 引擎设计](04-mrp-engine.md) | BOM、净需求、批量、日历、Pegging 算法 |
| [05-U8 集成设计](05-u8-integration.md) | 同步、水位、数据映射和受控写回 |
| [06-API 设计](06-api-design.md) | HTTP API、错误模型、分页和并发控制 |
| [07-任务与状态机](07-jobs-state-machine.md) | Celery任务、状态转换、检查点与重试 |
| [08-安全与审计](08-security-audit.md) | 权限、密钥、审计、数据安全 |
| [09-部署与运维](09-deployment-operations.md) | 环境、拓扑、监控、备份、容量 |
| [10-测试与验收](10-testing-acceptance.md) | 自动化测试、黄金数据集和验收指标 |
| [11-实施计划](11-implementation-plan.md) | 阶段、交付物、团队和上线门槛 |
| [12-架构决策记录](12-architecture-decisions.md) | 关键技术决策及其理由 |

## 4. 当前已知环境

- U8 数据库：`UFDATA_001_2020`
- SQL Server：2016 Enterprise SP1，约 154 GB
- 数据库仍承载 2026 年实时业务，并非单纯 2020 历史库
- 约 64,948 个存货、22,046 条 BOM 主记录
- 采购、销售、生产、BOM、库存及明显二开对象并存
- 正式开发前必须确认 U8 版本、标准接口许可和全部二开逻辑

## 5. 技术基线

- Python 3.13.x（锁定补丁版本）
- FastAPI、Pydantic 2、SQLAlchemy 2、Alembic
- SQL Server + Microsoft ODBC Driver 18 + pyodbc
- Celery + Redis（任务队列和短期缓存）
- Vue 3 + TypeScript（前端不在本目录展开代码设计）
- pytest、Hypothesis、Playwright、Ruff、mypy
- OpenTelemetry、structlog、Prometheus/Grafana 或企业现有监控

依赖必须通过锁文件固定，生产升级采用可回滚的变更流程。

