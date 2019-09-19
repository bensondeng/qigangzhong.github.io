---
layout: post
title: "SpringBoot拦截请求的方式"
categories: springboot
tags: springboot
author: 网络
---

* content
{:toc}

介绍SpringBoot中常见的拦截请求的方式，每种方式的特点，以及它们之间的区别











## 一、Filter

依赖servlet容器，可以处理request、response，但是无法处理controller、method或者更细的方法

```java
@Component
public class TimeFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("time filter init");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("time filter start");
        long startTime = System.currentTimeMillis();

        filterChain.doFilter(servletRequest, servletResponse);

        long endTime = System.currentTimeMillis();
        System.out.println("time filter consume " + (endTime - startTime) + " ms");
        System.out.println("time filter end");
    }

    @Override
    public void destroy() {
        System.out.println("time filter init");
    }
}
```

## 二、HandlerInterceptor

由spring框架提供，不依赖servlet容器，基于java反射机制

可以处理request、response，也可以处理controller、method（也就是handler对象），但是method的参数值无法获取，只能知道参数名称

```java
@Component
public class TimeInterceptor extends HandlerInterceptorAdapter {

    private final NamedThreadLocal<Long> startTimeThreadLocal = new NamedThreadLocal<>("startTimeThreadLocal");

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("time interceptor preHandle");

        HandlerMethod handlerMethod = (HandlerMethod) handler;
        // 获取处理当前请求的 handler 信息
        System.out.println("handler 类：" + handlerMethod.getBeanType().getName());
        System.out.println("handler 方法：" + handlerMethod.getMethod().getName());

        MethodParameter[] methodParameters = handlerMethod.getMethodParameters();
        for (MethodParameter methodParameter : methodParameters) {
            String parameterName = methodParameter.getParameterName();
            // 只能获取参数的名称，不能获取到参数的值
            //System.out.println("parameterName: " + parameterName);
        }

        // 把当前时间放入 threadLocal
        startTimeThreadLocal.set(System.currentTimeMillis());

        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("time interceptor postHandle");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

        // 从 threadLocal 取出刚才存入的 startTime
        Long startTime = startTimeThreadLocal.get();
        long endTime = System.currentTimeMillis();

        System.out.println("time interceptor consume " + (endTime - startTime) + " ms");

        System.out.println("time interceptor afterCompletion");
    }
}
```

拦截器与filter不同的是，需要添加才能生效

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
    //添加拦截器
    @Autowired
    private TimeInterceptor timeInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(timeInterceptor);
    }
}
```

## 三、AspectJ

可以处理controller、method，无法直接处理request、response，但是可以间接通过RequestContextHolder来处理

```java
@Aspect
@Component
public class TimeAspect {

    @Around("execution(* com.winchain.freshshare.salespartner.webapi.webapihost.controller.*.*(..))")
    public Object handleControllerMethod(ProceedingJoinPoint pjp) throws Throwable {

        System.out.println("time aspect start");

        //获取request
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        Object[] args = pjp.getArgs();
        for (Object arg : args) {
            System.out.println("arg is " + arg);
        }

        long startTime = System.currentTimeMillis();

        Object object = pjp.proceed();

        long endTime = System.currentTimeMillis();
        System.out.println("time aspect consume " + (endTime - startTime) + " ms");

        System.out.println("time aspect end");

        return object;
    }
}
```

## 四、ControllerAdvice

ControllerAdvice是对Controller的增强，一般用来配合@ExceptionHandler、@InitBinder、@ModelAttribute等注解来实现异常捕获、参数处理等功能，也可以当做filter，对request，response做处理。

### 全局参数处理、异常捕获

根据异常类型捕获异常，做相应的处理。

```java
/**
 * 自定义异常，测试ControllerAdvice
 */
public class CustomException extends RuntimeException {

    private long code;
    private String msg;

    public CustomException(Long code, String msg){
        this.code = code;
        this.msg = msg;
    }

    public long getCode() {
        return code;
    }

    public void setCode(long code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}

//通过ControllerAdvice捕获指定类型的异常信息
@ControllerAdvice
public class MyExceptionHandler {

    /**
     * 对指定的请求参数进行预处理
     * 可以设置参数的前缀，设置允许、禁止字段，设置必填字段以及验证器
     * 在controller中需要处理的参数前需要添加@ModelAttribute("param_name")注解
     * @param binder
     */
    @InitBinder("param1")
    public void initWebBinder(WebDataBinder binder){
        binder.setFieldDefaultPrefix("prefix1.");
    }
    @InitBinder("param2")
    public void initWebBinder(WebDataBinder binder){
        binder.setFieldDefaultPrefix("prefix2.");
    }

    /**
     * 绑定默认参数值
     * 在所有的controller>action方法中可以通过ModelMap modelMap, @ModelAttribute("attribute") String attribute两种方式获取到这个值
     * @param model
     */
    @ModelAttribute
    public void addAttribute(Model model) {
        model.addAttribute("attribute",  "The Attribute");
    }

