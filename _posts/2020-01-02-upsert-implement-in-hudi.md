### Hudi 中的 upsert 实现

#### HoodieWriteClient#upsert
更新或插入记录
```Java
public JavaRDD<WriteStatus> upsert(JavaRDD<HoodieRecord<T>> records, final String commitTime) {
    // 获取表并初始化上下文

    // 数据去重

    // 查找索引并用包含对应行的文件位置标记每个传入 record (如果实际存在这样的行)
    JavaRDD<HoodieRecord<T>> taggedRecords = index.tagLocation(dedupedRecords, jsc, table);

    // 插入更新记录并返回写入状态
    return upsertRecordsInternal(taggedRecords, commitTime, table, true);
}
```

#### HoodieWriteClient#upsertRecordsInternal
```Java
private JavaRDD<WriteStatus> upsertRecordsInternal(JavaRDD<HoodieRecord<T>> preppedRecords, String commitTime, HoodieTable<T> hoodieTable, final boolean isUpsert) {

    // 缓存

    // WorkloadProfile
    WorkloadProfile profile = null;
    if (hoodieTable.isWorkloadProfileNeeded()) {
     // 统计了每个分区的读写 record 数, 以及全局读写 record 数
      profile = new WorkloadProfile(preppedRecords);
      logger.info("Workload profile :" + profile);
      // 将 WorkloadProfile 保存到中间文件, 这对执行 MOR 数据集的回滚很有用 ???
      saveWorkloadProfileMetadataToInflight(profile, hoodieTable, commitTime);
    }

    // 获取分区器
    final Partitioner partitioner = getPartitioner(hoodieTable, isUpsert, profile);

    // 分区数据, 每个 bucket 为一个分区, 将 record 分区到对应的 bucket 中
    JavaRDD<HoodieRecord<T>> partitionedRecords = partition(preppedRecords, partitioner);

    // upsert 数据
    JavaRDD<WriteStatus> writeStatusRDD = partitionedRecords.mapPartitionsWithIndex((partition, recordItr) -> {
      if (isUpsert) {
        return hoodieTable.handleUpsertPartition(commitTime, partition, recordItr, partitioner);
      } else {
        return hoodieTable.handleInsertPartition(commitTime, partition, recordItr, partitioner);
      }
    }, true).flatMap(List::iterator);

    // ...
}
```

#### HoodieCopyOnWriteTable#$UpserPartitioner#<init>
```Java
UpsertPartitioner(WorkloadProfile profile) {
    // Map<fileId, bucketId>
    updateLocationToBucket = new HashMap<>();
    partitionPathToInsertBuckets = new HashMap<>();
    // Map<bucketId, bucketInfo>
    bucketInfoMap = new HashMap<>();
    globalStat = profile.getGlobalStat();
    rollingStatMetadata = getRollingStats();
    assignUpdates(profile);
    assignInserts(profile);

    // log
}
```

