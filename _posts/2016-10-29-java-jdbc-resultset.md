---
layout: post
category: java
date: 2016-10-29 02:30:55 UTC
title: Java基础之JDBC ResultSet
tags: [数据集，分批获取，缓冲，游标]
permalink: /java/jdbc/resultset
key: 8eaae8fb6e066834e417808115640553
description: 本文介绍了JDBC中ResultSet在Postgresql中的实现。
keywords: [数据集，分批获取，缓冲，游标]
---

对于JDBC中的结果集(ResultSet)，之前的对它理解是当次查询的结果集，它像一个游标一样，每次调用next的时候，就会将当前行的结果返回给我们。但是结果集会把所有的数据一次性拉倒内存中吗？如果不是，它又是如何实现分批获取的？调用**next()**时候背后又发生了什么？

这几年在开发过程中主要使用的是NoSQL，使用PostgreSQL(version 9.4)的时候太多用的是封装好的ORM，所以很少去深入了解这一块。今天在改动项目老代码的时候(新公司使用的是Oracle 11)，涉及到原生JDBC，进行的过程中碰到了点小问题，于是便趁机对于相关的知识点进行了整理。

事情的起因是一条查询(下表是实际场景中的简化版， <b style="color:red">所有的效率问题与当前环境有关，相应的结论仅供参考</b>)，`tbl_stu_teacher`(大概30几万行)记录的是学生和老师的对应关系，最终的汇总为某个学生被哪些老师教了，即输出`Map<StuId, List<TeacherId>>`。

<table style="margin-left: 26px">
    <tr>
        <th>stu_id</th>
        <th>teacher_id</th>
    </tr>
    <tr>
        <td>101</td>
        <td>201</td>
    </tr>
    <tr>
        <td>101</td>
        <td>202</td>
    </tr>
</table>


之前的做法是将全表数据拉出来(`select stu_id, teacher_id from tbl_stu_teacher`)，然后遍历，放到HashMap中。

改进方案是在数据库中先聚合(`select stu_id,  listagg("teacher_id", ',') within group (order by null) from tbl_stu_teacher`)， 改变ResultSet的**fetchSize**，遍历结果集，然后放到HashMap中(不用和上面一样，每次都要检查对应的key是否存在)。

改进之后大概快了7，8倍左右，开始以为是SQL优化的结果。 后来在Oracle中运行了上面两个查询，发现查询时间的差距很小。然后尝试<b style="color:red">调整初始方案的fetchSize</b>，发现速度也快了不少。所以，基本上可以认定提升的主要原因是**fetchSize的调整**。

###逻辑解释

结果集对应的是一次查询结果，可以理解为指向查询结果的游标(Cursor)，所以并没有将所有的查询结果加载到JVM内存。

对于结果的获取，我们可以"指挥"游标每次在查询结果上移动一次，就将结果加载到JVM内存，也可以"指挥"它每次移动100次，然后将对应的结果加载到JVM内存，有的数据库实现可能直接获取了所有查询结果，
假设查询命中100万数据，如果每次取1万，则需要100次; 如果每次取2万，则需要50次
这也就是**fetchSize**的意义所在。

每多取一次都会多一次网络传输，但也意味着每次取出的数据量会更大，占用更多的内存。但总的来说适当的增加**fetchSize**会提高效率，具体数字需要根据实际调试确定。

<b class="highlight">在结果集内部除了维护游标(Cursor)之外，还维护了如下几个重要变量:</b>

(1) 结果集中的索引 ---> **row_offset**，可以理解为游标在查询结果上的位移

(2) 每一次获取之后的ResultRow --> `List<byte[][]> rows`以及ResultRow中的索引`current_row(0-based)`

实际上每次从结果集中取出数据之后会放到ResultRow中，相当于当前批次的缓存。 <b style="color:red">当我们调用结果集的next方法，实际上是对于ResultRow进行遍历</b>。
而`current_row`则告诉我们，ResultRow中的数据已经被取出多少了。

