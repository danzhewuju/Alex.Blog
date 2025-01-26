---
title: Paimon Read 源码分析
date: '2025-01-26 10:36:10'
updated: '2025-01-26 23:06:34'
permalink: /post/paimon-read-source-code-analysis-apxgf.html
comments: true
toc: true
---

# Paimon Read 源码分析

# Abstract

本文主要是对paimon的写过程进行源码分析。

从官网上可以看到paimon读操作的配置，其中相关的配置都和snapshot相关。

​![image.png](https://raw.githubusercontent.com/danzhewuju/danzhewuju.github.io/blog/blog/source/images20250126192344.png)​

从下面的流程图中可以看到他们之间主要差别。

​![image.png](https://raw.githubusercontent.com/danzhewuju/danzhewuju.github.io/blog/blog/source/images20250126192340.png)​

对于manifest 的文件的读取主要是根据下面的plan 来实现

org.apache.paimon.operation.AbstractFileStoreScan#plan

```java
public Plan plan() {
        long started = System.nanoTime();
        ManifestsReader.Result manifestsResult = readManifests();
        Snapshot snapshot = manifestsResult.snapshot;
        List<ManifestFileMeta> manifests = manifestsResult.filteredManifests;

        Iterator<ManifestEntry> iterator = readManifestEntries(manifests, false);
        List<ManifestEntry> files = new ArrayList<>();
        while (iterator.hasNext()) {
            files.add(iterator.next());
        }

        if (wholeBucketFilterEnabled()) {
            // We group files by bucket here, and filter them by the whole bucket filter.
            // Why do this: because in primary key table, we can't just filter the value
            // by the stat in files (see `PrimaryKeyFileStoreTable.nonPartitionFilterConsumer`),
            // but we can do this by filter the whole bucket files
            files =
                    files.stream()
                            .collect(
                                    Collectors.groupingBy(
                                            // we use LinkedHashMap to avoid disorder
                                            file -> Pair.of(file.partition(), file.bucket()),
                                            LinkedHashMap::new,
                                            Collectors.toList()))
                            .values()
                            .stream()
                            .map(this::filterWholeBucketByStats)
                            .flatMap(Collection::stream)
                            .collect(Collectors.toList());
        }

        List<ManifestEntry> result = files;

        long scanDuration = (System.nanoTime() - started) / 1_000_000;
        if (scanMetrics != null) {
            long allDataFiles =
                    manifestsResult.allManifests.stream()
                            .mapToLong(f -> f.numAddedFiles() - f.numDeletedFiles())
                            .sum();
            scanMetrics.reportScan(
                    new ScanStats(
                            scanDuration,
                            manifests.size(),
                            allDataFiles - result.size(),
                            result.size()));
        }

        return new Plan() {
            @Nullable
            @Override
            public Long watermark() {
                return snapshot == null ? null : snapshot.watermark();
            }

            @Nullable
            @Override
            public Snapshot snapshot() {
                return snapshot;
            }

            @Override
            public List<ManifestEntry> files() {
                return result;
            }
        };
    }
```

读取文件的方法：org.apache.paimon.table.source.snapshot.SnapshotReaderImpl#read

此方法的功能主要是split的划分

```java
public Plan read() {
        FileStoreScan.Plan plan = scan.plan();
        @Nullable Snapshot snapshot = plan.snapshot();

        Map<BinaryRow, Map<Integer, List<DataFileMeta>>> files =
                groupByPartFiles(plan.files(FileKind.ADD));
        if (options.scanPlanSortPartition()) {
            Map<BinaryRow, Map<Integer, List<DataFileMeta>>> newFiles = new LinkedHashMap<>();
            files.entrySet().stream()
                    .sorted((o1, o2) -> partitionComparator().compare(o1.getKey(), o2.getKey()))
                    .forEach(entry -> newFiles.put(entry.getKey(), entry.getValue()));
            files = newFiles;
        }
        List<DataSplit> splits =
                generateSplits(snapshot, scanMode != ScanMode.ALL, splitGenerator, files);
        return new PlanImpl(
                plan.watermark(), snapshot == null ? null : snapshot.id(), (List) splits);
    }
```

文件读取，主要是从split中开始读取数据：

‍
