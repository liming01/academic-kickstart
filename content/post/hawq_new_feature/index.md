---
date: "2016-04-20T11:00:00"
draft: false
tags: ["hawq", "impala"]
title: "强者更强：HAWQ性能的思考"
summary: 本文是对比Impala的最新版本发布的性能特性，并总结了HAWQ自身的实现现状，列出一些HAWQ可以实现的性能方面的特性。
math: false
---

------ 作者：李明(email: mli@apache.org)

SQL on Hadoop产品非常繁多，从性能角度来看，比较好的有HAWQ、Impala。虽然目前HAWQ还是大幅领先于Impala（参见：[pivotal公布的hawq性能测试](https://blog.pivotal.io/big-data-pivotal/products/performance-benchmark-pivotal-hawq-beats-impala-apache-hive-part-1))，但是从最近的Impala的[新版本发布公告](https://www.cloudera.com/documentation/enterprise/release-notes/topics/impala_new_features.html#new_features_250)来看，它最近一直集中精力实现各种新的优化手段。为了使HAWQ继续保持竞争力，我们有必要学习其他竞争对手的优点，才能做到“知彼知己者，百战不殆”。

{{% alert note %}}
本文的观点仅代表本人观点，用于学习探讨交流。若因此出现经济、法律等方面的，与本人无关。
{{% /alert %}}

## HAWQ TO DO LIST

- Data skipping for I/O bound query: 
	- parquet/orc index( min/max/bloom filter) usage  
	- Runtime filter
	- Dynamic partition pruning
	
- Intra-Operator parallel processing vs vSeg number: 
	- IO intensive operator: base table scan: thread number = disk rotationar number 
	- CPU intensive operator: joins/aggregations/sort/top-N, thread number = cpu core number
	
- LLVM Codegen for CPU bound query:
	- Function call unrolling and branch pruning: Expression/loop/no switching 
	- CPU intensive operator: joins/aggregations/sort/top-N
	- more hotspots: counting the elements of a complex column, Checking for overflow in DECIMAL multiplication, ...

- Use HDFS caching feature to "pin" entire tables or individual partitions in memory, to speed up queries on frequently accessed data and reduce the CPU overhead of memory-to-memory copying.
- Duplicate small table to all segments to speed up join with other distributed tables.

- streaming pre-aggregation: decides at run time whether it is more efficient to do an initial aggregation phase and pass along a smaller set of intermediate data, or to pass raw intermediate data back to next phase of query processing to be aggregated there. 

- Optimization for small queries let Impala process queries that process very few rows without the unnecessary overhead of parallelizing and generating native code. 

- more parallel processing: SIMD, vectorization 
- Distributed by hash is one kind of partition?	
- better info for different level: explain/SUMMARY/profile
