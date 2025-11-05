+++
date = '2025-11-05T11:08:04+08:00'
draft = false
title = 'Ob源码初探（1）'
+++
OceanBase 源码导读路线：从启动、协议、SQL 到存储


一、整体地图（建议先扫一遍目录）
- 进程与服务：`src/observer/main.cpp:687`，`src/observer/main.cpp:503`，`src/observer/ob_server.cpp:234`，`src/observer/ob_server.cpp:931`
- 网络与协议：`src/observer/ob_srv_network_frame.h:38`，`src/observer/ob_srv_deliver.h:42`，`src/observer/mysql/obsm_handler.cpp:36`
- MySQL 请求处理：`src/observer/mysql/obmp_query.cpp:53`
- SQL 引擎入口：`src/sql/ob_sql.cpp:154`（`stmt_query`），`src/sql/ob_sql.cpp:2799`（`handle_text_query`）
- 解析器与语义：`src/sql/parser/ob_parser.cpp:24`，`src/sql/resolver/ob_resolver.cpp:13`
- 优化与代码生成：`src/sql/optimizer/ob_optimizer.cpp:27`，`src/sql/code_generator/ob_code_generator.cpp:23`
- 执行引擎（TSC 示例）：`src/sql/engine/table/ob_table_scan_op.cpp:1874`，`src/sql/engine/table/ob_table_scan_op.cpp:3763`
- DAS 与路由：`src/sql/das/ob_data_access_service.cpp:84`，`src/share/location_cache/ob_location_service.h:34`，`src/share/location_cache/ob_location_service.h:69`
- 存储访问：`src/storage/access/ob_table_scan_iterator.h:49`，`src/storage/blocksstable/ob_sstable.h:140`，`src/storage/memtable/ob_memtable.h:184`，`src/storage/memtable/mvcc/ob_mvcc_row.h:60`
- 事务与提交：`src/sql/ob_sql_trans_control.cpp:121`，`src/sql/ob_sql_trans_control.cpp:229`，`src/storage/tx/ob_trans_service.h:94`，`src/logservice/ob_log_service.h:94`
- 元数据与多租户：`src/share/schema/ob_multi_version_schema_service.h:113`，`src/share/ob_tenant_mgr.h:1`，`src/observer/omt/ob_multi_tenant.h:1`

二、从进程启动到对外服务
1) 入口与栈切换
- `main`：`src/observer/main.cpp:687` → 调用 `CALL_WITH_NEW_STACK(inner_main, ...)`
- `inner_main`：`src/observer/main.cpp:503` 完成信号栈、日志、参数解析，构造并驱动 `ObServer`

2) Server 初始化与启动
- `ObServer::init`：`src/observer/ob_server.cpp:234`
  - 关键初始化顺序（摘取常见关注点）：配置→计时器→日志→SQL 静态变量→全局上下文（`GCTX`）→模式版本→SQL Proxy→IO→KVCache→Schema→网络→RootService→SQL/PL→位置服务→存储→事务时间戳管理等。
- `ObServer::start`：`src/observer/ob_server.cpp:931` 启动信号线程与各子系统，observer 对外可用。

三、网络与协议层
1) 网络框架与交付
- `ObSrvNetworkFrame`：`src/observer/ob_srv_network_frame.h:38` 负责 easy 网络、RPC 和 MySQL NIO 的初始化与启动。
- `ObSrvDeliver`：`src/observer/ob_srv_deliver.h:42` 将请求分发到对应队列/线程组（RPC/MySQL/DDL/诊断等）。

2) MySQL 握手与连接管理
- `ObSMHandler`：`src/observer/mysql/obsm_handler.cpp:36` MySQL 协议握手、认证、连接生命周期与加密参数处理。

四、一次 SQL（以 MySQL 文本查询为例）
1) 协议层入口
- `ObMPQuery::process()`：`src/observer/mysql/obmp_query.cpp:53`
  - 取会话/租户→限流检查→解析 extra info 与 trace→初始化 `schema_guard_` 与本地/全局 schema 版本→调用 SQL 引擎：
  - `gctx_.sql_engine_->stmt_query(sql, ctx_, result)`：`src/observer/mysql/obmp_query.cpp:1090`

2) SQL 引擎主流程
- `ObSql::stmt_query()`：`src/sql/ob_sql.cpp:154`
  - 初始化 `ResultSet`→`handle_text_query()`：`src/sql/ob_sql.cpp:2799`
  - 处理 CCL、Plan Cache 命中→进入编译（未命中时）→设置结果集等。

