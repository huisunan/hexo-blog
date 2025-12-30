---
title: mapstruct 1.4.2和lombok 1.18.16之后版本，报错和mapstruct生成空的实现
date: 2021-05-14 12:42
tags: java
categories: 
---

<!--more-->

```xml
  <properties>
        <java.version>1.8</java.version>
        <org.mapstruct.version>1.4.2.Final</org.mapstruct.version>
    </properties>
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${org.mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>  <!-- 如果是0.1.0 有可能出现生成了maptruct的实现类，但该类只创建了对象，没有进行赋值 -->
                        </path>

                    </annotationProcessorPaths>
                </configuration>
            </plugin>
```