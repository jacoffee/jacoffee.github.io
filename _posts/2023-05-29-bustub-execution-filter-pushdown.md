---
layout: post
category: database
date: 2023-05-29 09:29:03 UTC
title: 【BusTub】执行层优化之Filter Pushdown In Join
tags: [数据库执行层、Filter Pushdown]
permalink: /bustub/execution/filter-pushdown
key:
description: 本文介绍了在BusTub中实现filter-pushdown
keywords: [数据库执行层、Filter Pushdown]
---

在《【数据库综述系列】CMU15-445课程数据库BusTub综述》中我们对于BusTub进行了概述，本文我们将针对查询优化中的Filter Pushdown In Join进行展开讲解。

假设我们在基础的Join(就是INNER JOIN，以及LEFT & RIGHT JOIN)实现这个优化的话，需要考虑如下问题:

# 1. 实现思考

+ 满足什么样条件的**逻辑操作符(AND、OR)**才可以下推，比如说`left.columnA > 1 or right.columnB > 1` 这种可以下推嘛？
+ 满足什么样条件的比较表达式才可以下推，比如说`left.columnA > right.columnB` 这种可以下推嘛？
+ 下推的时候，如何确定查询列下推的表， 比如说`left.columnA > 1 and right.columnB > 1`中，如何确定columnA是left表, 而columnB是right表的呢？
+ 下推对于JOIN类型有没有要求， INNER JOIN、LEFT JOIN、RIGHT JOIN等？
+ 针对具体的实现而言，BusTub中是如何针对某个Plan进行动态调整的?

# 2. 具体实现

## 2.1 树形遍历

逻辑计划的优化本质上就是TreeNode的转换、移动，比如说Spark SQL中就基于Scala的模式匹配、偏函数结合树遍历(pre-order、post-order)的方式来巧妙的完成了这一过程。

而BusTub采用是也是类似的方式，只不过采用的至下向上(post-order 左-右-Root)的遍历方式，如果当前节点有Children先优化Children，最后再到自己。基本代码就类似于：

+ Spark中类似的方法

```scala
/**
   * Returns a copy of this node where `rule` has been recursively applied first to all of its
   * children and then itself (post-order).  When `rule` does not apply to a given node, it is left unchanged.   
   */
def transformUpWithPruning(cond: TreePatternBits => Boolean, ruleId: RuleId = UnknownRuleId)(rule: PartialFunction[BaseType, BaseType])
: BaseType = {
      
}
```

+ BusTub中的实现

```c++
auto Optimizer::OptimizeNLJAsHashJoinWithFilterPushDown(const AbstractPlanNodeRef &plan) -> AbstractPlanNodeRef {
    
    std::vector<AbstractPlanNodeRef> children;
    
    // 先优化Children
    for (const auto &child : plan->GetChildren()) {
      children.emplace_back(OptimizeNLJAsHashJoinWithFilterPushDown(child));
    }
    
    auto optimized_plan = plan->CloneWithChildren(std::move(children));
    
    // 最后来优化当前节点，这样可以整体返回新的Node 
}
```

## 2.2 节点定位以及转换

回到综述篇，简化之后的语法树

```bash
						filter (FilterPlanNode)
                        ||
						join (NestedLoopJoinPlanNode)

					/			\
t1 (SeqScanPlanNode)          t2 (SeqScanPlanNode)
```

优化后变成了

```bash
    					join (HashJoinPlanNode)

					/				       \
				   /	
t1 (SeqScanPlanNode)    				filter (FilterPlanNode)      
												\
											t2 (SeqScanPlanNode)
```



所以当我们检测到 **当前节点类型为Filter 且 它的唯一Child为 NestedLoopJoin**，就可以开始着手两个方向的优化:

+ NestedLoopJoinPlan的效率偏低，将其转化成HashJoin，但BusTub中本人实现的还是比较原始的，就是简单遍历左边，Build HashTable，然后去右表Probe

```c++
HashJoinPlanNode(
    // 联表之后输出列
    SchemaRef output_schema, 
    // 左表
    AbstractPlanNodeRef left,
    // 右表
    AbstractPlanNodeRef right,
    // 左联表键
    AbstractExpressionRef left_key_expression, 
    // 右联表键
    AbstractExpressionRef right_key_expression,
    // 联表类型 LEFT、RIGHT、INNER
    JoinType join_type
)、
```

+ 基于Filter中的筛选条件能否下推，进行下推，也就是构造一个新的Filter with Child(SeqScanPlanNode)

```c++
FilterPlanNode(SchemaRef output, AbstractExpressionRef predicate, AbstractPlanNodeRef child)
```

## 2.3 下推实现

首先回顾下，优化之后的逻辑计划。

