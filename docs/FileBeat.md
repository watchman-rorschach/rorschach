# 概述
## filebeat和beats的关系
Beats在是一个轻量级日志采集器，filebeat是Beats中的一员。早期的ELK架构中使用Logstash收集、解析日志，但是Logstash对内存、cpu、io等资源消耗比较高。相比Logstash，Beats所占系统的CPU和内存几乎可以忽略不计。
Filebeat是用于转发和集中日志数据的轻量级传送工具。Filebeat监视您指定的日志文件或位置，收集日志事件，并将它们转发到Elasticsearch或 Logstash进行索引。
目前Beats包含六种工具：

- Packetbeat：网络数据（收集网络流量数据）
- Metricbeat：指标（收集系统、进程和文件系统级别的CPU和内存使用情况等数据）
- Filebeat：日志文件（收集文件数据）
- Winlogbeat：windows事件日志（收集Windows事件日志数据）
- Auditbeat：审计数据（收集审计日志）
- Heartbeat：运行时间监控（收集系统运行时的数据）
# 使用
## 安装启动

- 注意要和logstash和es的版本一致。
1. 下载地址：[https://www.elastic.co/cn/downloads/past-releases/filebeat-7-13-1](https://www.elastic.co/cn/downloads/past-releases/filebeat-7-13-1)
2. 监测配置文件是否正确  `./filebeat test config`
3. 启动命令  `./filebeat -e`

# 示例配置
## logstash作为输出
```yaml
- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:  #配置多个日志路径
    - /var/logs/es_aaa_index_search_slowlog.log
output.logstash:
  # The Logstash hosts #配多个logstash使用负载均衡机制
  hosts: ["192.168.110.130:5044","192.168.110.133:5044"]  
  loadbalance: true  #使用了负载均衡
```
## ES作为输出

```yaml
- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:  #配置多个日志路径
    - /var/logs/es_aaa_index_search_slowlog.log
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]
```
