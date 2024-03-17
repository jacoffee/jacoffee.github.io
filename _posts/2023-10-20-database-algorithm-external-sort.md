---
layout: post
category: database
date: 2023-10-20 08:45:03 UTC
title: 【数据库算法实践】基于C++简化实现支持外部排序的聚合操作
tags: [数据库算法、外部排序、Vocalno Model、Physical Plan、AggregationExecutor、Spark SQL]
permalink: /database/algorithm/external-sort
key:
description: 本文基于C++简化实现了数据库聚合的时候的外部排序
keywords: [数据库算法、外部排序、Vocalno Model、Physical Plan、AggregationExecutorr、Spark SQL]
---

# 1. 背景概述

## 1.1 数据库聚合操作的实现

常见的group by语句的实现，一般有如下两种思路：

### 1.1.1 HashBasedAggregation

+ 基于group by key构建内存HashTable

一般会hash分桶，将key哈希相同的分配到相同的bucket，为并行计算做好准备

+ 在bucket内部遍历分组key对应的数据，进行相应的聚合计算

比如说计算sum(column_name)操作，就需要累计key内部对应的column_name对应的值。当然这一步还需要区分key是否相同，因为hash相同的key，可能值是不同的。

这是比较理想情况的，如果**出现内存不足**，一般都会切换到第二种方式。

### 1.1.2 SortBasedAggregation

针对下面的示例数据，如果按照group key排好序了，那么我们通过读取行，发现group key发生变化了，就说明切换到了新的组。这样之前为group key=A累计的中间状态就可以进行计算。HashBasedAggregation之所以会导致内存溢出，一方面是group key太多，另外一方面是因为要同时存在的大量的中间存储，比如说A和B都要存在，而SortBasedAggregation可以边读边进行计算。

```bash
group key    value
A				1
A				2
A 				3
B				4
B				3
B				5
```

但是默认情况下，就个人所知，几乎所有的数据库默认都采用的是HashBasedAggregation，只不过各有各的优化。这个原因也很简单，因为对于大数据量的排序也是一个很重的操作。

## 1.2 外部排序(external sort)

上面提到的SortBasedAggregation显然会使用到某种排序算法，由于是因为数据量大而导致的，所以一般会采用外部排序。它实际上就是归并排序，然后还涉及到多路归并。

关于外部排序的实现网上的资料有很多，这里就不再赘述。

> External sorting is a technique used to sort data sets that **are too large to fit in main memory**. In this technique, the data set is divided into small blocks, and each block is sorted individually using merge sort. The sorted blocks are then merged to form the final sorted data set. External sorting is commonly used in database systems and file systems。

## 1.3 Spark SQL中实现概述

+ 当内存中HashMap不能再分配，则按照group key排序内存map的内容，落盘，形成ExternalSorter1，并维护相应的标识
+ 第一步中内存已经释放，可以继续存放聚合数据，直到再次出现内存不足，排序落盘，形成ExternalSorter2

....

+ 最终如果元素遍历完，如果检测到之前已经落过盘，则再次刷写，形成ExternalSorterN

之前的步骤形成了很多 ExternalSorter,  底层对应了很多sorted file，接下来就需要进入 N-way Merge Sort阶段(`switchToSortBasedAggregation()`)。

Spark SQL这个实现主要在`TungstenAggregationIterator.scala`中实现，虽然内部的注释非常详细，但由于各种封装，所以看起来还是很费劲的。本着纸上得来终觉浅，绝知此事要躬行的原则，加上在Bustub项目中实现过AggregationExecutor。

所以计划本篇先基于C++简化实现支持外部排序的聚合操作，然后再在Bustub原有的AggregationExecutor扩展支持SortBasedAggregation，期望通过这个过程来加深自己对于这两种方案融合的理解。

# 2. 实现思路

本着在弄清楚核心思想的情况下同时有一定的编码，所以会尽量简化各个流程。

## 2.1 场景引入

以下面的的学生Model为例，针对多个同学科目等分记录，按照subject分组，统计该门功课的平均值。

```c++
struct Student {
  string subject_;
  string name_;
  int score_;

  // note 操作符重载，以便落盘的时候，排序目前按照subject排序
  auto operator<(const Student& a) const -> bool { return subject_ < a.subject_; }

  auto toJson() const -> json {
    json j;
    j["subject"] = subject_;
    j["name"] = name_;
    j["score"] = score_;
    return j;
  }

  auto toJsonBySubject() const -> json {
    json ser;
    ser["key"] = subject_;
    ser["value"] = toJson();
    return ser;
  }

  /**
  	 用于反序列化
  */
  static void fromJsonLine(const std::string & line, Student& s) {
    auto j3 = json::parse(line);
    auto j = j3["value"].get<json>();
    j.at("name").get_to(s.name_);
    j.at("score").get_to(s.score_);
    j.at("subject").get_to(s.subject_);
  }

}


auto initializeStudents() -> std::vector<Student> {
  Student student1 = {"D", "s1", 90};
  Student student2 = {"B", "s2", 65};
  Student student3 = {"C", "s3", 73};
  Student student4 = {"E", "s4", 70};

  Student student5 = {"E", "s5", 70};
  Student student6 = {"A", "s6", 79};
  Student student7 = {"C", "s7", 100};
  Student student8 = {"B", "s8", 90};

  Student student9 = {"C", "s9", 54};
  Student student10 = {"E", "s10", 65};
  Student student11 = {"C", "s11", 76};
  Student student12 = {"A", "s12", 90};

  std::vector<Student> students = std::vector<Student>();
  students.push_back(student1);
  students.push_back(student2);
  students.push_back(student3);
  students.push_back(student4);
  students.push_back(student5);
  students.push_back(student6);
  students.push_back(student7);
  students.push_back(student8);
  students.push_back(student9);
  students.push_back(student10);
  students.push_back(student11);
  students.push_back(student12);
  return students;
}
```

