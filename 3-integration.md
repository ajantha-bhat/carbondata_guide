3 **integration module**

3.1 **spark integration**

In Spark 1.x, use CarbonContext as the entrance to the spark sql interface. CarbonContext extends HiveContext and customizes CarbonSqlParser, some LogicalPlan, CarbonOptimizer, and CarbonStrategy.

1. CarbonSqlParser used to parse CarbonData create table, load data and other sql statement, the specific call flow as shown in the figure, CarbonSQLDialect preferential use of CarbonSqlParser to parse sql (mainly include create table, load table and other sql statement), if it can not be resolved, continue Use HiveQL to parse sql (mainly select queries) and generate Carbon LogicalPlan.

<img src="media/3-1_1.png" width = "35%" alt="3-1_1" />

2. CarbonData LogicalPlan mainly has CreateTable (create table) and LoadTable (load data).

    CreateTable: TableNewProcessor is used to generate the carbondata table schema file (thrift format); CarbonMetastore.createTableFromThrift loads the schema file into the metadata cache; the last is to create the Spark table.
    
    <img src="media/3-1_2.png" width = "50%" alt="3-1_2" />
The
    LoadTable: Mainly divided into generateGlobalDictionary and loadCarbonData:
    
    <img src="media/3-1_3.png" width = "50%" alt="3-1_3" />
The
    generateGlobalDictionary is used to generate table-level dictionary encodings.
    loadCarbonData is used to generate carbondata and carbonindex files. First, it will be based on load,
The insert and update operations enter the loadDataFile, loadDataFrame, and loadDataFrameForUpdate load processes to generate the carbondata and carbonindex files. Second, CarbonUpdateUtil.updateTableMetadataStatus records the tablestatus information loaded by the data. If carbon.enable.auto.load.merge=true is configured, after loading multiple times, the handleSegmentMerging will be triggered before loading to circularly merge multiple smaller segments into one larger segment. The merge can effectively prevent Small file problem (alternatively, you can use the alter table compact command to trigger the merge).

3. CarbonOptimizer adjusts LogicalPlan to insert CarbonDictionaryCatalystDecoder before LogicalPlan which needs to decode the dictionary
LogcialPlan, delay decoding or not decoding as much as possible.

4. CarbonStrategy (CarbonTableScan) adapts CarbonData related LogicalPlan to generate SparkPlan, including: dictionary delay decoding CarbonDictionaryDecoder, ExecutedCommand (LoadTableByInsert) and table scan TableScan. TableScan is used to generate the CarbonScanRDD and read the CarbonData table data.

5. CarbonSource is carbondata datasource api, shortname is carbondata; Carbon table
Relation is CarbonDataSourceRelation.

6. CarbonEnv: The main role is to initialize the metadata information loaded with CarbonData.

3.2 **spark2 integration**

In Spark2, use CarbonSession as the entrance to the spark Sql interface. CarbonSession extends SparkSession, including CarbonSessionState, CarbonSparkSqlParser, CarbonLateDecodeRule and CarbonLateDecodeStartegy, DDLStartegy, and more.

Compared to spark 1.x, the differences are:

The call flow of CarbonSparkSqlParser has changed. The first call to Spark's AbstractSqlParser resolves sql. If it cannot be parsed, it will continue to use CarbonSpark2SqlParser to parse it and generate a LogicalPlan of Carbondata. CarbonSpark2SqlParser mainly deals with create table, load table and other sql parsing on the Carbondata table. Currently, Carbondata does not extend the select statement and is still parsed by spark AbstractSqlParser. This order adjustment helps reduce query parsing time.

<img src="media/3-2_1.png" width = "35%" alt="3-2_1" />

The implementation of InsertInto has changed. Spark adds Analyzer (CarbonPreInsertionCasts) and adapts LoadTableByInsert in DDLStrategy. Implemented by extending CarbonDatasourceHadoopRelation to InserttableRelation.

Carbon table relation changed to CarbonDatasourceHadoopRelation,
Use the buildScan method instead to generate the CarbonScanRDD.