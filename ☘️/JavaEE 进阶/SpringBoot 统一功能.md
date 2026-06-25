# SpringBoot 统一功能
> 相关笔记：[[JavaEE进阶|JavaEE进阶 知识总结]]


# 拦截器

<u>**Spring 拦截器是一种作用于 Spring MVC 层级的横切处理机制，通过拦截并增强 DispatcherServlet 调度的处理器执行链，实现对 HTTP 请求全生命周期的统一逻辑控制（如权限校验、日志记录、参数预处理等）**</u>

拿登录接口举例子：拦截器就像是 Controller 门口的保安，在请求进入核心逻辑前先查验登录状态，没带“证件”（Token/Session）的请求直接拦住并遣返，不让它打扰业务代码，避免了在业务代码中频繁验证状态的操作，增加开发效率。有了拦截器后只需要在一个地方写好登录校验逻辑

**类似于servlet的过滤器filter，功能类似**

## 功能实现

拦截器的使用分为两个步骤

1. 自定义拦截器

实现`HandlerInterceptor`接口，重写所有方法

![image](image-20260206004512-6k3cx4f.png)

以重写`preHandle`方法为例

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {

    private final ObjectMapper objectMapper;

    public LoginInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("目标方法执行前执行");
        HttpSession session = request.getSession();
        // 判断用户是否登录
        UserInfo attribute = (UserInfo)session.getAttribute(Constant.SESSION_USER_KEY);
        if (attribute == null || attribute.getId() <= 0) {
            // 给前端返回信息 便于判断
            response.setStatus(401); // 401 用户未登录
            ServletOutputStream outputStream = response.getOutputStream();
            Result<Object> objectResult = Result.unLogin();// 设置返回信息
            byte[] bytes = objectMapper.writeValueAsBytes(objectResult);
            outputStream.write(bytes);
            // 记得清缓存并关闭
            outputStream.flush();
            outputStream.close();
            return false; // false：拦截
        }
        return true; // true: 放行
    }
}
```

2. 配置拦截器

定义好拦截器的具体内容后，就应该配置拦截器应该出现在那些接口前面，拦截哪些请求

实现`WebMvcConfigurer`​接口，重写`addInterceptors`​方法，并加上注解`@Configuration`

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LoginInterceptor loginInterceptor;
    /**
     * 除了excludePathPatterns之外，addPathPatterns内文件路径内的请求都会被拦截
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns(
                        "/test/**", // 测试统一异常处理
                        "/user/login",
                        "/**/*.html",
                        "/css/**",
                        "/js/**",
                        "/**/**.ico",
                        "/pic/**");
    }
}
```

![image](image-20260206005655-5zb1e1n.png)

---

Spring是基于servlet封装的一个框架，所以在Spring中也能见到servlet的身影

# DispatcherServlet

# 统一数据返回格式

**<u>给所有成功的响应封装上统一的包装类，确保前端拿到的数据结构永远固定、可预期，当然也可以筛选接口</u>**

## 功能实现

通过`@ControllerAdvice`​ + `implement ResponseBodyAdvice`实现

```java
@ControllerAdvice
public class ResponseAdvice implements ResponseBodyAdvice {

    @Autowired
    private ObjectMapper objectMapper;

    // 1. 判断哪些接口需要包装（作筛选）
    // true: 需要处理，交给beforeBodyWrite
    // false: 不交给它处理，直接返回目标返回的对象
    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return true;
    }

    // 2. 在这里对返回的对象进行处理
    // body 为目标方法返回的结果
    @SneakyThrows
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // String 类型的特殊性
        if (body instanceof String) {
            return objectMapper.writeValueAsString(Result.success(body));
        }
        // 如果body已经是Result对象，则直接返回
        if (body instanceof Result<?>) return body;
        return Result.success(body);
    }
}
```

- **返回String 类型的特殊性**：因为`ResponseBodyAdvice`​ 处理 `String`​ 类型时，Spring 会优先使用 `StringHttpMessageConverter`​。这会导致它尝试把你的 `Result`​ 对象强制转成 `String`​，从而抛出 `ClassCastException`

# 统一异常处理

<u>**集中捕获并处理程序中抛出的各类异常，防止崩溃堆栈直接甩给用户，确保报错时也能返回规范的提示信息**</u>

## 功能实现

使用`@ControllerAdvice`​ + `@ExceptionHandler`​实现，并且可以分异常处理，Spring 会根据抛出的异常类型，“**就近原则**”匹配最合适的处理逻辑（这里的就近原则是在统一处理异常时列出的异常情况）

```java
/**
 * ResponseBody: 确保异常处理后的结果直接以 JSON 形式返回给前端，而不是去寻找 HTML 页面。
 */
@Slf4j
@ResponseBody
@ControllerAdvice
public class ExceptionAdvice {
    @ExceptionHandler
    public Object handler(Exception e) {
        log.error("发生内部错误, e:{}", e.getMessage());
        return Result.fail("内部错误");
    }

    @ExceptionHandler
    public Object handler(ArithmeticException e){
        log.error("发生算术错误, e:{}", e.getMessage());
        return Result.fail("发生算术异常");
    }

    @ExceptionHandler
    public Object handler(NullPointerException e){
        log.error("发生空指针, e:{}", e.getMessage());
        return Result.fail("发生空指针异常");
    }
}
```

测试接口

```java
@RestController
@RequestMapping("/test")
public class TestExceptionController {
    // 定位到Exception
    @RequestMapping("/t1")
    public String t1(){
        int[] aa = {1,2,3};
        System.out.println(aa[3]);
        return "t1";
    }
    // 定位到ArithmeticException
    @RequestMapping("/t2")
    public Integer t2(){
        int a = 10/0;
        return 10;
    }
    // 定位到NullPointerException
    @RequestMapping("/t3")
    public Boolean t3(){
        String aa = null;
        System.out.println(aa.length());
        return true;
    }
}
```

‍

SpringBoot的统一功能实现都体现了[[（重要)SpringAOP|SpringAOP]]思想——面向切面编程
