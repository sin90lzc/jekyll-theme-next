> 翻译自[quartz best practices](http://www.quartz-scheduler.org/documentation/best-practices.html)
> 
> 关于Quartz的特性请翻阅[quartz features](http://www.quartz-scheduler.org/overview/features.html)

## 生产环境使用建议

### 忽略更新检查

Quartz有自动更新机制，当检查到quartz有新的版本时进行下载。这个自动更新过程是异步的并且不影响quartz的启动和初始化。当发现可更新版本时，quartz会在日志中进行报告。

在生产环境可以通过配置文件/系统环境变量/jvm属性来兼用该功能。
	
	# 配置文件方式
	org.quartz.scheduler.skipUpdateCheck: true
	
	# 系统环境变量或jvm属性方式
	org.terracotta.quartz.skipUpdateCheck=true
	
## JobDataMap使用建议

1. JobDataMap只存储原生数据类型（包括字符串），以避免数据序列化问题。
2. 使用合并后的JobDataMap。可以在JobExecutionContext中方便地获取到JobDataMap对象，该对象是由JobDetail和Trigger两者的JobDataMap合并而成（相同属性情况下，会用Trigger的JobDataMap覆盖JobDetail）。当一个Job由多个Trigger触发的时候，Trigger中的JobDataMap则可以区分出Job由哪个Trigger触发了。因此在实现Job.execute(..)方法的时候，quartz推荐从JobExecutionContext中获取JobDataMap对象，而不是从JobDetail中获取。

## 使用JDBC作为JobStore

1. 使用scheduling API去操作数据库，而不是手动修改数据库中的数据。
2. 多个scheduler在使用同一个数据库作为JobStore的时候，scheduler的名字必须不能重复。
3. 数据源连接池应配置的最大值为worker线程池最大值+3。


## Job的实现

1. Job的execute方法应该由一个try-catch代码块所包含。因为当quartz捕获到一个由Job抛出的异常的时候，会重新执行job，而不是中断该次job的运行。
2. Job的实现应该是幂等的，因为有可能在schedule中断之后，Job的状态还是recoverable，这种情况Job有可能会被执行两次。
3. 所有Listener的实现都应该被一个try-catch代码块所包含，而且不能在Listener中完成大量复杂的任务。

## 通过应用的方式来暴露Scheduler功能时需要注意事项

通过应用的方式来暴躁scheduler的方式是推荐的，但是应该注意这种方式下的一些风险项

1. 确保暴露的功能中，不能让用户有能力去定义任务的类型和任务参数。比如说Quartz中有一个内建的org.quartz.jobs.NativeJob用于执行操作系统命令的。用户如果可以创建该类型的Job，将对系统造成致命的影响。
