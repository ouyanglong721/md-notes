# Goland使用wsl2输出提前换行问题的解决

## 问题描述

搭建好wsl2的go环境后，使用Goland进行开发，运行后出现日志换行的问题，如下:

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220519132647628.png" alt="image-20220519132647628" style="zoom:67%;" />

明明还有这么长的距离，但是出现了提前的换行。

## 解决办法

解决办法也很简单：

1. 首先按住`Ctrl`+`Shif`+`A`键

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220519132833643.png" alt="image-20220519132833643" style="zoom:67%;" />

2. 然后输入`Registry`，进入*Registry*界面

   <img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220519132937422.png" alt="image-20220519132937422" style="zoom: 50%;" />

3. 输入`run.process.with.pty`,找到*go.*run.process.with.pty和*run.process.with.pty*两个选项，全部取消勾选，关闭

   <img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220519133120292.png" alt="image-20220519133120292" style="zoom: 67%;" />

   4. 重启应用，输出日志恢复正常

   <img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220519133222783.png" alt="image-20220519133222783" style="zoom:67%;" />

   

