---
title: 闲着没事做，用js做了一个冒泡排序的动画
date: 2021-06-14 20:45
tags: java
categories: 
---

<!--more-->

![](https://raw.githubusercontent.com/huisunan/cdn/main/img/1410909-20210614204455766-1774322076_1730686672795.gif)

```html
<!DOCTYPE html>

<head>
    <script src="https://cdn.bootcdn.net/ajax/libs/jquery/3.6.0/jquery.min.js"></script>
    <script>
        let arr = [];
        function draw() {
            arr.forEach((item, index) => {
                // $("#context ol").append("<li style='height:" + item + "px;left:" + index * 30 + "px'>" + item + "</li>")
                $("#context ol").append("<li style='height:" + item + "px'>" + item + "</li>")
            })

        }

        function mark(i1, i2) {
            $($("#context ol li")[i1]).css("background-color", "#faa755");
            $($("#context ol li")[i2]).css("background-color", "#faa755");
        }
        function removeMark(i1, i2) {
            $($("#context ol li")[i1]).css("background-color", "black");
            $($("#context ol li")[i2]).css("background-color", "black");
        }
        function swap(i1, i2) {
            let liArr = $("#context ol").find("li").toArray();
            let temp = liArr[i1]
            liArr[i1] = liArr[i2]
            liArr[i2] = temp
            $("#context ol").empty()
            $("#context ol").append(liArr)
        }
        function sleep(time) {
            return new Promise((resolve) => setTimeout(resolve, time));
        }


        async function sort() {
            let next;
            //升序排序
            for (let i = 0; i < arr.length - 1; i++) {
                for (let j = 0; j < arr.length - 1 - i; j++) {

                    mark(j, j + 1)
                    if (arr[j] > arr[j + 1]) {
                        let temp = arr[j]
                        arr[j] = arr[j + 1]
                        arr[j + 1] = temp
                        swap(j,j+1)
                    }
                    await sleep(100)
                    removeMark(j, j + 1)

                }

            }

            arr.forEach(item => console.log(item))
            $("#message").text("成功了！")
        }



        $(function () {
            for (let i = 0; i < 20; i++) {
                arr.push(Math.floor(Math.random() * 80) + 20)
            }
            draw()
            sort()
        })
    </script>
    <style>
        #context {
            width: 1000px;
            height: 300px;
        }

        #context ol li {
            background-color: black;
            display: inline-block;
            color: white;
            margin-right: 10px;
            vertical-align: bottom;
        }
    </style>
</head>
<html>
<div id="context">
    <ol>

    </ol>
</div>
<div id="message">

</div>
</html>

```