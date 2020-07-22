---
title: 一文了解 Spring Starter 原理
date: 2020-04-22 21:15:38
tags:
- SpringBoot
---


#### 简介
Spring Starter相当于模块，它能将模块所需的依赖整合起来并对模块内的Bean根据环境（ 条件）进行自动配置。使用者只需要依赖相应功能的Starter，无需做过多的配置和依赖，Spring Boot就能自动扫描并加载相应的模块。


#### Quick Start

我们以创建一个钉钉机器人 starter 为例

1. 创建项目，添加必选依赖

```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.6.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>2.0.6.RELEASE</version>
        </dependency>
    </dependencies>
```

2. 创建配置类，用于读取配置信息

```
@ConfigurationProperties(prefix = "dingding.chatbot")
@Data
public class DingdingProperties {
    private String secret;
    private String accessToken;
    private String webHook = "https://oapi.dingtalk.com/robot/send?access_token=";
    private Double rate = 1D/6D;
}
```

@ConfigurationProperties 注解用于设置配置信息的前缀

例如：accessToken 在配置文件中`dingding.chatbot.access-token = 123456`

3. 创建自动装配类，创建想要生成的对象并装配到容器中

```
@Slf4j
@Configuration
@EnableConfigurationProperties(DingdingProperties.class)
public class DingdingAutoConfiguration {

    @Resource
    private DingdingProperties dingdingProperties;

    @Bean
    @ConditionalOnMissingBean(DingChatBot.class)
    public DingChatBot createDingdingBean() {
        return new DingChatBot(
                dingdingProperties.getSecret(),
                dingdingProperties.getAccessToken(),
                dingdingProperties.getWebHook() + dingdingProperties.getAccessToken(),
                dingdingProperties.getRate());
    }
}
```


4. 创建 spring.factories 文件，用于 SpringBoot 扫描

内容如下：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.souche.chatbot.dingding.DingdingAutoConfiguration

```


spring.factories 放在 resources/META-INF 目录下。
容器启动时，SpringFactoriesLoader 类会扫描该文件，将相应的类实例化装配到容器中。
关键代码如下：

```
    //解析成 CLASS 并实例化
	public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
		//获取 ClassLoader
		ClassLoader classLoaderToUse = classLoader;
		//解析成 CLASS
		List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);

		List<T> result = new ArrayList<>(factoryNames.size());
		for (String factoryName : factoryNames) {
		    //依次实例化
			result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
		}
		//排序后返回
		AnnotationAwareOrderComparator.sort(result);
		return result;
	}

	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}


	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	    //按 classLoader 保存已经解析出的类
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
		    //读取所有的 "META-INF/spring.factories" 文件
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			//按行读取 CLASS 绝对路径
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				//spring.factories 文件 解析成 KEY VALUE 形式
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					List<String> factoryClassNames = Arrays.asList(
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
					result.addAll((String) entry.getKey(), factoryClassNames);
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}

```


5. 添加 spring-boot-configuration-processor 依赖，可以在引用 starter 后自动读取相应的配置文件


```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
    <version>2.0.6.RELEASE</version>
</dependency>

```


6. 使用

1）添加上面的依赖

2）添加配置

```
# 钉钉
dingding.chatbot.secret= 1231231123121
dingding.chatbot.access-token = 1231231123121

```

3）使用demo

```
@Autowired
private DingChatBot chatBot;

public void method() {
    try {
        chatBot.sendMsg(errorMsg);
    } catch (RateLimiterException rateLimiterException) {
        log.error(rateLimiterException.getMessage(), rateLimiterException);
    }
}

```


[源码参考](https://github.com/JasonSongHoho/dingding-chatBot-spring-boot-starter)