## 2.2 关于聚合状态和落盘临界点

+ 聚合状态

这个本质上是维护的key - value对，而key一般是分组键，value一般是聚合值，所以可以分别使用Struct维护。

为了简化起见，key直接采用**string类型的subject**，value则维护整个**Student Struct**。聚合操作在ExtenalMergeSort完成之后，顺序读取的时候基于指定的aggregation逻辑去实现。

+ 落盘临界点

由于模拟超内存，不太方便实现，所以考虑内存中写入的键值对，**每超过N个，就触发一次落盘**，形成一个sorted file。结束之后，针对这个N个sorted file进行合并。


## 2.3 关于序列化和反序列化

序列化实际上就是需要将上面提到的聚合状态写入到磁盘，C++中的序列化有很多种选择，但是为了实现方便决定选择序列化成json，依赖选择为[nlohmann_json JSON for Modern C++](https://github.com/nlohmann/json?tab=readme-ov-file#serialization--deserialization)，是一个非常短小精悍，但是使用起来很方便的json依赖。

然后借助C++中基础的IO `std::ifstream`和`std::ofstream` 实现文件的读写。

## 2.4 关于多路归并的实现

```bash
B			A			A		
C			B			C
D			C			E
E           E
```

假设上面三组排好序的数据，需要按照升序从小到大排列，实现从逻辑上就讲就是: 

> 依次取这三堆数据的顶部元素，然后取最小的元素，接着移除，然后继续取头部元素比较，直到遍历完成所有的数据。

联想到取一组数据里面最小的元素，自然而然的会想到最小堆，C++中当然有相应的实现[std::priority_queue](https://en.cppreference.com/w/cpp/container/priority_queue)。

另外由于需要不断的比较首元素并且改变，所以需要借助系统`string.h`提供的getLine同时维护一个变量存储最近读出来的行，参见下面的`IOStringStack.h` &&  `IOStringStack.cpp`。

# 3. 核心代码梳理

## 3.1 优先队列中的元素实现类

因为是模拟栈这个数据结构的行为，所以命名上体现了这一点

+ `IOStringStack.h`

```c++
#include <fstream>
#include <string>

class IOStringStack {
public:
  explicit IOStringStack(std::string file_name);

  ~IOStringStack() = default;

  void close();

  bool empty();

  /**
   * 取出顶部行
   * @return
   */
  std::string pop();

  /**
  * 查看顶部行
  */
  std::string * peek();

private:
  std::ifstream fin_;
  std::string head_line_;
};

```

+ IOStringStack.cpp

```c++
#include <iostream>
#include <fstream>

using std::ios;
using std::ifstream;

IOStringStack::IOStringStack(std::string file_name) {
  ios::sync_with_stdio(false);
  fin_ = ifstream(file_name);
 getline(fin_, head_line_);
}

void IOStringStack::close() {
  fin_.close();
}

const bool IOStringStack::empty() {
  return head_line_.empty();
}

std::string IOStringStack::pop() {
  std::string result_line = head_line_;

  std::string line;
  getline(fin_, line);
  head_line_ = line;

  return result_line;
}

const std::string * IOStringStack::peek() {
  return &head_line_;
}
```



## 3.2 优先队列的比较函数实现

`std::priority_queue` 默认实现的是最大堆，也就是存放进去之后每次取出的是最大值，不过我们可以通过改变 比较函数方向来 变向实现最小堆，也就是每次最小的先出去， 考虑到队列的特定也就是最小的放在队头，最大的在队尾。 

> The [priority queue](https://en.wikipedia.org/wiki/Queue_(abstract_data_type)) is a [container adaptor](https://en.cppreference.com/w/cpp/container#Container_adaptors) that provides constant time lookup of the largest (by default) element, at the expense of logarithmic insertion and extraction.
>
> A user-provided `Compare` can be supplied to change the ordering, e.g. using [std::greater](http://en.cppreference.com/w/cpp/utility/functional/greater)<T> would cause the smallest element to appear as the [top()](https://en.cppreference.com/w/cpp/container/priority_queue/top).

```c++
using ComparingFunction = std::function<bool(std::shared_ptr<IOStringStack>, std::shared_ptr<IOStringStack>)>;
using IOStringStackQueue = std::priority_queue<std::shared_ptr<IOStringStack>, std::vector<std::shared_ptr<IOStringStack>>, ComparingFunction>;


// 使用priority_queue 这个构造器函数 初始化priority_queue, 从源码中也可以看到 默认使用的是less
template <class _Tp, class _Container = vector<_Tp>,
          class _Compare = less<typename _Container::value_type> >
class _LIBCPP_TEMPLATE_VIS priority_queue {
    
    explicit priority_queue(const value_compare& __comp)
        : c(), comp(__comp) {}
}

auto custom_comparator = [](std::shared_ptr<IOStringStack> prev, std::shared_ptr<IOStringStack> next) {
    auto first_original_key = *(prev->peek());
    auto second_original_key = *(next->peek());
    auto first_key = first_original_key.empty() ? first_original_key : json::parse(first_original_key)["key"].get<std::string>();
    auto second_key = second_original_key.empty() ? second_original_key : json::parse(second_original_key)["key"].get<std::string>();
    return first_key.compare(second_key) > 0;
};


IOStringStackQueue queue(custom_comparator);
```

## 3.3 N-way Merge-Sort实现

IOStringStack作为std::priority_queue中的基本元素，每次基于特定比较规则，然后最小的会放在顶部，直接取出来，然后取head_line。之后再次放回去，则会基于新的比较规则再次将**最小的IOStringStack**放在队头，如此反复，直到取出的IOStringStack为空，这时候开始下一个IOStringStack处理，最终完成全部排序。

```c++
auto flushSortedBuffer(std::vector<string> out_buffer, const string & path, const bool should_close) -> void {
  std::cout << "Flush "<< out_buffer.size() << " data " << '\n';
  std::ofstream merged_sort_file;

  // append mode
  merged_sort_file.open(path, std::ios_base::app);
  for (const auto &n : out_buffer) {
    std::cout << n << '\n';
    merged_sort_file << n << '\n';
  }
  if (should_close) {
    merged_sort_file.close();
  }
}

auto externalMergeSort(IOStringStackQueue & queue) -> std::string {
  std::vector<string> out_buffer;
  std::cout << queue.size() << '\n';
  while (!queue.empty()) {
    // 取出队列头元素  
    std::shared_ptr<IOStringStack> io_string_stack = queue.top();

    std::string result = io_string_stack->pop();
    if (!result.empty()) {
      out_buffer.emplace_back(result);

      //!  大于一定阈值然后落盘, 而不是等排序完成之后再进行落盘
      if (out_buffer.size() >= FLUSH_BUFFER_THRESHOLD) {
        flushSortedBuffer(out_buffer, output_path, false);
        out_buffer.clear();
      }
    }
    queue.pop();
    if (!io_string_stack->empty()) {
      queue.push(io_string_stack);
    } else {
      io_string_stack->close();
    }
  }

  if (!out_buffer.empty()) {
    flushSortedBuffer(out_buffer, output_path, true);
    out_buffer.clear();
  }


  return output_path;
}
```

## 3.4 顺序读取SortedFile，然后基于指定聚合函数逻辑进行计算

由于已经按照Group key排好了序，依次读取，group key发生变化则切换为一个组，知道所有文件读取完，整个计算完成。

```c++
void calculateDiskFileAvg(const std::string & output_path) {
  std::cout << "calculateDiskFileAvg" << '\n';

  std::ifstream fin(output_path);
  std::string line;

  std::string currentGroup;
  size_t scoreSum = 0;
  size_t studentCount = 0;
  while (getline(fin, line)) {
    if (line.empty()) {
      break;
    }

    Student s1;
    Student::fromJsonLine(line, s1);
    if (currentGroup.empty()) {
      currentGroup = s1.subject_;
    } else if (s1.subject_ != currentGroup) {
      // 前一组的数据已经全部输出完毕，准备开始统计
      std::cout << "Invoke aggregation function avg for group: [" << currentGroup << "] and result is: "<< (scoreSum / studentCount) << '\n';
      // 然后清空累计值
      scoreSum = 0;
      studentCount = 0;
      currentGroup = s1.subject_;
    }

    // 组内元素处理
    studentCount += 1;
    scoreSum += s1.score_;
  }

  if (
    !currentGroup.empty() && scoreSum > 0 && studentCount > 0
  ) {
    std::cout << "Invoke aggregation function avg for final group: [" << currentGroup << "] and result is: "<< (scoreSum / studentCount) << '\n';
  }

  fin.close();
}
```



# 4. 参考

\> Spark SQL 

\> [Merge Sort Algorithm: A Comprehensive Guide for Sorting Large Data Sets](https://cyberw1ng.medium.com/merge-sort-algorithm-a-comprehensive-guide-for-sorting-large-data-sets-70ad48282a3a)
