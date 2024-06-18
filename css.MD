## CSS 优化

1. 简写css

   简写css 意味着尽可能使用最不冗长的属性和属性值，在大型项目中可以达到累计减少空间的效果。
```
    p {
        width: 100px;
        margin-top: 10px;
        margin-left: 12px;
        margin-bottom：10px;
        margin-right: 12px;
    }
    
    p {
        width: 100px;
        margin: 10px 12px;
    }
```
    上面两个p 标签样式 第二个比第一个简单很多，可以减少字节长度，项目累计可以节省空间。

2. 使用css浅选择器减少层级

   减少css 选择器层级可以减少浏览器解析时间，与字符空间大小
```
    div ul li p .red {
        color: red;
    }

    .red {
        color: red;
    }
```
3. 分割css 

   根据特定模块拆分css，避免一个css文件过大。

4. 尽量避免使用占用过多cpu和内存的属性
```
    div{
        z-index：9999;
    }
```

5. 尽量避免使用耗电量大的属性
```
    css3 transforms opacity
```
6. 谨慎使用expensive 属性
```
    nth-child伪类 position: fixed
```

7. 尽量避免使用 css 表达式

```
    background-color: exoression((new Date()).getHours()%2? "#fff","#000")
```

8. 尽量减少使用通配符
```
    *{
        margin:0
    }
```

9. 使用外链的css

10. 内联css使js文件增大，外链可以把css 放cdn上面

11. 避免使用@import

    @import 串行加载会阻塞文件的加载

12. 精简css代码

13. 删除空格，注释

14. css 动画优化

* 避免同时使用动画
* 延迟动画初始化
* 使用svg 代替

15. css 放置在 head中

    css 放在head中避免用户看到的网页是无内容的,也可以在加载时提高页面的渲染性能,css放在后面会导致重新渲染和绘制dom