如果已经取完`current_row + 1 >= rows.size()`，则此时，结果集可能被读取完，执行结束或者触发下一次游标移动，获取下一批次的数据。

(3) 当前行的副本 --> **byte[][] this_row**，这个变量除了用于**rs.getXXX**，还可以用于可更新的结果集。以**getInt**为例(它调用了getFastInt), 可以清楚的看到**this_row**这个变量的使用。

```java
// org.postgresql.jdbc.PgResultSet
private int getFastInt(int columnIndex) throws SQLException, NumberFormatException {
    byte[] bytes = this_row[columnIndex - 1];
    ...
}
```

当弄清楚这几个变量的含义之后，整个数据的获取就非常清晰了，一句话总结(具体的实现可以参见`org.postgresql.jdbc.PgResultSet`的**next**方法):

> 建立数据库连接之后，通过创建Statement，执行SQL语句，并创建结果集(ResultSet)，它指向的是查询结果。 其中涉及到两个层面的遍历，一个是结果集中的游标对于查询结果的遍历，一个是对于ResultRow中的数据遍历，也就是真正将Row数据转换成Java对象的过程。所以数据的加载，是通过多次分批加载到JVM内存的。

###代码层面

```java
//org.postgresql.Driver
Connection makeConnection();  

//org.postgresql.jdbc.PgConnection
Statement createStatement();

// org.postgresql.jdbc.PgStatement --
java.sql.ResultSet executeQuery(String p_sql);
java.sql.ResultSet executeWithFlags();
...
void executeInternal();
  // 其中handleResultRows用于处理返回的Row，并且放到ResultWrapper(ResultSet)中
  StatementResultHandler handler = new StatementResultHandler();
       // connection.getQueryExecutor() -> org.postgresql.core.v3.QueryExecutorImpl
       connection.getQueryExecutor().execute(...handler .. fetchSize ...);    
            void processResults(ResultHandler handler, int flags);   
                // tuples用于存放每次fetch的结果
                handler.handleResultRows(currentQuery, fields, tuples);
  result = firstUnclosedResult = handler.getResults();
    
Boolean ResultSet.next()
  // 触发下一次ResultRow获取(更新Cursor)
  connection.getQueryExecutor().fetch(cursor, new CursorResultHandler(), fetchRows);
  // 没有触发下一次获取，将当前的Row放入到byte[][]this_row中
  void initRowBuffer()

ResultSet.getXXX()
// 实际就是从this_row中获取当前Row的结果，然后由于next()的循环调用，所以将结果全部取出来了。
rs.getInt() | rs.getString()
```

在上面的调用栈中，针对`org.postgresql.core.v3.QueryExecutorImpl.processResults`:

不论是第一次执行查询，还是后续游标移动都会调用该方法。发出查询或者是取数的请求之后，会根据返回的状态码来进行相应的处理。

```java
protected void processResults(ResultHandler handler, int flags) throws IOException {
    ...
    List<byte[][]> tuples = null;
    
    while (!endQuery) { 
        c = pgStream.receiveChar();
        
        switch (c) { 
            case 'D': // Data Transfer (ongoing Execute response)
              if (!noResults) {
                if (tuples == null) {
                  tuples = new ArrayList<byte[][]>();
                }
                tuples.add(tuple);
              }
            case 'C': // Command Status (end of Execute)  
              // There was a resultset.
              handler.handleResultRows(currentQuery, fields, tuples, null);
              ...
        }
    }
    ...
}
```

从上面的代码片段可以看出，若**pgStream**中接收到的是'D'，则将查询数据转移到**tuples**中，若收到的是'C'，则通过`handleResultRows`(回调)将tuples转移到Result中的**rows**中，以便后面使用。


## 参考

\> [ResultSet中fetchSize的真实作用](http://stackoverflow.com/questions/1318354/what-does-statement-setfetchsizensize-method-really-do-in-sql-server-jdbc-driv)

\> [ResultSet的取数逻辑](http://stackoverflow.com/questions/858836/does-a-resultset-load-all-data-into-memory-or-only-when-requested)