3) 解析与语义
- 预解析/快速判断：`ObParser` 构造与 `is_pl_stmt()` 等：`src/sql/parser/ob_parser.cpp:24`
- 语法解析（lexer/yacc 生成文件在 `src/sql/parser` 目录下，`sql_parser_mysql_mode_*.c`）
- 语义解析入口：`src/sql/resolver/ob_resolver.cpp:13`（按语句类型分发到 DML/DDL/TCL/Prepare 等具体 resolver）

4) 规则改写与优化
- 规则改写：`src/sql/rewrite/ob_transformer_impl.cpp:15` 汇聚常见改写（子查询上提、join 消除、谓词下推等）。
- 基本优化入口：`ObOptimizer::optimize()`：`src/sql/optimizer/ob_optimizer.cpp:27`
  - 生成逻辑计划、访问路径选择、代价估计、分布式执行信息等。

5) 代码生成与物理计划
- `ObCodeGenerator::generate()`：`src/sql/code_generator/ob_code_generator.cpp:23`
  - 生成表达式帧→算子树（静态引擎）→设置 batch/vectorize 相关信息。

五、执行引擎到数据访问（DAS）
1) 表扫描算子（TSC）为例
- 算子实现（获取下一行/批）：
  - `ObTableScanOp::inner_open()`：`src/sql/engine/table/ob_table_scan_op.cpp:1874`
  - `ObTableScanOp::inner_get_next_row()`：`src/sql/engine/table/ob_table_scan_op.cpp:3763`
  - 关键运行时定义（RTD/CTD）、Range 构造、BNLJ 参数、Runtime Filter 等在该文件分散实现，可结合 `init_table_scan_rtdef()`、`prepare_scan_range()` 搜索阅读。

2) DAS 分发与本地/远程执行
- `ObDataAccessService::execute_das_task()`：`src/sql/das/ob_data_access_service.cpp:84`
  - 本地直接执行任务，或通过 RPC 将子任务下发到目标节点；
  - 结合 `get_das_task_id()`、`execute_dist_das_task()` 理解并发与回包管理。
- 位置服务（定位 LS/Tablet Leader）：
  - `ObLocationService::get_leader()`：`src/share/location_cache/ob_location_service.h:69`

六、存储层：从 Tablet 到 Memtable/SSTable
1) Tablet 与 LS
- `ObLS`（日志流）承载一组 Tablet：`src/storage/ls/ob_ls.h:196`
- `ObTablet`（表/分区最小管理单元）：`src/storage/tablet/ob_tablet.h:154`

2) 访问路径与合并迭代
- 统一扫描迭代器：`ObTableScanIterator`：`src/storage/access/ob_table_scan_iterator.h:49`
  - 负责组织 Memtable 与多层 SSTable 的多路归并，构造 `ObTableAccessParam/Context`，`open_iter()` 打开各层迭代器。

3) Memtable（内存多版本）
- Memtable 类：`src/storage/memtable/ob_memtable.h:184`
- MVCC 版本链节点：`ObMvccTransNode`：`src/storage/memtable/mvcc/ob_mvcc_row.h:60`
  - 字段含义：`tx_id_`、`trans_version_`（可见性）、`scn_`、`seq_no_`、`prev_/next_` 等；理解读写冲突、提交可见性与快照读需从这里入手。

4) SSTable（磁盘有序表）
- 统一只读接口：`ObSSTable`：`src/storage/blocksstable/ob_sstable.h:140`
  - `scan/get/multi_scan/multi_get` 四组接口；
  - 宏/微块索引、BloomFilter、二级元数据扫描等接口在同文件继续向下；
  - 结合 `blocksstable/index_block/*` 观察索引组织与“先索引后数据”的 IO 流程。

七、事务融入执行
1) 语句级/事务级控制
- 显式开启：`ObSqlTransControl::explicit_start_trans()`：`src/sql/ob_sql_trans_control.cpp:121`，`src/sql/ob_sql_trans_control.cpp:138`
- 结束（提交/回滚）：`ObSqlTransControl::end_trans()`：`src/sql/ob_sql_trans_control.cpp:229`
- 关键服务：`ObTransService`：`src/storage/tx/ob_trans_service.h:94`

2) 日志提交与复制
- `ObLogService`：`src/logservice/ob_log_service.h:94` 提供提交/回放对接（底层封装 Palf），事务提交走 Redo→Prepare→Commit（两阶段/ELR）到位后可见。

八、Schema 与缓存
- 多版本 Schema 服务：`ObMultiVersionSchemaService`：`src/share/schema/ob_multi_version_schema_service.h:113`
  - 提供 `get_tenant_schema_guard()` 获取指定租户与版本的元数据视图，`ObMPQuery` 里会设置租户/系统两套版本参与重试控制。
