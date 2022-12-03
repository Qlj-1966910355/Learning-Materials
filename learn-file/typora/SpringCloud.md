### 一、分布式架构

#### 1. **分布式架构需要考虑的问题**

- 服务拆分粒度<如何做业务拆分>？
- 服务集群地址如何维护？
- 服务之间如何通讯？
- 服务健康状态如何感知？

#### 2. **微服务架构特征**

- 单一职责：微服务拆分粒度更小，每个服务都对应唯一的业务能力，做到单一职责，避免重复业务开发。
- 面向服务：微服务对外暴露业务接口。
- 自治：团队独立、技术独立、数据独立、部署独立。
- 隔离性强：服务调用做好隔离、容错、降级、避免出现级联问题。

#### 3. **微服务技术栈**

- Dubbo模式

  基于Dubbo的技术体系；服务接口采用Dubbo协议标准；服务调用采用Dubbo方式

- SpringCloud+Feign

  使用SpringCloud技术栈；服务接口采用Restful风格；服务调用采用Feign方式

- SpringCloudAlibaba+Feign

  使用SpringCloudAlibaba技术栈；服务接口采用Restful风格；服务调用采用Feign

- SpringCloudAlibaba+Dubbo

  使用SpringCloudAlibaba技术栈；服务接口采用Dubbo协议标准；服务调用采用Dubbo方式

#### 4. SpringCloud功能组件

- 服务注册发现<Eureka、Nacos、Consul>
- 服务远程调用<OpenFeign、Dubbo>
- 服务链路监控<Zipkin、Sleuth>
- 统一配置管理<SpringCloudConfig、Nacos>
- 统一网关路由<SpringCloudGateway、Zuul>
- 流控、降级、保护<Hystix、Sentinel>

#### 5. SpringCloud与SpringBoot版本兼容关系

	<左：SpringCloud版本；右：SpringBoot版本>
	2020.0.x aka llford			 2.4.x
	Hoxton					   2.2.x、2.3.x(Starting with SR5)
	Greenwith					2.1.x
	Finchley					2.0.x
	Edgware						1.5.x
	Dalston						1.5.x

### 二、服务的拆分与远程调用

#### 1. 拆分

- 不同微服务，不要重复开发相同业务
- 微服务数据独立，不要访问其他微服务的数据库
- 微服务可以将自己的业务暴露为接口，供其他微服务调用

#### 2. 远程调用

<使用Spring框架提供的远程调用类==`RestTemplate`==>(类似于HttpClient功能)

##### 2.1 ==RestTemplate==介绍

RestTemplate是由Spring框架提供的一个可用于应用中调用rest服务的类它简化了与http服务的通信方式，统一了RESTFul的标准，封装了http连接，我们只需要传入url及其返回值类型即可。相较于之前常用的HttpClient，RestTemplate是一种更为优雅的调用RESTFul服务的方式。

RestTemplate类的设计原则与许多其他Spring的模板类(例如JdbcTemplate)相同，为执行复杂任务提供了一种具有默认行为的简化方法。

RestTemplate默认依赖JDK提供了http连接的能力（HttpURLConnection），如果有需要的话也可以通过setRequestFactory方法替换为例如Apache HttpCompoent、Netty或OKHttp等其他Http libaray。

考虑到了RestTemplate类是为了调用REST服务而设计的，因此它的主要方法与REST的基础紧密相连就不足为奇了，后者时HTTP协议的方法：HEAD、GET、POST、PUT、DELETE、OPTIONS例如，RestTemplate类具有headForHeaders()、getForObject()、putForObject()，put()和delete()等方法。

##### 2.2 使用RestTemplate远程调用

- 注册RestTemplate

​	在消费者服务(Order服务)中注册RestTemplate。

```java
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication
public class OrderApplication {
	// 项目main方法
	public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
	}
	// 注册RestTemplate
	@Bean
	public RestTemplate restTemplate(){
		return new RestTemplate();
	}
}
```

- 利用RestTemplate对服务提供者发送请求

  - 首先注入RestTemplate类

  ```java
  @Autowired
  private RestTemplate restTemplate;
  ```

  - 在业务方法service中发送请求

  ```java
  // 1.查询订单
  Order order = orderMapper.findById(orderId);
  // 2.利用RestTemplate发起http请求，查询用户
  // 2.1.url路径
  String url = "http://localhost:8081/user/" + order.getUserId();
  // 2.2.发送http请求，实现远程调用
  User user = restTemplate.getForObject(url, User.class); // 参数2表示：请求响应返回值类型<json自动变成User>
  // 封装User对象到Order
  order.setUser(user);
  // 4.返回
  return order;
  ```

  **注意**：

  ​	重要的步骤是如何发送并响应信息。<步骤2>

  ​	url地址就是当前服务对服务提供者发送请求的地址，参数order.getUserId()是请求参数，是一种restful风格的调用方式。

  ​	getForObject()方法采用get请求类型进行发送。

  ​	User.class表示响应数据类型。

##### 2.3 服务提供者与消费者

提供者：接收请求并响应消费者所需的服务称为提供者。<暴露接口>

消费者：发送请求的服务称为消费者。<调用接口>

**注意**：

​	一个服务既可以是提供者又可以使消费者。<例：A 发送请求给 B，B 发送请求给 C。此时B既是提供者又是消费者>

### 三、==Eureka==注册中心

> 以上远程调用出现的问题：
>
> - url地址直接硬编码在服务中。在多环境的项目中，ip地址发生变化。在多服务项目中，一个业务可能有多个服务，端口号不同。

#### 1. eureka的作用

- 消费者如何获取服务提供者的信息？
  - 服务提供者启动时向eureka注册自己的信息；
  - eureka保存这些信息；
  - 消费者根据服务名称向eureka拉取提供者信息。
- 多个服务提供者，消费者该如何选择？
  - 服务消费者利用负载均衡算法，从服务列表中挑选一个。
- 消费者如何感知服务提供者的健康状态？
  - 服务提供者每个30s向EurekaServer发送心跳请求，报告健康状态；
  - eureka会更新记录服务列表信息，心跳不正常就会剔除；
  - 消费者就可以拉取到最新的信息。

#### 2. eureka架构角色

- EurekaServer：服务端，注册中心

  - 记录服务信息；
  - 心跳监控

- EurekaClient：客户端

  - Provider：服务提供者

    注册自己的信息到EurekaServer

    每个30s向EurekaServer发送心跳

  - Consumer：服务消费者

    根据服务名称从EurekaServer拉取服务列表

    基于服务列表做负载均衡，选中一个微服务后发起远程调用

#### 3. 搭建EurekaServer并注册服务

##### 3.1 创建一个eureka的微服务

- 在父工程下创建eureka服务

​	项目结构-->modules-->添加maven结构-->直接下一步(不勾选模板)-->选择父工程'parent'，填写名称-->完成

- 引入eureka服务端依赖

```xml
<dependencies>
    <!--eureka服务端-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

- 添加项目启动类

  在Java目录下添加启动类：com.sg.eureka.EurekaApplication

```java
@EnableEurekaServer     // Eureka自动装配开关
@SpringBootApplication
public class EurekaApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}
}
```

- 添加application.yml配置文件

```yaml
server:
	port: 10086 #服务端口
spring:
	application:
		name: eurekaserver    #当前微服务eureka名称
eureka:
	client:
		service-url:
			#eureka也是一个微服务，所以eureka将自己添加进eureka中
			defaultZone: http://127.0.0.1:10086/eureka  #多个eureka集群互相交流<当前是单机模式，此处应该是eureka集群地址>
```

- 通过启动类启动eureka服务

  浏览器访问：http://localhost:10086/

**注意**：

​	eureka微服务自己也是需要eureka进行管理的，所以才需要配置yml中信息。

​	访问页面中'Instances currently registered with Eureka'是eureka中当前管理的服务。<此时只有eureka这一个服务>

##### 3.2 将提供者与消费者服务注册到eureka中

- 提供者服务配置

  - 导入eureka客户端依赖

  ```xml
  <!--eureka客户端依赖-->
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

  - 配置eureka服务地址与当前服务名称

  ```yaml
  spring:
  	application:
  		name: userservice    #当前微服务user名称
  eureka:
  	client:
  		service-url:
  			defaultZone: http://127.0.0.1:10086/eureka
  ```

- 消费者服务配置

  - 导入eureka客户端依赖

  ```xml
  <!--eureka客户端依赖-->
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

  - 配置eureka服务地址与当前服务名称

  ```yaml
  spring:
  	application:
  		name: orderservice    #当前微服务order名称
  eureka:
  	client:
  		service-url:
  			defaultZone: http://127.0.0.1:10086/eureka
  ```

  **注意**：yml文件中不允许有同名的配置名称。所以要进行同名称合并。

##### 3.3 刷新/重启所有微服务。

此时eureka页面中有三个服务。<注意：可能会存在浏览器缓存问题，刷新页面时先清理浏览器缓存>

#### 4. 如何实现多实例(服务)运行

也就是说，将提供者服务进行多次启动，实现多实例部署，每个实例的端口号必须不同，要修改端口号。

<多实例就可以进行负载均衡>

例：
	需要多个userservice服务实例
步骤：

- 在idea最底部工具栏中，打开'Services'栏。
- 找到UserApplication启动类，右键‘copy configuration...’
- 修改名称<不重复即可>
- 在'Environment'栏的'VM options'(虚拟机选项)中添加“-Dserver.port=8082”(不冲突即可)
- 保存
- 此时在'Services'栏中多出一个未运行的服务，执行即可。
  	<也可以在idea顶部'编辑配置'中copy服务。>

**注意**：
	运行后，重新刷新eureka网页，可以发现‘USERSERVICE’服务的‘Status’中有两个实例。

#### 5. 消费者orderservice从注册中心拉取服务信息

服务拉取是基于服务名称获取服务列表，然后在对服务列表做负载均衡。<上面的userservice就是一个服务名称两个实例>

步骤：

- 修改代码

​	<在未使用注册中心eureka之前，我们使用了RestTemplate进行远程访问调用。此时我们采用eureka进行拉取服务信息>

```java
将:
	String url = "http://localhost:8081/user/" + order.getUserId();
修改为：
	String url = "http://userservice/user/" + order.getUserId();		// 采用微服务名称代替ip与端口号
```

- 在消费者服务的启动类中的RestTemplate添加负载均衡注解

```java
@Bean
@LoadBalanced			// 负载均衡注解<作用：拦截RestTemplate发送的请求>
public RestTemplate restTemplate(){
	return new RestTemplate();
}
```

**测试**：
	重新启动order服务，清空浏览器缓存，向order发送一个请求(这个请求到消费者服务器后，消费者服务器拉取提供者服务的信息)。
	**注意**：
		想要测试哪个实例被调用，可以清空idea中两个user提供者实例的执行日志，当前端执行成功，提供者实例就会执行，执行后就会产生日志。

#### 6. 总结

- 搭建EurekaServer
  - 引入eureka-server依赖
  - @EnableEurekaServer注解
  - 在application.yml中配置eureka地址

- 服务注册
  - 引入eureka-client依赖
  - 在application.yml中配置eureka地址

- 服务拉取/发现
  - 引入eureka-client依赖
  - 在application.yml中配置eureka地址
  - 给RestTemplate添加@LoadBalanced注解
  - 用服务提供者的服务名称远程调用

### 四、==Ribbon==负载均衡原理

> 学习文章：https://blog.csdn.net/tf835991342/article/details/122236383

小问题：我们知道，在消费者服务中采用的是服务提供者的服务名称代替了ip与端口号，那是否可以直接在浏览器执行这个被代替后的地址呢？

​	修改前可以，修改后不可以。

​	原因：服务名不是一个ip或地址，就算发给服务器也解析不了，何况浏览器解析不了也发不过去

那么消费者服务代码修改后，它是如何将http://userservice/user/解析出来并选择一个提供者实例的？

​	原因往下看。

#### 1. 负载均衡流程

- 消费者服务发起请求http://userservice/user/1
- 进入eureka-server中拉取userservice
- 返回指定服务名称对应的实例列表：localhost:8081、localhost:8082
- 轮询选择一个实例进行访问并响应

<img src=".\img\image-20221201150813927.png" alt="image-20221201150813927" style="zoom:75%;" />

<img src=".\img\image-20221202075239826.png" alt="image-20221202075239826" style="zoom:75%;" />

#### 2. Ribbon在哪里进行拦截了？

还记得我们在消费者服务中的启动类中配置了RestTemplate吗，我们在RestTemplate上添加了==`@LoadBalanced`==注解。

该注解作用就是`拦截RestTemplate发送的请求`。

#### 3. 基本流程

以消费者服务中的http://userservice/user/1请求为例

- **LoadBalancerIntercepor**拦截我们的RestTemplate请求http://userservice/user/1
- **RibbonLoadBalancerClient**会从请求url中获取服务名称，也就是user-service
- **DynamicServerListLoadBalancer**根据user-service到eureka拉取服务列表
- eureka返回列表，localhost:8081、localhost:8082
- **IRule**利用内置负载均衡规则<默认轮询>，从列表中选择一个，例如localhost:8081
- **RibbonLoadBalancerClient**修改请求地址，用localhost:8081替代userservice，得到http://localhost:8081/user/1，发起真实请求

<img src=".\img\image-20221202075320018.png" alt="image-20221202075320018" style="zoom:80%;" />

#### 4. 轮询的负载均衡算法

rest接口第几次请求数 % 服务器集群总数量 = 实际调用服务器位置下标 ，每次服务重启动后rest接口计数从1开始。

#### 5. 修改负载均衡规则

<常见：RoundRobinRule(轮询)、RandomRule(随机)、RetryRule(重试)...>(Rule接口有7个实现类)

通过定义`IRule`实现可以修改负载均衡规则，有两种方式：

- 方式一

​	在消费者服务中的启动类中定义一个新的IRule。

```java
// 随机负载均衡
@Bean
public IRule randomRule(){
	return new RandomRule();
}
```

​	**注意**：这种方式`全局配置`，只要是在该消费者服务中实现RestTemplate请求，都是这种负载均衡方式。

- 方式二

​	在配置文件消费者服务的.yml中配置。

```yaml
#修改IRule实现的负载均衡方式
userservice:		#这个是指定的服务提供者名称
	ribbon:
		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule  #'随机'负载均衡
