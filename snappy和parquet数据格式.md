### **Snappy 压缩文件、Parquet 文件及 `.snappy.parquet` 格式详解**

---

## **1. Snappy 压缩文件**
### **1.1 格式定义**
- **Snappy** 是 Google 开发的高性能压缩/解压缩库，**不追求最大压缩率**，而是以**极致速度**和**合理压缩率**为核心目标。
- **数据格式**：Snappy 压缩后的数据是二进制流，无自描述头信息（需依赖外部协议或封装格式，如 Parquet 的帧格式）。
- **压缩率**：相比 zlib 的最快模式，Snappy 速度快一个数量级，但压缩后的文件大小增加 20%-100%。

### **1.2 特点**
- **速度极快**：
  - **压缩速度**：250MB/s（64 位 Core i7 单核）。
  - **解压速度**：500MB/s（64 位 Core i7 单核）。
- **CPU 缓存友好**：算法设计考虑现代 CPU 缓存特性，指令级并行优化。
- **流式处理优化**：适合需要快速压缩/解压缩的流式数据处理场景（如 RPC 通信、日志处理）。
- **多语言支持**：提供 C++（核心）、Java（JNI）、Python、Go、Rust 等绑定。

### **1.3 数据存储与读取**
#### **存储数据为 Snappy 格式**
- **Python 示例**：
  ```python
  import snappy

  data = b"Hello, Snappy!" * 1000
  compressed_data = snappy.compress(data)  # 压缩
  with open("output.snappy", "wb") as f:
      f.write(compressed_data)
  ```

- **Java 示例**：
  ```java
  import org.xerial.snappy.Snappy;

  byte[] data = "Hello, Snappy!".getBytes();
  byte[] compressed = Snappy.compress(data);  // 压缩
  Files.write(Paths.get("output.snappy"), compressed);
  ```

#### **读取 Snappy 文件**
- **Python 示例**：
  ```python
  import snappy

  with open("output.snappy", "rb") as f:
      compressed_data = f.read()
  decompressed_data = snappy.uncompress(compressed_data)  # 解压
  print(decompressed_data.decode())
  ```

- **Java 示例**：
  ```java
  import org.xerial.snappy.Snappy;

  byte[] compressed = Files.readAllBytes(Paths.get("output.snappy"));
  byte[] decompressed = Snappy.uncompress(compressed);  // 解压
  System.out.println(new String(decompressed));
  ```

### **1.4 使用场景**
- **实时数据处理**：如 Kafka 消息压缩、Flink/Spark 流处理。
- **日志存储**：压缩日志文件以节省磁盘空间。
- **RPC 通信**：如 gRPC 默认使用 Snappy 压缩减少网络传输量。
- **大数据存储**：作为 Parquet、ORC 等列式存储格式的压缩后端。

---

## **2. Parquet 文件**
### **2.1 格式定义**
- **Parquet** 是一种开源的**列式存储文件格式**，由 Twitter 和 Cloudera 开发，2015 年成为 Apache 顶级项目。
- **文件结构**：
  - **Header**：4 字节魔术数字 `PAR1`，标识文件格式。
  - **Data Blocks（Row Groups）**：数据按行组（Row Group）划分，每个行组包含多个列块（Column Chunk）。
  - **Footer**：包含元数据（如 Schema、列统计信息、压缩算法等）。

### **2.2 特点**
- **列式存储**：
  - 数据按列存储，相同数据类型集中存储，显著提升分析查询效率（如仅读取部分列）。
  - 压缩率比行式格式（如 CSV）高 90% 以上。
- **高效压缩**：支持 Snappy、Gzip、Zstd、LZO 等算法。
- **自描述 Schema**：文件内嵌元数据（如数据类型、结构），确保跨系统一致性。
- **嵌套数据支持**：基于 Google Dremel 论文的算法处理复杂嵌套结构（如 JSON/Protocol Buffers）。

