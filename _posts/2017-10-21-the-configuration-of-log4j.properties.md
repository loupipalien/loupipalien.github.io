---
layout: post
title: "log4j.properties配置详解"
date: "2017-10-21"
description: "log4j.properties配置详解"
tag: [java,log4j]
---

**前提假设:** 了解log4j组件和简单使用

### 优先级
日志记录的优先级, 分为`OFF,TRACE,DEBUG,INFO,WARN,ERROR,FATAL,ALL`, Log4j建议只使用四个级别, 优先级从低到高分别是`DEBUG,INFO,WARN,ERROR`。通过定义日志的级别, 可以控制到应用程序中相应级别的日志信息的开关例如在这里定义了INFO级别, 则应用程序中所有DEBUG级别的日志信息将不被输出

### Logger (负责定义日志记录的优先级)
通常配置根Logger即可, 如果想把某个包或类的配置单独输出到指定文件, 则可配置包或类的Logger
- 配置根Logger  
`log4j.rootLogger = [level],appenderName,appenderName2,...`  
appenderName就是指定日志信息输出到哪个地方, 可同时指定多个输出目的。
- 配置包或类Logger  
`log4j.logger.com.ltchen.demo.log4j = [level],appenderName,appenderName2,...`  
`log4j.logger.com.ltchen.demo.log4j.App = [level],appenderName,appenderName2,...`  
对于包或类Logger, 还可设置additivity属性。标志子Logger是否继承父Logger的appender。默认情况下子Logger会继承父Logger的appender, 也就是说子Logger会在父Logger的appender里输出。若是additivity设为false, 则子Logger只会在自己的appender里输出, 而不会在父Logger的appender里输出。  
`log4j.additivity.com.ltchen.demo.log4j = false`  
`log4j.additivity.com.ltchen.demo.log4j.App = false`  

### Appender (负责指定日志的输出地)
Appender用于指定各Looger的日志输出地, 通用语法如下  
`log4j.appender.appenderName = fully.qualified.name.of.appender.class`  
`log4j.appender.appenderName.optionN = valueN`  

**Log4j提供的常用Appender**
- org.apache.log4j.ConsoleAppender(输出到控制台)
ConsoleAppender的选项属性
  - Threshold = DEBUG: 指定日志消息的输出最低层次
  - ImmediateFlush = TRUE: 默认值是true, 所有的消息都会被立即输出
  - Target = System.err: 默认值System.out, 输出到控制台(err为红色,out为黑色)
- org.apache.log4j.FileAppender(输出到文件)
FileAppender的选项属性
  - Threshold = INFO: 指定日志消息的输出最低层次
  - ImmediateFlush = TRUE: 默认值是true, 所有的消息都会被立即输出
  - File = C://log4j.log: 指定消息输出到C://log4j.log文件
  - Append = FALSE: 默认值true, 将消息追加到指定文件中, false指将消息覆盖指定的文件内容
  - Encoding = UTF-8: 可以指定文件编码格式
- org.apache.log4j.DailyRollingFileAppender(每天产生一个日志文件)
DailyRollingFileAppender的选项属性
  - Threshold = WARN: 指定日志消息的输出最低层次
  - ImmediateFlush = TRUE: 默认值是true, 所有的消息都会被立即输出
  - File = C://log4j.log: 指定消息输出到C://log4j.log文件
  - Append = FALSE: 默认值true, 将消息追加到指定文件中, false指将消息覆盖指定的文件内容
  - DatePattern='.'yyyy-ww: 每周滚动一次文件, 即每周产生一个新的文件。可以用以下参数:
    - '.'yyyy-MM: 每月
    - '.'yyyy-ww: 每周
    - '.'yyyy-MM-dd: 每天
    - '.'yyyy-MM-dd-a: 每天两次
    - '.'yyyy-MM-dd-HH: 每小时
    - '.'yyyy-MM-dd-HH-mm: 每分钟
  - Encoding = UTF-8: 可以指定文件编码格式
