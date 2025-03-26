---
layout: post
title: 记一次业务MySQL死锁
subtitle: 记一次业务MySQL死锁
excerpt_image: https://github.com/wtopps/wtopps.github.io/blob/master/images/Problem%20Investigation.jpeg?raw=true
categories: MySQL
tags: [MySQL]
top: 1

---

![banner](https://github.com/wtopps/wtopps.github.io/blob/master/images/Problem%20Investigation.jpeg?raw=true)

## 问题

线上服务出现服务告警，查询日志现场，错误信息如下：

```bash
Caused by: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:124)
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeInternal(ClientPreparedStatement.java:916)
	at com.mysql.cj.jdbc.ClientPreparedStatement.executeQuery(ClientPreparedStatement.java:972)
	at com.alibaba.druid.pool.DruidPooledPreparedStatement.executeQuery(DruidPooledPreparedStatement.java:213)
	at org.hibernate.engine.jdbc.internal.ResultSetReturnImpl.extract(ResultSetReturnImpl.java:57)
	... 113 common frames omitted
	
....

Deadlock found when trying to get lock; try restarting transaction
```

根据日志信息，很显然是一个MySQL死锁的case。



## 问题分析

根据报错信息，定位到报错的代码：

```java
for (AuditProcessRecord auditProcessRecord : auditProcessRecords) {
    if (auditProcessRecord.getId().equals(auditProcessRecordId)) {
        auditProcessRecord.setAuditStatus(AuditProcessStatus.AUDIT_PASS.getStatus());
        auditProcessRecordRepo.save(auditProcessRecord);
        break;
    }
}
// 检查是否存在审核中的流程，如果存在，继续等待剩余流程结束
// 事务中，加读锁，当前读
// 不加锁，快照读
int auditingRecordCount = auditProcessRecordRepo.existsAuditingRecordInLock(auditId);
if (auditingRecordCount > 0) {
    log.info("LotteryAudit process is pending auditItem, waiting. lotteryId={}, auditId={}",
            lotteryAuditRecord.getLotteryId(), auditId);
    return;
}
```

报错的方法是existsAuditingRecordInLock，看一下实现：

```java
@Query(value = "select count(*) from audit_process_record where audit_id = ?1 and audit_status = 0 LOCK IN SHARE MODE",nativeQuery = true)
Integer existsAuditingRecordInLock(String auditId);
```

existsAuditingRecordInLock的实现可以看到，使用了共享锁查询audit_process_record的行数，线上MySQL的事务级别使用RR，使用共享锁的目的，应该是希望避免快照读，而是使用当前读，并发场景下，可以查询到其他线程的最新数据，但问题也恰恰出在这里。

#### 执行过程分析

假定有两个线程，同时执行上面的代码，两个线程同时处理不同的 auditProcessRecordId，但是相同的 auditId：

线程A：

1. BEGIN;
2. 更新记录1：
   UPDATE audit_process_record SET audit_status = 1 WHERE id = 1  // 获得id=1记录的排他锁
3. SELECT COUNT(*) ... LOCK IN SHARE MODE   // 尝试获取audit_id=1的所有记录的共享锁
   需要获取id=2记录的共享锁，但被线程B的排他锁阻塞

线程B：
1. BEGIN;
2. 更新记录2：
   UPDATE audit_process_record SET audit_status = 1 WHERE id = 2  // 获得id=2记录的排他锁
3. SELECT COUNT(*) ... LOCK IN SHARE MODE   // 尝试获取audit_id=1的所有记录的共享锁
   需要获取id=1记录的共享锁，但被线程A的排他锁阻塞

死锁原因：

线程A先获得记录1的排他锁，然后要获取所有记录（包括记录2）的共享锁

线程B先获得记录2的排他锁，然后要获取所有记录（包括记录1）的共享锁

形成循环等待：

​	A持有记录1排他锁，等待记录2的共享锁

​	B持有记录2排他锁，等待记录1的共享锁



## 问题解决

方法一：

使用FOR UPDATE替代LOCK IN SHARE MODE

```java
SELECT COUNT(*) FROM audit_process_record WHERE audit_id = ?1 AND audit_status = 0 
FOR UPDATE
```

FOR UPDATE是独占锁（排他锁），它在读取数据时会锁定记录，并且同时防止其他事务读取或修改这些记录。在事务中使用 FOR UPDATE 时，其他事务将无法对加锁的记录执行 SELECT 或任何修改操作，直到当前事务提交或回滚。

方法二：

调整事物隔离级别，RR变为RC

方法三：

使用其他的分布式锁，例如Redis或者ZK，代替MySQL的锁操作。

