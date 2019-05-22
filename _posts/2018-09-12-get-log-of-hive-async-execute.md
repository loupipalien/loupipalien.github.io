---
layout: post
title: "获取 Hive 异步执行的日志"
date: "2018-09-12"
description: "获取 Hive 异步执行的日志"
tag: [hive]
---

### Hive-Jdbc 异步执行
Hive-Jdbc 在 java.sql.Statement 接口之外支持了 executeAsync 方法
```
/**
 * Starts the query execution asynchronously on the server, and immediately returns to the client.
 * The client subsequently blocks on ResultSet#next or Statement#getUpdateCount, depending on the
 * query type. Users should call ResultSet.next or Statement#getUpdateCount (depending on whether
 * query returns results) to ensure that query completes successfully. Calling another execute*
 * method, or close before query completion would result in the async query getting killed if it
 * is not already finished.
 * Note: This method is an API for limited usage outside of Hive by applications like Apache Ambari,
 * although it is not part of the interface java.sql.Statement.
 *
 * @param sql
 * @return true if the first result is a ResultSet object; false if it is an update count or there
 *         are no results
 * @throws SQLException
 */
public boolean executeAsync(String sql) throws SQLException {
    runAsyncOnServer(sql);
    if (!stmtHandle.isHasResultSet()) {
        return false;
    }
    resultSet =
            new HiveQueryResultSet.Builder(this).setClient(client).setSessionHandle(sessHandle)
                    .setStmtHandle(stmtHandle).setMaxRows(maxRows).setFetchSize(fetchSize)
                    .setScrollable(isScrollableResultset).build();
    return true;
}
```
此方法可以立即返回, 但是客户接着调用 ResultSet.next 或者 Statement.getUpdateCount, 在 SQL 未执行完成前阻塞, 因为在这两个方法中都会会调用 waitForOperationToComplete 方法
```
void waitForOperationToComplete() throws SQLException {
  TGetOperationStatusReq statusReq = new TGetOperationStatusReq(stmtHandle);
  TGetOperationStatusResp statusResp;

  // Poll on the operation status, till the operation is complete
  while (!isOperationComplete) {
    try {
      /**
       * For an async SQLOperation, GetOperationStatus will use the long polling approach It will
       * essentially return after the HIVE_SERVER2_LONG_POLLING_TIMEOUT (a server config) expires
       */
      statusResp = client.GetOperationStatus(statusReq);
      Utils.verifySuccessWithInfo(statusResp.getStatus());
      if (statusResp.isSetOperationState()) {
        switch (statusResp.getOperationState()) {
        case CLOSED_STATE:
        case FINISHED_STATE:
          isOperationComplete = true;
          isLogBeingGenerated = false;
          break;
        case CANCELED_STATE:
          // 01000 -> warning
          throw new SQLException("Query was cancelled", "01000");
        case TIMEDOUT_STATE:
          throw new SQLTimeoutException("Query timed out after " + queryTimeout + " seconds");
        case ERROR_STATE:
          // Get the error details from the underlying exception
          throw new SQLException(statusResp.getErrorMessage(), statusResp.getSqlState(),
              statusResp.getErrorCode());
        case UKNOWN_STATE:
          throw new SQLException("Unknown query", "HY000");
        case INITIALIZED_STATE:
        case PENDING_STATE:
        case RUNNING_STATE:
          break;
        }
      }
    } catch (SQLException e) {
      isLogBeingGenerated = false;
      throw e;
    } catch (Exception e) {
      isLogBeingGenerated = false;
      throw new SQLException(e.toString(), "08S01", e);
    }
  }
}
```
此方法会一直 while 循环, 直到服务端返回执行完成或者执行失败或异常; 对于异步的 SQL 操作, GetOperationStatus 在返回之前等待一个 HIVE_SERVER2_LONG_POLLING_TIMEOUT 的时长, 这是一个服务端的配置, 这个值默认是 5000ms; 这里 while 循环在执行结束前会一直向 HS2 发送 thrift 请求, HS2 的对于单个 HiveStatement 的 thrift 请求是 FIFO 处理的, 所以同时用另外的线程去获取日志, 会被阻塞到队列前的获取执行状态的请求完成才会被处理; 为了获取日志的请求能够尽快被处理, 可以在这个 while 循环里增加 Thread.sleep(), 减少获取执行状态的请求次数, 从而让获取日志的请求尽快被处理
