---
title: hexo图片路径问题
date: 2021-01-18 15:46:01
tags: hexo问题解决
---

* 今天尝试md中添加图片，可是最后通用的方式生成的图片地址很奇怪，找了一阵子博客，发现这个完美解决了我的问题，记录一下，以防后面忘记了。

  

* 下文转自博客：[传送门](https://blog.csdn.net/androidv/article/details/105781570)

  

* 使用hexo 插入图片，图片的标签会变为 设置的 url后缀+文件名，如`/.com/xxx.jpg` 等
  原因是因为 hexo 升级后，对应的插件hexo-asset-image 并未升级，且留下bug，这里全部修改为绝对路径 即 全域名

  修改 `node_module/hexo-asset-image/index.js` 中的大概60行附近的

```js
//这个是源码
$(this).attr('src', config.root + link + src);
//修改为如下，data.permalink 是全路径 如我的
//http://www.jiaway.com/2020/04/26/redis/1587899771231.png
$(this).attr('src', data.permalink +src);````
```

