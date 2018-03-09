5 **processing module**

5.1 **Global Dictionary**

Global dictionary encoding (currently a table-level dictionary encoding) uses a uniform integer to represent real data, which can serve as a data compression function. At the same time, the computation of grouping based on dictionary encoding is also more efficient. The global dictionary encoding generation process is as shown in the figure. CarbonBlockDistinctValuesCombinRDD calculates the distinct value list of the dictionary columns in each split, and then performs the shuffle partition according to the ColumnPartitioner, one partition per column. CarbonGlobalDictionaryGenerateRDD handles one dictionary column per task, generates a dictionary file (or appends a new dictionary value), refreshes the sortindex, and updates dictmeta.

<img src="media/5-1_1.png" width = "50%" alt="5-1_1" />

5.2 **DataLoading**

The CSV file data loads the main flow, as shown in the figure below. First, CarbonLoaderUtil.nodeBlockMapping partitions data blocks by node. NewCarbonDataLoadRDD uses this node for partitioning. Each node starts a task to handle the loading of data blocks. DataLoadExecutor.execute performs data loading and the main flow includes the four steps shown in the figure.

<img src="media/5-2_1.png" width = "50%" alt="5-2_1" />

Step1: InputProcessorStepImpl uses CSVInputFormat.CSVRecordReader to read and parse the CSV file.

Step2: DataConverterProcessStepImpl is used to convert data. FieldConverter mainly has the following implementation classes, including dictionary columns, non-dictionary columns, direct dictionaries, complex type columns, and measure column conversions.
![5-2_2](media/5-2_2.png)
Step3: SortProcessorStepImpl will be in accordance with the dimension sort, the default Sorter implementation class is ParallelReadMergeSorterImpl, sorter main flow as shown in the right figure.

The SortDataRows.addRowBatch method caches data, when the number of data records reaches the sort buffer size (default 100000), calls DataRorterAndWriter to sort and generates a tmp file to the local disk; when the number of tmpfile reaches the configured threshold (default 20), SortIntermediateFileMerger.startMerge calls these tmpfiles Sorting generates big tmp file. The input data in Step1 and Step2 are all sorted and generated files (some big tmpfile
And less than 20 tmpfile) into the tmp directory, SingleThreadFinalSortFilesMerger.startFinalMerge start finalmerge, stream merge and sort all tmpfile, the purpose is to make this node ordering of the loading data, and provide streaming data for subsequent Step4 .

<img src="media/5-2_3.png" width = "60%" alt="5-2_3" />

Step4: DataWriterProcessorStepImpl is used to generate carbondata and carbonindex files. The main flow is shown below. MultiDimKeyVarLengthGenerator.generateKey generates an MDK for each line of dictionary code dicesion. CarbonFactDataHandlerColumnar.addDataToStore caches data such as MDK encoding. After the number of records reaches the size of the blockletsize, the producer calls the Producer to generate a Blocklet object (NodeHolder).

BlockIndexerStorageForInt/Short handles the sorting (Array.sort) of the dimension column data in the blocklet, generates the RowId index (compressMyOwnWay), and uses RLE compression (compressDataMyOwnWay)

HeavyCompressedDoubleArrayDataInMemoryStore handles the compression of meause class data within the bloclet (using snappy).

CarbonFactDataWriterImplV2.writerBlockletData writes an existing blocklet data to a local data file. If the cumulative size of the blocklet has reached the size of table\_blocksize, create a new carbondata to write the data.

After the blocklet of the carbondata file is written, call the writeBlockletInfoToFile to complete the writing of the footer. After this node's task is completed, call writeIndexFile to generate the carbonindex file.

<img src="media/5-2_4.png" width = "60%" alt="5-2_4" />

5.3 **Compression Encoding**

1. Snappy Compressor

2. Number Compressor

3. Delta Compressor

4. Dictionary Encoding

5. Direct-Ditionary Encoding

6. RLE (Running Length Encoding)