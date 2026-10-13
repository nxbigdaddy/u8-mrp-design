# 04　MRP 引擎设计

## 1. 输入与输出

输入：物料计划参数、有效 BOM、独立需求、有效库存、采购/生产/委外/调拨供应、工作日历、冻结计划和方案参数。输出：采购建议、生产建议、相关需求、供需 Pegging、缺料和例外消息。

引擎接口必须与 U8 解耦：

```python
class MrpEngine(Protocol):
    def calculate(
        self,
        snapshot: SnapshotRef,
        scenario: Scenario,
        scope: PlanScope,
        checkpoint: CheckpointWriter,
    ) -> RunSummary: ...
```

## 2. 总体算法

```text
校验输入
→ 选择有效 BOM 版本
→ 检测环路并计算低层码
→ 建立独立需求池和有效供应池
→ level=0..max_level 逐层处理
   → 按 item_dim_key 分组
   → 按日期和优先级排序需求
   → 消耗期初可用量和确认供应
   → 对缺口应用批量和提前期规则
   → 生成计划订单和 Pegging
   → 按 BOM 展开计划订单，形成下一层相关需求
→ 生成例外消息与汇总指标
→ 原子完成 Run
```

层级之间存在依赖屏障；同一低层码内的独立物料可分片并行。每完成一层写检查点。

## 3. BOM 处理

1. 以计划日期选择生效且已审核的版本。
2. 使用深度优先搜索检测环路，返回完整环路路径。
3. 计算低层码：物料在所有 BOM 路径中出现的最大层级。
4. 子件需求：`父项计划数量 × 基本用量 × (1 + 损耗率) + 固定用量`。
5. 虚拟件不生成生产建议，直接继续展开。
6. 替代料第一阶段只按确定优先级处理；优化分配作为后续能力。
7. BOM版本、用量和单位换算全部进入快照，运行期间不可变化。

## 4. 时段净需求

```python
def net_item(demands, supplies, opening, policy, calendar):
    available = opening
    supply_queue = sorted(supplies, key=lambda x: x.available_date)
    for demand in sorted(demands, key=demand_sort_key):
        available += release_supply_until(supply_queue, demand.need_date)
        used = min(available, demand.open_qty)
        peg_existing_supply(demand, used)
        available -= used
        shortage = demand.open_qty - used
        if shortage <= 0:
            continue
        plan_qty = lot_size(shortage, policy)
        due_date = demand.need_date
        release_date = calendar.backward(due_date, policy.lead_time)
        plan = create_plan(plan_qty, release_date, due_date)
        peg_plan(demand, plan, shortage)
        available += plan_qty - shortage
```

实际实现应逐笔保存供应来源，不能只维护一个总 `available`。建议使用按到货日期排序的 deque，并记录每个 supply lot 的剩余量。

## 5. 排序与优先级

默认需求排序键：冻结需求优先 → 业务优先级 → 需求日期 → 客户等级 → 单据日期 → 单据号。排序规则参数化并保存版本；相同输入必须产生确定性结果。

## 6. 批量与日历

支持按需、固定批量、最小/最大批量、批量倍数、周期合并。所有舍入采用 `Decimal` 和明确的舍入模式。采购、生产、检验提前期分别计算；非工作日按配置向前或向后移动。

## 7. 冻结区和例外

冻结期内不自动调整已确认供应，只产生：提前、推迟、取消、加急、数量增加/减少、缺料和参数缺失。建议状态为 `PROPOSED → ADJUSTED → FROZEN → APPROVED → PUBLISHED`。

## 8. 性能策略

- 一次批量读取某层所需数据，避免循环查询数据库。
- 物料、BOM、日历使用紧凑内存结构和字典索引。
- 结果按分片使用 `executemany`/临时表批量写入。
- 先基准测试单进程，再按低层码内物料分片启用多进程。
- 不把完整 MRP 写成单条超大 SQL，也不依赖 Pandas DataFrame 逐行处理。

## 9. 确定性与重放

同一 `snapshot_id + scenario_version + engine_version` 必须产生一致结果。禁止在计算过程中读取当前时间或生产库；所有“今天”和截止时间由 Run 参数提供。

