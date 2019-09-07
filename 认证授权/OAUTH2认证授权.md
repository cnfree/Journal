# OATUH2认证授权


## Spring Cloud下基于OAUTH2认证授权的实现
https://github.com/wiselyman/uaa-zuul

在Spring Cloud需要使用OAUTH2来实现多个微服务的统一认证授权，通过向OAUTH服务发送某个类型的grant type进行集中认证和授权，从而获得access_token，而这个token是受其他微服务信任的，我们在后续的访问可以通过access_token来进行，从而实现了微服务的统一认证授权。

  * discovery-service:服务注册和发现的基本模块
  * auth-server:OAUTH2认证授权中心
  * order-service:普通微服务，用来验证认证和授权
  * api-gateway:边界网关(所有微服务都在它之后)

OAUTH2中的角色：
  * Resource Server:被授权访问的资源
  * Authotization Server：OAUTH2认证授权中心
  * Resource Owner： 用户
  * Client：使用API的客户端(如Android 、IOS、web app)

Grant Type：
  * authorization_code:
  * implicit:
  * password:
  * refresh_token: