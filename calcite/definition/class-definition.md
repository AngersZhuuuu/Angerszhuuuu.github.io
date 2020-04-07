[TOC]




## SQL Tree


下面的表格介绍一些类和其含义

|  类名   | 含义  |
|  ----  | ----  |
| SqlNode  | 整个SQL的AST树，非物理计划树 |
| RelNode  | Relation Node， SQL生成的物理执行计划树 |
| RexNode  | Row expression node，表达式节点 |
| RelOptUtil | Relation Optimize Util， 逻辑关系优化器工具类  |