# 集成Filter

在Spring MVC中，`DispatcherServlet`只需要固定配置到`web.xml`中，剩下的工作主要是专注于编写Controller。

但是，在Servlet规范中，我们还可以[使用Filter](../../../web/filter/index.html)。如果要在Spring MVC中使用`Filter`，应该怎么做？

有的童鞋在上一节的Web应用中可能发现了，如果注册时输入中文会导致乱码，因为Servlet默认按非UTF-8编码读取参数。为了修复这一问题，我们可以简单地使用一个EncodingFilter，在全局范围类给`HttpServletRequest`和`HttpServletResponse`强制设置为UTF-8编码。

可以自己编写一个EncodingFilter，也可以直接使用Spring MVC自带的一个`CharacterEncodingFilter`。配置Filter时，只需在`web.xml`中声明即可：

```xml
<web-app>
    <filter>
        <filter-name>encodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>

    <filter-mapping>
        <filter-name>encodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

因为这种Filter和我们业务关系不大，注意到`CharacterEncodingFilter`其实和Spring的IoC容器没有任何关系，两者均互不知晓对方的存在，因此，配置这种Filter十分简单。

我们再考虑这样一个问题：如果允许用户使用Basic模式进行用户验证，即在HTTP请求中添加头`Authorization: Basic email:password`，这个需求如何实现？

编写一个`AuthFilter`是最简单的实现方式：

```java
@Component
public class AuthFilter implements Filter {
    @Autowired
    UserService userService;

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        // 获取Authorization头:
        String authHeader = req.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Basic ")) {
            // 从Header中提取email和password:
            String email = prefixFrom(authHeader);
            String password = suffixFrom(authHeader);
            // 登录:
            User user = userService.signin(email, password);
            // 放入Session:
            req.getSession().setAttribute(UserController.KEY_USER, user);
        }
        // 继续处理请求:
        chain.doFilter(request, response);
    }
}
```

现在问题来了：在Spring中创建的这个`AuthFilter`是一个普通Bean，Servlet容器并不知道，所以它不会起作用。

如果我们直接在`web.xml`中声明这个`AuthFilter`，注意到`AuthFilter`的实例将由Servlet容器而不是Spring容器初始化，因此，`@Autowire`根本不生效，用于登录的`UserService`成员变量永远是`null`。

所以，得通过一种方式，让Servlet容器实例化的Filter，间接引用Spring容器实例化的`AuthFilter`。Spring MVC提供了一个`DelegatingFilterProxy`，专门来干这个事情：

```xml
<web-app>
    <filter>
        <filter-name>authFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>authFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

我们来看实现原理：

1. Servlet容器从`web.xml`中读取配置，实例化`DelegatingFilterProxy`，注意命名是`authFilter`；
2. Spring容器通过扫描`@Component`实例化`AuthFilter`。

当`DelegatingFilterProxy`生效后，它会自动查找注册在`ServletContext`上的Spring容器，再试图从容器中查找名为`authFilter`的Bean，也就是我们用`@Component`声明的`AuthFilter`。

`DelegatingFilterProxy`将请求代理给`AuthFilter`，核心代码如下：

```java
public class DelegatingFilterProxy implements Filter {
    private Filter delegate;
    public void doFilter(...) throws ... {
        if (delegate == null) {
            delegate = findBeanFromSpringContainer();
        }
        delegate.doFilter(req, resp, chain);
    }
}
```

这就是一个[代理模式](../../../design-patterns/structural/proxy/index.html)的简单应用。我们画个图表示它们之间的引用关系如下：

```ascii
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
  ┌─────────────────────┐        ┌───────────┐   │
│ │DelegatingFilterProxy│─│─│─ ─▶│AuthFilter │
  └─────────────────────┘        └───────────┘   │
│ ┌─────────────────────┐ │ │    ┌───────────┐
  │  DispatcherServlet  │─ ─ ─ ─▶│Controllers│   │
│ └─────────────────────┘ │ │    └───────────┘
                                                 │
│    Servlet Container    │ │  Spring Container
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

如果在`web.xml`中配置的Filter名字和Spring容器的Bean的名字不一致，那么需要指定Bean的名字：

```xml
<filter>
    <filter-name>basicAuthFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <!-- 指定Bean的名字 -->
    <init-param>
        <param-name>targetBeanName</param-name>
        <param-value>authFilter</param-value>
    </init-param>
</filter>
```

实际应用时，尽量保持名字一致，以减少不必要的配置。

要使用Basic模式的用户认证，我们可以使用curl命令测试。例如，用户登录名是`tom@example.com`，口令是`tomcat`，那么先构造一个使用URL编码的`用户名:口令`的字符串：

```plain
tom%40example.com:tomcat
```

对其进行Base64编码，最终构造出的Header如下：

```plain
Authorization: Basic dG9tJTQwZXhhbXBsZS5jb206dG9tY2F0
```

使用如下的`curl`命令并获得响应如下：

```plain
$ curl -v -H 'Authorization: Basic dG9tJTQwZXhhbXBsZS5jb206dG9tY2F0' http://localhost:8080/profile
> GET /profile HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.64.1
> Accept: */*
> Authorization: Basic dG9tJTQwZXhhbXBsZS5jb206dG9tY2F0
> 
< HTTP/1.1 200 
< Set-Cookie: JSESSIONID=CE0F4BFC394816F717443397D4FEABBE; Path=/; HttpOnly
< Content-Type: text/html;charset=UTF-8
< Content-Language: en-CN
< Transfer-Encoding: chunked
< Date: Wed, 29 Apr 2020 00:15:50 GMT
< 
<!doctype html>
...HTML输出...
```

上述响应说明`AuthFilter`已生效。

```alert type=warning title=注意
Basic认证模式并不安全，本节只用来作为使用Filter的示例。
```

### 练习

使用DelegatingFilterProxy实现AuthFilter。

[下载练习](spring-web-mvc-filter.zip)

### 小结

当一个Filter作为Spring容器管理的Bean存在时，可以通过`DelegatingFilterProxy`间接地引用它并使其生效。
