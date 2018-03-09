1 **Source code directory**

| Folders | Description |
|-------------|-------|
Assembly | building a single Jar package 
mvn clean -DskipTests package -Pdist 
or 
mvn clean -DskipTests -Pinclude-all package -Pdist |
Bin | provides two Linux shell scripts, carbon-spark-shell and carbon-spark-sql, which can be quickly and easily experienced in local mode |
Common | Common module, currently only contains logs |
| conf | Configuration Files carbon.properties.template, dataload.properties.template |
Core | core module, contains the query module code and some basic model types and tools 1.core: logical structure model such as table, dictionary, index, and dictionary cache, MDK generation, data decompression, file read and write, etc. .Scan: Implements query functions, including query data scanning, expression calculation and data filtering, detailed single query line result collection, query execution tools, etc.
| dev | Developer Tools, Java/scala Code style, findbugs configuration files |
| docs | Document Maintenance |
|Examples|  examples of runnable functions, including: flink, spark, spark2 |
Format | CarbonData file format definition, using Apache Thrift definition |
Hadoop | Hadoop interface implementation, such as: CarbonInputFormat, CarbonRecordReader, etc.
Integration | Integration module 1.Spark-common: spark and spark2 can reuse code module 2.Spark: CarbonContext, sqlparser, optimizer, sparkplan3.Spark2: CarbonSession, CarbonEnv, CarbonScan, CarbonSource4.Spark-common-test: spark and spark2 Reusable test cases |
Processing | Data Loading Module InputProcessorStep: Data Input Received DataConverteProcessorStep: Data Transformation (Type Conversion, Dictionary Encoding) SortProcessorStep: Sorting Data within the Node DataWriterProcessorStep: Generated Carbondata, Carbonindex File |
