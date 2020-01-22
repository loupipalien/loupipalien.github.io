### Hudi 中的 Bloom Filter

#### HoodieWriteClient#upsert
更新或插入记录
```Java
public JavaRDD<WriteStatus> upsert(JavaRDD<HoodieRecord<T>> records, final String commitTime) {
    // 获取表并初始化上下文

    // 数据去重

    // 查找索引并用包含对应行的文件位置标记每个传入 record (如果实际存在这样的行)
    JavaRDD<HoodieRecord<T>> taggedRecords = index.tagLocation(dedupedRecords, jsc, table);

    // TODO
    return upsertRecordsInternal(taggedRecords, commitTime, table, true);
}
```

#### tagLocation
为每个 record 标记文件位置
```Java
public JavaRDD<HoodieRecord<T>> tagLocation(JavaRDD<HoodieRecord<T>> recordRDD, JavaSparkContext jsc, HoodieTable<T> hoodieTable) {
    // 缓存输入的 record RDD
    if (config.getBloomIndexUseCaching()) {
      recordRDD.persist(config.getBloomIndexInputStorageLevel());
    }

    // 抽取出 (partitionPath, recordKey) 的 JavaPairRDD
    JavaPairRDD<String, String> partitionRecordKeyPairRDD = recordRDD.mapToPair(record -> new Tuple2<>(record.getPartitionPath(), record.getRecordKey()));

    // 为所有的 (partitionPath, recordKey) 查找索引
    JavaPairRDD<HoodieKey, HoodieRecordLocation> keyFilenamePairRDD = lookupIndex(partitionRecordKeyPairRDD, jsc, hoodieTable);

    // 缓存结果
    if (config.getBloomIndexUseCaching()) {
      keyFilenamePairRDD.persist(StorageLevel.MEMORY_AND_DISK_SER());
    }

    // 通过将 recordRDD 和 keyFilenamePairRDD 做连接区别出 insert 和 update 的 record
    JavaRDD<HoodieRecord<T>> taggedRecordRDD = tagLocationBacktoRecords(keyFilenamePairRDD, recordRDD);

    // 释放缓存

    // 返回 taggedRecordRDD
    return taggedRecordRDD;
}
```

#### lookupIndex
查找索引
```Java
private JavaPairRDD<HoodieKey, HoodieRecordLocation> lookupIndex(JavaPairRDD<String, String> partitionRecordKeyPairRDD, final JavaSparkContext jsc, final HoodieTable hoodieTable) {
    // 按 partitionPath 统计 record 的数
    Map<String, Long> recordsPerPartition = partitionRecordKeyPairRDD.countByKey();

    // 获得被影响的分区的 partitionPath 列表
    List<String> affectedPartitionPathList = new ArrayList<>(recordsPerPartition.keySet());

    // 加载被影响的文件, 返回 List<Tuple2<partitionPath, BloomIndexFileInfo>>
    List<Tuple2<String, BloomIndexFileInfo>> fileInfoList = loadInvolvedFiles(affectedPartitionPathList, jsc, hoodieTable);

    // 将上一步的结果转换为 Map<partitionPath, List<BloomIndexFileInfo>>
    final Map<String, List<BloomIndexFileInfo>> partitionToFileInfo = fileInfoList.stream().collect(groupingBy(Tuple2::_1, mapping(Tuple2::_2, toList())));

    // 计算每个文件可能存在与新来的  record 的个数的映射
    Map<String, Long> comparisonsPerFileGroup = computeComparisonsPerFileGroup(recordsPerPartition, partitionToFileInfo, partitionRecordKeyPairRDD);

    // 根据 record 计算安全的并行度, 由于 spark 每个分区有 2GB 的限制, Hudi 中设置每分区最大大小为 1.5GB, 并假定 Triple<partitionPath, fileId, recordKey> 最大字节数为 300 BYTE
    int safeParallelism = computeSafeParallelism(recordsPerPartition, comparisonsPerFileGroup);

    // 再从 partitionRecordKeyPairRDD.partitions.size(), hoodie.bloom.index.parallelism, safeParallelism 选出最大值
    int joinParallelism = determineParallelism(partitionRecordKeyPairRDD.partitions().size(), safeParallelism);

    // 返回每个 recordKey 所在文件的 location
    return findMatchingFilesForRecordKeys(partitionToFileInfo, partitionRecordKeyPairRDD, joinParallelism, hoodieTable, comparisonsPerFileGroup);
}
```

