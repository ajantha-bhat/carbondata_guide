2 **format module**

2.1 **File Directory Structure**

The CarbonData data is stored in the location specified by the carbon.storelocation configuration item (configured in carbon.properties; defaults to ../carbon.store if it is not configured).

 The file directory structure is shown in Figure 2-1_1:

<img src="media/2-1_1.png" width = "60%" alt="2-1_1" />

1. The carbon.store directory is where the CarbonData data is stored, under which is the ModifiedTime.mdt file and the database directory (default database name:default). ModifiedTime.mdt uses the file's modification time property to record the time version of the metadata. When the drop table and create table are updated, the modification time of the file is updated.

2. There is a user table directory (for example: user_table) belonging to the database in the default directory.

3. The user_table directory has the Metadata directory and the Fact directory
    
4. The Metadata directory stores schema files, tablestatus, and dictionary files (including .dict, .dictmeta, and .sortindex) in three categories of metadata data information files.

5. The Fact directory stores data and index files. The Fact directory has a Part0 partition directory, where 0 is the partition number.
    
6. There is a Segment_0 directory in the Part0 directory, where 0 is the segment number.
    
7. The Segment_0 directory has two types of files, carbondata and carbonindex.

2.2 **Detailed contents of the file**

When the table is created, the user_table directory is generated and a schema file is generated in the Metadata directory for recording the table structure.

When loading data in batches, each batch load generates a new segment directory. The scheduler tries to control starting a task on each node to process data load tasks. Each task will generate multiple carbon data files and a carbonindex file.

Regarding the global dictionary, if the two-pass scheme is adopted, before the data is loaded, the corresponding dict, dictmeta and sortindex three files are generated for each dictionary coded column. The pre-define dictionary may be used to provide some dictionary files to reduce the need. Generate dictionary-encoded columns by scanning the entire data; you can also use all dictionary to provide dictionary files for all dictionary-encoded columns to avoid scanning data. If the single-pass scheme is adopted, the global dictionary code is generated in real time during data loading. After the data is loaded, the dictionary is cured to a dictionary file.
 
