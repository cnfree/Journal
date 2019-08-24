# Apollo配置中心架构

# 官方架构图
  ![Apollo_Offical]
  
# 流程图
  ![流程图]  
  
# 核心模块
  * ConfigService
    * 提供配置获取接口
    * 提供配置推送接口
    * 服务于Apollo客户端
  * AdminService
    * 提供配置管理接口
    * 提供配置修改发布接口
    * 服务于管理界面Portal     
  * Client
    * 为应用获取配置，支持实时更新
    * 通过MetaServer获取ConfigService的服务列表
    * 使用客户端软负载SLB方式调用ConfigService 
  * Portal
    * 配置管理界面
    * 通过MetaServer获取AdminService的服务列表
    * 使用客户端软负载SLB方式调用AdminService   

# 辅助服务发现模块
  * Eureka
    * 用于服务发现和注册
    * Config/AdminService注册实例并定期报心跳
    * 和ConfigService住在一起部署
  * MetaServer
    * Portal通过域名访问MetaServer获取AdminService的地址列表
    * Client通过域名访问MetaServer获取ConfigService的地址列表
    * 相当于一个Eureka Proxy
    * 逻辑角色，和ConfigService住在一起部署
  * NginxLB
    * 和域名系统配合，协助Portal访问MetaServer获取AdminService地址列表
    * 和域名系统配合，协助Client访问MetaServer获取ConfigService地址列表
    * 和域名系统配合，协助用户访问Portal进行配置管理

# 不同视角的架构图
  ![apollo_arch]
  
  * ConfgService/AdminService/Client/Portal是Apollo的四个核心微服务模块，相互协作完成配置中心业务功能
  * Eureka/MetaServer/NginxLB是辅助微服务之间进行服务发现的模块
  * Apollo采用微服务架构设计，架构和部署都有一些复杂，但是每个服务职责单一，易于扩展
  * Apollo只需要一套Portal就可以集中管理多套环境(DEV/FAT/UAT/PRO)中的配置

# Apollo 通知流程图
  ![message-notification]
  * 用户在Portal操作配置发布
  * Portal调用Admin Service的接口操作发布
  * Admin Service发布配置后，发送ReleaseMessage给各个Config Service
  * Config Service收到ReleaseMessage后，通知对应的客户端
# Apollo 客户端设计
  ![client-architecture]
  * 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
  * 客户端还会定时从Apollo配置中心服务端拉取应用的最新配置。
    * 这是一个fallback机制，为了防止推送机制失效导致配置不更新
    * 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 - Not Modified
    * 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定System Property: apollo.refreshInterval来覆盖，单位为分钟。
  * 客户端从Apollo配置中心服务端获取到应用的最新配置后，会保存在内存中
  * 客户端会把从服务端获取到的配置在本地文件系统缓存一份
    * 在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置
  * 应用程序可以从Apollo客户端获取最新的配置、订阅配置更新通知  
 
# 实际业务架构图  
  ![Apollo分布式部署中心]
  
  * 实际部署时，不同的网络环境可能不会互通，除了Portal可以访问各个AdminService，客户端是不能访问任意集群的。
  * 改造Apollo客户端，根据不同的env，发现不同的MetaServer,进而访问不同的ConfigService
  * env按照apollo约定，配置在系统环境变量：
     * 使用Java启动参数添加java -Denv=YOUR-ENVIRONMENT -jar xxx.jar
     * 通过操作系统的System Environment
     * 通过配置文件：
       * 对于Mac/Linux，文件位置为/opt/settings/server.properties
       * 对于Windows，文件位置为C:\opt\settings\server.properties
     * 配置apollo-env.properties

# 启用Apollo配置
   * 创建app.properties
   
      * 请确保classpath:/META-INF/app.properties文件存在，并且其中内容为自己的项目名称或者项目编号，而且要保持唯一
          ```
          app.id=demo 
          ```
      * 为了保证app.id唯一，尽量对app.id进行统一管理，每个app.id需要走申请流程，最好为项目数字编号
      
   * 在启动类添加@EnableApolloConfig注解即可
       ```
        import org.springframework.boot.SpringApplication;
        import org.springframework.boot.autoconfigure.SpringBootApplication;
        
        @EnableApolloConfig
        @SpringBootApplication
        public class Application {
        
            public static void main(String[] args) {
                SpringApplication.run(Application.class, args);
            }
        
        }
       ```    
     
 # 关于 @ConfigurationProperties
  
  @ConfigurationProperties如果需要在Apollo配置变化时自动更新注入的值，需要配合使用EnvironmentChangeEvent或RefreshScope。
    
        
         

  [Apollo_Offical]: img/Apollo_Offical.png
  [apollo_arch]: img/apollo_arch.png
  [Apollo分布式部署中心]: img/Apollo分布式部署中心.png
  [message-notification]: img/message-notification.png
  [流程图]: img/流程图.png
  [client-architecture]: img/client-architecture.png