### **2.3 数据存储与读取**
#### **存储数据为 Parquet 格式**
- **Python（PyArrow）示例**：
  ```python
  import pyarrow as pa
  import pyarrow.parquet as pq
  import pandas as pd

  data = {"id": [1, 2, 3], "name": ["Alice", "Bob", "Charlie"], "age": [25, 30, 35]}
  df = pd.DataFrame(data)
  table = pa.Table.from_pandas(df)
  pq.write_table(table, "output.parquet", compression="SNAPPY")  # 使用 Snappy 压缩
  ```

- **PySpark 示例**：
  ```python
  from pyspark.sql import SparkSession

  spark = SparkSession.builder.appName("WriteParquet").getOrCreate()
  data = [("Alice", 25), ("Bob", 30), ("Charlie", 35)]
  df = spark.createDataFrame(data, ["name", "age"])
  df.write.parquet("output.parquet", compression="snappy")  # 使用 Snappy 压缩
  ```

#### **读取 Parquet 文件**
- **Python（PyArrow）示例**：
  ```python
  table = pq.read_table("output.parquet")
  df = table.to_pandas()
  print(df)
  ```

- **PySpark 示例**：
  ```python
  df = spark.read.parquet("output.parquet")
  df.show()
  ```

### **2.4 使用场景**
- **大数据分析**：如 Hive、Spark SQL、Presto 等查询引擎。
- **数据仓库**：如 Amazon Redshift、Snowflake 等支持 Parquet 格式。
- **机器学习**：作为特征存储格式（如 Feast、Hopsworks）。

---

## **3. `.snappy.parquet` 文件**
### **3.1 格式定义**
- **`.snappy.parquet`** 是 **Parquet 文件 + Snappy 压缩** 的组合格式，即：
  - **存储格式**：Parquet（列式存储）。
  - **压缩算法**：Snappy（快速压缩/解压缩）。
- **文件扩展名**：`.snappy.parquet` 或 `.parquet`（压缩算法通过元数据指定，扩展名可省略）。

### **3.2 特点**
- **结合 Parquet 和 Snappy 的优势**：
  - **高效查询**：列式存储 + 矢量化处理。
  - **快速压缩/解压缩**：Snappy 算法优化。
  - **合理存储空间**：压缩率比纯 Parquet（无压缩）高 30%-60%。
- **兼容性**：被 Hadoop、Spark、Hive、Presto 等大数据生态广泛支持。

### **3.3 数据存储与读取**
#### **存储数据为 `.snappy.parquet`**
- **Python（PyArrow）示例**：
  ```python
  pq.write_table(table, "output.snappy.parquet", compression="SNAPPY")
  ```

- **PySpark 示例**：
  ```python
  df.write.parquet("output.snappy.parquet", compression="snappy")
  ```

#### **读取 `.snappy.parquet` 文件**
- **Python（PyArrow）示例**：
  ```python
  table = pq.read_table("output.snappy.parquet")
  ```

- **PySpark 示例**：
  ```python
  df = spark.read.parquet("output.snappy.parquet")
  ```

### **3.4 使用场景**
- **实时分析**：如 ClickHouse、Doris 等 OLAP 引擎。
- **大数据管道**：如 Kafka + Flink + Parquet（Snappy 压缩）流批一体处理。
- **云存储**：如 S3、HDFS 存储压缩后的 Parquet 文件以降低成本。

---

## **4. 对比总结**
| **格式**         | **存储方式** | **压缩算法** | **速度** | **压缩率** | **适用场景**               |
|------------------|-------------|-------------|---------|-----------|---------------------------|
| **Snappy**       | 行式（二进制流） | Snappy      | 极快    | 中等      | 实时通信、日志压缩         |
| **Parquet**      | 列式        | 无/Gzip/Zstd | 中等    | 高        | 大数据分析、数据仓库       |
| **.snappy.parquet** | 列式        | Snappy      | 快      | 中高      | 实时分析、流批一体、云存储 |

---

## **5. 推荐实践**
1. **选择 `.snappy.parquet` 的场景**：
   - 需要**快速查询** + **合理存储空间** + **低延迟**（如实时数仓）。
   - 结合 **PyArrow/Spark** 写入，**Presto/Trino** 查询。
