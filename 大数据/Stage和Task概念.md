你好！我是大数据编程专家。很高兴为你梳理这份关于 Spark 核心调度机制的教材。

这份教材将以 **“从原理到实战，从内核到调优”** 为逻辑主线，不仅涵盖你提供的核心概念，还会补充 Shuffle 细节、资源调度、数据倾斜处理等进阶知识，并配合完整的 PySpark 代码示例。

---

# 《Spark 核心调度机制深度解析：Stage 与 Task》

## 前言：为什么要学习 Stage 和 Task？
如果把 Spark 作业比作**“一家工厂的生产流水线”**：
*   **Job（作业）**：是一份完整的订单。
*   **Stage（阶段）**：是流水线上的一个个**车间**（如：切割车间、组装车间）。车间之间需要传送带（Shuffle）来传递半成品。
*   **Task（任务）**：是车间里的**工人**，每个工人只负责处理一件产品（一个数据分区）。
*   **Executor（执行器）**：是工人的**工作台**（包含 CPU 和内存）。

理解 Stage 和 Task，是从“写代码”进阶到“写高性能代码”的关键，也是面试和实际生产中排查性能瓶颈的基石。

---

## 第一章：核心概念与 DAG 划分

### 1.1 RDD 的依赖关系（Dependency）
Spark 的核心是 RDD（弹性分布式数据集）。RDD 之间的关系决定了 Stage 的划分。

#### 1.1.1 窄依赖 (Narrow Dependency)
*   **定义**：父 RDD 的**每一个分区**最多被子 RDD 的**一个分区**使用。
*   **形象理解**：一对一，或者“父分区”的数据不需要跨分区就能计算出“子分区”。
*   **常见算子**：`map()`, `filter()`, `flatMap()`, `union()`, `mapPartitions()`。
*   **特性**：**不需要 Shuffle**，数据可以在同一个 Stage 内像管道一样流式处理。

#### 1.1.2 宽依赖 (Wide Dependency / Shuffle Dependency)
*   **定义**：父 RDD 的**一个分区**的数据可能被发送到子 RDD 的**多个分区**。
*   **形象理解**：多对多，或者“父分区”的数据需要根据 Key 重新分发到不同的“子分区”。
*   **常见算子**：`reduceByKey()`, `groupByKey()`, `join()`, `sortByKey()`, `repartition()`。
*   **特性**：**必须触发 Shuffle**，这是 Stage 的边界。

### 1.2 DAG (有向无环图) 的构建
Spark Driver 会将用户的代码转换为 DAG。
*   **顶点 (Vertex)**：RDD。
*   **边 (Edge)**：依赖关系。
*   **划分依据**：从后往前（从 Action 算子往前回溯），遇到**宽依赖**就切断，划分出一个新的 Stage。

> **专家提示**：DAG 的划分是**从后往前**的。因为 Action（如 save/collect）是最终目标，为了达到这个目标，我们需要哪些前置步骤？遇到 Shuffle 就意味着必须等上一个阶段全部完成才能开始下一个阶段。

---

## 第二章：Stage 详解 —— 作业的逻辑单元

### 2.1 Stage 的两种类型
在 DAG 中，Stage 主要分为两类：

1.  **ShuffleMap Stage**：
    *   **位置**：位于 DAG 的中间。
    *   **产出**：输出结果**不是**给用户的，而是作为下一个 Stage 的输入（需要经过 Shuffle）。
    *   **例子**：`reduceByKey` 的前半部分（Map 端）。
2.  **Result Stage**：
    *   **位置**：位于 DAG 的末端（最后一个 Stage）。
    *   **产出**：输出结果**直接返回给 Driver** 或写入外部存储（如 HDFS、MySQL）。
    *   **例子**：`collect()`, `count()`, `saveAsTextFile()` 之前的那个 Stage。

### 2.2 代码示例：观察 Stage 划分
我们通过代码来直观感受 Stage 的生成。

```python
from pyspark import SparkConf, SparkContext

conf = SparkConf().setAppName("StageDemo").setMaster("local[4]")
sc = SparkContext(conf=conf)

# 1. 创建数据，4个分区
rdd = sc.parallelize([("a", 1), ("b", 1), ("a", 2), ("c", 1), ("b", 2)], 4)

# 2. map 是窄依赖，不会划分 Stage
rdd_mapped = rdd.map(lambda x: (x[0], x[1] + 1))

# 3. reduceByKey 是宽依赖，会触发新的 Stage
# 注意：这里会产生 Shuffle
rdd_reduced = rdd_mapped.reduceByKey(lambda x, y: x + y)

# 4. collect 是 Action，触发 Job 执行
result = rdd_reduced.collect()

print(result)
sc.stop()
```

