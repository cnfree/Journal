# Prometheus结合Consul监控架构

## 架构图
 
  ![prometheus_consul]

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
 
  
  [prometheus_consul]: img/prometheus_consul.webp
  [consul_sd]: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#consul_sd_config