2. **避免 `.snappy.parquet` 的场景**：
   - 存储空间极度受限（优先选 Zstd/Gzip）。
   - 需要与已有压缩格式兼容（如 gzipped CSV）。

3. **性能优化**：
   - **行组大小**：调整 `parquet.block.size`（默认 128MB）以匹配集群内存。
   - **分区裁剪**：在 Hive/Spark 中按列过滤减少 I/O。
   - **矢量化读取**：启用 `parquet.vectorized.reader.enabled=true`（Spark 3.0+）。
  


--- 


### **`.snappy.parquet` 文件格式详解：存储与读取全指南**

`.snappy.parquet` 是 **Parquet 列式存储文件** + **Snappy 压缩算法** 的组合格式，广泛应用于大数据分析、实时数仓和云存储场景。本文将详细介绍如何通过 **Python（PyArrow/Pandas）、PySpark、Java（Hadoop API）、命令行工具** 等多种方式存储和读取 `.snappy.parquet` 文件，并对比不同方法的优缺点。

---

## **1. `.snappy.parquet` 文件的核心特性**
- **列式存储**：数据按列分块存储，支持高效查询（仅读取需要的列）。
- **Snappy 压缩**：
  - **压缩速度**：250MB/s（压缩） / 500MB/s（解压）。
  - **压缩率**：比无压缩 Parquet 小 30%-60%，但比 Gzip 慢且压缩率低。
- **自描述元数据**：文件内嵌 Schema、统计信息（如最小值、最大值），无需外部依赖。
- **跨语言支持**：通过 Arrow/Spark/Hive 等生态工具无缝读写。

---

## **2. 存储数据为 `.snappy.parquet` 文件**
### **方法 1：Python（PyArrow + Pandas）**
**适用场景**：本地开发、小规模数据处理、与 Pandas 生态集成。

#### **步骤 1：安装依赖**
```bash
pip install pyarrow pandas
```

#### **步骤 2：写入 `.snappy.parquet` 文件**
```python
import pyarrow as pa
import pyarrow.parquet as pq
import pandas as pd

# 创建示例数据
data = {
    "id": [1, 2, 3],
    "name": ["Alice", "Bob", "Charlie"],
    "age": [25, 30, 35],
    "score": [88.5, 92.0, 85.5]
}
df = pd.DataFrame(data)

# 转换为 PyArrow Table
table = pa.Table.from_pandas(df)

# 写入 Parquet 文件（使用 Snappy 压缩）
pq.write_table(
    table,
    "output.snappy.parquet",
    compression="SNAPPY",  # 关键参数：指定压缩算法
    version="2.6"          # 可选：指定 Parquet 版本（默认最新）
)
```

#### **关键参数说明**
- `compression`：可选 `"SNAPPY"`、`"GZIP"`、`"ZSTD"`、`"LZ4"`、`"UNCOMPRESSED"`。
- `row_group_size`：控制每个行组（Row Group）的行数（默认 128MB），影响查询性能。
- `version`：Parquet 格式版本（`"1.0"`、`"2.4"`、`"2.6"`），新版支持更多特性。

---

### **方法 2：PySpark**
**适用场景**：大规模数据处理、与 Spark 生态集成（如 HDFS、S3）。

#### **步骤 1：启动 Spark Session**
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("WriteSnappyParquet") \
    .getOrCreate()
```

#### **步骤 2：创建 DataFrame 并写入 `.snappy.parquet`**
```python
# 创建示例数据
data = [("Alice", 25), ("Bob", 30), ("Charlie", 35)]
df = spark.createDataFrame(data, ["name", "age"])

# 写入 Parquet 文件（使用 Snappy 压缩）
df.write \
    .mode("overwrite") \
    .parquet(
        "output.snappy.parquet",  # 输出路径（本地/HDFS/S3）
        compression="snappy"      # 关键参数：指定压缩算法
    )