```bash
 === PLANNER ===                                                                                                                             
 Projection { exprs=[#0.0, #0.1, #0.2, #0.3] } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)                                
   Filter { predicate=(#0.0=100) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)                                            
     NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)                    
       SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER)                                                                                
       SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)                                                                                           
 === OPTIMIZER ===                                                                                                                           
 HashJoin { type=Inner, left_key=#0.0, right_key=#1.0 } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)                      
   SeqScan { table=t1, filter=(#0.0=100) } | (t1.v1:INTEGER, t1.v2:INTEGER)                                                                   
   SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER) 
```

首先，我们需要明确下可以下推的条件

### 2.3.1 下推条件确认

+ 逻辑操作符为AND且至少有某 子比较表达式，只包含一个表的列

```bash
# 子比较表达式 各属于一个表
t1.v1 = 3 and t2.v3 > 4

# 子比较表达式 有一个属于一个表，这种就在只能下推部分，然后剩余部分继续留在原始Filter部分
t1.v1 = 3 and t1.v2 > t2.v4
```

这样的子表达式也有一个专门的术语: **conjunction**

我们把过滤条件按照最外层的AND拆分之后的单元叫做conjunction，比如 `((A & B) | C) & D & E` 就是由 `((A & B) | C)， D，E` 三个conjunction组成的。只所以这么定义是，conjunction是是否下推到存储的最小单元。
一个conjunction里面的条件要么都下推，要么都不下推

+ 逻辑操作符为OR且所有的子比较表达式 都属于一个表

```bash
# 子比较表达式 都属于左表
t1.v1 = 3 or t1.v2 > 4

# 子比较表达式 都属于右表
t2.v3 = 3 or t2.v4 > 4
```

### 2.3.2 下推条件拆分

这部分如果对于Spark SQL比较熟悉的，可以去参考`org.apache.spark.sql.catalyst.expressions.PredicateHelper#splitConjunctivePredicates`实现。当然大概率所有数据库都有这块相关的优化。

+ 根据AND表达式去拆分逻辑表达式 -- `std::vector<AbstractExpressionRef> subExpressions` 

```c++
auto SplitConjunctivePredicates(const AbstractExpressionRef &expr, std::vector<AbstractExpressionRef> &collector) -> void {
    if (
      const auto *logic_expr = dynamic_cast<const LogicExpression *>(expr.get());
      (logic_expr != nullptr && logic_expr->logic_type_ == LogicType::And)
    ) {
      SplitConjunctivePredicates(logic_expr->GetChildAt(0), collector);
      SplitConjunctivePredicates(logic_expr->GetChildAt(1), collector);
    } else {
      collector.emplace_back(expr);
    }
}
```

+ 遍历拆分的逻辑表达式集合`subExpressions`，针对每一个`subExpression`也就是如果
+ 类型是`ComparisonExpression`,  则收集它的Children中(**也就是ColumnValueExpression**)的col_index。如果所有的col_index都属于一个表，则视为单表条件；反之则视为涉及到两表的条件;  
当然也要注意Right Children是否也为ColumnValueExpression，如果是的话，那么不能下推到Join内部， 需要联表之后再次筛选
+ 类型是`LogicExpression`， 则说明是OR逻辑表达式，因为没有被拆分，则递归遍历收集Children中(**也就是ColumnValueExpression**)中的col_index，如果都满足一边表的，则可分配到一边表的筛选条件，反之则说明无法下推

上面逻辑大致实现

```c++
struct LeftRightCommonConditions {

  std::vector<AbstractExpressionRef> left;

  std::vector<AbstractExpressionRef> right;

  std::vector<AbstractExpressionRef> common;

};

auto Split(const AbstractExpressionRef &expr, size_t left_column_cnt, size_t right_column_cnt) -> LeftRightCommonConditions {
    std::vector<AbstractExpressionRef> collector{};
    
    SplitConjunctivePredicates(expr, collector);
    
    std::vector<AbstractExpressionRef> left_conditions{};
    std::vector<AbstractExpressionRef> right_conditions{};
    // left.A1 > right.A2
    std::vector<AbstractExpressionRef> common_conditions{};
    
    for (const auto &expression : collector) {
        // ! 判断是否为比较表达式 要是有模式匹配 就很方便了
        if (
           const auto *comp_expression = dynamic_cast<const ComparisonExpression *>(expression.get()); comp_expression != nullptr 	
        ) {
            if (
              const auto *column_value_expression = dynamic_cast<const ColumnValueExpression *>(comp_expression->GetChildAt(0).get());
              column_value_expression != nullptr
            ) {
             	bool right_expression_is_column =
		            dynamic_cast<const ColumnValueExpression *>(comp_expression->GetChildAt(1).get()) != nullptr;   
                
                auto col_idx = column_value_expression->GetColIdx();
                if (col_idx < left_column_cnt && !right_expression_is_column) {
                    //! 左边的条件
		            left_conditions.emplace_back(expression);
                } else if (
           			(col_idx >= left_column_cnt && col_idx < left_column_cnt + right_column_cnt) && !right_expression_is_column
		        ) {
                    //! note 改写column value，使得其在对应的表中，列处于正确的位置
                    right_conditions.emplace_back(
                      std::make_shared<ComparisonExpression>(
                        std::make_shared<ColumnValueExpression>(
                          1, col_idx - left_column_cnt, column_value_expression->GetReturnType()
                        ),
                        comp_expression->GetChildAt(1),
                        comp_expression->comp_type_
                      )
		            );
                } else {
                    common_conditions.emplace_back(expression);
                }
            }
        }
    }
    
    // 如果此时三部分条件都为空，说明第一部分是AND逻辑表达式，需要进一步拆分
    ...
}
```

