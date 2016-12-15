---
layout: post
category: spark
date: 2016-12-14 07:51:22 UTC
title: Spark基础之函数在分区上的执行一点解释
tags: [可复用线程池，一分区一函数一线程，一核一任务]
permalink: /spark/func-on-execution
key: c68b5153ef0a0b926c8d10b2260bf184
description: "本文阐述了函数在分区上执行时，线程数的一个小问题"
keywords: [可复用线程池，一分区一函数一线程，一核一任务]
---

在使用分区进行数据库操作的过程中，我们通常会使用foreachPartition来减小数据库连接的建立，因为我们定义的函数是对于整个**Iterator**执行的。但是在实现一个批量插入数据的函数时让我产生了一个疑问: <b style="color">同一个分区中，多少线程在同时调用这个函数？</b>

下面这个方法的逻辑很简单，利用PreparedStatement的addBatch来合并插入语句的执行，因此需要定义一个计数器，来统计每一批次的语句数，但是又担心计数器被同时修改，所以产生了上面的疑问。

```java
public abstract class HikariBatchInsertFunc implements VoidFunction<...> {
    ....
    
    public abstract void invoke(String[] record, PreparedStatement ps) throws Exception;
    
    @Override
    public void call(Iterator<String> lineIterator) throws Exception {
        // 获取当前执行任务的相关信息
        TaskContext ctx = TaskContext.get();
        Integer stageId = ctx.stageId();
        Integer partId = ctx.partitionId();
        String hostName = ctx.taskMetrics().hostname();
        System.out.println(stageId + "  " + hostName +  "  " + partId + "  " + Thread.currentThread().getName());

        AtomicInteger counter = new AtomicInteger(1);
        try (
            Connection conn = dbInstance.getConnection();
            PreparedStatement ps = conn.prepareStatement(insertSQL)
        ) {

            while (lineIterator.hasNext()) {
                String[] record = lineIterator.next().split(DELIMITER);
                invoke(record, ps);
                ps.addBatch();
                if (counter.intValue() >= MAX_BATCH) {
                    ps.executeBatch();
                    counter.set(0);
                } else {
                    counter.incrementAndGet();
                }
            }

            ps.executeBatch();
            counter.set(0);
        }        
    }
}
```

由于我们可以在分区函数的执行过程中获取相应的上下文信息，因此可以打印出Stage，PartitionId以及执行Executor的hostname。

```bash
// 部分输出
12  server106  12  Executor task launch worker-1
12  server106  5  Executor task launch worker-0

12  server106  31  Executor task launch worker-1
12  server106  30  Executor task launch worker-0

12  server106  36  Executor task launch worker-1
12  server106  40  Executor task launch worker-0
12  server106  42  Executor task launch worker-1
12  server106  46  Executor task launch worker-0
......
```

另外SPARK UI中Stage页面的Event Timeline也可以提供很多信息(**executor_cores="2"**)。

![Spark UI Event Timeline](http://static.zybuluo.com/jacoffee/xbh2pk47m8rw73jipb5vftx8/image_1b3u7da1k1ike10snds51ll2dq99.png)

基于上述两方面的信息，基本上可以确定: <b class="highlight">对于同一Stage的同一分区，我们定义的函数同一时间只会被一个线程执行，并没有并发的问题，所以AtomicInteger可以去掉</b>。

**(1)** 每一个Excecutor对应一个JVM进程，会维护一个线程池(threadpool)。任务提交后，会形成下面的调用，`launchTask ==> threadPool.execute(TaskRunner)`。也就是向线程池提交了一个任务，对于分区中的元素调用相应的操作，比如说上面的`call`。接下来，线程池会分配一个线程去执行这个任务。

**(2)** 这样看来线程池的利用率似乎不是很高。因此，在Spark中提供了另外一个参数`executor-cores`(对应同时在Executor上运行的Task数)，它们可以共享同一个线程池。
上面的案例中，一个Executor上配置了两个CPU Core，可以同时执行两个任务，因此Event Timeline上同一时间出现了两个绿条，控制行输出中work index始终停留在了0和1。

关于0和1的出现其实还涉及到Executor线程池的实现，`org.apache.spark.util.ThreadUtils.newDaemonCachedThreadPool`，它会根据需要来创建线程，并且复用之前的线程。由于同时最多有两个任务运行，所以只创建了两个线程。后续的Task提交上来的时候，之前的任务已经完成所以会复用之前的线程。