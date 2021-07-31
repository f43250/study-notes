# SpringBoot

## 1.思想及应用
  - SpringBoot的核心思想是约定优于配置.
  - 是一个服务于框架的框架,服务范围是简化配置文件.

## 2.Bean加载顺序
  - 根据package扫描出需要被Spring管理的类
  - 讲这些类封装成BeanDefinition并注册到BeanFactory
  - 实例化扫描到的BeanDefinition,其中包括解决循环依赖、延迟加载问题.