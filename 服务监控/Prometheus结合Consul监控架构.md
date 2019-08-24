# Prometheus结合Consul监控架构

## 架构图
 
  ![prometheus_consul]

## Prometheus关于Consul服务自动获取的配置

  参见 [Prometheus configuration consul_sd 章节][consul_sd]
  
  * __meta_consul_address: the address of the target
  * __meta_consul_dc: the datacenter name for the target
  * __meta_consul_tagged_address_<key>: each node tagged address key value of the target
  * __meta_consul_metadata_<key>: each node metadata key value of the target
  * __meta_consul_node: the node name defined for the target
  * __meta_consul_service_address: the service address of the target
  * __meta_consul_service_id: the service ID of the target
  * __meta_consul_service_metadata_<key>: each service metadata key value of the target
  * __meta_consul_service_port: the service port of the target
  * __meta_consul_service: the name of the service the target belongs to
  * __meta_consul_tags: the list of tags of the target joined by the tag separator
 
  
  [prometheus_consul]: img/prometheus_consul.webp
  [consul_sd]: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config