```

​	**注意**：这种配置方式，可以指定一个服务提供者进行配置。

#### 6. Ribbon负载均衡'饥饿加载'

Ribbon默认采用的是懒加载，即第一次访问时才会创建LoadBalanceClient，请求时间很长。

饥饿加载：

​	在项目启动时就会创建，降低第一次访问的耗时。

配置饥饿加载：

```yaml
<在消费者服务中的yml中进行配置>ribbon:
eager-load:
	enabled: true     #开启饥饿加载
	clients: userservice    #指定对userservice这个服务饥饿加载
```

### 五、==Nacos==注册中心

> 官网：https://nacos.io/zh-cn/		文档：https://nacos.io/zh-cn/docs/what-is-nacos.html
>
> 下载：
> 		https://github.com/alibaba/nacos --> releases --> Tags --> 选择一个版本下载			
>
> ​		https://github.com/alibaba/nacos/tags
>
> 安装：直接解压即可
>
> windows启动：在bin目录下的startup.cmd启动文件。
>
> ​	管理员模式下在当前目录下，执行：`startup.cmd -m standalone`		// 单机启动<非集群模式>
>
> 客户端：
>
> ​	在cmd中启动成功后，nacos图标后有客户端网页链接地址，直接访问这个html即可。
>
> ​	账号：nacos	密码：nacos

#### 1. 将服务注册在Nacos中

- 在父工程的pom文件中添加spring-cloud-alibaba的管理依赖

```xml
<!--引入spring-cloud-alibaba管理依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>
```

- 如果你是修改上面的项目，需要注释掉原有的eureka依赖<如果不是可以忽略该步骤>

- 添加nacos的客户端依赖<服务提供者与消费者都要添加>

```xml
<!--nacos-client依赖-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	<version>2.2.5.RELEASE</version>
</dependency>
```

- 修改服务提供者与消费者的yml文件，添加nacos地址、微服务名称

```yaml
spring:
	application:
		name: userservice   #当前微服务的名称
	cloud:
		nacos:
			server-addr: localhost:8848  #nacos服务地址
```

- 消费者服务中使用RestTemplate发送请求

  - 在消费者服务(Order服务)的启动类中注册RestTemplate。

  ```java
  // 注册RestTemplate
  @Bean
  @LoadBalanced		// 负载均衡注解<作用：拦截RestTemplate发送的请求>
  public RestTemplate restTemplate(){
  	return new RestTemplate();
  }
  ```

  - 利用RestTemplate发送远程请求<与eureka一样>

  ```java
  String url = "http://userservice/user/" + order.getUserId();
  User user = restTemplate.getForObject(url, User.class);
  ```

- 启动测试

​	启动后，在nacos网页客户端中查看启动后的服务已经注册到注册中心。

#### 2. Nacos服务分级存储模型<集群>

场景：

​	对于一个服务可以部署多个实例，但不能将多个实例放在同一个地方，因此需要将多个实例部署到多个'机房'<容灾>。

​	例：上海、西安、北京三个位置都部署了一个服务的实例，且一个位置有多个实例，每个位置构成了一个集群，多个位置有多	个集群，共同协调。

分级：

​	服务(一级) --> 集群(二级) --> 实例(三级)

##### 2.1 服务跨集群调用问题

服务调用时尽可能优先选择本地集群的实例，跨集群调用延迟较高。本地集群不能调用时才去访问其他集群实例。

##### 2.2 服务集群属性配置

<目前userservice服务提供者有3个实例，将两个放在一个集群，剩余一个放在一个集群>

1. 修改application.yml配置信息

```properties
spring.cloud.nacos.discovery.cluster-name=xian		// 注意合并同名称<修改为yml格式>
```

**注意**：配置完成后，如果想让user1与user2组成一个集群，则重启这两个实例。重新将'xian'集群名称修改为'weinan'，重启user3实例。此时，集群'xian'中有两个实例，集群'weinan'中有一个实例。<配置好'weinan'名称后不要重启user1和user2>

2. 在Nacos网页监控页面查看集群变化。

<目前orderservice服务消费者想要获取本地集群中的实例，所以也需要配置集群属性>

1. 配置orderservice服务实例所在的位置，按照userservice集群名称配置。

​	例：消费者服务目前位于'xian'，所以就想调用userservice的'xian'集群实例。

```properties
spring.cloud.nacos.discovery.cluster-name=xian    #当前消费者服务集群名称<与服务提供者某个集群名称一致>
```

​	注意：

​		我们知道消费者访问提供者时完全由Ribbon决定，选择实例也由IRule的负载均衡规则决定，所以此时order访问时还是进		行'轮询'访问。

​		目前并不能优先调用本地集群的提供者实例。

2. 设置负载均衡的IRule为NacosRule，这个规则优先会寻找与自己同集群的服务<优先随机调用同集群实例>

```yaml
userservice:		#这个是指定的服务提供者名称
	ribbon:
		NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule	#负载均衡规则(优先会寻找与自己同集群的服务)
```

测试：

​	将所有提供者服务实例开启，调用发现，消费者只会调用当前同集群的提供者服务，而且是随机调用。

​	停止同集群下的提供者服务实例，发现第一次访问会找不到外地集群位置，再次访问就会成功访问。

##### 2.3 按权重负载均衡

<服务器设备性能有差异，部分实例所在机器性能较好，另一些较差，我们希望性能好的机器承担更多的用户请求。>

==Nacos提供了权重配置来控制访问频率，权重越大则访问频率越高==

- 设置服务实例权重

  - 在Nacos网页控制台可以设置实例的权重值，首先选中实例后面的'编辑'按钮。
  - 将权重设置为0.1，测试可以发现8081被访问到的频率大大降低。

  **注意**：权重值在[0,1]之间；设置为0后不会访问；同集群内的多个实例，权重越高被访问的频率越高。

  ​		  默认情况下所有实例权重为1。<与其说是权重，不如说时调用实例的概率>

- 使用场景<平滑升级>

  在单实例项目中想要升级应用/程序时，必须下线后才能再次替换。而在多实例中，可以调节权重，权重调节为0后，用户就不会进行访问，此时可以进行程序升级，但不会影响用户访问其他实例。升级完成后，重新设置权重为0.1，此时放进来一少部分用户进行测试，成功后再调节权重完成升级。

#### 3. Nacos环境隔离-namespace

> 学习文章：https://www.kancloud.cn/qingshou/aaa1/2667182

##### 3.1 结构

namespace 包含 group 包含 service/dataid

<img src=".\img\image-20221201180902204.png" alt="image-20221201180902204" style="zoom:50%;" />

- 一个namespace命名空间可以包含多个分组
- 一个分组可以有多个子项目或者子模块的微服务
- 每个子项目有自己的dataid，dataId命名规则带后缀，如：xxxx-dev.yaml、xxxx-pro.yaml，可以产生对某一个子项目部署环境配置文件加载隔离的效果。

##### 3.2 创建并使用namespace

微服务在没有明确指定配置的情况下， 默认使用的是Public命名空间。

- 首先创建一个namespace

  在nacos网页控制台中创建namespace

  步骤：

  ​	左侧栏'命名空间' --> 新建命名空间 --> 填写命名空间名与描述，id可以不填<默认会指定>(例：dev 开发环境) --> 确定

- 使用创建的dev命名空间

  在服务中配置服务的namespace。

  <可以服务隔离配置与配置隔离配置>

  - 服务隔离配置：微服务属于不同的namespace，彼此之间不能远程调用

  ```properties
  spring.cloud.nacos.discovery.namespace=5279d5f1-eb9d-4ef4-abc1-86293aeb38a9		// 命名空间
  ```

  - 配置隔离配置：同一个namespace下的配置文件可以为该namespace下的多个子项目共享，跨namespace则无法实现配置文件共享

  ```properties
  spring.cloud.nacos.config.namespace=5279d5f1-eb9d-4ef4-abc1-86293aeb38a9
  ```

  下面只测试服务隔离配置，配置隔离不使用。

- 测试
  - 我们只在消费者服务配置了命名空间，看一下namespace的效果。
  - 重启消费者服务orderservice，刷新nacos网页控制台，可以看到服务列表中的dev模块下有orderservice服务。
  - 向消费者服务发送请求，测试是否可以执行成功<不同的namespace，彼此之间不能远程调用>
    - 此时出现500错误，由于namespace不同，会导致找不到userservice。

> **注意**：
>
> - 每个namespace都有唯一id
> - 服务配置namespace时要写id而不是名称
> - 不同namespace下的服务互相不可见

#### 4. Nacos与eureka的原理区别

<img src=".\img\image-20221202075003758.png" alt="image-20221202075003758" style="zoom:75%;" />

- nacos注册中心感知服务健康状态时分为临时实例与非临时实例感知。
  - 临时：<服务向nacos发送心跳>服务被动/主动关闭后，nacos接收的心跳信息异常，直接下线对应服务
  - 非临时：<nacos主动获取服务心跳>如果发现服务异常关闭，处于不健康状态，则不会将服务删除下线，只会标记不健康。
  - <非临时实例需要配置，且若配置后不删除这个实例一直在nacos中>
  - <eureka只有临时实例。>

- 消费者服务会拉取nacos服务列表，并形成缓存，不会每次请求都要获取列表。
  - 服务列表缓存会定时进行更新。更新分为两种：
    - <pull>消费者定时拉取nacos中的服务列表。
    - <push>nacos中的服务发生变更，主动向消费者服务发送变更信息，及时更新缓存列表。
  - eureka只能做拉取操作。
- nacos集群默认采用AP模式，当集群中存在非临时实例时，采用CP模式。
  - eureka只有AP模式。

非临时实例配置：

```yaml
# 在yml中配置
spring.cloud.nacos.discovery.ephemeral=false	# 设置非临时实例
```

### 六、Nacos统一配置管理

#### 1. 基本介绍

在单体应用中，配置管理可能不是什么大的事情，通常会以配置文件的方式。常见的方法比如将配置通过打包脚本打入应用包中，或者直接放到运行应用的服务器的特定目录下，或者存储到数据库中。这种方式在传统的单体应用中简单有效，但是也会有些比较棘手的问题。

> 例如：
>
> - 配置变化频繁时，需要频繁的打包部署应用。
> - 不同环境的配置需要分开管理(比如测试环境与生产环境)。
> - 而在分布式微服务架构中，服务数量剧增，如果还是手动去实现配置信息的修改或数据的迁移等，效率是很低的，而且手动操作配置也极有可能出现错误的情况。

复杂的业务对应大量的配置项，对集群部署的应用配置进行修改时需要修改每个节点上的应用配置，在这种背景下，中心化的配置服务即配置中心应运而生。

**配置中心**：

> 配置中心就是一种统一管理各种应用配置的基础服务组件，配置中心可以把业务开发者从复杂以及繁琐的配置中解脱出来，只需专注于业务代码本身，从而能够显著提升开发以及运维效率。同时将配置和发布包解藕也进一步提升发布的成功率，并为运维的细力度管控、应急处理等提供强有力的支持。

在微服务架构中，微服务的统一配置管理一般有以下需求：

- 集中管理配置：一个使用微服务架构的应用系统可能会包括成千上万个微服务，因此集中管理配置是非常有必要的。
- 不同环境不同配置：例如，数据源配置在不同的环境（开发、测试、预发布、生产等）中是不同的。
- 运行期间可动态调整：例如，可根据各个微服务的负载情况，动态调整数据源连接池大小或熔断阈值，并且在调整配置时不重启微服务。
- 配置修改后自动更新：如配置内容发生变化，微服务能够自动更新配置。

综上所述，对于微服务架构而言，一个通用的配置管理机制必不可少，常见做法是使用配置服务器管理配置。

#### 2. 实现配置管理

##### 2.1 在nacos中创建配置文件

- 在nacos网页客户端中，打开左侧'配置管理'栏 --> '配置列表'

- 点击右边'+'，进行配置。

  - data id：一般使用'服务名称-运行环境.yaml'名称，例：userservice-dev.yaml

  - group：分组，默认即可

  - 描述：随意。例：userservice的开发环境配置文件。

  - 配置格式：yaml

  - 配置内容：

    <注意：配置内容是将来用于热更新的，所以配置内容是一些经常进行修改的配置信息。例：数据库地址一般不需要热更新>

    例：配置日期格式

    ```yaml
    pattern:
    	dateformat: yyyy-MM-dd HH:mm:ss		// 日期格式配置 <以后可以更改此处实现日期的热更新>
    ```

##### 2.2 服务读取创建的配置文件

问题：spring是在什么时候读取项目中的application.yml文件的？

答：项目启动 ---> 读取项目配置文件application.yml ---> 创建Spring容器 ---> 加载Bean对象

> **nacos配置文件读取流程**：
>
> ​	项目启动 ---> 先读取nacos配置文件 ---> 读取项目配置文件application.yml ---> 创建Spring容器 ---> 加载Bean对象

思考：既然优先读取的是nacos配置文件，而nacos的地址却在application.yml中配置着，想要读取到外部的nacos配置文件，肯定是要先知道nacos地址。

解决方案：将nacos地址单独取出来，放到读取nacos配置文件之前。

​	spring中提供了一个bootstrap.yml文件，这个配置文件spring会优先读取。

> **所以**：
>
> ​	**项目读取nacos配置文件流程**：
>
> - 项目启动 ---> 读取bootstrap.yml文件 ---> 读取nacos配置文件 ---> 读取项目配置文件application.yml ---> 创建Spring容器 ---> 加载Bean

步骤：

- 引入nacos配置管理客户端依赖<在userservice服务中>

```xml
<!--nacos配置管理客户端依赖-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
	<version>2.2.5.RELEASE</version>
