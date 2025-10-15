+++
date = '2025-10-15T22:23:13+08:00'
draft = false
title = 'MINIOB算子实现流程'
+++
# 0. 脑子里要有的总地图（一次过）

SQL 的执行在 miniob 里基本是这几步（你也已经知道的那条线）：

Parser

词法/语法把 SQL 字符串变成解析树（ParsedSqlNode，比如 SelectSqlNode）。

Resolver（绑定/类型解析）

把未绑定的表达式（比如 UnboundFieldExpr、UnboundAggregateExpr）变成可执行的表达式（FieldExpr、AggregateExpr），决定类型、长度，必要时做 cast。

Logical Plan（逻辑算子）

基于 SelectSqlNode 搭建逻辑计划：扫描、谓词、投影、聚合、分组、连接等。这里只决定“做什么”，还没决定“怎么做/用什么算法”。

Physical Plan（物理算子）

选择具体实现（比如 GROUP_BY_VEC、AGGREGATE_VEC、HASH_JOIN…），实现 open/next/close 的迭代器接口。

miniob 的物理算子是向量化风格：next(Chunk &chunk) 每次吐一批数据列（Column），而不是一行行吐。

执行（Executor）

从根算子开始 open()，不断 next() 直到 RECORD_EOF，最后 close()。

接下来每一步用已经改的代码把“做了什么”和“为什么这么做”讲清楚。

# 1. Parser：让 SQL 能“写出来并被理解”
## 1.1 语法：在 yacc_sql.y 支持聚合与 GROUP BY
已经有：

aggregate_expression：

```
aggregate_expression:
    ID LBRACE expression RBRACE {
      $$ = create_aggregate_expression($1, $3, sql_string, &@$);
    }
  | ID LBRACE '*' RBRACE {
      $$ = create_aggregate_expression($1, new StarExpr(), sql_string, &@$);
    }
  ;
```
这让 COUNT(*) / SUM(col) / AVG(col) 等进入解析树里，先是 Unbound（UnboundAggregateExpr），类型还没最终确认（语义阶段做）。
group_by：
你可以用“语义阶段禁 *”或“语法直接禁 *”两种方案之一。我们给了“语法直接禁 *”的版本（group_by_expression 不包含 '*' 和 aggregate_expression），这样 GROUP BY 列就不会接受 * 或聚合。
## 1.2 解析树：ParsedSqlNode / SelectSqlNode
在 parse_defs.h 你看到：
```
struct SelectSqlNode {
  vector<unique_ptr<Expression>> expressions;  // SELECT 列（表达式）
  vector<string>                 relations;    // FROM 表
  vector<ConditionSqlNode>       conditions;   // WHERE
  vector<unique_ptr<Expression>> group_by;     // GROUP BY
};
```

这一步的职责是：把 SQL 文本“抄写”成一棵结构化的语法树，表达式是 Expression* 的抽象（此时可能是 UnboundFieldExpr、UnboundAggregateExpr、ArithmeticExpr…）。

重点：Parser 阶段不检查业务规则（比如 SUM(*) 是否允许），它的目标是“尽量把语法树搭起来”；真正的限制交给 resolver/语义分析。
# 2. Resolver：把“未绑定表达式”变成“可执行表达式”
你给的 expression.h 里，有一对“未绑定/已绑定”的类型：

未绑定：UnboundFieldExpr、UnboundAggregateExpr

已绑定：FieldExpr（持有 Field，知道 table/column 与类型长度）、AggregateExpr（知道聚合类型/返回类型，比如 COUNT -> INTS、AVG -> FLOATS）

Resolver 的典型工作（伪代码示意）：
```
Expression* bind(Expression *e, const FromInfo &from) {
  switch (e->type()) {
    case ExprType::UNBOUND_FIELD: {
      auto *u = static_cast<UnboundFieldExpr*>(e);
      const Field &f = resolve_field(u->table_name(), u->field_name(), from);
      return new FieldExpr(f);  // 绑定字段 → 类型/长度明确
    }
    case ExprType::UNBOUND_AGGREGATION: {
      auto *u = static_cast<UnboundAggregateExpr*>(e);
      AggregateExpr::Type t = parse_agg_name(u->aggregate_name()); // COUNT/SUM/...
      unique_ptr<Expression> child(bind(u->child().get(), from));
      // 可在这儿检查 SUM(*) 合不合规，COUNT(*) 的 child 是否 StarExpr 等
      return new AggregateExpr(t, std::move(child));
    }
    case ExprType::ARITHMETIC: { ... 递归 bind 左右子树，做类型提升/CAST ... }
    ...
  }
}
```
为什么要绑定？

执行时必须知道每个表达式的值类型（value_type()）、长度（value_length()）。你看 AggregateExpr::value_type()：COUNT->INTS，AVG->FLOATS，其余跟 child 一样。

