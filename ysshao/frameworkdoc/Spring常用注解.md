# 常用注解

在spring boot中，摒弃了spring以往项目中大量繁琐的配置，遵循**约定大于配置**的原则，通过自身默认配置，极大的降低了项目搭建的复杂度。同样在spring boot中，大量注解的使用，使得代码看起来更加简洁，提高开发的效率。这些注解不光包括spring boot自有，也有一些是继承自spring的。

​    **下面列举 spring boot项目中常用的一些核心注解归类总结**

## Spring Bean 相关

### 1.注入Bean的注解

**`@Autowired`**

 可以注解到set方法或属性上。此注解用于bean的field、setter方法以及构造方法上，显式地声明依赖。根据type来注入。

**`@Qualifier`**

此注解是和@Autowired一起使用的。使用此注解可以让你对注入的过程有更多的控制。

@Qualifier可以被用在单个构造器或者方法的参数上。当上下文有几个相同类型的bean, 使用@Autowired则无法区分要绑定的bean，此时可以使用@Qualifier来指定名称。

**`@Resource`**

@Resource(name="name",type="type")：  没有括号内内容的话，默认byName。与@Autowired干类似的事。

### 2.声明Bean的注解

- `@Component` ：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。

- `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。

- `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。

- `@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 Service 层返回数据给前端页面。

- `@Configuration`一般用来声明配置类，可以使用 `@Component`注解替代，不过使用`@Configuration`注解声明配置类更加语义化。

### 3.基础注解

@CompentScan :扫描组件路径，指定Spring扫描注解的package。如果没有指定包，那么默认会扫描此配置类所在的package。

@Lazy 此注解使用在Spring的组件类上。默认的，Spring中Bean的依赖一开始就被创建和配置。如果想要延迟初始化一个bean，那么可以在此类上使用Lazy注解，表示此bean只有在第一次被使用的时候才会被创建和初始化。

@Configuration：说明这个类是配置文件

@scope：用来声明Bean的Scope，有singleton,prototype,session,request。默认singleton

@PropertySource：读取配置文件地址

@ConfigurationProperties(prefix) 配置文件映射对象

@Value :作用于变量或方法上，将配置文件中的变量设置到该变量上

## Spring MVC和HTTP相关

**5 种常见的请求类型:**

- **GET** ：请求从服务器获取特定资源。举个例子：`GET /users`（获取所有学生）
- **POST** ：在服务器上创建一个新的资源。举个例子：`POST /users`（创建学生）
- **PUT** ：更新服务器上的资源（客户端提供更新后的整个资源）。举个例子：`PUT /users/12`（更新编号为 12 的学生）
- **DELETE** ：从服务器删除特定的资源。举个例子：`DELETE /users/12`（删除编号为 12 的学生）
- **PATCH** ：更新服务器上的资源（客户端提供更改的属性，可以看做作是部分更新），使用的比较少，这里就不举例子了。

### 1.GET 请求

`@GetMapping("users")` 等价于`@RequestMapping(value="/users",method=RequestMethod.GET)`

### 2.POST 请求

`@PostMapping("users")` 等价于`@RequestMapping(value="/users",method=RequestMethod.POST)`

### 3.PUT 请求

`@PutMapping("/users/{userId}")` 等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.PUT)`

### 4.DELETE 请求

`@DeleteMapping("/users/{userId}")`等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.DELETE)`

## 前后端传值

### `@PathVariable` 和 `@RequestParam`

`@PathVariable`用于获取路径参数，`@RequestParam`用于获取查询参数。

举个简单的例子：

```java
@GetMapping("/klasses/{klassId}/teachers")
public List<Teacher> getKlassRelatedTeachers(
         @PathVariable("klassId") Long klassId,
         @RequestParam(value = "type", required = false) String type ) {
...
}
```

如果我们请求的 url 是：`/klasses/{123456}/teachers?type=web`

那么我们服务获取到的数据就是：`klassId=123456,type=web`。

### `@RequestBody`

用于读取 Request 请求（可能是 POST,PUT,DELETE,GET 请求）的 body 部分并且**Content-Type 为 application/json** 格式的数据，接收到数据之后会自动将数据绑定到 Java 对象上去。系统会使用`HttpMessageConverter`或者自定义的`HttpMessageConverter`将请求的 body 中的 json 字符串转换为 java 对象。也可以直接转换为JSON对象。

```java
 //接受外部传入的JSON字符串。
@PostMapping(value = "/json/test1" , produces = "application/json;charset=UTF-8")
public  ServiceMsg helloMsg(@RequestBody JSONObject json ){
    System.out.println(json.toJSONString());

    SysHead sysHead = new SysHead();
    sysHead.setOrgId("11");
    sysHead.setUserId("test11");
    BodyObject bodyObject = new BodyObject();
    bodyObject.setAmAttribute("cus", "200");
    bodyObject.setAmAttribute("idddd", "2222");
    smsg.setBodyObject(bodyObject);
    smsg.setSysHead(sysHead);

    return smsg;
}
```

**@CookieValue**

此注解用在@RequestMapping声明的方法的参数上，可以把HTTP cookie中相应名称的cookie绑定上去。

```java
@ReuestMapping("/cookieValue")
      public void getCookieValue(@CookieValue("JSESSIONID") String cookie){
}
```

cookie即http请求中name为JSESSIONID的cookie值。

## 读取配置信息

### @PropertySource注解

引入单个properties文件：

@PropertySource(value = {"classpath : xxxx/xxx.properties"})

引入多个properties文件：

@PropertySource(value = {"classpath : xxxx/xxx.properties"，"classpath : xxxx.properties"})

**注入authorSetting.properties配置文件内容** 

```properties
author.name=zhagnsan
author.age=15
```

```java
@Component
@PropertySource(value = {"classpath:static/config/authorSetting.properties"},
        ignoreResourceNotFound = false, encoding = "UTF-8", name = "authorSetting.properties")
public class AuthorTest {

    @Value("${author.name}")
    private String name;
    @Value("${author.age}")
    private int age;

    //省略 setter 和 getter
}
```

> 用于指定读取属性文件所使用的编码，我们通常使用的是UTF-8；ignoreResourceNotFound含义是当指定的配置文件不存在是否报错，默认是false;比如上文中指定的加载属性文件是authorSetting.properties。如果该文件不存在，则ignoreResourceNotFound为true的时候，程序不会报错，如果ignoreResourceNotFound为false的时候，程序直接报错。实际项目开发中，最好设置ignoreResourceNotFound为false。该参数默认值为false。
>
> 也可以通过`@ConfigurationProperties`读取配置信息并与 bean 绑定。

## SpringCloud常用注解

| 注解                    | 功能                                                         |
| ----------------------- | ------------------------------------------------------------ |
| @EnableEurekaServer     | 把当前微服务标记为Eureka注册中心 接收其他微服务的注册        |
| @EnableEurekaClient     | 注册该微服务到Eureka中                                       |
| @LoadBalanced           | 该注解写在配置RestTemplate的配置类方法上来启动ribbon负载均衡 |
| @EnableFeignClients     | 写在主程序上来支持feign                                      |
| @HystrixCommand         | 服务熔断和降级配置                                           |
| @EnableCircuitBreaker   | 启用对Hystrix熔断机制的支持                                  |
| @FeignClient            | springboot调用外部接口                                       |
| @EnableHystrixDashboard | 加在主程序上启动服务监控                                     |

