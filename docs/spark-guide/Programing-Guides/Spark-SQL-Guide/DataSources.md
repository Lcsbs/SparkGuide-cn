# 数据源

Spark SQL支持通过DataFrame接口对各种数据源进行操作。DataFrame可以使用关系转换进行操作，也可以用于创建临时视图。将DataFrame注册为临时视图使您可以对其数据运行SQL查询。本节介绍了使用Spark数据源加载和保存数据的一般方法，然后介绍了可用于内置数据源的特定选项。

- 通用加载/保存功能
  - [手动指定选项](http://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#manually-specifying-options)
  - [直接在文件上运行SQL](http://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#run-sql-on-files-directly)
  - [保存模式](http://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#save-modes)
  - [保存到永久表](http://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#saving-to-persistent-tables)
  - [分组，分类和分区](http://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html#bucketing-sorting-and-partitioning)
- 通用文件源选项
  - [忽略损坏的文件](http://spark.apache.org/docs/latest/sql-data-sources-generic-options.html#ignore-corrupt-iles)
  - [忽略丢失的文件](http://spark.apache.org/docs/latest/sql-data-sources-generic-options.html#ignore-missing-iles)
  - [路径全局过滤器](http://spark.apache.org/docs/latest/sql-data-sources-generic-options.html#path-global-filter)
  - [递归文件查找](http://spark.apache.org/docs/latest/sql-data-sources-generic-options.html#recursive-file-lookup)
- 实木复合地板文件
  - [以编程方式加载数据](http://spark.apache.org/docs/latest/sql-data-sources-parquet.html#loading-data-programmatically)
  - [分区发现](http://spark.apache.org/docs/latest/sql-data-sources-parquet.html#partition-discovery)
  - [模式合并](http://spark.apache.org/docs/latest/sql-data-sources-parquet.html#schema-merging)
  - [Hive Metastore Parquet表转换](http://spark.apache.org/docs/latest/sql-data-sources-parquet.html#hive-metastore-parquet-table-conversion)
  - [组态](http://spark.apache.org/docs/latest/sql-data-sources-parquet.html#configuration)
- [ORC文件](http://spark.apache.org/docs/latest/sql-data-sources-orc.html)
- [JSON文件](http://spark.apache.org/docs/latest/sql-data-sources-json.html)
- 蜂巢表
  - [指定Hive表的存储格式](http://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html#specifying-storage-format-for-hive-tables)
  - [与Hive Metastore的不同版本进行交互](http://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html#interacting-with-different-versions-of-hive-metastore)
- [JDBC到其他数据库](http://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
- Avro文件
  - [部署中](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#deploying)
  - [加载和保存功能](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#load-and-save-functions)
  - [to_avro（）和from_avro（）](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#to_avro-and-from_avro)
  - [数据源选项](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#data-source-option)
  - [组态](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#configuration)
  - [与Databricks spark-avro的兼容性](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#compatibility-with-databricks-spark-avro)
  - [Avro-> Spark SQL转换支持的类型](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#supported-types-for-avro---spark-sql-conversion)
  - [Spark SQL支持的类型-> Avro转换](http://spark.apache.org/docs/latest/sql-data-sources-avro.html#supported-types-for-spark-sql---avro-conversion)
- [整个二进制文件](http://spark.apache.org/docs/latest/sql-data-sources-binaryFile.html)
- [故障排除](http://spark.apache.org/docs/latest/sql-data-sources-troubleshooting.html)