#### loadInvolvedFiles
```Java
List<Tuple2<String, BloomIndexFileInfo>> loadInvolvedFiles(List<String> partitions, final JavaSparkContext jsc, final HoodieTable hoodieTable) {
    // 根据最新的 commitTime, 查找每个分区中最不同 fileId 最靠近 commitTime 的文件, 返回 Pait<partitionPath, fileId> 的列表
    List<Pair<String, String>> partitionPathFileIDList =
           jsc.parallelize(partitions, Math.max(partitions.size(), 1)).flatMap(partitionPath -> {
             Option<HoodieInstant> latestCommitTime =
                 hoodieTable.getMetaClient().getCommitsTimeline().filterCompletedInstants().lastInstant();
             List<Pair<String, String>> filteredFiles = new ArrayList<>();
             if (latestCommitTime.isPresent()) {
               filteredFiles = hoodieTable.getROFileSystemView()
                   .getLatestDataFilesBeforeOrOn(partitionPath, latestCommitTime.get().getTimestamp())
                   .map(f -> Pair.of(partitionPath, f.getFileId())).collect(toList());
             }
             return filteredFiles.iterator();
           }).collect();    
    // 返回 Pait<partitionPath, BloomIndexFileInfo> 的列表
    if (config.getBloomIndexPruneByRanges()) {
      // 使用范围来剪枝 (recordKey 如果是单调的, 通过读取 parquet 文件 footer 中存放的 minKey 和 maxKey 来确定 recordKey 是否在其中)
      return jsc.parallelize(partitionPathFileIDList, Math.max(partitionPathFileIDList.size(), 1)).mapToPair(pf -> {
        try {
          HoodieRangeInfoHandle<T> rangeInfoHandle = new HoodieRangeInfoHandle<T>(config, hoodieTable, pf);
          String[] minMaxKeys = rangeInfoHandle.getMinMaxKeys();
          return new Tuple2<>(pf.getKey(), new BloomIndexFileInfo(pf.getValue(), minMaxKeys[0], minMaxKeys[1]));
        } catch (MetadataNotFoundException me) {
          logger.warn("Unable to find range metadata in file :" + pf);
          return new Tuple2<>(pf.getKey(), new BloomIndexFileInfo(pf.getValue()));
        }
      }).collect();
    } else {
      return partitionPathFileIDList.stream()
          .map(pf -> new Tuple2<>(pf.getKey(), new BloomIndexFileInfo(pf.getValue()))).collect(toList());
    }
}
```

#### computeComparisonsPerFileGroup
为每个文件组计算要执行的布隆过滤器比较的估计数
```Java
private Map<String, Long> computeComparisonsPerFileGroup(final Map<String, Long> recordsPerPartition, final Map<String, List<BloomIndexFileInfo>> partitionToFileInfo, JavaPairRDD<String, String> partitionRecordKeyPairRDD) {
    if (config.getBloomIndexPruneByRanges()) {
        // 利用 Bloom Index 计算可能在此文件中的 record 数
        fileToComparisons = explodeRecordRDDWithFileComparisons(partitionToFileInfo, partitionRecordKeyPairRDD).mapToPair(t -> t).countByKey();
    } else {
        // 每个文件与可能在本文件中的 record 数 (简单的计算为所有进入此分区的 record 数)
        fileToComparisons = new HashMap<>();
        partitionToFileInfo.entrySet().stream().forEach(e -> {
            for (BloomIndexFileInfo fileInfo : e.getValue()) {
              // each file needs to be compared against all the records coming into the partition
              fileToComparisons.put(fileInfo.getFileId(), recordsPerPartition.get(e.getKey()));
            }
          });
    }
    // 返回 Map<fileId, Count>
    return fileToComparisons;
}
```

#### explodeRecordRDDWithFileComparisons
```Java
JavaRDD<Tuple2<String, HoodieKey>> explodeRecordRDDWithFileComparisons(final Map<String, List<BloomIndexFileInfo>> partitionToFileIndexInfo, JavaPairRDD<String, String> partitionRecordKeyPairRDD) {
    // 根据 hoodie.bloom.index.use.treebased.filter 选择使用基于间隔树的索引过滤器还是基于列表的索引过滤器
    IndexFileFilter indexFileFilter = config.useBloomIndexTreebasedFilter()
      ? new IntervalTreeBasedIndexFileFilter(partitionToFileIndexInfo)
      : new ListBasedIndexFileFilter(partitionToFileIndexInfo);
    // 过滤出 partitionPath 下可能包含 recordKey 的文件, 返回 Tuple2<fileId, HoodieKey> 的 RDD
    return partitionRecordKeyPairRDD.map(partitionRecordKeyPair -> {
      String recordKey = partitionRecordKeyPair._2();
      String partitionPath = partitionRecordKeyPair._1();

      return indexFileFilter.getMatchingFiles(partitionPath, recordKey).stream()
          .map(matchingFile -> new Tuple2<>(matchingFile, new HoodieKey(recordKey, partitionPath)))
          .collect(Collectors.toList());
    }).flatMap(List::iterator);
}
```

