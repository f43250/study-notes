# SpringCloud

## 1.SpringCloudBus如何知道配置文件刷新了?
- 消息总线
  1. 提交代码,利用Git的webhook触发请求到bus/refresh接口
  2. server端接收到请求并发送给Spring Cloud Bus
  3. Spring Cloud Bus接受到请求通知给其他客户端
  4. 其他客户端全部接受到通知, 请求server端获取最新配置
  5. 所有客户端全部升级配置完成.
- 引入actuator
  1. 使用了配置的地方加上@RefreshScope注解
  2. 项目根路径/refresh手动刷新