## XA MYSQL CONNECTION

### 背景

由于当前使用的第三方 mysql 包 没有计划去支持 XA事务，需要自己设计并实现 MYSQL Conn 用来支持 XA 分布式事务

### MYSQL 原生 XA 事务命令

实现 seata-go XA 分布式事务的前提是，底层数据库支持 XA 本地事务。因为 MYSQL 原生支持，所以可以集成 MYSQL

```shell
XA START '自定义事务id';

SQL语句...

XA END '自定义事务id';
XA PREPARE '自定义事务id';         # 对事务进行持久化
XA COMMIT\ROLLBACK '自定义事务id';
```

### 时序图

借鉴 AT 事务执行流程，通过分析 seata AT 事务代码，做出如下时序图， 此图从用户开始执行 ExecContext 开始绘制，向 TC 注册 XID 阶段省略。执行事务时，和 AT 不一样的部分主要在 driver.Conn:

1. 在执行 SQL 前，需要执行 XA START（不同数据库可能实现不同，MSYQL 执行 XA START）
2. 根据 Executor 的结果，执行 XA PREPARE or XA ROLLBACK（包括 XA END ）
3. 在Commit阶段时，注册分支事务，并在每个分支事务中执行 XA COMMIT，如有异常则回滚，最终向 TC 进行汇报 
4. 最后在提交全局事务的时候，TC 通知 RM 调用 XASourceManager 开始提交分支事务

<img width="1327" alt="image" src="https://user-images.githubusercontent.com/32014420/218318215-5c23814a-c19c-42be-91d3-afef3a08699e.png">


### 类图
todo