```

#### **高级配置**
- **分区写入**（按列分区存储）：
  ```python
  df.write \
      .partitionBy("age") \  # 按 age 列分区
      .parquet("output_partitioned.snappy.parquet", compression="snappy")
  ```
- **控制行组大小**（通过 Spark 配置）：
  ```python
  spark.conf.set("spark.sql.parquet.rowGroupSize", "134217728")  # 128MB
  ```

---

### **方法 3：Java（Hadoop API）**
**适用场景**：集成到 Java/Scala 应用中，直接操作 HDFS 或本地文件系统。

#### **步骤 1：添加 Maven 依赖**
```xml
<dependency>
    <groupId>org.apache.parquet</groupId>
    <artifactId>parquet-hadoop</artifactId>
    <version>1.12.3</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>3.3.4</version>
</dependency>
```

#### **步骤 2：写入 `.snappy.parquet` 文件**
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.parquet.example.data.Group;
import org.apache.parquet.example.data.simple.SimpleGroupFactory;
import org.apache.parquet.hadoop.ParquetWriter;
import org.apache.parquet.hadoop.example.ExampleParquetWriter;
import org.apache.parquet.hadoop.metadata.CompressionCodecName;
import org.apache.parquet.schema.MessageType;
import org.apache.parquet.schema.Types;

import static org.apache.parquet.schema.PrimitiveType.PrimitiveTypeName.BINARY;
import static org.apache.parquet.schema.PrimitiveType.PrimitiveTypeName.INT32;

public class WriteSnappyParquet {
    public static void main(String[] args) throws Exception {
        // 定义 Schema
        MessageType schema = Types.buildMessage()
                .required(INT32).named("id")
                .required(BINARY).named("name")
                .required(INT32).named("age")
                .named("example_schema");

        // 配置写入路径和压缩算法
        Path path = new Path("output.snappy.parquet");
        Configuration conf = new Configuration();

        // 创建 ParquetWriter（使用 Snappy 压缩）
        try (ParquetWriter<Group> writer = ExampleParquetWriter.builder(path)
                .withType(schema)
                .withConf(conf)
                .withCompressionCodec(CompressionCodecName.SNAAPPY)  // 关键参数
                .build()) {

            // 写入数据
            SimpleGroupFactory groupFactory = new SimpleGroupFactory(schema);
            writer.write(groupFactory.newGroup()
                    .append("id", 1)
                    .append("name", "Alice")
                    .append("age", 25));
            writer.write(groupFactory.newGroup()
                    .append("id", 2)
                    .append("name", "Bob")
                    .append("age", 30));
        }
    }
}
```

---

### **方法 4：命令行工具（Parquet CLI）**
**适用场景**：快速测试或脚本化操作。

#### **步骤 1：安装 Parquet CLI**
```bash
# 通过 Homebrew（macOS）
brew install parquet-cli

# 或通过 Docker
docker run -it --rm parquet/parquet-cli
```

#### **步骤 2：从 CSV 转换并压缩为 `.snappy.parquet`**
```bash
# 假设 input.csv 内容：
# id,name,age
# 1,Alice,25
# 2,Bob,30

parquet-tools csv --compression SNAPPY input.csv output.snappy.parquet
```

---

## **3. 读取 `.snappy.parquet` 文件**
### **方法 1：Python（PyArrow + Pandas）**
#### **步骤 1：读取文件**
```python
table = pq.read_parquet("output.snappy.parquet")
df = table.to_pandas()
print(df)
```

#### **高级操作**
- **读取特定列**：
  ```python
  table = pq.read_table(
      "output.snappy.parquet",
      columns=["name", "age"]  # 仅读取 name 和 age 列
  )
  ```
- **读取元数据**：
  ```python
  metadata = pq.read_metadata("output.snappy.parquet")
  print(f"行数: {metadata.num_rows}")
  print(f"列数: {len(metadata.schema.names)}")
  ```

---

### **方法 2：PySpark**
#### **步骤 1：读取文件**
```python
df = spark.read.parquet("output.snappy.parquet")
df.show()
```

#### **高级操作**
- **读取分区数据**：
  ```python
  # 假设文件按 age 分区存储在 output_partitioned/ 目录下
  df = spark.read.parquet("output_partitioned/age=25")  # 仅读取 age=25 的分区
  ```
