# Dubbo灰度发布

## Dubbo有三种路由方式：
 * 通过版本号version控制
 * 通过分组group控制
 * 自定义dubbo loadbalance实现

### version
   
   当一个接口的实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用
      
   ```xml
    <!-- 机器A提供1.0.0版本服务 --> 
    <dubbo:service interface="com.foo.BarService" version="1.0.0" />
    <!-- 机器B提供2.0.0版本服务 --> 
    <dubbo:service interface="com.foo.BarService" version="2.0.0" />
    <!-- 机器C消费1.0.0版本服务 -->
     <dubbo:reference id="barService" interface="com.foo.BarService" version="1.0.0" />
    <!-- 机器D消费2.0.0版本服务 --> 
    <dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />
   ```

   此外，消费者消费服任意版本的服务时：
   ```xml
   <dubbo:reference id="barService" interface="com.foo.BarService" version="*" />
   ```
## group

  当一个接口有多种实现时，可以用group区分
  ```xml
    <!-- dubbo group 使用示例 --> 
    <bean id="demoA" class="com.xxx.IndexServiceImpl1" /> 
    <dubbo:service group="feedback" interface="com.xxx.IndexService" ref="demoA" /> 
    <bean id="demoB" class="com.xxx.IndexServiceImpl2" /> 
    <dubbo:service group="member" interface="com.xxx.IndexService" ref="demoB" />
  ```

  此外，dubbo消费者也可以设置为：消费任意一个group的服务。
  ```xml
  <dubbo:reference id="barService" interface="com.foo.BarService" group="*" />
  ```

Version方式和Group方式都不适合灰度发布，因为需要修改代码，因此灰度发布需要采用第三种方式，自定义 dubbo loadbalance来实现。

## dubbo负载均衡包含如下四种：
  * RandomLoadBalance：默认的负载策略，随机负载
  * ConsistentHashLoadBalance：一致性 Hash负载
  * LeastActiveLoadBalance：最少活跃数负载
  * RoundRobinLoadBalance：轮询负载

## springboot-dubbo实现自定义负载方法
  
  springboot-dubbo使用自定义负载其实很简单，大致分为如下几步：

  1. 创建自定义负载类，继承AbstractLoadBalance，重写doSelect方法，这个方法就是定义算法规则的地方。
  2. 添加dubbo负载扩展点，在src/main/resources目录下创建META-INFO/dubbo目录，在目录下创建org.apache.dubbo.rpc.cluster.LoadBalance文件，里面配置对应的负载算法类，如下：
     ```properties
     gray=com.dalaoyang.balance.GrayLoadBalance
     ```
  3. 配置文件中使用，如下：
     ```properties
     dubbo.provider.loadbalance=gray
     ```

