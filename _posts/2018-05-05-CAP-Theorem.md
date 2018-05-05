---
layout:     post
title:      A brief introduction to CAP 
discription: 本文通过一个具体的例子来解释什么是CAP。
date:       2018-05-05 18:09:00
categories: 高并发
tags:       CAP 高并发
---

* 文章目录
{:toc}

### Introduction

> CAP is the theorem foundation of distribution system. It is impossible for a web service to provide the three following guarantees : Consistency, Availability and Partition-tolerance.




#### Description

**Consistency**

If system response success for a write request, then afterward requests have to read the new data; otherwise, all read requests can not read the new data. Data is strong consistency for caller.


**Availability**

All request(read/write) must receive response in a while, it can be terminated but not be waiting.


**Partition-tolerance**

The separated node can serve well in network partition.

 
#### Results
If system satisfy AP and the separated node still work will lead to inconsistency, which not satisfy C. 

But if system satisfy CP,  in order to satisfy C in network partition, the request have to be waiting, thus not satisfy A.

And If system satisfy CA, in order to ready node consistency in a period, the system demand no network partition, so, can not satisfy P.

 

So, we can not satisfy CAP both in a distribution system(particularly distributed storage system), we can ready two of CAP at least.

In the engineer practice, we will guarantee  AP and eventual consistency. The common practice is called asynchronous replication.