#### HoodieCopyOnWriteTable#assignInserts
```Java
private void assignInserts(WorkloadProfile profile) {
    // 获取 record 将写入的所有分区
    Set<String> partitionPaths = profile.getPartitionPaths();

    // 基于 hhoodie.copyonwrite.record.size.estimate 和已写入 record 数和字节, 计算每条 record 的平均大小

    // 处理每个分区
    for (String partitionPath : partitionPaths) {
        // 获取分区的 WorklaodStat
        WorkloadStat pStat = profile.getWorkloadStat(partitionPath);

        if (pStat.getNumInserts() > 0) {
            // 在当前分区下小于等于最后一次 commit 的每个文件的最后一个版本中, 返回小于 hoodie.parquet.small.file.limit 的文件, 并额外记录在一个全局小文件列表中
            List<SmallFile> smallFiles = getSmallFiles(partitionPath);

            // 总的未分配的插入数
            long totalUnassignedInserts = pStat.getNumInserts();
            List<Integer> bucketNumbers = new ArrayList<>();
            List<Long> recordsPerBucket = new ArrayList<>();

             for (SmallFile smallFile : smallFiles) {
                 // 计算当前文件应该追加多少 record
                 long recordsToAppend = Math.min((config.getParquetMaxFileSize() - smallFile.sizeBytes) / averageRecordSize, totalUnassignedInserts);
                 if (recordsToAppend > 0 && totalUnassignedInserts > 0) {
                     // 根据 fileId 在 updateLocationToBucket 找到对应的 bucketId, 没有则在 updateLocationToBucket 新增一个

                     // 记录此分区 record 的所使用的 bucketId, 以及追加的 record 数
                     bucketNumbers.add(bucket);
                     recordsPerBucket.add(recordsToAppend);
                     totalUnassignedInserts -= recordsToAppend;
                 }
             }

             // 当小文件剩余容量使用完时, 则创建新文件
             if (totalUnassignedInserts > 0) {
                 //  hoodie.copyonwrite.insert.split.size 和 hoodie.parquet.max.file.size 决定每个 bucket 应该放多少 record

                 // 计算还需使用多少 bucket

                 // 循环创建 bucket
             }

             // 遍历 bucketNumbers, 基于桶插入数占比分区插入数分配权重
             List<InsertBucket> insertBuckets = new ArrayList<>();

             // ...

             // Map<partitionPath, insertBuckets>
             partitionPathToInsertBuckets.put(partitionPath, insertBuckets);
        }
    }
}
```

#### HoodieCopyOnWriteTable#$UpserPartitioner#getPartition
```Java
public int getPartition(Object key) {
    Tuple2<HoodieKey, Option<HoodieRecordLocation>> keyLocation = (Tuple2<HoodieKey, Option<HoodieRecordLocation>>) key;

    if (keyLocation._2().isPresent()) {
        // 有 location 即为更新数据, 返回 updateLocationToBucket.get(fileId)
    } else {
        // 所有插入 record 的 bucket
        List<InsertBucket> targetBuckets = partitionPathToInsertBuckets.get(keyLocation._1().getPartitionPath());
        // 基于权重选择目标 bucket
        double totalWeight = 0.0;
        final long totalInserts = Math.max(1, globalStat.getNumInserts());
        final long hashOfKey = Hashing.md5().hashString(keyLocation._1().getRecordKey(), StandardCharsets.UTF_8).asLong();
        final double r = 1.0 * Math.floorMod(hashOfKey, totalInserts) / totalInserts;
        // 感觉这里的计算不能将插入 bucket 对应的文件均匀化 ???
        for (InsertBucket insertBucket : targetBuckets) {
          totalWeight += insertBucket.weight;
          if (r <= totalWeight) {
            return insertBucket.bucketNumber;
          }
        }
        // 默认返回
        return targetBuckets.get(0).bucketNumber;
    }
}
```

#### HoodieCopyOnWriteTable#handleUpsertPartition
```Java
public Iterator<List<WriteStatus>> handleUpsertPartition(String commitTime, Integer partition, Iterator recordItr, Partitioner partitioner) {
    // 根据 partition 索引获取对应的 bucketInfo
    UpsertPartitioner upsertPartitioner = (UpsertPartitioner) partitioner;
    BucketInfo binfo = upsertPartitioner.getBucketInfo(partition);
    BucketType btype = binfo.bucketType;
    // 分别处理 insert 和 update
    try {
        if (btype.equals(BucketType.INSERT)) {
        // 分区无记录返回空迭代器, 否则创建一个迭代器返回
        return handleInsert(commitTime, binfo.fileIdPrefix, recordItr);
      } else if (btype.equals(BucketType.UPDATE)) {
        return handleUpdate(commitTime, binfo.fileIdPrefix, recordItr);
      } else {
        throw new HoodieUpsertException("Unknown bucketType " + btype + " for partition :" + partition);
      }
    } catch (Throwable t) {
        // 异常处理
    }
}

```
