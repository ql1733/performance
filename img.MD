## 图片的优化

图片格式

* jpg
* png
* gif
* svg
* webp

jpg格式适合banner，颜色丰富的背景大图片，压缩质量会损失，缺乏透明度是可以接受的。不支持透明度

png格式适合纯色，透明，图标，无损压缩体积较jpg大，支持透明度。

gif 适合动画图片，不适合彩色图片 压缩性能底

svg 矢量图 不可压缩，可缩放

webp 适合图像与半透明格式与png 相同，不适合彩色图片，有更好的压缩性，但有兼容问题


图片压缩工具

* [在线工具]()
* [png压缩包node-pngquant-native]()
* [jpg压缩包jpegtran]()

  在不同网络环境下加载不同尺寸的图片

  响应式图片,在不同窗口大小加载不同尺寸图片

* javascript 事件动态绑定图片
* css 媒体查询
* img 标签属性
```
    <img srcset="img-320w.jpg,img-640w.jpg,img-960w.jpg 3x" src="img-960w.jpg">
```

逐步加载图片，使用统一占位符(图片未加载完成时默认显示一个低质量图片)

  小图标可是使用svg 与iconfont 字体库替换

  小图可以直接base64，减少请求

