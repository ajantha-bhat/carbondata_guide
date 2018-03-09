4 **core module**

4.1 **Scan (Query)**

CarbonScanRDD is a Carbon table dataset used to read Carbon table data; the key methods are getPartitions (block blocks that get filter hits for some of the blocklets) and compute (results that filter the blocklet to filter out and stitch out data rows).

1. The getPartition implementation flow chart shows. The CarbonInputFormat.getSplits method is used to get the block block of the query hit, which SegmentStatusManager.getValidAndInvalidSegments method can get all the segments in the carbon table is valid, through the SegmentTaskIndexStore.loadAndGetTaskIdtoSegmentsMap load segment's carbonindex index file, then FilterExpressionProcessor.getFilterredBlocks from the index information to find out All blocks that were hit. The distributeSplits method is used to split the splits evenly by cluster nodes according to the data localization calculation principle, and then generate SparkPartitions for all splits according to the grouping result.

<img src="media/4-1_1.png" width = "60%" alt="4-1_1" />

2. Compute using the implementation process as shown. The CarbonRecordReader internally wraps the DetailQueryExecutor to implement the detail query of the carbon table.

<img src="media/4-1_2.png" width = "60%" alt="4-1_2" />

getBlockExecutionInfos Gets the information needed for query execution on each block, including blockindex, startkey, endkey, startBlockletIndex, numberOfBlockletToScan, filterExecuter, and so on. The Executor side BlockIndex is built by the BlockIndexStore.getAll method.

DataBlockIteratorImpl.next scan block to get batchsize row data. BlockletScanner is divided into FilterFilter and NonFilterScanner according to whether or not there are filter conditions. BlockletScanner.scanBlocklet uses the cached BlocketBTreeLeafNode metadata information to load the hit blocklet (the number of startblocklet and blocklet passed on the driver side and the minmax check of the blocklet level on the execution side). The required column data is used to populate the ScannedResult. For the FilterScanner, the index of the row number of the data that the filter condition hits is also generated, and FilterQueryScannedResult or NonFilterQueryScannedResult is returned. The DictionaryBasedResultCollector calls the collectData method from the result of the previous step, stitching the required column data into the last needed row data.

4.2 **Filter Expression**

Inside the executor side, a set of methods for storing data in columns within a blocklet and filtering them according to expressions is implemented.

First, it was pushdown to filter in CarbonScanRDD
The expression is converted to the expression of carbondata as shown. Implement the code: CarbonFilters.createCarbonFilter.

![4-2_1](media/4-2_1.png)

Second, generate FilterResolverIntf based on the previous Expression tree
Tree, subclasses as shown below. The conversion process code is: CarbonInputFormatUtil.resolveFilter, which performs Expression expression calculation and encapsulates the result in different FilterResolverIntf depending on the expression type.

![4-2_2](media/4-2_2.png)

Next, generate a FilterExecuter based on the FilterResolverIntf tree from the previous step.
Tree, subclasses are shown below. The conversion process code is: FilterUtil.getFilterExecuterTree, and the corresponding FilterExecuter is generated according to the type of the Executer in FilterResolverIntf.

![4-2_3](media/4-2_3.png)

Finally, FilterScanner.fillScannedResult calls FilterExecuter.applyFilter to calculate the row number of the internal data of the blocklet that the filter condition hits, using RowId.
Index is converted to a row number index.

4.3 **LRU Cache**

There are four types of CarbonLRUCache depending on the purpose, as shown in the following figure, which are:

<img src="media/4-3_1.png" width = "80%" alt="4-3_1" />

ForwardDictionaryCache of Key-\>Value: Content is ColumnDcitionaryInfo
Reverse-DictionaryCache of Value-\>Key: Content is ColumnReverseDcitonaryInfo
Driver-side BTree Index Cache SegmentTaskIndexStoree: Content is SegmentTaskIndex
Executor Side Btree Index Cache BlockIndexStore: Content is BlockIndex

4.4 **BTree Index**

Use the cache's index to quickly hit the required blocks and blocklets. The index model is shown in the following figure. IndexKey contains dicitoary and nodicitonary columns, which is also the current carbondata
The sequence used for dataloading. The IndexKey in the BTreeNode is the startKey, endKey of the related Block/Blocklet. AbstractBTreeLeaf contains min/max values ​​for each column in the NodeBlock/Blocklet. BlockBTreeLeafNode is used for the driver side cache carbonindex information, BlockletBTreeLeafNode is used for the executor side cache footer information, and encapsulates the read from the hdfs column
Data chunk method.

<img src="media/4-4_1.png" width = "60%" alt="4-4_1" />

1. The Driver side index cache is managed by the SegmentTaskIndexStore. The index load build process is shown in the figure.

<img src="media/4-4_2.png" width = "60%" alt="4-4_2" />

CarbonUtil.readCarbonIndexFile reads the carobnindex file and SegmentTaskIndex.buildIndex constructs an index tree for each task of each segment.

The index tree structure is shown in the following figure. Each BTreeNonLeafNode has no more than 32 child nodes. The startkey and endkey are respectively the startkey of the leftmost child node and the endkey of the rightmost child node.

<img src="media/4-4_3.png" width = "60%" alt="4-4_3" />

In the query, the startkey, endkey, and minmax values ​​are first calculated through the filter conditions. Then, startkey, endkey are used to search out the qualified leafNode node range from the tree. Finally, minmax is used for the leafNode (corresponding to a block) to check out some of the blocks.

1. The Executer side index cache is managed by the BlockIndexStore. The index load build process is shown in the figure below. CarbonUtil.readMetadataFile reads the data file footer.

<img src="media/4-4_4.png" width = "60%" alt="4-4_4" />

Metadata information, BlockletBtreeBuilder.build builds an index tree for each block. The index tree structure is similar to the above figure. The leaf node is BlockletBtreeLeafNode.

When querying, first use the number of the start blocklet and the number of blocklets passed from the driver side, combine the blockletBtree on the side of the executer to determine the blocklets to be scanned, and use the minmax check of the blocklet level to skip the blocklets that do not meet the condition.