#### findMatchingFilesForRecordKeys
```Java
JavaPairRDD<HoodieKey, HoodieRecordLocation> findMatchingFilesForRecordKeys(final Map<String, List<BloomIndexFileInfo>> partitionToFileIndexInfo, JavaPairRDD<String, String> partitionRecordKeyPairRDD, int shuffleParallelism, HoodieTable hoodieTable, Map<String, Long> fileGroupToComparisons) {
    // 利用 Bloom Index 计算可能在此文件中的 record 数
    JavaRDD<Tuple2<String, HoodieKey>> fileComparisonsRDD = explodeRecordRDDWithFileComparisons(partitionToFileIndexInfo, partitionRecordKeyPairRDD);

    // 是否启用桶式布隆过滤 (默认为 true)

    // 确定 recordKey 所在的文件, 并返回 RDD
    return fileComparisonsRDD.mapPartitionsWithIndex(new HoodieBloomIndexCheckFunction(hoodieTable, config), true)
       .flatMap(List::iterator).filter(lr -> lr.getMatchingRecordKeys().size() > 0)
       .flatMapToPair(lookupResult -> lookupResult.getMatchingRecordKeys().stream()
           .map(recordKey -> new Tuple2<>(new HoodieKey(recordKey, lookupResult.getPartitionPath()),
               new HoodieRecordLocation(lookupResult.getBaseInstantTime(), lookupResult.getFileId())))
           .collect(Collectors.toList()).iterator());
}
```

#### HoodieBloomIndexCheckFunction$LazyKeyCheckIterator#computeNext
```Java
protected List<HoodieKeyLookupHandle.KeyLookupResult> computeNext() {
    List<HoodieKeyLookupHandle.KeyLookupResult> ret = new ArrayList<>();
      try {
        // 遍历每个 Tuple2<fileId, HoodieKey>
        while (inputItr.hasNext()) {
          Tuple2<String, HoodieKey> currentTuple = inputItr.next();
          String fileId = currentTuple._1;
          String partitionPath = currentTuple._2.getPartitionPath();
          String recordKey = currentTuple._2.getRecordKey();
          Pair<String, String> partitionPathFilePair = Pair.of(partitionPath, fileId);

          // 懒加载初始化状态
          if (keyLookupHandle == null) {
            keyLookupHandle = new HoodieKeyLookupHandle(config, hoodieTable, partitionPathFilePair);
          }

          // 如果还是当前文件, 直接检查 recordKey 是否为候选 recordKey
          if (keyLookupHandle.getPartitionPathFilePair().equals(partitionPathFilePair)) {
            keyLookupHandle.addKey(recordKey);
          } else {
            // 由于 RDD 是以 fileId 排序的, 当进入此分支表示上一个 file 已经结束, 这时在这里再将候选 recordKey 过滤一次
            ret.add(keyLookupHandle.getLookupResult());
            // 为当前 file 生成新的 HoodieKeyLookupHandle
            keyLookupHandle = new HoodieKeyLookupHandle(config, hoodieTable, partitionPathFilePair);
            keyLookupHandle.addKey(recordKey);
            break;
          }
        }

        // handle case, where we ran out of input, close pending work, update return val
        if (!inputItr.hasNext()) {
          ret.add(keyLookupHandle.getLookupResult());
        }
      } catch (Throwable e) {
        if (e instanceof HoodieException) {
          throw e;
        }
        throw new HoodieIndexException("Error checking bloom filter index. ", e);
      }

      return ret;
}
```

#### HoodieKeyLookupHandle#getLookupResult
对于候选的 recordKey, 检查文件进行再次确认
```Java
public KeyLookupResult getLookupResult() {
    if (logger.isDebugEnabled()) {
      logger.debug("#The candidate row keys for " + partitionPathFilePair + " => " + candidateRecordKeys);
    }

    HoodieDataFile dataFile = getLatestDataFile();
    List<String> matchingKeys =
        checkCandidatesAgainstFile(hoodieTable.getHadoopConf(), candidateRecordKeys, new Path(dataFile.getPath()));
    logger.info(
        String.format("Total records (%d), bloom filter candidates (%d)/fp(%d), actual matches (%d)", totalKeysChecked,
            candidateRecordKeys.size(), candidateRecordKeys.size() - matchingKeys.size(), matchingKeys.size()));
    return new KeyLookupResult(partitionPathFilePair.getRight(), partitionPathFilePair.getLeft(),
        dataFile.getCommitTime(), matchingKeys);
  }
```