- 计划缓存绑定 Schema 版本（见 `ObSql::handle_text_query()`：`src/sql/ob_sql.cpp:2799` 中 plan cache 逻辑），发生切主/变更可能触发重编译。

九、推荐阅读路径与断点
Step 0（运行/attach）：启动 observer 后，用 gdb/lldb 设断点并 attach 到进程。

Step 1（启动期）
- 断点：`src/observer/main.cpp:503`，`src/observer/ob_server.cpp:234`，`src/observer/ob_server.cpp:931`
- 关注：配置装载、网络启动、GCTX 填充。

Step 2（协议接入）
- 断点：`src/observer/mysql/obsm_handler.cpp:69`（on_connect），`src/observer/mysql/obmp_query.cpp:53`
- 关注：连接态、租户/会话、trace_id 与超时设置。

Step 3（编译链）
- 断点：`src/sql/ob_sql.cpp:154`，`src/sql/ob_sql.cpp:2799`，`src/sql/parser/ob_parser.cpp:24`，`src/sql/resolver/ob_resolver.cpp:13`，`src/sql/rewrite/ob_transformer_impl.cpp:15`，`src/sql/optimizer/ob_optimizer.cpp:27`，`src/sql/code_generator/ob_code_generator.cpp:23`
- 关注：Plan Cache 命中/失效、AST→逻辑计划→物理计划各阶段输入输出。

Step 4（执行与存储）
- 断点：`src/sql/engine/table/ob_table_scan_op.cpp:1874`，`src/sql/engine/table/ob_table_scan_op.cpp:3763`，`src/sql/das/ob_data_access_service.cpp:84`，`src/share/location_cache/ob_location_service.h:69`，`src/storage/access/ob_table_scan_iterator.h:49`，`src/storage/blocksstable/ob_sstable.h:140`
- 关注：DAS 本地/远程、LS/Tablet 定位、Memtable/SSTable 归并读取。

Step 5（事务与提交）
- 断点：`src/sql/ob_sql_trans_control.cpp:121`，`src/sql/ob_sql_trans_control.cpp:229`，`src/storage/tx/ob_trans_service.h:94`，`src/logservice/ob_log_service.h:94`
- 关注：显式/隐式事务、autocommit、提交路径与可见性。

十、按主题的深入清单（可分支阅读）
- 执行引擎（静态向量化）：`src/sql/code_generator/ob_static_engine_cg.cpp:1`，`src/sql/engine/expr/*`（表达式计算与向量化批处理）
- PX 与并行：`src/sql/engine/px/*`（GI/GI task 切分、DFO 调度、Exchange）
- Compaction：`src/storage/compaction/*`（合并策略、Medium Info、Builder）
- LOB/外表：`src/storage/lob/*`，`src/sql/engine/table/*external*`（外部格式访问）
- 权限与审计：`src/sql/privilege_check/*`，`src/sql/monitor/ob_security_audit_utils.h:1`

十一、常见阅读技巧
- ripgrep 搜索调用链：`rg -n "ObTableScanOp::inner_get_next_row|execute_das_task|ob_sstable"` 定位热路径。
- 打印 trace_id：在 `ObMPQuery::process` 与 `ObSql::stmt_query` 附近观察 `ObCurTraceId` 与 FLT 标签。
- 查看 Schema 版本：`ObMPQuery::process` 中 `retry_ctrl_` 的本地/全局版本设置。

附：一次 SELECT 简化调用图（关键节点）
- MySQL 收包 → `ObMPQuery::process`（`src/observer/mysql/obmp_query.cpp:53`）
- → SQL 引擎 `ObSql::stmt_query`（`src/sql/ob_sql.cpp:154`）/`handle_text_query`（`2799`）
- → 解析（`src/sql/parser/ob_parser.cpp:24`）→ 语义（`src/sql/resolver/ob_resolver.cpp:13`）→ 改写（`src/sql/rewrite/ob_transformer_impl.cpp:15`）→ 优化（`src/sql/optimizer/ob_optimizer.cpp:27`）→ 代码生成（`src/sql/code_generator/ob_code_generator.cpp:23`）
- → 执行（`ObTableScanOp::inner_open/get_next_row`）
- → DAS（`execute_das_task`）→ 位置服务（`get_leader`）→ 存储访问（`ObTableScanIterator`→`ObMemtable`/`ObSSTable`）
- → 返回结果集并编码为 MySQL 包。

说明
- 文中行号为便于设断点的“函数起始附近”位置，随版本可能轻微偏移；若不匹配，可先 `rg` 搜索函数名再下断点。
