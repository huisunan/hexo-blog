---
title: word模板生成，基于freemarker
date: 2021-10-20 11:05
tags: java
categories: 
---

<!--more-->

# word模板生成，基于freemarker

## 创建word模板文档

文档的格式为docx，如果不是另存为docx文档  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020101408861-898722344_1730686630335.png)

**注意是word文档\(.docx\)**

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020101511629-1885950339_1730686630335.png)

## 提取需要替换的文件

docx文档其实是zip格式的，修改后缀就可以看到文档里的实际内容  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020101710009-1064901690_1730686630335.png)  
用压缩软件打开  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020110407836-925296304_1730686641760.png)

里面的内容符合[微软officeopenxml规范](http://www.officeopenxml.com/)

会用到的文件：

- \[Content\_Types\].xml 文件格式的声明，如果加入新的图片格式，需要再这里添加
- word/media 图片存储的文件夹
- word/\_rels/document.xml.rels 文档图片的声明
- word/document.xml 文档的内容

## 创建freemarker模板

将word/\_rels/document.xml.rels和word/document.xml复制并修改后缀名  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020102930684-1987465277_1730686641760.png)

### 替换一个标题

在模板里找到标题  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020103028034-1546279334_1730686641760.png)

### 渲染图片列表

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020105406015-208164383_1730686641760.png)

处理图片引用

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020105717810-74506227_1730686641760.png)

## 渲染

ImageVo.class

```java
public class ImageVo {
    private String id;
    private String path;
    private String name;


    public ImageVo(String id,String name,String file){
        this.id = id;
        this.name = name;
        this.path = file;
    }


    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getPath() {
        return path;
    }

    public void setPath(String path) {
        this.path = path;
    }
}
```

word工具类

```java
public class WordUtil {
    private static final Configuration configuration = new Configuration(Configuration.VERSION_2_3_23);

    static {
        //设置编码
        configuration.setDefaultEncoding("utf-8");
        //ftl模板文件
        configuration.setClassForTemplateLoading(WordUtil.class, "/template/");
    }

    public static void createWord(Map<String,Object> dataMap, String templateName, OutputStream outputStream) throws Exception{
        //获取模板
        Template template = configuration.getTemplate(templateName);
        //生成文件
        template.process(dataMap, new BufferedWriter(new OutputStreamWriter(outputStream, StandardCharsets.UTF_8)));
    }
}
```

代码逻辑

```java
public class Main {

    private static final String outFile = "out.docx";

    private static final List<ImageVo> imageVos = new LinkedList<>();

    static {
        String absolutePath = new ClassPathResource("image3.jpeg").getAbsolutePath();
        imageVos.add(new ImageVo("image1","image20.jpeg",absolutePath));
        imageVos.add(new ImageVo("image2","image21.jpeg",absolutePath));
        imageVos.add(new ImageVo("image3","image22.jpeg",absolutePath));
        imageVos.add(new ImageVo("image4","image23.jpeg",absolutePath));
    }



    public static void main(String[] args) {
        Map<String,Object> params = new HashMap<>();
        params.put("title","填入的标题");//标题
        params.put("imageList",imageVos);//图片列表
        //创建Zip流
        try (ZipInputStream zipInputStream = new ZipInputStream(new ClassPathResource("template/template.docx").getStream());
             ZipOutputStream zipOutputStream = new ZipOutputStream(new FileOutputStream(outFile));
        ) {
            ZipEntry entryIn;
            while ((entryIn = zipInputStream.getNextEntry()) != null) {
                String entryInName = entryIn.getName();
                ZipEntry entryOut = new ZipEntry(entryIn.getName());

                zipOutputStream.putNextEntry(entryOut);
                if ("word/_rels/document.xml.rels".equals(entryInName)) {
                    //处理图片引用
                    WordUtil.createWord(params, "rels.ftl", zipOutputStream);
                } else if ("word/document.xml".equals(entryInName)) {
                    //填充word模板
                    WordUtil.createWord(params, "document.ftl", zipOutputStream);
                } else {
                    //拷贝原始的字节
                    IoUtil.copy(zipInputStream, zipOutputStream);
                }
                zipOutputStream.closeEntry();
            }

            //写入图片
            for (ImageVo imageVo : imageVos) {
                ZipEntry e = new ZipEntry("word/media/" + imageVo.getName());
                zipOutputStream.putNextEntry(e);
                try (InputStream openStream = new FileInputStream(imageVo.getPath())) {
                    IoUtil.copy(openStream, zipOutputStream);
                }
                zipInputStream.closeEntry();
            }

        } catch (Exception e) {
            throw new RuntimeException("导出异常");
        }
    }
}

```

## 生成的效果图

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20211020110113264-1489170074_1730686650382.png)

表格也是有一定的逻辑，可以自己探索一下

## Git地址

<https://gitee.com/huisunan/word-template-freemarker>