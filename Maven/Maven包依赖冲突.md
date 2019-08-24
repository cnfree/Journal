# Maven包依赖冲突 

## Maven的依赖原则
  * 第一声明优先原则：
     
     在pom.xml配置文件中，如果有两个名称相同版本不同的依赖声明，那么先写的会生效。所以，先声明自己要用的版本的jar包即可。

  * 路径近者优先：
 
     直接依赖优先于传递依赖，如果传递依赖的jar包版本冲突了，那么可以自己声明一个指定版本的依赖jar，即可解决冲突。

  * 排出原则：
     
     传递依赖冲突时，可以在不需要的jar的传递依赖中声明排除，从而解决冲突。例子：
   
       ```
        <dependency>
            <groupId>org.apache.struts</groupId>
            <artifactId>struts2-spring-plugin</artifactId>
            <version>2.3.24</version>
            <exclusions>
              <exclusion>
                <groupId>org.springframework</groupId>
                <artifactId>spring-beans</artifactId>
              </exclusion>
            </exclusions>
        </dependency>
      ```

   * 版本锁定原则（最常使用）

     在配置文件pom.xml中先声明要使用哪个版本的相应jar包，声明后其他版本的jar包一律不依赖。解决了依赖冲突。例子：
    
      ```
        <properties>
            <spring.version>4.2.4.RELEASE</spring.version>
            <hibernate.version>5.0.7.Final</hibernate.version>
            <struts.version>2.3.24</struts.version>
        </properties>
        <!-- 锁定版本，struts2-2.3.24、spring4.2.4、hibernate5.0.7 -->
        <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring-context</artifactId>
                    <version>${spring.version}</version>
                </dependency>
            </dependencies>
        </dependencyManagement>
      ```
## Maven包依赖查看命令
**_mvn dependency:tree -Dverbose_**

  ![依赖冲突]

从图上可以看到：

* 包含 **_version managed from_** 的，是来自parent dependencyManagement里的配置(**版本锁定原则**)
    * 第三方包里定义的version都不会被采用，比如自定义spring boot starter的version
    * **dependencyManagement 父亲的优先级最高**，因此如果采用了BOM管理方式，继承了spring boot bom，都将依赖于spring-boot-dependencies,里面包含了所有常用的jar版本信息，可能会非常旧
    * **无法在自定义spring boot starter中覆盖spring-boot-dependencies中的版本信息**
    * 如果要覆盖父dependencyManagement的版本信息，需要在parent bom里非dependencyManagement段的dependencies直接指定版本进行覆盖
* 包含 **_omitted for conflict with_** 的，代表着版本冲突，需要使用exclusion标签将冲突的jar排除
* 包含 **_omitted for duplicate_** 的，代表着重复依赖   


最容易让人疑惑的就是版本锁定原则，会发现无论怎么定义版本号，package的时候使用的都是 root dependencyManagement 里的版本配置

## 如何方便的查看版本依赖冲突
  * Eclipse的Dependency Hierarchy看起来很方便，一目了然
  * Idea需要安装插件Maven Helper，可以快速查看冲突jar包，但是缺少冲突原因分析，无法定位是如何造成冲突的，没有eclipse方便

 
  [依赖冲突]:img/verbose.png