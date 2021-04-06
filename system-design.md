> 系统设计题，核心参考资料 [文档](https://github.com/donnemartin/system-design-primer/blob/master/README-zh-Hans.md)

### 如何处理系统设计题目

1. 描述使用的场景、约束和假设
   * 谁使用它，怎么使用它？
   * 系统的输入、输出是什么？
   * 处理的数据量？TPS、读写比
2. 创造高层次的设计
   * 画出主要的组件和连接
3. 设计核心组件
   * 核心组件进行详细深入的分析
4. 扩展设计 (评估解决方案与利弊)
   * 负载均衡
   * 水平扩展
   * 缓存
   * 数据库分片

### 设计短域名服务

1. 描述场景，约束与假设
   * 用户：使用它保存文本的内容，生成短链接进行访问
   * 输入：文本。输出：短链接
   * 数据量假设
     * 一千万的用户量
     * 每个月一千万的 paste 写入量
     * 每个月一亿的 paste 读取量
     * 读写比例在 10:1

### 朋友圈系统设计

[参考](http://www.woshipm.com/pd/249869.html)

twitter 时间线设计：[链接](https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/twitter/README.md)