    /**
     * 捕获CustomException
     * @param e
     * @return 返回json格式数据
     */
    @ResponseBody
    @ExceptionHandler({CustomException.class}) //指定拦截异常的类型
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) //自定义浏览器返回状态码
    public Map<String, Object> customExceptionHandler(CustomException e) {
        Map<String, Object> map = new HashMap<>();
        map.put("code", e.getCode());
        map.put("msg", e.getMsg());
        return map;
    }
}

@RestController
public class TestController {
    @GetMapping("/test")
    public String test(ModelMap modelMap, @ModelAttribute("attribute") String attribute){
        throw new CustomException(400L, "出问题啦, " + modelMap.get("attribute") + " " + attribute);
        //return modelMap.get("attribute") + " " + attribute;
    }
}
```

### 处理request、response

* 可以直接注入`HttpServletRequest`，`HttpServletResponse`对象，对request和response进行处理

```java
@ControllerAdvice
@ResponseBody
public class ApiExceptionAdvice {
    public static final Logger logger = LoggerFactory.getLogger(ApiExceptionAdvice.class);

    @ExceptionHandler(value=Exception.class)
    @IgnoreApiResult
    public ApiResult<String> allExceptionHandler(HttpServletRequest request, Exception exception)
    {
        logger.error("Unhandled exception, request uri:"+request.getRequestURI(), exception);
        return new ApiResult<>(ErrorCode.ServerError.getCode(), ErrorCode.ServerError.getDescription(), false, ExceptionUtils.getStackTrace(exception));
    }

    @ExceptionHandler(value = ApiException.class)
    @IgnoreApiResult
    public ApiResult<Object> customExceptionHandler(HttpServletRequest request, ApiException exception){
        logger.error("ApiException, request uri:"+request.getRequestURI(), exception);
        return new ApiResult<>(exception.getCode(), exception.getMsg(), false, exception.getValue());
    }
}
```

* 也可以实现`RequestBodyAdvice`、`ResponseBodyAdvice`接口来对request、response进行处理

`RequestBodyAdvice`可以用来对请求的内容进行加密解密或者其它预处理

`ResponseBodyAdvice`可以在请求返回到客户端之前对响应数据进行处理，内容加解密，同一封装等操作

```java
//示例对controller返回结果做同一封装(同一返回json数据格式为{"code":0,"msg":"this is message","data":"this is data"})
@ControllerAdvice
public class ApiResponseAdvice implements ResponseBodyAdvice<Object> {
    //这个方法可以判断是否继续执行下面的beforeBodyWrite方法
    @Override
    public boolean supports(MethodParameter methodParameter, Class<? extends HttpMessageConverter<?>> aClass) {
        String className = methodParameter.getMethod().getDeclaringClass().getName();
        if (className.equalsIgnoreCase(HttpConfigContract.SECURITY_OAUTH2_TOKEN_ENDPOINT)
                || className.equalsIgnoreCase(HttpConfigContract.SWAGGER_WEB_APIRESOURCE_CONTROLLER)
                || className.equalsIgnoreCase(HttpConfigContract.SWAGGER_WEB_SWAGGER2_CONTROLLER)) {
            //token相关的接口以及swagger相关接口不进行封装
            return false;
        }
        List<Annotation> classAnnotations = Arrays.asList(methodParameter.getMethod().getDeclaringClass().getDeclaredAnnotations());
        List<Annotation> methodAnnotations = Arrays.asList(methodParameter.getMethodAnnotations());
        if (methodAnnotations.stream().anyMatch(mAnnotation -> mAnnotation.annotationType().equals(IgnoreApiResult.class) ||
                classAnnotations.stream().anyMatch(cAnnotation -> cAnnotation.annotationType().equals(IgnoreApiResult.class)))) {
            return false;
        }
        return true;
    }


    //这个方法可以修改response
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends
            HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {

        ServletServerHttpResponse sshResp = (ServletServerHttpResponse) response;
        HttpServletResponse httpServletResponse = sshResp.getServletResponse();
        if (httpServletResponse.getStatus() != HttpConfigContract.HTTPCODE_SUCCESS_200 || body instanceof ApiResult) {
            //仅封装200的请求
            return body;
        }

        return new ApiResult(ErrorCode.Success.getCode(), ErrorCode.Success.getDescription(), true, body);
    }
}
```

## 五、几种方式的处理顺序

![springboot-request-process.jpg](/images/spring/springboot-request-process.jpg)

## 参考

[spring boot RESTFul API拦截 以及Filter和interceptor 、Aspect区别](https://www.cnblogs.com/bingshu/p/7819932.html)

[@ControllerAdvice的使用详解3（请求参数预处理 @InitBinder）](https://www.hangge.com/blog/cache/detail_2483.html)