### 2.3.3 优化逻辑计划看下推效果

+ 子查询本身没有filter，联表之后一个基础的AND，一个条件是一个表、一个条件是两个表都涉及到了

```sql
explain select * from t1 inner join t2 on v1 = v3 where v1 > 5 and v2 > v3;
```

```bash
=== PLANNER ===
Projection { exprs=[#0.0, #0.1, #0.2, #0.3] } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  Filter { predicate=((#0.0>5)and(#0.1>#0.2)) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
    NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
      SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER)
      SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
=== OPTIMIZER ===
Filter { predicate=(#0.1>#0.2) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  HashJoin { type=Inner, left_key=#0.0, right_key=#1.0 } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
    SeqScan { table=t1, filter=(#0.0>5) } | (t1.v1:INTEGER, t1.v2:INTEGER)
    SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
```

可以看到优化的执行计划中，一部分条件被下推到t1的SeqScan，另外一部分还是停留在Join之后的筛选条件

+ 子查询本身没有filter, 联表之后 一个基础的OR, 所有的列来自于同一个表

```sql
statement ok
explain select * from t1 inner join t2 on v1 = v3 where v2 = 10 or v1 > 5;
```

```bash
=== PLANNER ===
Projection { exprs=[#0.0, #0.1, #0.2, #0.3] } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  Filter { predicate=((#0.1=10)or(#0.0>5)) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
    NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
      SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER)
      SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
=== OPTIMIZER ===
HashJoin { type=Inner, left_key=#0.0, right_key=#1.0 } | (t1.v1:INTEGER, t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  SeqScan { table=t1, filter=((#0.1=10)or(#0.0>5)) } | (t1.v1:INTEGER, t1.v2:INTEGER)
  SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
```

虽然是OR，但是来源于同一个表，所以可以直接下推。


+ 子查询本身有filter, 联表之后 一个基础的AND, 所有的列来自于同一个表

```sql
statement ok
explain select * from (select * from t1 where v1 = 1000) t1 inner join t2 on v1 = v3 where v2 = 10 and v1 > 5;
```

```bash
=== PLANNER ===
Projection { exprs=[#0.0, #0.1, #0.2, #0.3] } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  Filter { predicate=((#0.1=10)and(#0.0>5)) } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
    NestedLoopJoin { type=Inner, predicate=(#0.0=#1.0) } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
      Projection { exprs=[#0.0, #0.1] } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER)
        Projection { exprs=[#0.0, #0.1] } | (t1.v1:INTEGER, t1.v2:INTEGER)
          Filter { predicate=(#0.0=1000) } | (t1.v1:INTEGER, t1.v2:INTEGER)
            SeqScan { table=t1 } | (t1.v1:INTEGER, t1.v2:INTEGER)
      SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
=== OPTIMIZER ===
HashJoin { type=Inner, left_key=#0.0, right_key=#1.0 } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER, t2.v3:INTEGER, t2.v4:INTEGER)
  SeqScan { table=t1, filter=((#0.0=1000)and((#0.1=10)and(#0.0>5))) } | (t1.t1.v1:INTEGER, t1.t1.v2:INTEGER)
  SeqScan { table=t2 } | (t2.v3:INTEGER, t2.v4:INTEGER)
```

可以看到除了下推，还进行了筛选条件合并，`filter=((#0.0=1000)and((#0.1=10)and(#0.0>5)))`。









# 3. 总结

至此基于BusTub，扩展优化规则之Join情况下的筛选条件下推，就基本梳理清楚了。当然还是偏学习型的，生产级别的数据库这块肯定比这里复杂更大，但对于理解基本原理足以。


# 参考

\> Spark SQL 

\> [aliyun 一文读懂AnalyticDB MySQL过滤条件智能下推原理](https://developer.aliyun.com/article/1183032)