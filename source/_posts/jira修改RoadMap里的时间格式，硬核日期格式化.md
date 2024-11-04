---
title: jira修改RoadMap里的时间格式，硬核日期格式化
date: 2024-07-04 02:39
tags: other
categories: 
---

<!--more-->

## jira修改roadMap里的时间格式

在插件目录找到portfolio-plugin-9.16.1.jar将他下载到本地

使用zip解压软件解压jar包

全局搜索 DD/MM/YY 将其替换YYYY/MM/DD

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240704021643685-138945748_1730686593219.png)

修改后效果图

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240704021902180-1363806614_1730686593219.png)  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240704021925960-489974832_1730686593219.png)

全局搜索

return`${l()(o.getUTCDate().toString(),2,"0")}/${t}`

替换

return `${o.getUTCMonth()+1}/${l()(o.getUTCDate().toString(), 2, "0")}`

效果图

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240704022125055-91458395_1730686593219.png)

全局搜索

return`${t} ${l()(""+o.getUTCDate(),2,"0")}/${a}`

替换

return `${o.getUTCMonth()}/${l()("" + o.getUTCDate(), 2, "0")} 星期${['日','一','二','三','四','五','六'][o.getUTCDay()]}`

效果  
![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20240704024640829-57160194_1730686593219.png)

全部替换完成后，将文件压缩成zip包，将后缀修改成jar，并上传到插件目录下替换

这里是已经替换好的文件包