- **推断 Schema**：
  ```python
  schema = spark.read.parquet("output.snappy.parquet").schema
  print(schema)
  ```

---

### **方法 3：Java（Hadoop API）**
#### **步骤 1：读取文件**
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.parquet.hadoop.ParquetReader;
import org.apache.parquet.hadoop.example.ExampleParquetReader;
import org.apache.parquet.hadoop.metadata.ParquetMetadata;
import org.apache.parquet.schema.MessageType;

public class ReadSnappyParquet {
    public static void main(String[] args) throws Exception {
        Path path = new Path("output.snappy.parquet");
        Configuration conf = new Configuration();

        // 读取元数据
        ParquetMetadata footer = ParquetFileReader.readFooter(conf, path, ParquetMetadataConverter.NO_FILTER);
        MessageType schema = footer.getFileMetaData().getSchema();
        System.out.println("Schema: " + schema);

        // 创建 Reader 并读取数据
        try (ParquetReader<Group> reader = ExampleParquetReader.builder(path).withConf(conf).build()) {
            Group group;
            while ((group = reader.read()) != null) {
                System.out.println("ID: " + group.getInteger("id", 0));
                System.out.println("Name: " + group.getString("name", 0));
            }
        }
    }
}
```

---

### **方法 4：命令行工具（Parquet CLI）**
#### **步骤 1：查看文件内容**
```bash
parquet-tools head output.snappy.parquet  # 查看前几行
parquet-tools schema output.snappy.parquet  # 查看 Schema
```

---

## **4. 性能优化建议**
1. **行组大小（Row Group Size）**：
   - 默认 128MB，调整至与集群内存匹配（如 256MB 或 512MB）。
   - 在 PySpark 中通过 `spark.sql.parquet.rowGroupSize` 配置。

2. **分区裁剪（Partition Pruning）**：
   - 在 Spark/Hive 中按分区列过滤，避免全表扫描。

3. **谓词下推（Predicate Pushdown）**：
   - Parquet 支持在读取时过滤数据，减少 I/O：
     ```python
     # PyArrow 示例：过滤 age > 30 的行
     table = pq.read_table(
         "output.snappy.parquet",
         filters=[("age", ">", 30)]
     )
     ```

4. **矢量化读取（Vectorized Reading）**：
   - 在 Spark 3.0+ 中启用：
     ```python
     spark.conf.set("spark.sql.parquet.enableVectorizedReader", "true")
     ```

---

## **5. 常见问题解决**
### **Q1：读取 `.snappy.parquet` 文件时报错 `Not a Parquet file`**
- **原因**：文件扩展名可能被篡改或文件损坏。
- **解决**：
  - 使用 `file` 命令检查文件类型：
    ```bash
    file output.snappy.parquet
    ```
  - 验证文件头是否为 `PAR1`：
    ```bash
    xxd output.snappy.parquet | head -n 1
    ```

### **Q2：如何选择压缩算法？**
| **算法**   | **速度** | **压缩率** | **适用场景**               |
|------------|---------|-----------|---------------------------|
| Snappy     | 极快    | 中等      | 实时查询、低延迟           |
| Gzip       | 慢      | 高        | 长期存储、归档             |
| Zstd       | 快      | 高        | 平衡速度与压缩率           |

---

## **6. 总结**
| **方法**       | **工具链**       | **适用场景**               | **关键代码/参数**                     |
|----------------|------------------|---------------------------|---------------------------------------|
| **Python**     | PyArrow + Pandas | 本地开发、小规模数据        | `pq.write_table(..., compression="SNAPPY")` |
| **PySpark**    | Spark SQL         | 大规模数据处理              | `df.write.parquet(..., compression="snappy")` |
| **Java**       | Hadoop API       | 集成到 Java 应用            | `CompressionCodecName.SNAAPPY`        |
| **命令行**     | Parquet CLI       | 快速测试、脚本化操作        | `parquet-tools csv --compression SNAPPY` |

通过本文的详细指南，您可以根据实际需求选择最适合的方法存储和读取 `.snappy.parquet` 文件，并优化性能以满足业务场景。