In the following sections, the Java objects generated after compiling the thrift files describing the format of the carbondata files are used to explain the contents of each file one by one (you can also read the thrift file defined by the format directly) (https://github.com/apache/incubator-carbondata/tree). /master/format/src/main/thrift or [Introduction to the official website data format] (https://github.com/apache/incubator-carbondata/blob/master/docs/file-structure-of-carbondata.md) Learn about the carbondata data format).

2.2.1 **Schema file format**

The contents of the schema file are shown in the TableInfo class in Figure 2-2_1:

<img src="media/2-2_1.png" width = "60%" alt="2-2_1" />

1. TableSchema class
    The TableSchema class does not record the table name, and the table name is determined by the user_table directory name.
    TableProperties is used to record table-related properties, such as table_blocksize.

2. ColumnSchema class
    Encoders used to record the encoding used by the column store
    columnProperties is used to record column related attributes.

3. BucketingInfo class
    When creating a bucket table, you can specify the bucket number of the table and the column to splitbuckets.

4. DataType class
    Describes the data types supported by CarbonData.

5. Encoding class
    Several encodings that CarbonData files may use.

2.2.2 **carbondata file format**

The carbondata file consists of multiple blocklets and footer sections. The blocklet is the internal dataset of the carbondata file (the latest V3 format, the default configuration is 64MB), each blocklet contains a ColumnChunk for each column, and a ColumnChunk may contain one or more Column Pages.

Carbondata files currently support versions V1, V2, and V3. The main difference is the change in the blocklet section.

**blocklet section**
 V1:
 The blocket consists of all the column's data page, RLE page, and rowID page. Since the pages in the blocklet are grouped according to the page type, the three pieces of data of each column are stored separately in the blocklet. In the footer part, the offset and length information of all pages need to be recorded.

<img src="media/2-3_1.png" width = "25%" alt="2-3_1" />

V2:
A blocklet consists of ColumnChunks for all columns. A column's ColumnChunk consists of a ColumnPage. The ColumnPage includes a data chunk header, a data page, an RLE page, and a rowID page. Since ColumnChunk aggregates the 3 types of Page data of the column, it can use less readers to complete the column data reading. Since the header part records the length information of all pages, the footer part only needs to record the offset and length of the ColumnChunk and also reduce the amount of footer data.

<img src="media/2-3_2.png" width = "50%" alt="2-3_2" />

V3:
The blocklet also consists of ColumnChunks for all columns. What's changed is that a ColumnChunk consists of one or more Column Pages, and ColumnPage adds a new BlockletMinMaxIndex.

Compared with V2: V2 format blocklet data volume defaults to 120,000 lines, while the V3 format blocklet data volume defaults to 64MB, the same size of the data file, footer part of the index metadata information may be further reduced; V3 format added page The level of data filtering, and the default data volume per page is only 32,000 lines, a lot less than the 120,000 lines in the V2 format. The hit accuracy of data filtering further suggests that more data can be filtered out before decompressing data.

<img src="media/2-3_3.png" width = "50%" alt="2-3_3" />

**footer section**

Footer records every carbondata
All blocklet data distribution information within the file and statistics related metadata information (minmax, startkey/endkey).

<img src="media/2-3_4.png" width = "80%" alt="2-3_4" />

BlockletInfo3 is used to record the offset and length of all ColumnChunk3s.

2. SegmentInfo is used to record the number of columns and the cardinality of each column.

BlockletIndex includes BlockletMinMaxIndex and BlockletBTreeIndex.

BlockletBTreeIndex is used to record the startkey/endkey of all the blocklets in the block; the query is used to generate the query's startkey/endkey through filtering conditions combined with mdkey. With BlocketBtreeIndex, the blocklet range that meets the conditions in each block can be delimited.

BlockletMinMaxIndex is used to record the min/max value of all columns in the blocklet; by checking min/max on the filter condition when querying, blocks/blocklets that do not satisfy the condition can be skipped.

2.2.3 **carbonindex file format**

The carbonindex file is generated by extracting the BlockletIndex section of the footer section. Bulk load data, scheduling as much as possible to control a node to start a task, each task generates multiple carbondata files and a carbonindex file. The carbonindex file records the index information of all the blocklets inside all carbondata files generated by the task.

As shown in the figure, the index information corresponding to a block is recorded by a BlockIndex object, including carbondata filename, footer offset, and BlockletIndex. The BlockIndex data size is less than the footer. When the query is used, the file is directly used to build an index on the driver side without skipping more data in the data file.

<img src="media/2-4_1.png" width = "25%" alt="2-4_1" />

2.2.4 **dictionary file format**
    
For each dictionary code column, 3 files are used to store the dictionary metadata for that column.
 
1. dict file records the distinct value list of a column

When dataloading is used for the first time, the file is generated using a column's distinct value list. The value in the file is unordered; In dataloading step (Data Convert Step), the dictionary encoding column will use the dictionary key to replace the data's real value.

<img src="media/2-5_1.png" width = "25%" alt="2-5_1" />

The
2. dictmeta records the metadata description of the newly added distinct value for each dataloading

The dictionary cache uses this information to incrementally refresh the cache.

<img src="media/2-5_2.png" width = "25%" alt="2-5_2" />
The
3. sortindex record dictionary coded according to the value of the key result set.

In dataLoading, if there is a new dictionary value, the entire dictionary code will be used to regenerate the sortindex file.

Filtering queries based on dictionary encoding columns requires that the filter bar of value be converted to the filter condition of the key, and the sortindex file can be used to rapidly construct an ordered value sequence so that the key value corresponding to value can be quickly found, thereby speeding up the conversion process.

<img src="media/2-5_3.png" width = "30%" alt="2-5_3" />

2.2.5 **tablestatus file format**

Tablestatus records information about each loaded and merged segment (in gson format), including load time, load status, segment name, whether it was deleted, and the name of the merged segment. After each load or merge is complete, regenerate tablestatusfile.

<img src="media/2-6_1.png" width = "25%" alt="2-6_1" />