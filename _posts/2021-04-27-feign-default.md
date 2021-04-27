---
title: "Feign interface包含default方法"
---

# 1. Feign interface包含default方法
通常情况下，会写个feign interface约定Spring Cloud微服务API，服务提供会实现该接口。服务依赖方则直接扫描改interface，得到feign client bean。

如果feign interface的方法，有default实现，结果会怎样？

## 1.1 测试过程

- Feign interface

```java
@FeignClient(name = "demo-spring-feign-server", contextId = "feignDemo")
@RequestMapping("/demo/feign")
public interface FeignDemoFeign {

    @GetMapping("/test")
    default String test() {
        return "default";
    }

    @GetMapping("/test/no/default")
    String testNoDefault();
}
```

- Feign 服务实现

```java
@RestController
public class FeignDemoController implements FeignDemoFeign {

    @Override
    public String test() {
        return "Implementation";
    }

    @Override
    public String testNoDefault() {
        return "Implementation, no-default";
    }
}
```

- 服务调用

FeignClientApplication
```java
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "com.vbobot.spring.demo.feign.api")
@SpringBootApplication
public class FeignClientApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(FeignClientApplication.class)
                .web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

```java
@Slf4j
@RestController
@RequestMapping("/feign/demo")
public class FeignDemoClientController {
    @Resource FeignDemoFeign feignDemoFeign;

    /**
     * 接口的default方法，会直接本地调整，不触发远程调用
     * ReflectiveFeign.java:59
     */
    @GetMapping("/test")
    public String test() {
        final String test = feignDemoFeign.test();
        log.info("Result:{}", test);
        return test;
    }

    @GetMapping("/test/no/default")
    public String testNoDefault() {
        final String noDefault = feignDemoFeign.testNoDefault();
        log.info("Result:{}", noDefault);
        return noDefault;
    }
}

```

- 结果

```shell
$ curl http://127.0.0.1:8080/feign/demo/test
default

$ curl http://127.0.0.1:8080/feign/demo/test/no/default
Implementation, no-default
```

有默认实现的，直接调用了返回了默认实现。没有默认实现的，则调用对应了远程方法。

## 1.2 源码分析

- 在调用处理打断点
- 进入实际调用的方法
  实际调用的是：feign.ReflectiveFeign.FeignInvocationHandler.invoke

    ```Java
    // feign-core-10.1.0-sources.jar!/feign/ReflectiveFeign.java:88
  
  static class FeignInvocationHandler implements InvocationHandler {
  
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          if ("equals".equals(method.getName())) {
              try {
                  Object otherHandler =
                    args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
                return equals(otherHandler);
              } catch (IllegalArgumentException e) {
                return false;
              }
          } else if ("hashCode".equals(method.getName())) {
              return hashCode();
          } else if ("toString".equals(method.getName())) {
              return toString();
          }
      
          return dispatch.get(method).invoke(args); 
      }
    }
    ```
  
    实际从dispatch从取出`MethodHandler`，dispatch由构造方法初始化，继续找构造方法调用
  
    ```java
    // feign-core-10.1.0-sources.jar!/feign/ReflectiveFeign.java:80
  
    static class FeignInvocationHandler implements InvocationHandler {
    
        private final Target target;
        private final Map<Method, MethodHandler> dispatch;
    
        FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
          this.target = checkNotNull(target, "target");
          this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
        }
        ...
    }
    ```
  
    继续往下找哪里调用了FeignInvocationHandler的构造方法，在`feign.InvocationHandlerFactory.Default.create`
    ```java
    // feign.InvocationHandlerFactory.Default.create
  
        @Override
        public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
          return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
        }
    ```

    再往下找调用位置，`feign.ReflectiveFeign.newInstance`，在这里根据Method的不同，使用了`DefaultMethodHandler`或者`SynchronousMethodHandler`

  ```java
  @Override
    public <T> T newInstance(Target<T> target) {
      Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
      Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
      List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
  
      for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
          continue;
        } else if (Util.isDefault(method)) { // 这里
          DefaultMethodHandler handler = new DefaultMethodHandler(method);
          defaultMethodHandlers.add(handler);
          methodToHandler.put(method, handler);
        } else {
          methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
      }
      InvocationHandler handler = factory.create(target, methodToHandler);
      T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
          new Class<?>[] {target.type()}, handler);
  
      for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
      }
      return proxy;
    }
  ```

  ```java
  /**
     * Identifies a method as a default instance method.
     */
    public static boolean isDefault(Method method) {
      // Default methods are public non-abstract, non-synthetic, and non-static instance methods
      // declared in an interface.
      // method.isDefault() is not sufficient for our usage as it does not check
      // for synthetic methods. As a result, it picks up overridden methods as well as actual default
      // methods.
      final int SYNTHETIC = 0x00001000;
      return ((method.getModifiers()
          & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC | SYNTHETIC)) == Modifier.PUBLIC)
          && method.getDeclaringClass().isInterface();
    }
  ```

# 2. 涉及源代码

[源码](https://github.com/HuaboTang/spring-demo/tree/master/spring-feign)