</dependency>
```

- 在userservice中的resource目录添加一个bootstrap.yml文件，这个文件是引导文件，优先级高于application.yml

```yaml
spring:
  application:
	name: userservice   #服务名称
  profiles:
	active: dev   #开发环境
  cloud:
	nacos:
	  server-addr: localhost:8848  #nacos服务地址
	  config:
		file-extension: yaml  #文件后缀名
```

- 删除application.yml中重复的配置信息

​	例：当前微服务的名称；nacos服务地址。<因为bootstrap中已经存在>

- 在controller中写一个接口，用于测试nacos配置文件是否能够使用。

  - 注入nacos配置文件的配置信息，使用dateformat接收

  ```java
  @Value("${pattern.dateformat}")
  private String dateformat;
  ```

  - 编写接口 (配置信息实际是'yyyy-MM-dd HH:mm:ss')

  ```java
  @GetMapping("/now")
  public String now() {
  	return new SimpleDateFormat(dateformat).format(new Date());
  }
  ```

- 重启服务，测试是否正常响应

​	浏览器输入：http://localhost:8081/user/now

###### 2.2.1 实现配置文件热更新

<虽然在nacos配置管理中创建了配置信息，但是如果修改后，依然不能实施生效>

- 方式一

  通过`@Value`注入后，结合注解`@RefreshScope`刷新配置。

  > 例：
  > 	在controller类上面添加：
  > 		**@RefreshScope**           //配置文件热更新<配合@Value一起使用>

- 方式二

  通过 `@ConfigurationProperties` 注入配置，同时配合 `@Component` 将配置交给spring管理生效，自动刷新。

  1. 首先创建一个配置类，专门用于接收nacos配置信息。

  ```java
  @Data
  @Component
  @ConfigurationProperties(prefix = "pattern")		//  prefix 绑定配置文件中的配置，相当于捕捉到外部的配置信息
  public class NacosProperties {
  	private String dateformat;
  }
  ```

  > **注意**：
  >
  > prefix参数拼接属性名称，必须与配置信息一致。配置类名称不能为"NacosProperties"。
  >
  > 进行Spring Boot配置文件部署时，发出警告Spring Boot Configuration Annotation Processor not configured，但是不影响运行。
  >
  > <添加spring-boot-configuration-processor依赖即可解决>

  2. 将配置类使用@Autowired注入到controller类中，使用get方法获取这个实体类接收的配置信息。

  ```java
  @Autowired
  private NacosDateProperties nacosProperties;
  ...
  @GetMapping("/now")
  public String now() {
  	return new SimpleDateFormat(nacosProperties.getDateformat()).format(new Date());
  }
  ```

  > **注意**：
  >
  > 不是所有的配置都适合放到配置中心，维护起来比较麻烦。
  >
  > 建议将一些关键参数，需要运行时调整的参数放到nacos配置中心，一般都是自定义配置。

#### 3. 多环境配置共享

场景：项目在多个环境下，不需要对每个环境都进行配置nacos配置信息，也可以创建一个，多环境都会共享的配置文件<只需配置一次，环境切换不影响>。

微服务启动时会从nacos读取多个配置文件：

- 指定环境下的配置文件：服务名称-环境名.yaml。<例：userservice-dev.yaml>
- 多环境配置文件共享：服务名称.yaml。<例：userservice.yaml>

**步骤**：

- 创建nacos配置文件

  - Data ID：userservice；

  - 描述：userservice的多环境共享配置

  - 格式：yaml

  - 配置内容：

    ```yaml
    pattern:
    	girlFriend: zmq
    ```

- 读取配置信息

  在实现热部署的配置文件中'NacosProperties.class'添加girlFriend属性。<prefix都是pattern>

- 在controller中写一个前后端接口，用于测试配置文件正常加载输出。

  ```java
  @GetMapping("/girl")
  public String girl(){
  	return nacosProperties.getGirlFriend();
  }
  ```

- 将userservice的bootstrap.yml配置文件中原有'dev'环境换成'test'环境。

- 测试结果

  nacos中的userservice-dev.yaml配置文件不能在test环境下执行。

  而userservice.yaml配合文件可以正常加载输出。<因为它是多环境共享的配置文件>

> **问题：**
>
> 如果在dev环境中userservice-dev.yaml、userservice.yaml、本地环境application.yml中存在同名称的配置信息，三个配置文件会优先执行哪一个？
>
> 优先级：
>
> - userservice-dev.yaml(nacos环境配置) > userservice.yaml(nacos共享配置) > application.yml(项目本地配置)
>
> 即：
>
> - 远程/线上配置 > 本地配置		环境配置 > 共享配置

#### 4. 多服务共享配置

不同微服务之间可以共享配置文件。

- 方式一
  - 通过shared-configs指定
- 方式二
  - 通过extension-configs指定

优先级：

- 服务名-profile.yaml > 服务名称.yaml > extension-config > shared-config > 本地配置

### 七、Nacos集群搭建

> 具体可以查看：Nacos集群搭建.md

#### 1. 集群搭建步骤

- **搭建Mysql集群并初始化数据库表**

  Nacos默认数据存储在内嵌数据库Derby中，不属于生产可用的数据库。

  官方推荐的最佳实践是使用带有主从的高可用数据库集群。

  由于没有主从数据库集群，此处使用单个数据库。

  - 创建一个数据库

    我没有创建数据库，直接使用旧的数据库springcloud。

  - 执行sql语句

    数据表结构是由官方提供的。

    ```sql
    CREATE TABLE `config_info` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
      `data_id` varchar(255) NOT NULL COMMENT 'data_id',
      `group_id` varchar(255) DEFAULT NULL,
      `content` longtext NOT NULL COMMENT 'content',
      `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
      `src_user` text COMMENT 'source user',
      `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
      `app_name` varchar(128) DEFAULT NULL,
      `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
      `c_desc` varchar(256) DEFAULT NULL,
      `c_use` varchar(64) DEFAULT NULL,
      `effect` varchar(64) DEFAULT NULL,
      `type` varchar(64) DEFAULT NULL,
      `c_schema` text,
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = config_info_aggr   */
    /******************************************/
    CREATE TABLE `config_info_aggr` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
      `data_id` varchar(255) NOT NULL COMMENT 'data_id',
      `group_id` varchar(255) NOT NULL COMMENT 'group_id',
      `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
      `content` longtext NOT NULL COMMENT '内容',
      `gmt_modified` datetime NOT NULL COMMENT '修改时间',
      `app_name` varchar(128) DEFAULT NULL,
      `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';
    
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = config_info_beta   */
    /******************************************/
    CREATE TABLE `config_info_beta` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
      `data_id` varchar(255) NOT NULL COMMENT 'data_id',
      `group_id` varchar(128) NOT NULL COMMENT 'group_id',
      `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
      `content` longtext NOT NULL COMMENT 'content',
      `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
      `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
      `src_user` text COMMENT 'source user',
      `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
      `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = config_info_tag   */
    /******************************************/
    CREATE TABLE `config_info_tag` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
      `data_id` varchar(255) NOT NULL COMMENT 'data_id',
      `group_id` varchar(128) NOT NULL COMMENT 'group_id',
      `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
      `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
      `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
      `content` longtext NOT NULL COMMENT 'content',
      `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
      `src_user` text COMMENT 'source user',
      `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = config_tags_relation   */
    /******************************************/
    CREATE TABLE `config_tags_relation` (
      `id` bigint(20) NOT NULL COMMENT 'id',
      `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
      `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
      `data_id` varchar(255) NOT NULL COMMENT 'data_id',
      `group_id` varchar(128) NOT NULL COMMENT 'group_id',
      `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
      `nid` bigint(20) NOT NULL AUTO_INCREMENT,
      PRIMARY KEY (`nid`),
      UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
      KEY `idx_tenant_id` (`tenant_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = group_capacity   */
    /******************************************/
    CREATE TABLE `group_capacity` (
      `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
      `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
      `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
      `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
      `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
      `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
      `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
      `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_group_id` (`group_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = his_config_info   */
    /******************************************/
    CREATE TABLE `his_config_info` (
      `id` bigint(64) unsigned NOT NULL,
      `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
      `data_id` varchar(255) NOT NULL,
      `group_id` varchar(128) NOT NULL,
      `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
      `content` longtext NOT NULL,
      `md5` varchar(32) DEFAULT NULL,
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
      `src_user` text,
      `src_ip` varchar(50) DEFAULT NULL,
      `op_type` char(10) DEFAULT NULL,
      `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
      PRIMARY KEY (`nid`),
      KEY `idx_gmt_create` (`gmt_create`),
      KEY `idx_gmt_modified` (`gmt_modified`),
      KEY `idx_did` (`data_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';
    
    
    /******************************************/
    /*   数据库全名 = nacos_config   */
    /*   表名称 = tenant_capacity   */
    /******************************************/
    CREATE TABLE `tenant_capacity` (
      `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
      `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
      `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
      `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
      `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
      `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
      `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
      `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
      `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_tenant_id` (`tenant_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';
    
    
    CREATE TABLE `tenant_info` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
      `kp` varchar(128) NOT NULL COMMENT 'kp',
      `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
      `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
      `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
      `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
      `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
      `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
      PRIMARY KEY (`id`),
      UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
      KEY `idx_tenant_id` (`tenant_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';
    
    CREATE TABLE `users` (
    	`username` varchar(50) NOT NULL PRIMARY KEY,
    	`password` varchar(500) NOT NULL,
    	`enabled` boolean NOT NULL
    );
    
    CREATE TABLE `roles` (
    	`username` varchar(50) NOT NULL,
    	`role` varchar(50) NOT NULL,
    	UNIQUE INDEX `idx_user_role` (`username` ASC, `role` ASC) USING BTREE
    );
    
    CREATE TABLE `permissions` (
        `role` varchar(50) NOT NULL,
        `resource` varchar(255) NOT NULL,
        `action` varchar(8) NOT NULL,
        UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
    );
    
    INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);
    
    INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
    ```

    

- **下载解压nacos**

  为了不破坏之前的nacos文件，此处重新下载解压nacos。

- **修改集群配置(节点信息)，数据库配置**

  进入nacos的conf目录，修改配置文件cluster.conf.example，重命名为cluster.conf。

  - cluster.conf文件中：

  ```yaml
  #多个nacos的ip及端口，将原有的配置删除。
  127.0.0.1:8845
  127.0.0.1.8846
  127.0.0.1.8847
  ```

  - application.properties文件中，添加数据库配置：

  ```properties
  spring.datasource.platform=mysql
  
  db.num=1
  
  db.url.0=jdbc:mysql://127.0.0.1:3306/springcloud?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
  db.user.0=root
  db.password.0=123456
  ```

  > 源文件中将这些注释掉了，直接放开即可。将原有数据库地址及账户密码修改掉即可。<数据库指的是第一步创建的数据库>

- **分别启动多个nacos节点	<记得清理内存>**

  将nacos文件夹复制三份，分别命名为：nacos1、nacos2、nacos3。<想要配置一个3个节点的nacos集群>

  在每个application.properties中配置每个nacos的端口号：

  - nacos1、nacos2、nacos3分别对应端口号：8845、8846、8847	<就是上面配置的>

  在各自的bin目录下启动nacos。

  - 注意：

    此时是`集群模式启动`。<1.4.1默认启动为集群模式>

    命令：startup.cmd

- **nginx反向代理**

  - 下载nginx压缩包，并解压
  - 修改nginx.conf文件，配置如下：	<注意：将配置粘贴到http{}内即可>

  ```conf
  upstream nacos-cluster {
  	server 127.0.0.1:8845;
  	server 127.0.0.1:8846;
  	server 127.0.0.1:8847;
  }
  
  server {
  	listen       80;
  	server_name  localhost;
  
  	location /nacos {
  		proxy_pass http://nacos-cluster;
  	}
  }
  ```

  - 启动nginx服务。<启动后可关闭cmd窗口>

    直接在nginx.exe文件目录下的cmd中，执行命令：start nginx.exe

    检查启动成功：

    - nginx -s reopen		重启nginx，不报错说明成功
    - nginx启动后会在logs目录下生成nginx.pid文件，停止后会自动删除。
    - netstat -an|findstr 80	查看端口启动
    - 步骤6，启动网页

- **在浏览器中打开nacos的网页客户端，说明集群已经配置完成。**

- **修改之前的nacos项目**

  userservice微服务中：

  ​	由于之前将nacos的配置信息写到了bootstrap.yml中，所以在bootstrap中将端口号改为80端口(nginx端口)

  orderservice没有配置bootstrap，所以在application中将nacos端口号修改为80端口号。并且将服务名改为order_service

- **测试集群**

  在nacos网页客户端中，发布一个配置文件。<例：在配置列表中添加userservice.yaml文件>

  发布后，查看数据库中config_info表

  <此时配置信息都保存到了我们创建的数据库当中>

### 八、http客户端==Feign==

==<font color="red"><只是替换RestTemplate请求方式></font>==

> **RestTemplate方式调用微服务资源存在的问题**：
>
> <上面eureka与nacos都使用RestTemplate方式>
>
> ```java
> String url = "http://userservice/user/" + order.getUserId();
> User user = restTemplate.getForObject(url, User.class);
> ```
>
> - 代码可读性差，编程体验不统一。
> - 参数复杂的url难以维护<参数较多>。

#### 1. 介绍Feign

> 官网：https://github.com/OpenFeign/feign

Feign是一个声明式的web service客户端。类似controller调用service。接口 + 注解的方式发起HTTP请求调用

作用：==帮助我们优雅的实现http请求的发送，解决RestTemplate发送方式的问题。==

#### 2. 如何使用

- 消费者服务引入openfeign依赖

  ```xml
  <!--Feign客户端依赖-->
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

- 开启Feign自动装配功能

  在消费者orderservice服务的启动类上添加注解：@EnableFeignClients

- 编写Feign客户端

  创建clients包，定义接口UserClient

  ```java
  @FeignClient("userservice")				// 服务提供者名称
  public interface UserClient {
  	@GetMapping("/user/{id}")
  	User fingById(@PathVariable("id") Long id);
  }
  ```

  主要是基于SpringMVC的注解来声明远程调用的信息，比如：

  - 服务名称：userservice
  - 请求方式：GET
  - 请求路径：/user/{id}
  - 请求参数：Long id
  - 返回值类型：User

  > **注意**：
  >
  > ​    实际上相当于RestTemplate中的：
  >
  > ```java
  > String url = "http://userservice/user/" + order.getUserId();
  > User user = restTemplate.getForObject(url, User.class);
  > ```

- 调用Feign实现的请求方法

  - 向order服务中引入feign-api依赖<可以直接引入，不需要将feign-api打成jar包>

    ```xml
    <!--引入feign-api依赖-->
    <dependency>
    	<groupId>cn.itcast.demo</groupId>
    	<artifactId>feign-api</artifactId>
    	<version>1.0</version>
    </dependency>
    ```

  - 注入接口参数

    ```java
    @Autowired
    private UserClient userClient;
    ```

  - 调用Feign实现的方法

    ```java
    // 1.查询订单
    Order order = orderMapper.findById(orderId);
    // 2.利用Feign发送远程请求
    User user = userClient.fingById(order.getUserId());
    // 3.封装User对象
    order.setUser(user);
    // 4.返回
    return order;
    ```

- 浏览器端调用controller中实现的地址

  <feign只是消费者服务中发送给消费者服务请求，并不是改变消费者的controller请求，两者注意区分>

> **注意**：
>
> <font color="red">RestTemplate并没有负载均衡，而Feign只是替换了RestTemplate这种风格的请求方式。真正实现负载均衡的是Ribbon，它拦截请求，然后获取注册中心的服务列表，并使用负载均衡算法原则一个服务的。与RestTemplate和Feign都没有关系。</font>

> 注意：上面的对OpenFeign的使用，是直接将Feign集成在了消费者服务中了。而在开发中我们一般将Feign创建的所有服务间调用的接口整合在一个微服务中，这个微服务可以不需要启动，只充当api的作用，相当于消费者服务引用的一个jar包，使用时将坐标导入即可<**最佳实践**>。

#### 3. 自定义Feign的配置

- feign.logger.Level	<作用：修改`日志级别`>

  包含四种不同的级别：

  - NONE：不输出日志
  - BASIC：输出请求方法及url，响应的状态码及响应时间
  - HEADERS：输出请求和响应的头信息
  - FULL：输出请求和响应的请求头，消息体及元数据

- feign.codec.Decoder	<作用：`响应结果的解析器`>

  http远程调用的结果做解析，例如：解析json字符串为java对象。

- feign.codec.Encoder	<作用：`请求参数编码`>

  将请求参数编码，便于通过http请求发送。

- feign.Contract	<作用：`支持的注解格式`>

  默认是SpringMVC的注解。

- feign.Retryer	<作用：`失败重试机制`>

  请求失败的重试机制，默认是没有，不过会使用Ribbon的重试。

==***此处介绍Feign的日志配置***==

1. 方式一：配置文件方式

   分为全局生效/局部生效

   ```yaml
   #Feign日志配置
   feign:
   	client:
   		config:
   			default:    #这里用default就是全局配置，如果是写服务名称，则是针对某个微服务的配置
   			#userservice:	#局部生效
   				loggerLevel: FULL     #日志级别<输出请求和响应的请求头，消息体及元数据>
   ```

2. 方式二：java代码方式

   - 创建一个Configuration类，但使用@Configuration注解，对方法添加@Bean

     ```java
     public class DefaultFeignConfig {
     	@Bean
     	public Logger.Level logLevel() {
     		return Logger.Level.BASIC;
     	}
     }
     ```

     <目前还没有生效>

   - 日志配置生效

     如果是全局配置，则把它放到@EnableFeignClients这个注解中：

     ```java
     @EnableFeignClients(defaultConfiguration = DefaultFeignConfig.class) 		<启动类上的feign启动注解>
     ```

     如果是局部配置，则把它放到@FeignClient这个注解中：

     ```java
     @FeignClient(value = "userservice", configuration = DefaultFeignConfig.class) 		<client接口中的注解>
     ```

     > 注意：defaultConfiguration属性值是步骤①创建的config类。

   配置结束直接在消费者服务的输出控制台查看。

#### 4. Feign底层实现原理

> 学习文章：https://zhuanlan.zhihu.com/p/393053564

- 基于面向接口的动态代理方式生成实现类

  @EnableFeignClients开启扫描所有@FeignClient注解的接口，将所有的访问路径以及接口信息封装成一个FeignClientFactoryBean注册到Spring容器中。

- 根据接口类的注解声明规则，解析出底层MethodHandler

- 基于RequestBean，动态生成request

- Encoder将Bean包装成请求

- 拦截器负责对请求和返回进行装饰处理

- 日志记录

  在发送和接收请求的时候，Feign定义了统一的日志门面来输出日志信息 , 并且将日志的输出定义了四个等级：NONE、BASIC、HEADERS、FULL

- 基于重试器发送http请求

  Feign内置了一个重试器，当HTTP请求出现IO异常时，Feign会有一个最大尝试次数发送请求

- 发送http请求

  Feign真正发送HTTP请求是委托给feign.Client来做的。

  Feign默认底层通过JDK的java.net.HttpURLConnection实现了feign.Client接口类,在每次发送请求的时候，都会创建新的HttpURLConnection链接，这也就是为什么默认情况下Feign的性能很差的原因。可以通过拓展该接口，使用ApacheHttpClient或者OkHttp3等基于连接池的高性能Http客户端。

  而这个client会委托给org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient进行负载均衡地调用，采用了观察者模式。

> 故：核心就是**通过一系列的封装和处理，将以JAVA注解的方式定义的远程调用API接口，最终转换成HTTP的请求形式，然后将HTTP的请求的响应结果，解码成JAVA Bean，放回给调用者。**

#### 5. Feign的性能优化

Feign底层的客户端实现：

- HttpURLConnection：默认实现，不支持连接池
- ApacheHttpClient：支持连接池
- OKHttp3：支持连接池

优化方法：

- 使用连接池代替默认的HttpURLConnection
- 日志级别，最好使用Basic和none

**如何将默认JDK连接方式替换为连接池？**

- 引入依赖

```xml
<!--feign-httpclient依赖-->
<dependency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign-httpclient</artifactId>
</dependency>
```

- 配置连接池与日志

```yaml
feign:
	#Feign日志配置
	client:
		config:
			default:    #这里用default就是全局配置，如果是写服务名称，则是针对某个微服务的配置
				loggerLevel: BASIC     #日志级别<基本的请求与响应信息>
	#配置feign连接池
	httpclient:
		enabled: true     # 开启feign对httpclient的支持
		max-connections: 200    #最大连接数
		max-connections-per-route: 50   #每个路径的最大连接数
```

#### 6. Feign的最佳实践

<最好的使用方式/如何使用最好>

方式一：对消费者的FeignClient和提供者的controller定义统一的父接口作为标准。

- 定义一个公共的接口，让消费者的FeignClient客户端继承这个接口，提供者的controller实现这个接口，但是不推荐这样做，因为提供者的controller实现这个接口时候，参数不可继承，得再写一次，并且服务的耦合度高。

方式二：将FeignClient抽取为独立模板，并且把接口有关的pojo、默认的Feign配置都放在这个模块中，提供给所有消费者使用。

- 当存在多个消费者服务时，每个消费者都需要远程请求同一个资源，此时每个消费者服务都应该写属于自己的FeignClient接口，注意，由于它们远程资源相同，所以FeignClient接口都是相同的。
- 所以，我们将所以FeignClient提取出来，统一放在一个模块(微服务)中，消费者服务想要使用FeignClient时，只需要引入入依赖，直接调用FeignClient使用。

**实现方式二**：

- 首先创建一个module，命名feign-api，然后引入feign的starter依赖

  只需要引入Feign客户端依赖<修改底层连接池的依赖不需要>。

  ```xml
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

  > 注意：继承父类服务

- 将orderservice中编写的UserClient、User、DefaultFeignConfig都复制到feign-api项目中

  - 创建包名cn.itcast.feign。
  - 将消费者中的FeignClient以及与它相关的实体类、配置类复制进来。
  - 将消费者服务中的被复制资源删除。

- 在消费者服务中引入feign-api的依赖
- 修改order-service中的所有与上述三个组件有关的import部分，改成导入feign-api中的包
- 重启消费者服务测试<要修改扫描包位置>

> **注意：**
>
> > 重启后发现出现了错误，重启服务失败。
> >
> > 在报错日志中：
> >
> > ```
> > Description:
> > 	Field userClient in cn.itcast.order.service.OrderService required a bean of type 'cn.itcast.feign.clients.UserClient' that
> > 	could not be found.
> > 描述：cn.itcast.order.service中的字段userClient。OrderService需要“cn.itcast.feign.clients”类型的bean。找不到UserClient。
> > ```
> >
> > 原因：
> >
> > - 我们之前在定义UserClient后使用了@FeignClient注解，然后通过在启动类中添加@EnableFeignClients(开启Feign自动装配功能)，也就是扫描和注册@FeignClient注解的bean。但是当我们将UserClient移到了feign-api中，此时扫描不到@FeignClient这个注解，也就没有了对应的bean。
> >
> >   <注意：@EnableFeignClients扫描的是启动类所在的包的所有注解，而启动类在当前服务的cn.itcast.order中，所以肯定扫描不到@FeignClient。同时在最初使用UserClient时，我们先是@Autowired注入UserClient对象(也就是@EnableFeignClients扫描到的bean对象)，但是当我们删除了消费者服务中的UserClient接口时，就不能注入UserClient对象了，所以会启动错误。>
> >
> > 解决方案：
> >
> > - 方案一
> >
> >   @EnableFeignClients既然在启动类上做配置，并且在启动类的包内进行扫描(cn.itcast.order)，我们修改包扫描范围。
> >
> >   ```java
> >   @EnableFeignClients(basePackages = "cn.itcast.feign.clients")		// 指定@FeignClient所在包
> >   ```
> >
> > - 方案二
> >
> >   ```java
> >   @EnableFeignClients(clients = {UserClient.class})			// 直接指定@FeignClient所在类
> >   ```

### 九、统一网关==Gateway==

> 学习网址：https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#gateway-request-predicates-factories
>
> https://zhuanlan.zhihu.com/p/380599509

#### 1. 关于网关

**场景**：目前我们多个微服务可以获取数据库信息、各微服务间通讯、服务的注册发现、配置读取等，而用户只需要通过http请求进行获取服务资源。但是存在一些问题：

- 项目存在多个微服务时，我们只想让内部用户看到特定服务，也就是说不同用户需要身份验证和权限验证。
- 用户请求被拦截后，网关需要通过服务路由手段判断请求的微服务到底是哪一个，同时同一个服务可能有多个实例，还需要负载均衡选择服务实例。
- 用户访问较多，流量如何限制。

**网关作用：**

- 防止内部服务关注暴露给外部客户端
- 为我们内部多个服务添加了额外的安全层
- 减低微服务访问的复杂性

**网关功能**：

- 身份验证和权限校验
- 服务路由、负载均衡
- 请求限流

SpringCloud中提供的网关分类：`gateway`、`zuul`。

1. ==Zuul：基于Servlet实现的，属于阻塞式编程。==
2. ==gateway：SpringCloudGateway则是基于Spring5中提供的WebFlux，属于响应式编程的实现，具备更好的性能。==

#### 2. 请求发送流程

- 浏览器向网关服务发送请求：http://localhost:10010/user/1
- 通过路由断言判断，此时满足条件，该请求可以通过，并拼接uri为新的请求：http://userservice/user/1
- 拉取nacos注册中心的服务列表，多实例下通过负载均衡算法拿到一个userservice实例
- 根据新的请求路径获取请求资源

<img src=".\img\image-20221202075746675.png" alt="image-20221202075746675" style="zoom:80%;" />

> 1. **GatewayClient** 向 **GatewayServer** 发出请求
> 2. 请求首先会被 **HttpWebHandlerAdapter** 进行提取组转成网关上下文
> 3. 然后网关的上下文会传递到 **DispatcherHandler**，它负责将请求分发给 **RoutePredicateHandlerMapping**
> 4. **RoutePredicateHandlerMapping**负责路由查找，并更具路由断言判断路由是否可用
> 5. 如果断言成功，由 **FilteringWebHandler** 创建过滤器链并调用
> 6. 请求会一次经过 **PreFilter** ---> **微服务** ---> **PostFilter** 的方法，最终返回响应

#### 3. 路由route的组成部分

- id：路由的id，区别于其他route
- uri：匹配路由的转发地址，即客户端请求最终被转发到的微服务
- predicates：配置该路由的断言，通过PredicateDefinition类进行接收设置<用来进行条件判断，只有断言都返回真，才会真正的执行路由>
- order：路由的优先级，数字越小，优先级越高
- Filter：过滤器，过滤一些请求并处理，满足则转发

#### 4. 搭建网关Geteway

- 创建新的module(gateway服务)，引入SpringCloudGateway的依赖和nacos的服务发现依赖。

```xml
<!--nacos服务注册发现依赖-->
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
	<version>2.2.5.RELEASE</version>
</dependency>
<!--网关geteway依赖-->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

- 创建启动类

```java
package cn.itcast.gateway;
@SpringBootApplication
public class GatewayApplication {
	public static void main(String[] args) {
		SpringApplication.run(GatewayApplication.class, args);
	}
}
```

- 编写路由配置及nacos地址

```yaml
#复制粘贴时可能缩进关系出现问题
server:
	port: 10010   #服务端口
spring:
	application:
		name: gateway     #服务名称
	cloud:
		nacos:
			server-addr: localhost:8848     #nacos地址
		gateway:
			routes:   #网关路由配置(可设置多个)
				-   id: user-service    #路由id，自定义，只要唯一即可
					#uri: http://127.0.0.1:8081   #路由的目标地址 http就是固定地址
					uri: lb://userservice   #路由的目标地址 lb就是负载均衡(loadbalance)，后面跟服务名称
					predicates:   #路由断言(断定，一种布尔表达式)，也就是判断请求是否符合路由规则的条件。<可设置多个>
						-   Path=/user/**   #这个是按照路径匹配，只要以请求/user/开头就符合要求

				-   id: order-service
					uri: lb://orderservice
					predicates:
						-   Path=/order/**
```

> 注意：'-'与字符之间的空格

- 测试网关gateway

  在浏览器中：

  - 测试userservice的请求是否可以通过

    http://localhost:10010/user/1

  - 测试orderservice的请求是否可以通过

    http://localhost:10010/order/101

#### 5. 路由断言工厂Route Predicate Factory

我们在配置文件中写的断言规则只是字符串，这些字符串会被Predicate Factory读取并处理，转变为路由判断的条件。

例如：

```yaml
Path=/user/**		// 按照路径匹配
```

这个规则是由org.springframework.cloud.gateway.handler.predicate.**PathRoutePredicateFactory**类来处理的。

Spring提供了11种基本的Predicate工厂：

- After			// 是某个时间点后的请求
- Before			// 是某个时间点之前的请求
- Between			// 是某两个时间点之前的请求
- Cookie			// 请求必须包含某些cookie
- Header			// 请求必须包含某些header
- Host			// 请求必须是访问某个host(域名)
- Method			// 请求方式必须是指定方式
- Path			// 请求路径必须符合制定规则
- Query			// 请求参数必须包含指定参数
- RemoteAddr		// 请求者的ip必须是指定范围
- Weight			// 权重处理

#### 6. 路由过滤器GatewayFilter

> 学习网站：https://zhuanlan.zhihu.com/p/488790949

##### 6.1 生命周期

Spring Cloud Gateway的Filter生命周期很简单，它只有：`pre` 和 `post` 。

**PRE**：这种过滤器在请求被路由之前调用。可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。

**POST**：这种过滤器在路由到微服务后执行。可用来为响应添加标准的HTTPHeader、收集统计信息和指标、将响应从微服务发送给客户端等。

##### 6.2 分类

根据作用范围：`GatewayFilter`与`GlobalFilter`。

**局部过滤器(GatewayFilter)**：

​	针对单个路由的过滤器，对访问的URL过滤，切面处理。在Spring Cloud Gateway中通过GatewayFilter的形式内置了很多不同类型的局部过滤器。

**全局过滤器(GlobalFilter)**：

​	用于所有的路由，可以通过自定义实现自己的GlobalFilter。通过全局过滤器实现对所有请求的的统一校验。

**默认过滤器：**

​	通过配置文件来配置，对所有的请求和响应做过滤

##### 6.3 存在多个过滤器时执行顺序

请求进入网关存在上面三种过滤器。

请求路由后(请求路由后才知道有哪些过滤器存在于网关中)，会将三种过滤器合并到一个过滤器链(集合)中，排序后依次执行每个过滤器。

- 当配置order参数时

  > 全局过滤器可以实现Ordered接口或者使用@Order注解，默认过滤器和路由过滤器默认按照配置的顺序从上到下从1依次递增。
  >
  > 过滤器会按照order值从小到大的顺序依次执行过滤。

- 当过滤器的order值一样或者没有配置order时

  > (defaultfilters)默认过滤器 > 路由过滤器(GatewayFilter) > 全局过滤器(GlobalFilter)	由大到小顺序执行

  > 对于：
  >
  > ```yaml
  > default-filters:        # 默认过滤器，会对所有的路由请求都生效
  > 	-	AddRequestHeader=Zmq, zmq nb!!		# 先声明为 order=1
  > 	-	AddRequestHeader=Qlj, qlj nb!!		# 后声明递增 order+=1
  > ```
  >
  > 实际上路由过滤器与默认过滤器的order未指定时由Spring指定的，默认是按照声明顺序从1递增，也就是说先声明先执行

##### 6.4 Spring中提供了31种不同的路由过滤器工厂

- AddRequestHeader：给当前请求请求添加一个请求头
- AddResponseHeader：给响应结果中添加一个响应头
- RemoveRequestHeader：移除请求中的一个请求头
- RemoveResponseHeader：从响应结果中移除一个响应头
- RequestRateLimiter：限制请求的流量
- ......

###### 6.4.1 局部过滤器

==给进入userservice的请求添加一个请求头<局部过滤器>==

1. **实现**：在gateway服务中修改application.yml文件

```yaml
# AddRequestHeader给所有请求添加一个请求头<没有添加路由断言>
spring:
	cloud:
		gateway:
			routes: # 网关路由配置
				-   id: add_request_header_route
					uri: lb://userservice
					predicates:
						-   Path=/**
					filters:
						-   AddRequestHeader=QLJ, niubi!
```

>  **注意**：一个请求不能存在于两个route中。<否则只会进入最上面的那个>

2. **验证**：在uri对应的userservice服务的请求上添加请求头获取参数。

```java
@GetMapping("/{id}")
public User queryById(@PathVariable("id") Long id, 
						// value：请求头名称<对应filters参数>；
						// required：false 如果存在就传递，不存在就不传递
					  @RequestHeader(value = "Qlj", required = false) String reqHead) {
	System.out.println(reqHead);
	return userService.queryById(id);
}
```

###### 6.4.2 默认过滤器

==对所有路由都生效的过滤器。<通过配合文件进行设置>==

```yaml
spring:
	cloud:
		gateway:
			routes:
			-   id: user-service    #路由id，自定义，只要唯一即可
				uri: lb://userservice
				predicates:
					-   Path=/user/**
				filters:
					-   AddRequestHeader=Qlj, qlj nb!!

			-   id: order-service
				uri: lb://orderservice
				predicates:
					-   Path=/order/**
			default-filters:        # 默认过滤器，会对所有的路由请求都生效
				- AddRequestHeader=Zmq, zmq nb!!
```

> **注**：default-filters对userservice与orderservice的请求都进行过滤。

###### 6.4.3 全局过滤器

全局过滤器的作用也是处理一切进入网关的请求和微服务响应，与默认过滤器的作用一样。

区别在于默认过滤器是通过配置定义，处理逻辑是固定的。而全局过滤器的逻辑需要自己定义类实现。

实现方式：必须实现**GlobalFilter**接口。

**源码：**

```java
public interface GlobalFilter {
	/*
		exchange：请求上下文，里面可以获取Request、Response等信息。<从请求进入网关到结束为止，整个过程共享该对象>
		chain：用来把请求委托给下一个过滤器
		return：返回标示当前过滤器业务结束
	*/
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

**实现：**

> 需求：定义全局过滤器，拦截请求，判断请求的参数是否满足下面的条件：
>
> 1. 参数中是否有authorization
>
> 2. authorization参数值是否为admin
>
> 如果同时满足则放行，否则拦截。

```java
// 登录认证
@Order(-1)  // 越小优先级越高<也可以多实现一个Order接口>
@Component
public class AuthorizeFilter implements GlobalFilter {
	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		// 1.获取请求参数
		ServerHttpRequest request = exchange.getRequest();      // 与servlet中的request不同
		MultiValueMap<String, String> params = request.getQueryParams();    // key:参数名称，value:对应值
		// 2.获取参数中的 authorization 参数
		String auth = params.getFirst("authorization");
		// 3.判断参数值是否等于admin
		if ("admin".equals(auth)) {
			// 4.是，放行
			return chain.filter(exchange);     // 当调用filter()时，实际是从过滤器链中获取下一个过滤器
		}
		// 5.否，拦截
		// 5.1.设置状态码<直接拦截返回时没有任何结果，一般会设置状态码>
		exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);		// UNAUTHORIZED：401错误
		// 5.2.拦截请求
		return exchange.getResponse().setComplete();
	}
}
```

**测试：**

- 重启gateway服务。

- 发送请求：http://localhost:10010/user/1

  <此时发现访问报错，401错误，需要验证用户>

- 重新发送：

  http://localhost:10010/user/1?authorization=admin	// 响应成功

#### 7. 网关的跨域问题处理

跨域：域名不一致。

- 域名不一致：www.qljweb.com、www.qljweb.org、www.zmqweb.com等，它们的域名都不同。
- 域名相同，端口不同：localhost:8080、localhost:8081

> 此前，我们userservice与orderservice的端口号不同时，完全可以进行互相通讯访问，完全没有问题。
>
> 跨域问题是指浏览器禁止请求的发起者与服务端发生跨域ajax请求，请求被浏览器拦截的问题。
>
> 具体可前往AJAX文档查看。

**解决方案：**<CORS方案>

```yaml
spring:
  cloud:
	gateway:
	  globalcors: # 全局的跨域处理
		add-to-simple-url-handler-mapping: true # 解决options请求被拦截问题
		corsConfigurations:
		  '[/**]':
			allowedOrigins: # 允许哪些网站的跨域请求
			  - "http://localhost:8090"
			  - "http://www.leyou.com"
			allowedMethods: # 允许的跨域ajax的请求方式
			  - "GET"
			  - "POST"
			  - "DELETE"
			  - "PUT"
			  - "OPTIONS"
			allowedHeaders: "*" # 允许在请求中携带的头信息
			allowCredentials: true # 是否允许携带cookie
			maxAge: 360000 # 这次跨域检测的有效期
```

### 十、微服务异步通讯<==RabbitMQ==> 

#### 1. 同步与异步通讯

##### 1.1 同步通讯

同步简单的讲：指你要做一件事，等着他完成后，然后再做下一件事。<feign发送请求通讯时就是同步执行>

消费者服务器向提供者服务器发送通讯，消费者服务器的其他请求通讯不能执行，只能等服务提供者响应完成后，其他通讯请求才能发送。

优点：

- 时效性较强，可以立即得到结果

缺点：

- 耦合度高。

  服务之间直接发送请求通讯时，如果新添加一个业务服务，就造成了每次加入新业务就需要修改支付服务里面原先的代码。存在耦合度高的问题。

- 性能和吞吐能力下降。

  当用户量大，服务通讯只允许同步，耗时较长，并发量大的时候处理的非常慢。就存在一个性能和吞吐的能力下降的问题。

- 存在额外的资源消耗。

  其他通讯请求也想与这个服务通讯时，在等待的时候会占用cpu与内存，造成额外资源消耗。

- 有级联失败问题。

  当多个服务组成一个业务时，如果一个服务挂掉，则此时消费者服务的通讯请求阻塞，不能及时清理，占用内存资源。虽然消费者服务还可以访问其他服务，但是用户量巨大，不停的给挂掉的服务发送请求，最终导致消费者服务资源耗尽，其他服务也不能接收到任务请求了。

##### 1.2 异步通讯

异步指你要发起一件事，但不太关心他做没做完，继续自己的。

消费者服务不关心提供者服务是否响应，消费者服务中的其他请求依然可以进行发送。

<实际上与Ajax的效果一样>

**异步调用方案：**

> 异步通讯最常见的实现就是事件驱动模式。
>
> 在同步通讯中，服务间通讯是直接通讯，并且存在先后。
>
> 所以我们在此引入一个**Broker**，Broker就是**事件代理者**。服务间通讯直接交给Broker，**由Broker统一进行管理**。
>
> 下游服务(提供者)通过去订阅Broker<订阅事件>，接收请求并处理，处理后再推送给Broker。
>
> ==**Broker作用：**==
>
> - **接收Producer(生产者)发过来的消息**、**处理Consumer(消费者)的消费消息请求**、**消息的持久化存储**、**消息的HA机制以及服务端过滤功能**等

> 注意：此处在消息队列中的生产者与消费者，不等同于上面提到的提供者与消费者。
>
> - 提供者与消费者：
>
>   基于服务器中的资源。<在远程调用时这样称呼>
>
> - 生产者与消费者：
>
>   基于请求消息的，生产者就是请求的发送者，生产消息的。而消费者则是接收消息的。

优点：

- 耦合度极低，每个服务都可以灵活插拔，可替换。

- 性能提升，吞吐量提高。

- 故障隔离<服务没有强依赖，不担心级联失败问题>。

- 流量消峰。

  当用户请求量较多时，消息的生产者需要给消费者发送请求，而消费者每次处理的请求量有限，此时Broker充当缓存作用，消费者可以处理几个就获取几个。

缺点：

- 依赖Broker的可靠性、安全性、吞吐能力。<一旦Droker挂掉，整个微服务承受不了高并发请求>
- 架构复杂了，业务没有明显的流程线，不好追踪管理。

#### 2. 常见MQ

MQ(MessageQueue)，消息队列。存放消息的队列。<也就是上面提到的Broker>。

常见MQ：RabbitMQ、ActiveMQ、RocketMQ、Kafka

<img src="./img/SpringCloud/image-20221203104648076.png" alt="image-20221203104648076" style="zoom:80%;" />

#### 3. RabbitMQ

> 官网：https://www.rabbitmq.com/

RabbitMQ是基于Erlang语言开发的开源消息通信中间件。

RabbitMQ 整体上是一个**生产者与消费者模型**，主要**负责接收**、**存储和转发消息**。

##### 3.1 下载安装

- 下载镜像

  在远程镜像仓库中pull

  ```shell
  docker pull rabbitmq:3-management
  ```

  使用事先下载好的mq.tar文件

  ```shell
  # 将mq.tar压缩文件上传到/tmp中<临时文件存放目录>
  # 在命令行中执行命令：
  	cd /tmp
  	docker load -i mq.tar
  # 查看镜像
  	docker images			# rabbitmq:3-management
  ```

- 首次安装MQ

  ```shell
  docker run \
  	-e RABBITMQ_DEFAULT_USER=qlj \			# 用户名
  	-e RABBITMQ_DEFAULT_PASS=123321 \		# 密码		<访问或登录管理平台MQ时需要>
  	--name mq \							  # 容器名称
  	--hostname mq1 \						# 主机名<单机部署也可以不配，但是集群部署就要配置>
  	-p 15672:15672 \						# 端口映射	<MQ管理平台的端口>
  	-p 5672:5672 \							# 端口映射 <消息通讯端口>
  	-d \									# 后台运行
  	rabbitmq:3-management					# 镜像名称
  ```

- 查看安装成功

  查看容器是否启动：docker ps		<查看运行中的容器>

  > 退出/运行容器：docker stop/start 容器名称

- 登录MQ管理平台

  http://192.168.10.128:15672/		<注意：端口号是管理平台的端口号>

##### 3.2 RabbitMQ结构

RabbitMQ就是一个Broker，RabbitMQ将整个Broker划分成了多个Vhost(Virtual Host)，Vhost可以理解为一个个逻辑空间，它们之间相互隔离。

<一般生产与消费过程只会使用一个Vhost>

- **Vhost**构成

  Vhost由多个exchange和queue构成。

  ![image-20221203110909763](img/SpringCloud/image-20221203110909763.png)

  - **Exchange**

    `消息交换器`。用来接收生产者发送的消息并将这些消息路由给服务器中的队列(queue)。它指定消息按什么规则，路由到哪个队列。

  - **Queue**

    `消息的载体`，每个消息都会被投到一个或多个队列，等待消费者连接到这个队列将其取走。它是消息的容器，也是消息的终点。

  - **routing Key**

    `路由关键字`，exchange根据这个关键字进行消息投递。

  - **binding key**

    用于消息队列和交换器之间的关联。

    一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。

    它的作用就是把exchange和queue按照路由规则绑定起来。

##### 3.3 常见消息模型

> 官网上为我们提供了7种消息模型。https://www.rabbitmq.com/getstarted.html

常见的5种消息模型分类：

- ==无交换机==

  - **基本消息队列**<BasicQueue>

    结构：生产者P、消息队列Queue、消费者C(单个消费者)

    - 生产者P：消息发送者，将消息发送到队列Queue中
    - 消息队列Queue：负责接收并缓存信息
    - 消费者C：订阅队列，处理队列中的消息

    <img src="img/SpringCloud/image-20221203112938160.png" alt="image-20221203112938160" style="zoom:60%;" />

  - 工作消息队列<WorkQueue>

    结构：生产者P、消息队列Queue、消费者C(多个消费者)

    <img src="img/SpringCloud/image-20221203112919913.png" alt="image-20221203112919913" style="zoom:60%;" />

- ==发布订阅==<Publish、Subscribe>，根据交换机类型分为三种

  - **广播**<Fanout Exchange>

    结构：生产者P、交换机Exchange、消息队列Queue、消费者C(多个消费者)

    <img src="img/SpringCloud/image-20221203112856163.png" alt="image-20221203112856163" style="zoom:60%;" />

  - **路由**<Direct Exchange>

    结构：生产者P、交换机Exchange、消息队列Queue、消费者C(多个消费者)

    <img src="img/SpringCloud/image-20221203112804628.png" alt="image-20221203112804628" style="zoom:60%;" />

  - **主题**<Topic Exchange>

    结构：生产者P、交换机Exchange、消息队列Queue、消费者C(多个消费者)

    <img src="img/SpringCloud/image-20221203112632595.png" alt="image-20221203112632595" style="zoom:60%;" />

    ![image-20221203112525644](img/SpringCloud/image-20221203112525644.png)



##### 3.4 直连模式实现<基本消息队列>

- 生产者发送消息

  步骤：

  - 建立连接<生产者与MQ建立连接，在连接工厂中获取连接>

    ① 设置连接参数，分别是：主机名、端口号、vhost(虚拟主机)、用户名、密码

    ② 建立连接<建立连接后，管理平台connections中就存在一个连接>

  - 创建通道Channel<通道创建后，管理平台channels中就会存在一个通道>

  - 创建队列<创建消息队列，管理平台queues就存在一个消息队列>

  - 发送消息<消息发送后，在管理平台进入指定的消息队列中，get message 查看发送的消息>

  - 关闭通道和连接<关闭后管理平台就不存在连接与通道了>

  案例：

  ```java
  public void testSendMessage() throws IOException, TimeoutException {
  	// 1.建立连接<生产者与MQ建立连接，在连接工厂中获取连接>
  	ConnectionFactory factory = new ConnectionFactory();
  	// 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
  	factory.setHost("192.168.10.128");
  	// MQ消息通讯端口<不是MQ管理平台的端口>
  	factory.setPort(5672);
  	// 虚拟主机<在MQ管理平台中的admin中查看虚拟主机>
  	factory.setVirtualHost("/");
  	// MQ用户名
  	factory.setUsername("qlj");
  	// 密码
  	factory.setPassword("123321");
  	// 1.2.建立连接<建立连接后，管理平台connections中就存在一个连接>
  	Connection connection = factory.newConnection();
  
  	// 2.创建通道Channel<通道创建后，管理平台channels中就会存在一个通道>
  	Channel channel = connection.createChannel();
  
  	// 3.创建队列<创建消息队列，管理平台queues就存在一个消息队列>
  	String queueName = "simple.queue";
  	channel.queueDeclare(queueName, false, false, false, null);
  
  	// 4.发送消息<消息发送结束后，在管理平台进入指定的消息队列中，get message 查看发送的消息>
  	String message = "hello, rabbitmq!";
  	channel.basicPublish("", queueName, null, message.getBytes());
  	System.out.println("发送消息成功：【" + message + "】");
  
  	// 5.关闭通道和连接<关闭后管理平台就不存在连接与通道了>
  	channel.close();
  	connection.close();
  }
  
  ```

  

- 消费者订阅接收处理消息

  步骤：

  - 建立连接

    ① 设置连接参数，分别是：主机名、端口号、vhost(虚拟主机)、用户名、密码

    ② 建立连接

  - 创建通道Channel

  - 创建队列

    为什么生产者创建了消息队列，这里又创建了一次消息队列？

    - 原因：生产者与消费者启动顺序不确定，如果消费者先启动，保证消息队列一定存在。
    - 注意：如果生产者优先创建了消息队列，此处不会重复创建的，除非消息队列名称不一致。

  - 订阅消息<Consume：消费消息>

    ```java
    /*
     queueName：指定的消息队列
     true：是否自动确认消息已经消费，如果为false则需要手动确认消费信息。
     new DefaultConsumer(channel){}：消费消息的行为
     	handleDelivery()：相当于一个回到函数。一旦指定的消息队列中有个消息，这个方法就会被触发调用。<异步执行>
     	<实际上就是实现封装好消费消息的行为，然后将这个消费行为挂到消息队列上，将来消息队列中一旦有了消息，这个消费	  行为就会触发>
     	body：消息体，字节数组。生产者发送到RabbitMQ后就变成了字节数组。
    */
    String basicConsume(String queue, boolean autoAck, Consumer callback)   // 订阅消息方法
    ```

  - 处理消息<消息被接受执行后会立即销毁，控制平台在消息队列中就看不到这条消息>

    注意：处理消息的方法是回调执行的，不是立即必须执行，相当于挂起状态。而挂起后不能影响后续代码执行，所以方法一定是异步的。

  ```java
  public static void main(String[] args) throws IOException, TimeoutException {
  	// 1.建立连接
  	ConnectionFactory factory = new ConnectionFactory();
  	// 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
  	factory.setHost("192.168.10.128");
  	factory.setPort(5672);
  	factory.setVirtualHost("/");
  	factory.setUsername("qlj");
  	factory.setPassword("123321");
  	// 1.2.建立连接
  	Connection connection = factory.newConnection();
  
  	// 2.创建通道Channel
  	Channel channel = connection.createChannel();
  
  	// 3.创建队列
  	/**
  	 * 为什么生产者创建了消息队列，这里又创建了一次消息队列？
  	 *  原因：生产者与消费者启动顺序不确定，如果消费者先启动，保证消息队列一定存在。
  	 *  注意：如果生产者优先创建了消息队列，此处不会重复创建的，除非消息队列名称不一致。
  	 */
  	String queueName = "simple.queue";
  	channel.queueDeclare(queueName, false, false, false, null);
  
  	// 4.订阅消息<Consume：消费消息>
  	/**
  	 * channel.basicConsume()   // 订阅消息方法
  	 * 参数：
  	 *  queueName：指定的消息队列
  	 *  true：是否自动确认消息已经消费，如果为false则需要手动确认消费信息。
  	 *  new DefaultConsumer(channel){}：消费消息的行为
  	 *      handleDelivery()：相当于一个回调函数。一旦指定的消息队列中有个消息，这个方法就会被触发调用。<异步执行>
  	 *      <实际上就是事先封装好消费消息的行为，然后将这个消费行为挂到消息队列上，将来消息队列中一旦有了消息，这个消费行为就会触发>
  	 *      参数：
  	 *          body：消息体，字节数组。生产者发送到RabbitMQ后就变成了字节数组。
  	 */
  	channel.basicConsume(queueName, true, new DefaultConsumer(channel){
  		@Override
  		public void handleDelivery(String consumerTag, Envelope envelope,
  								   AMQP.BasicProperties properties, byte[] body) throws IOException {
  			// 5.处理消息<消息被接受执行后会立即销毁，控制平台在消息队列中就看不到这条消息>
  			String message = new String(body);
  			System.out.println("接收到消息：【" + message + "】");
  		}
  	});
  	// 注意：上面的方法是回调执行的，不是立即必须执行，相当于挂起状态。而挂起后不能影响后续代码执行，所以方法一定是异步的。
  	System.out.println("等待接收消息。。。。");
  }
  
  ```

##### 3.5 SpringAMQP

- AMQP

  Advanced Message Queuing Protocol(高级消息队列协议)。

  是用于应用程序或应用程序之间传递业务消息的开放标准。

  该协议与语言和平台无关，更符合微服务中独立性的要求。

- Spring AMQP

  > 网站：https://spring.io/projects/spring-amqp

  是基于AMQP协议定义的一套API规范，提供了模板来发送和接收消息。

  包含两个部分：

  - spring-amqp是基础抽象
  - spring-rabbit是底层默认实现

  功能：

  - 用于异步处理入站消息的监听器容器

  - 用于发送和接收消息的RabbitTemplate
  - RabbitAdmin用于自动声明队列、交换和绑定<自动创建队列>

##### 3.6 使用SpringAMQP实现RabbitMQ消息模型

> **注意：**以下5种消息发送都采用RabbitTemplate进行发送消息的。<使用@Autowired>

==**<font color="red">无交换机(消息即用即删)</font>**==

###### 3.6.1 直连模式<基本消息队列>

步骤：

- 引入spring-amqp的依赖。

  由于我们是微服务工程，直接在父工程中引入依赖

  ```xml
  <!--AMQP依赖，包含RabbitMQ-->
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-amqp</artifactId>
  </dependency>
  ```

- 在生产者中利用RabbitTemplate发送消息到消息队列中

  - 在生产者中的yaml中配置mq连接信息

    ```yaml
    spring:
      rabbitmq:
        host: 192.168.10.128    #mq运行服务器地址
        port: 5672  #端口
        virtual-host: /   # 虚拟主机
        username: qlj   # 用户名
        password: 123321    # 密码
    ```

  - 编写消息发送程序

    下面使用单元测试做消息的发送，实际业务在service中写的

    ```java
    /**
     * Runner是Junit的运行器，即运行测试用例的工具。
     *  在测试类之上，使用@RunWith()注解为这个类指定一个特定的Runner。当我们没有指定@RunWith()的时候，
     * 会自动使用Junit的默认Runner——BlockJunit4ClassRunner。
     */
    //@RunWith(SpringRunner.class)
    @SpringBootTest
    public class SpringAmqpTest {
    	@Autowired
    	private RabbitTemplate rabbitTemplate;
    
    	@Test
    	void basicMessageSend(){
    		// 消息队列名称
    		String queueName = "simple.queue";
    		// 消息
    		String message = "hello,spring amqp!";
    		// 转换并发送
    		rabbitTemplate.convertAndSend(queueName, message);
    	}
    }
    
    ```

    > 注意：queueName这个消息队列是前面传统发送消息时创建的。

- 在消费者中编写消费逻辑，绑定消息队列。

  <之前的项目中已经创建了一个消息队列，此处先使用这个消息队列(simple.queue)>

  - 在消费者的yaml中配置mq连接

    ```yaml
    spring:
      rabbitmq:
        host: 192.168.10.128    #mq运行服务器地址
        port: 5672  #端口
        virtual-host: /   # 虚拟主机
        username: qlj   # 用户名
        password: 123321    # 密码
    ```

  - 在消费者中新建一个类，编写消费逻辑<在src中>

    ```java
    @Component
    public class SpringAmqpRabbitListener {
    	@RabbitListener(queues = "simple.queue")
    	public void listenerSimpleQueue(String msg) {
    		System.out.println("接收消息：" + msg);
    	}
    }
    ```

  - 启动服务，启动成功时就会执行listenerSimpleQueue()方法。

    原因：

    在生产者中，我们已经发送了一次消息，并且消息存放在了simple.queue队列中。

    而被@RabbitListener(queues = "simple.queue")注解标注后，当在指定的消息队列中一旦发现消息，就会自动执行被标注的方法。

    方法形参就是接收消息内容，方法体用于消费者逻辑编写。

    注意：

    消息被消费后就会从消息队列中删除。

###### 3.6.2 工作消息队列(WorkQueue)

作用：提高消费者处理消息的速度，避免消息队列消息堆积。

特点：消息队列后可以有多个消费者。

> **注意**：在基本消息队列中，消息一旦被消费就会立即被删除，所以消息队列后只能有一个消费者订阅。
>
> **场景**：当生产者生产消息量较大时，比如1s发送100个消息，而消费者服务只能一个一个进行处理，1s只能处理20个消息，此时消息就会滞留在消息队列中，最终内存被占满。<所以我们需要多个消费者进行消费消息>

例：工作消息队列并没有出现新的API，在基本消息队列的基础上进行多消费者更改

模拟WorkQueue，实现一个队列绑定多个消费者。

- 引入spring-amqp的依赖。

- 编写消息发送程序<模拟消息发送50条>

  ```java
  // 工作消息队列发送<消息队列绑定多个消费者，避免消息堆积>
  @Test
  public void workMessageSend() throws InterruptedException {
  	// 消息队列名称
  	String queueName = "simple.queue";
  	// 消息
  	String message = "hello,spring amqp_";
  	for (int i = 1; i <= 50; i++) {
  		// 转换并发送
  		rabbitTemplate.convertAndSend(queueName, message + i);
  		Thread.sleep(20);		// 让程序执行慢一点
  	}
  }
  ```

  > **注意**：先不要运行，让消费者服务先运行。<消费者服务启动后，生产者生产消息后会立即被消费者获取到并处理>

- 配置mq地址；新建一个类，编写消费逻辑

  - yaml配置

    ```yaml
    spring:
      rabbitmq:
    	host: 192.168.10.128    #mq运行服务器地址
    	port: 5672  #端口
    	virtual-host: /   # 虚拟主机
    	username: qlj   # 用户名
    	password: 123321    # 密码
    	listener:
    	  simple:
    		prefetch: 1   # 消息预期上限
    ```

  - 编写消费者处理类

    ```java
    /**
     * 工作消息队列接收
     * <创建了两个消费者，它们的能力不同，消费者1每秒处理50条消息，消费者2每秒处理5条消息>
     * <测试：两个消费者消费消息是否按照自身能力大小进行消费，也就是说是否按照能力越大消费越多>
     * @param msg 消息内容
     * @throws InterruptedException
     * <结果：默认情况下多个消费者拿到的消息量相同，与能力大小无关></>
     * <原因：
     *  默认采用‘消息预取机制’。
     *  	生产者发送50条消息时，消息队列会先将消息进行投递，不管消费者能力大小或者能不能执行，提前将消息一人一个的
     *    方式分配给消费者(所以出现奇数位消息与偶数位消息分别由两个消费者执行)。而处理能力小的消费者就会执行很慢。
     *  >
     *  <解决：消息预取机制默认预取上限是无穷。有多少分配多少。通过yaml进行配置预期上限。
     *      listener.simple.prefetch: 1     // 预期上限为1
     *  >
     */
    @RabbitListener(queues = "simple.queue")
    public void listenerSimpleQueue1(String msg) throws InterruptedException {
    	System.out.println("workqueue2接收消息：【" + msg + "】" + LocalTime.now());
    	Thread.sleep(20);
    }
    @RabbitListener(queues = "simple.queue")
    public void listenerSimpleQueue2(String msg) throws InterruptedException {
    	// err：表示输出为红色<为了与上面的区分>
    	System.err.println("workqueue2...接收消息：【" + msg + "】" + LocalTime.now());
    	Thread.sleep(200);
    }
    ```

    

==**<font color="red">发布订阅模式<有交换机></font>**==

前面两个模板案例中的消息在发送后接收就会被删除。不能实现同一条消息推送给多个消费者的业务。

实现方式：

​	添加交换机。<在生产者发送消息给消息队列间添加一个交换机>

​	<注意：交换机Exchange负责的是消息的路由，并不是存储，路由失败时消息就会丢失，发不到消息队列中>

常见交换机：

- Fanout：广播
- Direct：路由
- Topic：话题

交换机作用：

- 接收生产者发送的消息；
- 将消息按照规则路由到与之绑定的队列；
- 不能缓存消息，路由失败消息丢失；

> **区别：**
>
> - fanoutexchange：将消息路由到每个被绑定的消息队列
> - directexchange：根据规则路由到指定消息队列<存在routingkey与bindingkey对应关系>
> - topicexchange：与direct类似，区别在于routingkey必须由多个单词组成，单词之间必须以'.'分隔，同时bindingkey可以使用通配符代替。

###### 3.6.3 Fanout广播消息模型

之所以称之为广播，是因为交换机在路由消息时，是将消息路由到与之绑定的所有消息队列中的，所以称作广播。

步骤：

- 消息接收

  我们先将消费者配置完成并启动，生产者一旦发送消息就会接收。

  - 首先需要创建配置类config

    ① 创建交换机对象

    ② 创建消息队列对象<多个消息队列>

    ③ 绑定交换机与消息队列<一个交换机可以绑定多个消息队列>

  - 创建消费者消费业务，实现消费逻辑

    <每个消息队列都可以有一个或多个消费者>

  - 启动服务，MQ管理平台检验交换机绑定是否成功

    启动成功后，管理平台中的Exchanges栏就会存在创建好的交换机名称(itcast.fanout)。

    进入Bindings中就可以看到被绑定在交换机上的所有消息队列。

- 消息发送

  - 指定交换机名称
  - 编辑消息并发送

  注：我们将交换机与消息队列的创建与绑定都放在了消费者所在的服务中。与之前在生产者所在服务创建消息队列不同。

案例：

<生产者发送一条消息，两个消费者同时接收并显示>

- 消息发送<依然在测试类中发送消息(方便)>

  - 创建配置类，创建交换机与消息队列并绑定。

    ```java
    // 下面配置类创建了一个交换机itcast.fanout，两个消息队列fanout.queue1、fanout.queue2
    @Configuration
    public class FanoutConfig {
    	// 交换机fanout
    	@Bean
    	public FanoutExchange fanoutExchange() {
    		return new FanoutExchange("itcast.fanout");
    	}
    	// 消息队列queue1
    	@Bean
    	public Queue fanoutQueue1() {
    		return new Queue("fanout.queue1");
    	}
    	// 交换机与消息队列绑定
    	@Bean
    	public Binding fanoutBanding1(Queue fanoutQueue1, FanoutExchange fanoutExchange) {
    		// 将消息队列fanoutQueue1绑定到fanoutExchange交换机上
    		return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    	}
    	// 消息队列queue1
    	@Bean
    	public Queue fanoutQueue2() {
    		return new Queue("fanout.queue2");
    	}
    	// 交换机与消息队列绑定
    	@Bean
    	public Binding fanoutBanding2(Queue fanoutQueue2, FanoutExchange fanoutExchange) {
    		// 将消息队列fanoutQueue2绑定到fanoutExchange交换机上
    		return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    	}
    }
    ```

  - 创建接收消息逻辑并启动

    ```java
    // 每个消息队列都可以绑定一个或多个消费者，下面两个消费者方法分别绑定了不同
    @RabbitListener(queues = "fanout.queue1")
    public void listenerFanoutQueue1(String msg) {
    	System.out.println("消费者接收到fanout.queue1的消息：【" + msg + "】");
    }
    @RabbitListener(queues = "fanout.queue2")
    public void listenerFanoutQueue2(String msg) {
    	System.out.println("消费者接收到fanout.queue2的消息：【" + msg + "】");
    }
    ```

- 消息发送<依然在测试类中发送消息(方便)>

  ```java
  // 与之前消息发送不同的是需要按照交换机名称发送，不需要知道消息队列名称
  @Test
  public void fanoutMessageSend() {
  	// 交换机名称
  	String exchangeName = "itcast.fanout";
  	// 消息
  	String message = "hello,fanoutMessage...";
  	// 发送消息
  	rabbitTemplate.convertAndSend(exchangeName, "",message);	// 第二个参数RoutingKey在后面路由交换机有具体讲解
  }
  ```



###### 3.6.4 Direct路由消息模型

DirectExchange(路由交换机)会将接收到的消息根据规则路由到指定的Queue。<与广播路由的区别就是路由规则>

> **消息路由原理：**
>
> 每一个消息队列都与路由器设置一个BindingKey。
>
> 消息生产者发送消息时，指定消息的RoutingKey。
>
> 交换机将消息路由到BindingKey与消息RoutingKey一致的队列。
>
> <每个消息队列可以有多个BindingKey，只要路由器发送的Key存在于消息队列的BindingKey中，消息队列就能拿到消息>
>
> <当被绑定的消息队列上都包含某个BindingKey时，实际上就是广播交换机消息模式>
>
> **故：**
>
> 我们在交换机与消息队列绑定时会设置一个BindingKey，而生产者发送消息时会携带一个RoutingKey，如果消息队列的BindingKey中包含RoutingKey，则这条消息就会发送到被绑定的消息队列中。
>
> **注意：**在广播消息对列模式中，我们采用config配置类创建交换机与消息队列并绑定，而路由消息模式中我们直接使用注解进行创建绑定。

<不需要写config类了>

- 消息接收

  ```java
  /**
   * 路由交换机模式消息接收
   *  在广播消息对列模式中，我们采用config配置类创建交换机与消息队列并绑定，而路由消息模式中我们直接使用注解进行创建绑定
   * @param msg 消息
   */
  @RabbitListener(bindings = @QueueBinding(
  		// 消息队列创建
  		value = @Queue(name = "direct.queue1"),
  		// 交换机创建<默认类型就是Direct类型的交换机>
  		exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
  		// BindingKey
  		key = {"k1", "k2"}
  ))
  public void listenerDirectQueue1(String msg) {
  	System.out.println("消费者接收到direct.queue1[k1,k2]的消息：【" + msg + "】");
  }
  @RabbitListener(bindings = @QueueBinding(
  		value = @Queue(name = "direct.queue2"),
  		exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
  		key = {"k1", "k3"}
  ))
  public void listenerDirectQueue2(String msg) {
  	System.out.println("消费者接收到direct.queue2[k1,k3]的消息：【" + msg + "】");
  }
  ```

  

- 消息发送

  ```java
  @Test
  public void directMessageSend() {
  	// 交换机名称
  	String exchangeName = "itcast.direct";
  	// 消息
  	String message = "hello_k2,DirectMessage...";
  	// 发送消息
  	rabbitTemplate.convertAndSend(exchangeName, "k2",message);		// 参数2就是routingkey
  }
  ```

  > 注意：routingkey = k2时，只有消息队列direct.queue1拿到了这条消息。

###### 3.6.5 Topic话题消息模式

- 消息接收

```java
/**
 * 话题交换机消息模型
 *  Topic与DirectExchange类似，区别在于routingkey必须由多个单词组成，单词之间必须以'.'分隔。
 *      例：routingkey = “china.news”
 *  BindingKey可以使用通配符：
 *      #：0个或多个单词
 *      *：1个单词 <注意不是字符>
 *  例如：bindingkey="china.news"   替代：bindingkey="china.*" | bindingkey="china.#"
 *  routingkey与bindingkey对应关系：
 *      例：
 *          1、routingkey = "china.news" ==> bindingkey = "china.#"
 *          2、routingkey = "china.weather" ==> bindingkey = "china.#"
 *          3、routingkey = "japan.news" ==> bindingkey = "*.news"
 *          4、routingkey = "china.news" ==> bindingkey = "*.news"
 *      可以发现：1和2的routingkey首单词一致，可以匹配到china.#。
 * @param msg 消息
 */
@RabbitListener(bindings = @QueueBinding(
		// 消息队列
		value = @Queue(name = "topic.queue1"),
		// 话题交换机<注意更换交换机类型，默认是Direct>
		exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
		// bindingkey
		key = "china.#"
))
public void listenerTopicQueue1(String msg) {
	System.out.println("消费者接收到topic.queue1的消息：【" + msg + "】");
}
@RabbitListener(bindings = @QueueBinding(
		value = @Queue(name = "topic.queue2"),
		exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
		key = "*.news"
))
public void listenerTopicQueue2(String msg) {
	System.out.println("消费者接收到topic.queue2的消息：【" + msg + "】");
}
```

- 消息发送

```java
// 话题交换机模式消息发送
@Test
public void topicMessageSend() {
	// 交换机名称
	String exchangeName = "itcast.topic";
	// 消息
	String message = "hello_china.news,TopicMessage...";
	// 发送消息
	rabbitTemplate.convertAndSend(exchangeName, "china.news",message);
}
```

##### 3.7 RabbitTemplate序列化问题

> **问题：**
>
> 使用SpringAMQP发送消息时，rabbitTemplate.convertAndSend()方法中接收的消息参数是Object类型，那么如果传入消息参数是一个java对象，消费者接收到消息是否可以正常使用这个对象？
>
> **答案：**
>
> MQ消息队列中不能查看java对象<被序列化>，但是消费者使用对应的类型接收时可以使用<反序列化>。
>
> **原因：**
>
> 与Redis的使用SpringDataRedis存取数据一样，java序列化与反序列化采用的工具一样<底层使用jdk自带的序列化>，所以消费者使用时正常。但是，Rabbit在将消息存储到消息队列中时，使用的序列化与反序列化工具与jdk的不同，所以在消息队列中看到的消息乱码。
>
> **解决问题：**
>
> Spring的消息对象的处理由org.springframework.amqp.support.converter.MessageConverter来处理的。而底层默认实现是jdk自带的消息序列化工具<ObjectOutputStream>。
>
> 所以我们只需要**修改MessageConverter类型的Bean**即可，将ObjectOutputStream实现的序列化替换掉。

步骤：

- 在父工程中引入依赖<生产者消费者都需要用到>

  ```xml
  <!--Rabbit序列化依赖-->
  <dependency>
  	<groupId>com.fasterxml.jackson.dataformat</groupId>
  	<artifactId>jackson-dataformat-xml</artifactId>
  </dependency>
  ```

- 在生产者所在服务创建配置类

  也可以直接在启动类中添加@Bean方法，因为启动类也是一个配置类

  ```java
  @Configuration
  public class SerializedConfig {
  	@Bean
  	public MessageConverter messageConverter() {
  		// 注意：MessageConverter：org.springframework.amqp.support.converter.MessageConverter;
  		return new Jackson2JsonMessageConverter();
  	}
  }
  // 此时可以测试以下发送消息后，在消息队列中的消息已经不是乱码。而是一个json类型的数据。
  ```

- 消费者所在服务创建配置类

  ```java
  @Configuration
  public class SerializedConfig {
  	@Bean
  	public MessageConverter messageConverter() {
  		return new Jackson2JsonMessageConverter();
  	}
  }
  ```

**测试**：

上面5种消息模型总共简单分为3种方式实现的。

​	<1与2是使用事先准备好的消息队列，3是使用config类将消息队列与交换机绑定方式创建消息队列的，4与5是使用注解方式实现交换机与消息队列>

​	由于我们消息一旦发送就会被消费者消费掉，所以不能使用第三种方式实现测试用例，会被立刻消费掉，在MQ管理平台中就看不到了。

步骤：

- 在消费者的config类中创建一个test用的消息队列

  ```java
  @Bean
  public Queue testQueue() {
  	return new Queue("object.queue");
  }
  ```

- 消息发送

  ```java
  @Test
  public void testObject() {
  	// 消息队列名称
  	String queueName = "object.queue";
  	// 消息
  	Map<String, Object> message = new HashMap<>();
  	message.put("name", "屈刘杰");
  	message.put("age", 23);
  	// 转换并发送
  	rabbitTemplate.convertAndSend(queueName, message);
  }
  ```

- 在MQ控制平台查看存入消息队列中的map对象

  我们发现：存入消息类型不是HashMap类型。

  simple.queue消息队列中存入的一条消息是一串乱码。

  消息内容类型‘content_type:application/x-java-serialized-object’ <序列化后对象>。

- 测试消费者是否可以用map对象

  ```java
  @RabbitListener(queues = "object.queue")
  public void listenerTestQueue(Map msg) {
  	System.out.println(msg.get("name"));			// 屈刘杰
  	System.out.println("消费者接收到fanout.queue1的消息：【" + msg.toString() + "】");
  	// 消费者接收到fanout.queue1的消息：【{name=屈刘杰, age=23}】
  }
  ```

  可以发现，消费者使用Map类型接收正常。



### 十一、Elasticsearch









