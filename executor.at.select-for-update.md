## SELECT FOR UPDATE EXECUTOR

### the role of executor
众所周知, 为了实现分布式事务在 seata-go 中存在三种角色: `TM`、`RM`、`TC`。

- ` TC ` : 事务协调者, 负责维护全局和分支事务的状态, 驱动全局事务提交和回滚
- ` TM ` : 事务管理器, 负责开启、提交和回滚全局事务, 用户一般是和 TM 打交道
- ` RM ` : 资源管理器, 管理分支事务处理的资源, 驱动分支事务提交和回滚

在一个分布式事务中, 通常涉及多个 RM, 每个 RM 代理一个 DB, TM 会根据业务 SQL 远程调用 RM 开启本地事务并执行 SQL, 同时生成 undo log, 然后 RM 再向 TC 注册分支事务并在 TC 端进行持久化(需要全局锁), 成功以后 RM 再持久化 undo log, 并提交本地事务, 当所有 TM 得到所有 RM 的执行结果以后, 再执行 全局事务的提交 or 回滚

如果执行全局事务的提交, 当所有 RM 负责的分支事务都成功时, TM 则进行全局事务提交, TC 通知所有的 RM 进行第二阶段提交, TC 删除分支事务记录而 RM 删除 undo log, 最终 TC 删除全局事务记录

如果执行全局事务的回滚, 当有至少一个 RM 执行分支事务报错时, TM 只能执行回滚保持全局数据的一致性, 具体做法：TM 通知 TC 进行回滚, TC 则所有 RM 进行回滚, 每个 RM 响应 TC 以后, TC 都会删除对应的全局锁、分支事务, 并更新全局事务状态为 Rollbacked, 然后再删除全局事务

上面说了一堆, 和 executor 有啥关系呢? 实际上 executor 是 RM 的核心之一，负责执行业务 SQL 并生成undo log, 基本上它是第一阶段的全部内容, 从开启本地事务到执行分支业务 SQL 到最后提交本地事务

### select for update
> 而select … for update 语句是我们经常使用手工加锁语句。在数据库中执行select … for update ,大家会发现会对数据库中的表或某些行数据进行锁表，在mysql中，如果查询条件带有主键，会锁行数据，如果没有，会锁表。
>
> 由于InnoDB预设是Row-Level Lock，所以只有「明确」的指定主键，MySQL才会执行Row lock (只锁住被选取的资料例) ，否则MySQL将会执行Table Lock (将整个资料表单给锁住)。
>
> 注意：
> 1、FOR UPDATE仅适用于InnoDB，且必须在事务处理模块(BEGIN/COMMIT)中才能生效。
>
> 2、要测试锁定的状况，可以利用MySQL的Command Mode(命令模式) ，开两个视窗来做测试。
>
> 3、Myisam 只支持表级锁，InnerDB支持行级锁 添加了(行级锁/表级锁)锁的数据不能被其它事务再锁定，也不被其它事务修改。是表级锁时，不管是否查询到记录，都会锁定表。

总结: for update 会给查询到的记录加锁(行锁 or 表锁)，在RR级别下防止幻读


### select for update executor implementation
在 RM 中，executor 先开启本地事务，并执行业务 SQL，然后分析业务 SQL 中的主键
1. 如果存在主键就向 TC 申请全局锁，锁住这些记录防止修改
2. 如果不存在就直接返回


### Q & A

#### 01 为什么需要全局锁锁住这些记录呢, for update 不是已经锁住了记录吗?
executor 负责开启本地事务、执行业务 SQL 和 提交事务, 在该 RM 执行完分支业务 SQL 后，会提交本地事务，for update 加的锁已经释放了，如果存在其他 RM 修改这部分记录，就需要全局锁，等所有 RM 执行完所有分支事务以后再一个个的释放 全局锁

#### 02 上文提到 RM 会执行业务 SQL 并生成 undo log, 但实现中貌似没有 undo log?
是的，因为 select for update 只是对记录加锁，并没有修改数据，并不需要回滚

#### 03 在分布式事务场景上, 存在使用该 executor 的场景吗?
待补充


### reference
- [seata AT 时序图](https://www.processon.com/view/link/62dd28a1e0b34d1e9bd98766)
- [seata 官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
- [for udpate 的作用和用法](https://www.cnblogs.com/banma/p/11797560.html)
- [select for update 的场景以及死锁陷阱](https://www.cnblogs.com/qilong853/p/9427145.html)