- org.apache.log4j.RollingFileAppender(文件大小到达指定尺寸的时候产生一个新的文件)
RollingFileAppender的选项属性
  - Threshold = ERROR: 指定日志消息的输出最低层次
  - ImmediateFlush = TRUE: 默认值是true, 所有的消息都会被立即输出
  - File = C://log4j.log: 指定消息输出到C://log4j.log文件
  - Append = FALSE: 默认值true, 将消息追加到指定文件中, false指将消息覆盖指定的文件内容
  - MaxFileSize = 100KB: 后缀可以是KB,MB,GB.在日志文件到达该大小时, 将会自动滚动,如:log4j.log.1
  - MaxBackupIndex = 2: 指定可以产生的滚动文件的最大数
  - Encoding = UTF-8: 可以指定文件编码格式
- org.apache.log4j.WriterAppender(将日志信息以流格式发送到任意指定的地方)  

### Layout (负责对输出的日志格式化)
Layout用于格式化日志的输出样式, 通用语法如下    
`log4j.appender.appenderName.layout = fully.qualified.name.of.layout.class`    
`log4j.appender.appenderName.layout.optionN = valueN`

**Log4j提供的Layout**
- org.apache.log4j.HTMLLayout(以HTML表格形式布局)
  - LocationInfo = TRUE: 默认值false,输出java文件名称和行号
  - Title=Struts Log Message: 默认值 Log4J Log Messages
- org.apache.log4j.PatternLayout(可以灵活地指定布局模式, 推荐使用)
  - ConversionPattern = %m%n: 格式化指定的消息(参数意思下面有)
- org.apache.log4j.SimpleLayout(包含日志信息的级别和信息字符串)
- org.apache.log4j.TTCCLayout(包含日志产生的时间、线程、类别等等信息)
- org.apache.log4j.xml.XMLLayout(以XML形式布局)
  - LocationInfo = TRUE: 默认值false,输出java文件名称和行号

Log4j采用类似C语言中的printf函数的打印格式格式化日志信息, 打印参数如下

|参数|含义|
|--|--|
|%c|输出所属的类目, 通常是所在类的全限定名|
|%C|输出所属的类的全限定名|
|%d|输出日志时间点的日期或时间, 默认格式为ISO8601, 也可以在其后指定格式, 如：%d{yyyy年MM月dd日 HH:mm:ss,SSS}, 输出类似：2012年01月05日 22:10:28,921|
|%F|输出日志消息产生时所在的文件名称|
|%l|输出日志事件的发生位置, 包括类名以及在代码中的行数, 如：Testlog.main(TestLog.java:10)|
|%L|输出日志事件的发生的代码行数, 会影响性能不建议使用|
|%m|输出代码中指定的消息|
|%M|输出日志事件的发生所在的方法名, 会影响性能不建议使用|
|%n|输出一个回车换行符, Windows平台为"\r\n", Unix平台为"\n"|
|%p|输出优先级, 即DEBUG,INFO,WARN,ERROR,FATAL|
|%r|输出自应用启动到输出该log信息耗费的毫秒数|
|%t|输出产生该日志事件的线程名|
|%x|输出和当前线程相关联的NDC(嵌套诊断环境)|
|%X|输出和当前线程相关联的MDC(映射诊断环境)|
|%%|输出一个"%"字符|

可以在%与模式字符之间加上修饰符来控制其最小宽度、最大宽度、和文本的对齐方式
- %5c: 输出category名称, 最小宽度是5, category<5, 默认的情况下右对齐
- %-5c: 输出category名称, 最小宽度是5, category<5, "-"号指定左对齐,会有空格
- %.5c: 输出category名称, 最大宽度是5, category>5, 就会将左边多出的字符截掉，<5不会有空格
- %20.30c: category名称<20补空格, 并且右对齐, >30字符, 就从左边较远输出的字符截掉

>**参考:**  
[log4j 1.2 manual](https://logging.apache.org/log4j/1.2/manual.html)  
[log4j 1.2 apidocs](https://logging.apache.org/log4j/1.2/apidocs/index.html)    
[log4j.properties配置详解与实例](http://blog.sina.com.cn/s/blog_5ed94d710101go3u.html)  
[log4j将不同Package的日志输出到不同的文件的方法](http://www.crazyant.net/1931.html)    
[log4j配置文件中的additivity属性](http://www.cnblogs.com/edgedance/p/6979622.html)  
