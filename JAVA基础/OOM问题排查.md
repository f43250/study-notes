### 11.29三峡151环境OOM问题排查.

***
###### 问题发现及日志排查过程: 

- 周一151环境发现机器人导入导出接口超时导致调用失败,经日志排查,从basic服务调用outbound导入变量接口进入熔断状态,随后排查outbound日志,发现日志中两种关键字: `java.io.IOException: Broken pipe`及 `java.lang.OutOfMemoryError: Java heap space`
  
***

###### 诊断工具排查过程:
  - 连接151服务器执行`jps -l` 列出java服务进程号
  - 启动Arthas, `java -jar arthas-boot.jar 指定服务进程pid` 监控出现问题的outbound服务进程, 执行dashboard操作, 发现`cms-old-gen`为99%. 考虑OOM老年代回收可能有问题.

***

###### 代码排查过程
 - 本地启动对应Flow-Engine服务,连接到OOM环境所用数据库,复制日志入参,进入接口调试,发现执行下列代码段卡住.
```java
List<DialogVariantDict> dialogVariantDictList = dialogVariantDictExtMapper.exportByTenantId(tenantId);
```
 - 查看sql,发现根据租户id从数据库`dialog_variant_dict`表全量导出,随统计`dialog_variant_dict`表条数,130万条,思考字典表不太应该数据量太大.
 - 查看表结构,发现包含`(tenant_id, variant_hash_id, origin_value, delete_time) 字段的联合唯一索引`,然后比较已有数据,发现所有数据delete_time字段值均为`NULL`.根据已有经验回想NULL值可能造成联合唯一索引失效,随查询资料,的确有这个可能性.
 - 比较63开发环境与Stest3测试环境表字段,发现delete_time字段均设置了`不为NULL且默认值为'1970-01-01 00:00:00'`,考虑可能三峡环境数据库表结构不太对.

***

###### 尝试修复过程
- 备份151环境`dialog_variant_dict`数据表,对原表进行设置字段默认值,不能为NULL值等结构修改操作.
- 进行数据清洗,根据联合索引分组去重.
- 自测机器人导入导出功能,恢复正常.
- 再次查看监控,发现`cms-old-gen`恢复为8%,初步感觉修复成功.随同步修复方式给三峡现场的海龙,修复SIT和UAT环境.


***

###### 经验记录
- 由于是索引问题,不是代码或者死锁之类的问题,发现得比较顺利,有收获也有遗憾和纠结:
  - 收获包括Arthas基本使用、巩固了数据库索引失效的场景之一、问题排查的流程和经验.
  - 遗憾和纠结包括假如是代码逻辑问题或者线程死锁问题也许又学到一部分东西,不过自己能不能解决好.

