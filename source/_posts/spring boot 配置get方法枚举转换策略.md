---
title: spring boot 配置get方法枚举转换策略
date: 2023-12-19 18:05
tags: java
categories: 
---

<!--more-->

配置转换器

```java
@SuppressWarnings({"rawtypes", "unchecked"})
public class CompositeEnumConverterFactory implements ConverterFactory<String, Enum<?>> {
	@Override
	public <T extends Enum<?>> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToEnum<>(targetType);
	}
}
```

注入spring容器中

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(WebMvcConfigurer.class)
public class ZbMvcConfig implements WebMvcConfigurer {

	@Override
	public void addFormatters(FormatterRegistry registry) {
		registry.addConverterFactory(new CompositeEnumConverterFactory());
	}

}

```