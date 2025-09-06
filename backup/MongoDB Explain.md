MongoDB 的 `explain` 命令用于分析查询的执行过程和优化情况，帮助开发者理解查询是否命中索引、执行效率如何，以及优化空间。`explain` 报告会详细展示查询计划、索引使用、扫描量、返回文档数等信息。

---

## 一、基本用法

```js
db.collection.find({条件}).explain()
```
或
```js
db.collection.find({条件}).explain("executionStats")
```

---

## 二、常见 explain 级别

- `"queryPlanner"`：只显示查询计划（不实际执行查询）。
- `"executionStats"`：显示查询计划和实际执行统计（推荐）。
- `"allPlansExecution"`：显示所有尝试过的查询计划及统计。

---

## 三、主要字段介绍

### 1. `queryPlanner`
- **`winningPlan`**：实际采用的查询计划（是否用索引、索引类型、扫描方式等）。
- **`parsedQuery`**：解析后的查询条件。
- **`indexFilterSet`**：是否使用了索引过滤。
- **`inputStage`**：具体执行阶段（如 COLLSCAN 全集合扫描，IXSCAN 索引扫描）。

### 2. `executionStats`
- **`nReturned`**：返回文档数量。
- **`executionTimeMillis`**：查询实际耗时（毫秒）。
- **`totalKeysExamined`**：索引项扫描数量。
- **`totalDocsExamined`**：文档扫描数量。
- **`executionStages`**：执行各阶段详细统计。

### 3. 关键阶段类型
- **COLLSCAN**：全表扫描（无索引命中，性能较低）。
- **IXSCAN**：索引扫描（命中索引，性能较高）。
- **FETCH**：根据索引获取文档。
- **SORT**：需要额外排序操作。
- **LIMIT / SKIP**：分页相关操作。

---

## 四、典型 explain 结果示例

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "IXSCAN",               // 用了索引
      "indexName": "age_1",            // 索引名字
      ...
    }
  },
  "executionStats": {
    "nReturned": 10,                   // 返回10条文档
    "executionTimeMillis": 2,          // 用时2ms
    "totalKeysExamined": 10,           // 扫描了10个索引项
    "totalDocsExamined": 10,           // 扫描了10个文档
    ...
  }
}
```

---

## 五、如何解读结果


一个典型的执行计划文档可能包含以下关键字段：

* **`queryPlanner`：** 包含关于查询优化器选择的信息。
  * `winningPlan`：优化器最终选择并执行的计划。这是最重要的部分。
  * `rejectedPlans`：优化器考虑但最终放弃的备选计划。
* **`winningPlan`** (获胜计划)下的关键信息：
  * **`stage`：** 代表查询执行过程中的一个阶段。常见的阶段包括：
    * `IXSCAN`：表示使用了​**索引扫描**​。这是最理想的情况，意味着查询高效地利用了索引。
    * `COLLSCAN`：表示进行了​**全集合扫描**​（Collection Scan）。这意味着查询没有找到合适的索引，不得不遍历整个集合的所有文档，性能通常较差。
    * `SORT`：表示对结果进行了排序操作。如果这个操作没有在索引中完成，可能会消耗大量内存和CPU。理想情况是看到 `IXSCAN` 后跟着一个 `SORT`，这表明排序是通过索引完成的。
    * `FETCH`：表示从磁盘中获取文档。
  * **`inputStage`：** 如果一个阶段是另一个阶段的输入，这个字段会描述输入阶段。一个复杂的查询计划通常由多个嵌套的阶段组成。
  * `indexName`：如果使用了索引，这里会显示索引的名称。
* **`executionStats`** (执行统计)下的关键信息：
  * `nReturned`：查询返回的文档数。
  * `totalKeysExamined`：在索引中扫描的键数。
  * `totalDocsExamined`：在集合中扫描的文档数。
  * `executionTimeMillis`：查询实际执行花费的时间（毫秒）。


---

## 六、常见优化建议

- 查询出现 `COLLSCAN` 时，应考虑加索引。
- `totalDocsExamined` 远大于 `nReturned` 时，说明查询效率低。
- 如果有 `SORT`，可尝试增加排序字段索引。

---
