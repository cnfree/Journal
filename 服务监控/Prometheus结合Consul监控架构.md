# Prometheus结合Consul监控架构

## 架构图
 
  ![prometheus_consul]

## Prometheus组成部分
  * **_Prometheus Server_**：负责采集监控数据，并且对外提供PromQL实现监控数据的查询以及聚合分析；
  * **_Exporters_**：用于向Prometheus Server暴露数据采集的endpoint，Prometheus轮训（**Pull模式**）这些Exporter采集并且保存数据；
  * **_AlertManager_** 以及其它组件
  
## Pull模式的优点
  * 只要Exporter在运行，可以在任何地方（比如在本地）搭建监控系统
  * 可以更容易的去定位Instance实例的健康状态以及故障定位  
  
## 基于服务发现注册监控服务

   Prometheus支持多种服务发现机制：文件、DNS、Consul、Kubernetes、OpenStack、EC2等等。
   
   基于服务发现的过程并不复杂，通过第三方提供的接口，Prometheus查询到需要监控的Target列表，然后轮训这些Target获取监控数据。  
  
## Prometheus关于Consul服务自动获取的配置

  参见 [Prometheus configuration consul_sd 章节][consul_sd]
  
  * ___meta_consul_address_: the address of the target
  * ___meta_consul_dc_: the datacenter name for the target
  * ___meta_consul_tagged_address\_\<key>_: each node tagged address key value of the target
  * ___meta_consul_metadata\_\<key>_: each node metadata key value of the target
  * ___meta_consul_node_: the node name defined for the target
  * ___meta_consul_service_address_: the service address of the target
  * ___meta_consul_service_id_: the service ID of the target
  * ___meta_consul_service_metadata\_\<key>_: each service metadata key value of the target
  * ___meta_consul_service_port_: the service port of the target
  * ___meta_consul_service_: the name of the service the target belongs to
  * ___meta_consul_tags_: the list of tags of the target joined by the tag separator
  
## Prometheus relabel_config
  relabel_config的作用就是将时间序列中 label 的值做一个替换，具体的替换规则有配置决定
  
  ```
    relabel_configs:
          - source_labels: [__address__]
            target_label: __param_target
          - source_labels: [__param_target]
            target_label: instance
          - target_label: __address__
            replacement: 127.0.0.1:9115
  ```  
  意思是：
  ```  
  __param_target = __address__ , instance = __param_target ,__address__ = 127.0.0.1:9115
  ```  
    
  * separator: 如果有多个source_label([\_\_address\_\_,jod])的时候用separator去连接几个值
  * regex: 符合这个正则表达式的source_label会被赋值给replacement再赋值给target_label
  * replacement: 取匹配了正则表达式的第几串字符串
  
  * action:
    * keep/drop: 是否需要过滤Target目标，keep 保留， drop 过滤
    * replace: 将采集数据写入到新的label
    
    例：当匹配到Target的元数据标签_metaconsul_tags中匹配到“.，development,.”，则keep当前实例
    ```
      relabel_configs:
      - source_labels: ["__meta_consul_tags"]
        regex: ".*,development,.*"
        action: keep   
    ```  



 
 ## Consul注册服务和删除服务
 
   * [Consul服务API文档][3]
   * 注册服务：	/agent/service/register
   * 删除服务：	/agent/service/deregister/:service_id
   * 可以通过docker_compose自动注册服务或者Prometheus Exporter到Consul上
 
## 参考文章

  * [Prometheus中动态发现Target和Relabel的应用][1]
  * [node exporter完整版][2]
  
  [prometheus_consul]: img/prometheus_consul.webp
  [consul_sd]: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config
  
  [1]: https://blog.csdn.net/M2l0ZgSsVc7r69eFdTj/article/details/79124770
  [2]: https://blog.51cto.com/1000682/2374406
  [3]: https://www.consul.io/api/agent/service.html