绑定还会把 Unbound* 换成能 get_value/ get_column 的表达式。比如 FieldExpr::get_column 会直接从 Chunk 里拎列；AggregateExpr 在向量化管线里通常由专门算子计算。

补充：Expression::pos() 的含义

某些表达式会在下层算子中就计算好，并把结果作为一列放进 Chunk；这时上层拿到表达式时可以检查 pos()，如果 >=0，说明在输入 Chunk 的那一列已经有结果，无需重复算。

聚合常常是上层算子算的（例如 GROUP_BY_VEC/AGGREGATE_VEC），所以它们的 pos() 由上层产出后再设置。
#   3. Logical Plan：决定“做什么”
基于 SelectSqlNode，逻辑计划会识别：

没有 GROUP BY 但有聚合 → “全局聚合”

有 GROUP BY → “分组聚合”

没有聚合 → 普通投影/筛选/连接

逻辑节点不关心算法，只关心语义。例如它会有 “GroupBy( keys=[…], aggs=[…] )” 这样的抽象节点。
# 4. Physical Plan：决定“怎么做 & 用哪个算子”
miniob 已经有两条路：

全局聚合（无 GROUP BY）：AggregateVecPhysicalOperator
你贴的 aggregate_vec_physical_operator.cpp 就是它——把所有输入当成一个组，聚合成一行输出。

分组聚合（GROUP BY）：GroupByVecPhysicalOperator
我给你的 .h/.cpp 实现，就是哈希分组 + 分组内聚合，输出多行（每组一行）。
## 4.1 物理算子的通用接口

每个算子都要实现：
```
RC open(Trx *trx);
RC next(Chunk &chunk);  // 向量化：一次吐一批列
RC close();
```

执行时框架会：
```
root->open(trx);
while (root->next(output_chunk) == RC::SUCCESS) { ...消费... }
root->close();
```
## 4.2 全局聚合（你现成那份）
AggregateVecPhysicalOperator 的关键点：

构造里：为每个聚合创建 state_ptr（create_aggregate_state），准备输出列

open()：循环 child->next(chunk_)，用 value_expressions_[i]->get_column(chunk_, col) 得到每个聚合的输入列，把整列喂给 aggregate_state_update_by_column(state, type, child_type, col) 做批量累积

next()：把每个 state finalize 到 output_chunk_，只吐一次（一行），下次就是 RECORD_EOF

这就是“全局聚合只有一个组”的版本。
## 4.3 分组聚合（我们新写的）
GroupByVecPhysicalOperator 的职责：

open()：

一直从 child 拉批：

计算分组键列 → groups_chunk

计算聚合输入列（每个聚合的 child）→ aggrs_chunk

hash_table_->add_chunk(groups_chunk, aggrs_chunk)

构建扫描器（AggregateHashTable::Scanner）用于按组吐结果

next()：

scanner_->next(chunk)：把若干分组的“分组键 + 聚合结果”写到一个 chunk 里

close()：

关闭 child，销毁扫描器/哈希表（state 也被释放）

你已经有 AggregateHashTable 的抽象和一个标准实现 StandardAggregateHashTable，我们只需要把 add_chunk 和 Scanner 的输出写完整，它会管理 “key -> state[]” 这件事。
# 5. 聚合哈希表：把“多行多组”累积成“每组一行”的结果
你给的头文件已经定义好了接口：
```
class AggregateHashTable {
public:
  virtual RC add_chunk(Chunk &groups_chunk, Chunk &aggrs_chunk) = 0;
  class Scanner { virtual RC next(Chunk &chunk) = 0; ... };
  vector<AggregateExpr::Type> aggr_types_;
  vector<AttrType>            aggr_child_types_;
};
```

StandardAggregateHashTable 的核心是：
```
unordered_map<vector<Value>, vector<void *>> aggr_values_;
```

key = 该行的分组键（按列取第 i 行得到 Value，合成 vector<Value>）

value = 这组的聚合 state 指针数组（每个聚合一个）
## 5.1 add_chunk（把一批 rows 落进哈希表）
思路：

对 groups_chunk 和 aggrs_chunk，先取行数 n（一般两者相等等于子算子吐出的批大小）

for i in [0..n)：

组装 key：对每个 group-by 列取第 i 个 Value → 拼成 vector<Value>

查哈希表：

首次遇到该 key：vector<void*> states(aggr_types_.size())，对每个聚合 create_aggregate_state(aggr_types_[k], aggr_child_types_[k])

然后把第 i 行对应聚合输入值更新到 state：

简单可靠的做法：把该输入列第 i 个元素做成“单元素视图”的 Column（或者直接构造 1 个元素的 Column），调用 aggregate_state_update_by_column(states[k], ...)

