---
title: SpringBoot自定义逗号分隔转集合注解
date: 2021-10-13 13:27
tags: java
categories: 
---

<!--more-->

## 注解

```java
package com.ccsa.common.core.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author hsn
 * 2021/10/13
 * SpiltList
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface SpiltList {
    //分割符
    String value() default ",";
}

```

## 自定义解析器

```java
package com.ccsa.common.data.resolver;

import cn.hutool.core.util.StrUtil;
import com.ccsa.common.core.annotation.SpiltList;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.MethodParameter;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.Arrays;
import java.util.Objects;
import java.util.stream.Collectors;

/**
 * Get方法里将逗号转成list
 * 1,2,3 ----> [1,2,3]
 * @author hsn
 * 2021/10/13
 * SplitListHandlerMethodArgumentResolver
 */
@Slf4j
public class SplitListHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
    /**
     * Whether the given {@linkplain MethodParameter method parameter} is
     * supported by this resolver.
     * 当参数里有SpiltList注解则调用该解析器
     * @param parameter the method parameter to check
     * @return {@code true} if this resolver supports the supplied parameter;
     * {@code false} otherwise
     */
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(SpiltList.class);
    }

    /**
     * Resolves a method parameter into an argument value from a given request.
     * A {@link ModelAndViewContainer} provides access to the model for the
     * request. A {@link WebDataBinderFactory} provides a way to create
     * a {@link WebDataBinder} instance when needed for data binding and
     * type conversion purposes.
     * 解析过程
     * @param parameter     the method parameter to resolve. This parameter must
     *                      have previously been passed to {@link #supportsParameter} which must
     *                      have returned {@code true}.
     * @param mavContainer  the ModelAndViewContainer for the current request
     * @param webRequest    the current request
     * @param binderFactory a factory for creating {@link WebDataBinder} instances
     * @return the resolved argument value, or {@code null} if not resolvable
     * @throws Exception in case of errors with the preparation of argument values
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        String requestParameter = webRequest.getParameter(Objects.requireNonNull(parameter.getParameterName()));
        SpiltList spiltList = parameter.getParameterAnnotation(SpiltList.class);
        assert requestParameter != null;
        assert spiltList != null;
        //获取list里的泛型
        ParameterizedType parameterizedType = (ParameterizedType) parameter.getParameter().getParameterizedType();
        if (parameterizedType.getActualTypeArguments().length == 0){
            throw new RuntimeException("not found parameterizedType in List<>");
        }
        Type actualTypeArgument = parameterizedType.getActualTypeArguments()[0];
        if (!(actualTypeArgument instanceof Class)){
            throw new RuntimeException("get class fail");
        }
        if (StrUtil.isBlank(requestParameter)){
            return Collections.emptyList();
        }
        Class<?> type =(Class<?>) actualTypeArgument;
        String[] split = requestParameter.split(spiltList.value());
        //调用泛型的valueOf方法，所以只支持基本的包装类型，这里也可以根据自己的需要进行处理
        Method valueOfMethod = type.getMethod("valueOf", String.class);
        return Arrays.stream(split).map(item -> {
            try {
                return valueOfMethod.invoke(null, item);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }).collect(Collectors.toList());
    }

}
```

## 配置

```java
package com.ccsa.common.data.mvc;

import com.ccsa.common.data.resolver.SplitListHandlerMethodArgumentResolver;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

/**
 * @author hsn
 * 2021/10/13
 * WebConfig
 */
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new SplitListHandlerMethodArgumentResolver());
    }
}
```

## 使用

```java
    @GetMapping("analyze")
    public R analyze(@Size(min = 1,message = "专业id不能为空") @SpiltList List<Integer> majors) {
        return R.ok(reviewService.analyse(majors));
    }
```

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211013132720570-1302493286_1730686650382.png)