**运行后的 Spark UI (http://localhost:4040) 观察：**
你会看到一个 Job，里面包含 **2 个 Stage**：
*   **Stage 0** (ShuffleMap Stage): 包含 `parallelize` 和 `map` 算子。有 4 个 Task（对应 4 个分区）。
*   **Stage 1** (Result Stage): 包含 `reduceByKey` 和 `collect` 算子。假设 reduceByKey 后默认分区数是 2（或配置的数量），则有 2 个 Task。
*   **Stage 1 必须等待 Stage 0 全部完成后才能开始**。

---

## 第三章：Task 详解 —— 最小执行单元

### 3.1 Task 的生命周期
Task 是分配给 Executor 上的具体工作单元。

1.  **创建 (Creation)**：Driver 将 Stage 切分为 TaskSet（一组 Task），Task 的数量 = 该 Stage 输入 RDD 的分区数。
2.  **调度 (Scheduling)**：Driver 通过 TaskScheduler 将 Task 发送给集群管理器（YARN/K8s/Standalone），再由集群管理器分配给 Executor 的 CPU 核心。
3.  **执行 (Execution)**：
    *   Executor 启动线程执行 Task。
    *   读取分区数据。
    *   执行算子逻辑（map/reduce 等）。
    *   **如果是 ShuffleMap Task**：将计算结果写入本地磁盘文件（Shuffle File），并向 Driver 注册位置信息。
    *   **如果是 Result Task**：将结果直接发送给 Driver 或写入外部存储。
4.  **完成/失败 (Completion/Failure)**：
    *   成功：返回状态给 Driver。
    *   失败：Driver 会重新调度该 Task（默认重试 4 次），如果重试失败则整个 Job 失败。

### 3.2 并行度 (Parallelism) 的控制
**核心公式：Task 数量 = 分区数量**。

如何控制 Task 数量（即并行度）？

```python
# 1. 创建 RDD 时指定分区数
rdd1 = sc.parallelize(range(1000), numSlices=10)  # 强制 10 个分区 -> 10 个 Task

# 2. 在 Shuffle 算子中指定分区数 (最常用)
# reduceByKey 的第二个参数 numPartitions 决定了下一个 Stage 的 Task 数量
rdd2 = rdd1.map(lambda x: (x % 10, x)) \
           .reduceByKey(lambda a, b: a + b, numPartitions=5) # 下一 Stage 有 5 个 Task

# 3. 使用 repartition 或 coalesce
rdd3 = rdd1.repartition(20)   # 全量 Shuffle，增加或减少分区
rdd4 = rdd1.coalesce(5)       # 仅减少分区，避免全量 Shuffle（在某些情况下）

# 4. SQL 配置 (非常重要)
spark.conf.set("spark.sql.shuffle.partitions", "200")  # Spark SQL 默认 200 个分区
```

---

## 第四章：进阶 —— Shuffle 与数据倾斜

### 4.1 Shuffle 的物理过程
Shuffle 是 Spark 中最昂贵的操作，涉及磁盘 I/O、网络 I/O 和序列化。
*   **Map 端**：每个 Task 计算后，按 Key 的哈希值将数据写入不同的磁盘文件（Buffer -> Spill -> Merge）。
*   **Reduce 端**：每个 Task 去所有 Map 端拉取属于自己的数据（Fetch），然后进行聚合。

### 4.2 数据倾斜 (Data Skew) —— Task 的噩梦
**现象**：
*   某几个 Task 处理的数据量极大（例如几 GB），而其他 Task 只有几 KB。
*   Spark UI 中看到大部分 Task 很快完成，只有 1-2 个 Task 运行极慢。
*   报错：OOM (Out Of Memory) 或 Shuffle Fetch Failed。

**原因**：
Key 的分布不均匀。例如：大部分数据 Key 是 "unknown" 或空值，或者某些热点 Key（如大V的ID）。

**解决方案（加盐 Salting）**：
将倾斜的 Key 打散，变成多个 Key，二次聚合。

```python
# 假设 key "A" 数据量巨大
rdd = sc.parallelize([("A", 1), ("A", 2), ("B", 1), ("A", 3)])

# 1. 普通聚合（会倾斜）
# rdd.reduceByKey(lambda x, y: x + y).collect()

# 2. 加盐聚合（两阶段聚合）
import random

# 第一阶段：给 Key 加上随机前缀，打散数据
rdd_salted = rdd.map(lambda x: (str(random.randint(0, 9)) + "_" + x[0], x[1])) \
                .reduceByKey(lambda x, y: x + y)

# 第二阶段：去掉前缀，进行最终聚合
rdd_final = rdd_salted.map(lambda x: (x[0].split("_")[1], x[1])) \
                      .reduceByKey(lambda x, y: x + y)

print(rdd_final.collect())
```

---

## 第五章：实战调优指南

### 5.1 如何查看 Stage/Task 详情？
1.  **Spark UI (端口 4040)**：
    *   **Jobs Tab**：看作业的依赖关系。
    *   **Stages Tab**：看每个 Stage 的 Task 数量、输入输出数据量、Shuffle 读写大小、耗时分布（最重要！看是否有慢 Task）。
    *   **Storage Tab**：看缓存的 RDD 分区情况。
    *   **Executors Tab**：看内存使用情况，是否有 OOM。

2.  **日志分析**：
    *   搜索 `TaskSetManager`：查看 Task 的调度日志（如 "Lost task 1.0 in stage 0.0"）。
    *   搜索 `Executor`：查看 OOM 错误栈。

### 5.2 常见调优策略

| 问题场景 | 现象 | 解决方案 |
| :--- | :--- | :--- |
| **Task 数量过少** | CPU 利用率低，大文件处理慢 | 增加分区数：`repartition()` 或调整 `spark.default.parallelism` |
| **Task 数量过多** | 调度开销大，小文件过多 | 减少分区数：`coalesce()` 或减少输入文件块大小 |
| **Shuffle 过多** | 作业慢，磁盘 I/O 高 | 1. 使用 `Broadcast Join` 替代 `Shuffle Join` (小表广播)<br>2. 使用 `mapPartitions` 预聚合<br>3. 开启 Kryo 序列化 |
| **数据倾斜** | 少数 Task 极慢，OOM | 1. Key 加盐 (Salting)<br>2. 过滤掉无效 Key (如 null)<br>3. 单独处理倾斜 Key |
| **内存不足** | Executor Lost, OOM | 1. 增加 `spark.executor.memory`<br>2. 调整 `spark.memory.fraction` (存储/执行内存比例)<br>3. 使用持久化策略 `MEMORY_AND_DISK` |

---

## 第六章：综合实战案例 —— 词频统计优化

我们来写一个完整的、包含调优思想的词频统计程序。

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("WordCountOptimized") \
      .config("spark.sql.shuffle.partitions", "10") \
      .getOrCreate()

sc = spark.sparkContext

# 1. 读取数据
lines = sc.textFile("hdfs:///data/large_text_file.txt", minPartitions=20)

# 2. 预处理 + Map 阶段 (Stage 0)
# 扁平化映射
words = lines.flatMap(lambda line: line.split(" "))
# 过滤空字符串
filtered_words = words.filter(lambda w: len(w) > 0)
# 初步映射为 (word, 1)
pairs = filtered_words.map(lambda w: (w, 1))

# 3. Reduce 阶段 (Stage 1) - 关键点
# 假设我们知道 "the", "a" 这种停用词会导致数据倾斜，先过滤
# 或者如果不确定，直接进行聚合。这里演示调整分区数
# 假设集群有 10 个核心，我们设置 10 个分区以最大化并行
word_counts = pairs.reduceByKey(lambda a, b: a + b, numPartitions=10)

# 4. 输出 (Stage 2 - Result Stage)
# 排序并取 Top 10 (这会触发新的 Stage)
sorted_counts = word_counts.map(lambda x: (x[1], x[0])) \
                           .sortByKey(ascending=False) \
                           .map(lambda x: (x[1], x[0]))

top_10 = sorted_counts.take(10)

for word, count in top_10:
    print(f"{word}: {count}")

spark.stop()
```

### 代码解析：
1.  `minPartitions=20`：在读取时就指定初始并行度，避免分区太少。
2.  `spark.sql.shuffle.partitions`：显式设置 Shuffle 后的分区数为 10，匹配集群资源（假设 10 个核心），避免默认 200 导致的小任务过多。
3.  `filter` 提前：在 Shuffle 前过滤掉无效数据，减少网络传输量。
4.  `sortByKey`：这是一个宽依赖，会触发额外的 Stage。如果只是为了 Top N，可以考虑用 `top()` 或 `takeOrdered()` 在 Driver 端排序，避免全量排序的 Shuffle。

---

## 课后习题与思考

1.  **思考题**：如果一个 Job 有 3 个 Stage，第 1 个 Stage 有 100 个 Task，第 2 个 Stage 有 50 个 Task，第 3 个 Stage 有 10 个 Task。请问总共需要多少个 CPU 核心同时运行才能达到理论上的最大并行度？（假设无资源限制）
2.  **代码题**：编写一个 Spark 程序，处理包含用户 ID 和消费金额的日志，计算每个用户的总消费。并模拟一个“数据倾斜”场景（例如人为制造 100 万条 user_id=0 的数据），观察 Spark UI 的表现，并尝试用加盐法修复。
3.  **面试题**：请解释 Spark 中窄依赖和宽依赖的区别，以及它们对 Stage 划分的影响。什么是 Shuffle？为什么要尽量避免 Shuffle？

---

## 总结

*   **Stage** 是逻辑边界，由**宽依赖**（Shuffle）划分。
*   **Task** 是物理执行单元，数量等于**分区数**。
*   **性能瓶颈**通常出现在 Shuffle 和数据倾斜上。
*   **调优核心**：控制分区数（并行度）、减少 Shuffle、均衡数据分布。

希望这份教材能帮助你和学生们彻底掌握 Spark 的调度内核！如果有具体的报错或场景需要分析，欢迎继续提问。
