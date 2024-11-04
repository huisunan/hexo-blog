---
title: swagger忽略方法里参数的方法
date: 2021-05-07 11:25
tags: java
categories: 
---

<!--more-->

```java
 @Bean
    public Docket createRestApi(){
        return new Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any()).build()
                .ignoredParameterTypes(UserInfo.class, HttpSession.class);//在此忽略参数的class
    }
```