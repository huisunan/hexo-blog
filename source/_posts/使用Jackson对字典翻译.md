---
title: 使用Jackson对字典翻译
date: 2021-05-25 14:08
tags: java
categories: 
---

<!--more-->

# 使用Jackson对字典翻译

> 参考:<https://juejin.cn/post/6844904053844115470>

## 字典项注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@Documented
public @interface DictValue {
    //字典名称
    DictEnum value();
}

```

## Bean序列化更改器

```java
public class DictSerializerModifier extends BeanSerializerModifier {
    //这个方法在类第一次序列化时会调用一次，确定后就不会再更改了
    @Override
    public List<BeanPropertyWriter> changeProperties(SerializationConfig config, BeanDescription beanDesc, List<BeanPropertyWriter> beanProperties) {
        for (BeanPropertyWriter beanProperty : beanProperties) {
            DictValue dictValue = beanProperty.getAnnotation(DictValue.class);
            if (dictValue != null){
                DictFieldSerializer dictFieldSerializer = new DictFieldSerializer(dictValue.value().getName());
                //自定以序列器
                beanProperty.assignSerializer(dictFieldSerializer);
                //null值序列器
                beanProperty.assignNullSerializer(NullSerializer.instance);
            }
        }
        return beanProperties;
    }
}
```

## 自定义序列器

```java

public class DictFieldSerializer extends JsonSerializer<Object> {

    private String key;

    public DictFieldSerializer(String key){
        this.key = key;
    }

    //自定义写入方法
    @Override
    public void serialize(Object value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        IDictService dictService = SpringTool.getBean(IDictService.class);
        String dictValue = dictService.getDictValue(key, value.toString());
        gen.writeString(dictValue);
    }
}

```

## SpringBoot配置

```java

@Configuration
public class JacksonConfig {


    @Bean
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper objectMapper = builder.createXmlMapper(false).build();
        objectMapper.setSerializerFactory(objectMapper.getSerializerFactory().withSerializerModifier(new DictSerializerModifier()));
        return objectMapper;
    }
}

```