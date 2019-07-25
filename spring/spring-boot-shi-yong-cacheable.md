# Spring Boot 使用 Cacheable

### 相关文档：

{% embed url="https://docs.spring.io/spring/docs/5.0.4.BUILD-SNAPSHOT/javadoc-api/org/springframework/cache/annotation/CacheConfig.html" %}

{% embed url="https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-cache/" %}

{% embed url="https://www.cnblogs.com/yueshutong/p/9381540.html" %}

### 示例:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@EnableCaching
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@CacheConfig(cacheNames = "Demo")
public class DemoService {
    @Cacheable(key = "#name")
    public String hrefPermission(Long name) {
        log.info("测试缓存, userId: {} ", name);
        return "Hello Cache";
    }
}

```