## 代码实现
  ```java
    /**
     * 灰度负载均衡，消费者根据 dubbo.gray 属性和 dubbo.gray.filter 属性来判断是否走灰度服务，
     * 灰度服务需要设置属性：dubbo.provider.parameters.status=gray
     */
    @Component
    @Slf4j
    @ConditionalOnBean(name = "grayFilter")
    public class GrayLoadBalance extends AbstractLoadBalance implements EnvironmentAware {
    
        private static final String GRAY = "gray";
    
        private static final Map<String, Pair<Integer, Class[]>> methodMap = new ConcurrentHashMap<>();
    
        private Environment environment;
    
        @Override
        public void setEnvironment(Environment environment) {
            this.environment = environment;
        }
    
        @Override
        protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            boolean isGray = environment.getProperty("dubbo.gray", Boolean.class, Boolean.FALSE);
    
            List<Invoker<T>> list = new ArrayList<>();
            for (Invoker invoker : invokers) {
                list.add(invoker);
            }
    
            Object grayKeyValue = null;
    
            Object grayFilter = SpringUtil.getBean("grayFilter");
            if (grayFilter instanceof GrayFilter) {
                grayKeyValue = getGrayKeyValue(url, invocation);
            }
    
            boolean routeGray = (grayFilter instanceof GrayFilter && ((GrayFilter) grayFilter).isGray(grayKeyValue)) || isGray;
    
            List<Invoker<T>> grayInvokers = new ArrayList<>();
    
            Iterator<Invoker<T>> iterator = list.iterator();
            while (iterator.hasNext()) {
                Invoker<T> invoker = iterator.next();
                String providerStatus = invoker.getUrl().getParameter("status", "prod");
    
                //如果是灰度请求数据，走灰度
                if (routeGray) {
                    if (Objects.equals(providerStatus, GRAY)) {
                        grayInvokers.add(invoker);
                        continue;
                    } else {
                        //如果是灰度请求，移除非灰度服务
                        iterator.remove();
                        continue;
                    }
                } else {
                    if (Objects.equals(providerStatus, GRAY)) {
                        //如果不是灰度请求，移除灰度服务
                        iterator.remove();
                        continue;
                    }
                }
            }
    
            if (routeGray) {
                if (grayInvokers.isEmpty()) {
                    throw new RuntimeException("No available " + (routeGray ? "gray" : "prod") + " dubbo provider");
                } else {
                    return randomSelect(grayInvokers, url, invocation);
                }
            } else {
    
                if (list.isEmpty())
                    throw new RuntimeException("No available " + (routeGray ? "gray" : "prod") + " dubbo provider");
                else {
                    return randomSelect(list, url, invocation);
                }
            }
        }
    
        private Object getGrayKeyValue(URL url, Invocation invocation) {
            Object grayKeyValue = null;
            String clazz = url.getParameter("interface");
            String fullName = clazz + "." + invocation.getMethodName();
            if (methodMap.get(fullName) != null && matchCache(methodMap.get(fullName), invocation.getParameterTypes())) {
                int index = methodMap.get(fullName).getLeft();
                if (invocation.getArguments() != null && invocation.getArguments().length > index) {
                    grayKeyValue = invocation.getArguments()[index];
                }
            } else {
                try {
                    Method method = Class.forName(clazz).getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    GrayKey grayKey = method.getAnnotation(GrayKey.class);
    
                    if (grayKey != null) {
                        String key = grayKey.name();
                        if (!StringUtils.isEmpty(key)) {
                            Parameter[] parameters = method.getParameters();
                            if (parameters != null) {
                                for (int i = 0; i < parameters.length; i++) {
                                    if (parameters[i].getName().equals(key)) {
                                        grayKeyValue = invocation.getArguments()[i];
                                        break;
                                    }
                                }
                            }
                        }
    
                        if (invocation.getArguments() != null && invocation.getArguments().length > grayKey.index()) {
                            grayKeyValue = invocation.getArguments()[grayKey.index()];
                            methodMap.put(fullName, new ImmutablePair<>(grayKey.index(), invocation.getParameterTypes()));
                        }
                    }
                } catch (NoSuchMethodException e) {
                    log.error("Can't find method {} at class {}", invocation.getMethodName(), clazz, e);
                } catch (ClassNotFoundException e) {
                    log.error("Can't find class {}", clazz, e);
                }
            }
            return grayKeyValue;
        }
    
        private boolean matchCache(Pair<Integer, Class[]> integerPair, Class<?>[] parameterTypes) {
            return Arrays.equals(integerPair.getRight(), parameterTypes);
        }
    
    
        /**
         * 重写了一遍随机负载策略
         *
         * @param invokers
         * @param url
         * @param invocation
         * @param <T>
         * @return
         */
        private <T> Invoker<T> randomSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            int length = invokers.size();
            boolean sameWeight = true;
            int[] weights = new int[length];
            int firstWeight = this.getWeight((Invoker) invokers.get(0), invocation);
            weights[0] = firstWeight;
            int totalWeight = firstWeight;
    
            int offset;
            int i;
            for (offset = 1; offset < length; ++offset) {
                i = this.getWeight((Invoker) invokers.get(offset), invocation);
                weights[offset] = i;
                totalWeight += i;
                if (sameWeight && i != firstWeight) {
                    sameWeight = false;
                }
            }
    
            if (totalWeight > 0 && !sameWeight) {
                offset = ThreadLocalRandom.current().nextInt(totalWeight);
    
                for (i = 0; i < length; ++i) {
                    offset -= weights[i];
                    if (offset < 0) {
                        return (Invoker) invokers.get(i);
                    }
                }
            }
            return (Invoker) invokers.get(ThreadLocalRandom.current().nextInt(length));
        }
    }

  ```
     