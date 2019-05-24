---
title: "Spark RDD Shuffle internals"
date: 2019-05-24T23:45:10+03:00
draft: false
---

## group-by is using an aggregator to combineValuesByKey


### At the DAG level configuration:

See [PairRDDFunctions.scala#L503-L505](https://github.com/apache/spark/blob/5fae8f7b1d26fca3cbf663e46ca0da6d76c690da/core/src/main/scala/org/apache/spark/rdd/PairRDDFunctions.scala#L503-L505)

```java
    val createCombiner = (v: V) => CompactBuffer(v)
    val mergeValue = (buf: CompactBuffer[V], v: V) => buf += v
    val mergeCombiners = (c1: CompactBuffer[V], c2: CompactBuffer[V]) => c1 ++= c2
```

which eventually calls:
PairRDDFunctions.combineByKeyWithClassTag

```java
val aggregator = new Aggregator[K, V, C](
      self.context.clean(createCombiner),
      self.context.clean(mergeValue),
      self.context.clean(mergeCombiners))
```

which in turn, boils down to:

```java
new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
```

----------------------------------------------
#### Reduce side

On the reduce side, when the DAG is actually materialized:

`ShuffledRDD#compute` is the actual RDD implm for a shuffled RDD.

This in turn uses `BlockStoreShuffleReader#read` which has the following important points:

1. Read the shuffle blocks from mappers (network via Netty or local): `ShuffleBlockFetcherIterator`
2. Deserialize: `recordIter`
3. Aggregate the (K,V)s by key:  `aggregatedIter`  `aggregator#combineValuesByKey`
    
    ```
    val combiners = new ExternalAppendOnlyMap[K, V, C]
    (createCombiner, mergeValue, mergeCombiners)
    ```
    
    ExternalAppendOnlyMap will accumulate in memory the (K,agg(V)) till spill limit is hit, then spills (keep a reference to an on-disk partial sorted iterator)
4. The final `resultIter` is actually provided by `ExternalSorter` which wraps the `aggregatedIter` and does the final merge sort of partial sorted results. See `ExternalSorter#merge`  
   For `groupBy` the `aggregator.isDefined=false`, only `ordering.isDefined=true`


## References:

[1] ExternalAppendOnlyMap:
https://github.com/JerryLead/SparkInternals/blob/master/markdown/english/4-shuffleDetails.md#externalappendonlymap




## repartitionAndSortWithPartition is ligher(?)