也可以直接提供 “按单值更新”的 API（如果你们有的话）

实现要点：

Value 的相等与哈希要正确，VectorHash/VectorEqual 你们已经写了

state 的生命周期由哈希表析构统一 free（你们的析构里已经 free(state) 了）
## 5.2 Scanner::next（把分组结果吐成 Chunk）
思路：

维护一个迭代器（begin → end）

每次 next(chunk)：

清空并准备输出列：先把分组键列也建上（通常 SELECT 里要用到），再加聚合列

迭代若干组（你可以一次性全吐，也可以控制批大小）：

把 key 的每个 Value 写到对应键列的尾部一个单元

对每个聚合 finalize 把结果写到对应聚合列的尾部一个单元

移动迭代器；全部吐完时返回 RECORD_EOF

这和我们在 GroupByVecPhysicalOperator::next 里使用扫描器的契约一致。
#   你写/改这些代码时的心智模型
表达式（Expression）：谁来算、在哪一层算？

字段/算术/常量可以在子算子就算成列（get_column）；

聚合必须在聚合算子里算（AGGREGATE_VEC 或 GROUP_BY_VEC），不指望子算子提供它（除非上游做了预聚合）。

Chunk/Column 的流动：

每个算子接入一个或多个 child；

next(child_chunk) → 评估自己负责的表达式 → 产出新的列或引用已有列 → chunk.reference(...) 或 chunk.add_column(...)；

只要你理解“列是连续内存的一段数组，Chunk 是多列的并排集合”，就很容易写对。

state 的生命周期：

聚合 state 创建在 open/build 阶段，更新在读入批时，finalize 在吐结果的时候，最后在 close/析构回收。

全局聚合是一组 state；分组聚合是“很多组，每组有一组 state”。
# 7. 怎么验证自己没写歪（最小自测集）
解析 + 绑定能过：

SELECT COUNT(*) FROM t;

SELECT dept, AVG(salary) FROM emp GROUP BY dept;

SELECT DATE(ts), SUM(x) FROM tab GROUP BY DATE(ts);

运行结果行数/列数正确：

无 GROUP BY → 一行

有 GROUP BY → 组数行；键列 + 聚合列顺序正确

边界：

空输入：COUNT(*) = 0；SUM/AVG/MIN/MAX 的空组行为按你们 finalize 的约定（通常是 NULL/未定义）；

键为相同值的行都被归到同一组；

多键（两列分组）与表达式键（DATE(ts)）也生效。
# 8. 如果以后你要“自己独立做一个新算子”，照这个配方走
这是一个通用 Checklist（照抄就能落地）：

语法（可选，看算子类型）

在 yacc_sql.y 增加语法产出，必要时更新 l_sql.l 关键字

把新东西装进 ParsedSqlNode（或新建一个）

表达式/解析树

如果引入新表达式，先给出 Expression 的派生类定义（type()/value_type()/get_value()/get_column()）

决定哪些在解析阶段是 Unbound，哪些在 resolver 阶段绑定

Resolver（绑定/类型推导）

把 Unbound* 变成可执行表达式，做类型/长度决定与校验

把语义限制放在这里报错（比如 SUM(*) 不允许，GROUP BY * 不允许）

Logical Operator

定义逻辑节点（比如 Filter/Project/Aggregate/Join 等），能表达“要做什么”

在 builder 里根据 ParsedSqlNode 选择逻辑结构

Physical Operator

选择实现策略（向量化/算子组合/索引利用等），实现 open/next/close

约定好 Chunk 的输入输出列、表达式在哪一层计算

写必要的数据结构（比如 HashTable、状态机、临时缓冲）

执行/集成

把物理算子挂进物理计划器，保证 SELECT 走到你的算子

写自测 SQL，确认 EXPLAIN 能看到你的算子

性能/扩展（后续）

批处理、SIMD、内存复用、下推计算（把 pos() 设定好减少重复求值）

DISTINCT、NULL 语义、类型提升、溢出处理
# 9. 你现在手上的代码都在做什么（和这套思路一一对上）
yacc_sql.y：把聚合与 group by 语法接进来，产出 UnboundAggregateExpr、Expression 列表、group_by 列表

expression.h：定义了所有表达式类型与行为；Resolver 会把 Unbound* 变成 FieldExpr/AggregateExpr；执行时 get_column() 提供列向量

aggregate_vec_physical_operator.cpp：无分组的聚合算子（一个组 → 一行）

group_by_vec_physical_operator.h/.cpp（我们补的）：分组聚合算子（多个组 → 多行，靠 AggregateHashTable 聚合）

aggregate_hash_table.h/.cpp：分组聚合的核心容器（add_chunk 做 build，Scanner 做输出）；你还补了一个标量版本的 LinearProbingAggregateHashTable::add_batch，作为 SIMD 优化的基础版本